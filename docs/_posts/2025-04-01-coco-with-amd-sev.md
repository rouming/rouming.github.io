---
title: Confidential Computing with AMD-SEV
date: 2025-04-01
---

This document aims to provide a clear understanding of AMD-SEV technology by offering practical examples on how to set up and use various components, including BIOS, UEFI (OVMF), QEMU, and SEV tools.

The main documents from AMD for reference are [1] and [2].

---

## Table of Contents

1.  [Variations of AMD-SEV Technology](#variations-of-amd-sev-technology)
2.  [Configuration of the BIOS and Kernel for AMD-SEV](#configuration-of-the-bios-and-kernel-for-amd-sev)
    * [Host BIOS](#host-bios)
    * [Host Kernel](#host-kernel)
    * [Guest OVMF Firmware](#guest-ovmf-firmware)
    * [Guest Kernel](#guest-kernel)
3.  [AMD-SEV Tooling from VirTEE](#amd-sev-tooling-from-virtee)
4.  [Configuration and Usage of the SEV and SEV-ES](#configuration-and-usage-of-the-sev-and-sev-es)
    * [Building sevctl](#building-sevctl)
    * [System Probing for SEV Support](#system-probing-for-sev-support)
    * [Platform Ownership](#platform-ownership)
    * [Guest Ownership](#guest-ownership)
    * [Guest Verification by the Guest Owner](#guest-verification-by-the-guest-owner)
    * [Launch Secret](#launch-secret)
5.  [Configuration and Usage of the SEV-SNP](#configuration-and-usage-of-the-sev-snp)
    * [Building snphost and snpguest](#building-snphost-and-snpguest)
    * [System Probing for SEV-SNP Support](#system-probing-for-sev-snp-support)
    * [QEMU](#qemu)
    * [VCEK and VLEK Public Signing Keys](#vcek-and-vlek-public-signing-keys)
    * [Guest Policy](#guest-policy)
    * [Attestation](#attestation)
    * [Measurement](#measurement)
    * [Launch Guest With ID-Block and Auth-Block](#launch-guest-with-id-block-and-auth-block)
6.  [AMD-SNP Attestation Procedure](#amd-snp-attestation-procedure)
7.  [References](#references)
8.  [Other Resources](#other-resources)

---

## Variations of AMD-SEV Technology

Secure Encrypted Virtualization (SEV) is a technology developed by AMD that enhances the security of virtual machines (VMs) by encrypting their memory with a unique key for each VM. SEV comes in three variations, each offering a different level of security:

1.  **Secure Encrypted Virtualization (SEV)**

    At this basic level, the CPU encrypts the VM's memory, safeguarding virtual RAM (vRAM) against unauthorized access.

2.  **Secure Encrypted Virtualization-Encrypted State (SEV-ES)**

    This version enhances protection by encrypting both the VM's memory and its registers stored in the VM save area (VMSA).

3.  **Secure Encrypted Virtualization-Secure Nested Paging (SEV-SNP)**

    The most advanced level, SEV-SNP, not only encrypts memory and registers but also ensures the integrity of guest memory. This feature is crucial in defending against hypervisor-based attacks, which may attempt to exploit guest data through corruption, aliasing, or replay.

    * **Aliasing attacks** occur when a hypervisor assigns the same physical memory page to different virtual memory locations within guests, potentially allowing it to access or alter data improperly.

    * **Replay attacks** involve the hypervisor capturing a guest's memory page at a specific time and restoring outdated or tampered data later, misleading the guest into processing inaccurate information.

SEV and SEV-ES are covered by AMD's "Secure Encrypted Virtualization API" [2] specification, while SEV-SNP is described in a separate document, the "SEV Secure Nested Paging Firmware ABI Specification" [3]. Not only are the API and ABI different, but so is the set of tools for AMD-SEV platform or guest administration. More on this later.

---

## Configuration of the BIOS and Kernel for AMD-SEV

### Host BIOS

1.  Ensure the BIOS is correctly configured. Typically, all SEV-related settings can be found in the Processor Settings section. Make sure the following options are properly set up:
    ```text
    Secure Memory Encryption             = Enabled
    Minimum SEV non-ES ASID              = 509
    Secure Nested Paging                 = Enabled
    SNP Memory Coverage                  = Enabled
    Transparent Secure Memory Encryption = Disabled (default)
    ```

2.  Ensure UEFI is enabled, otherwise SEV-SNP won't work and the following error may appear in the dmesg:
    ```text
    SEV-SNP: failed to INIT rc -5, error 0x3
    ```

### Host Kernel

1.  Make sure your kernel is configured with `CONFIG_KVM_AMD_SEV=y`.

2.  The `kvm_amd` module doesn't need extra options; all features are enabled by default. Refer to the file `linux/arch/x86/kvm/svm/sev.c` - it shows that `sev_enabled`, `sev_es_enabled`, and `sev_snp_enabled` are all set to true by default.

3.  To use SEV-SNP, you must fully enable IOMMU; simply using passthrough mode (`pt`) is insufficient. If not set up correctly, you'll encounter this error in dmesg:
    ```text
    AMD-Vi: SNP: IOMMU disabled or configured in passthrough mode, SNP cannot be supported.
    ```
    IOMMU is enabled by default, so you don't need to add any options to the cmdline. Just ensure that `iommu=pt` is **not** present.

### Guest OVMF Firmware

`OVMF` firmware should be built with AMD-SEV support enabled, specifically for the [Launch Secret](#launch-secret) feature. The `SEV-SNP` requires the OVMF binary to include `SNP_KERNEL_HASHES`, which is enabled by the same AmdSevX64 OVMF build.

1.  Pull latest stable:
    ```bash
    git clone \
      --branch edk2-stable202511 \
      --single-branch \
      --recurse-submodules \
      https://github.com/tianocore/edk2.git
    ```

2.  Build tools and apply environment:
    ```bash
    make -C BaseTools
    source ./edksetup.sh
    ```

3.  Touch the `grub.efi`:
    ```bash
    touch OvmfPkg/AmdSev/Grub/grub.efi
    ```

4.  Build OVMF for the AMD-SEV platform:
    ```bash
    build -p OvmfPkg/AmdSev/AmdSevX64.dsc -b RELEASE -a X64 -t GCC5
    ```

* Resulting binary `./Build/AmdSev/RELEASE_GCC5/FV/OVMF.fd` will be used by the QEMU and should be shared with the platform owner.

### Guest Kernel

Ensure the guest kernel is configured with the `EFI_SECRET` option enabled for the [Launch Secret](#launch-secret) feature. The latest Ubuntu Noble 24.04 includes the `efi_secret` module, but you need to install an additional modules package. For example:

```bash
apt install linux-modules-extra-6.8.0-55-generic
modprobe efi_secret
```

---

## AMD-SEV Tooling from VirTEE

VirTEE is an open community focused on creating open-source tools for establishing, verifying, and managing Trusted Execution Environments (TEEs). Within its array of projects, VirTEE includes:

* **sev**: This library provides implementations of the AMD Secure Encrypted Virtualization (SEV/SEV-ES) APIs and the SEV Secure Nested Paging Firmware (SEV-SNP) ABIs. [Explore on GitHub](https://github.com/virtee/sev).

For AMD SEV/SEV-ES technologies:

* **sevctl**: A command-line tool designed to manage the AMD Secure Encrypted Virtualization (SEV) platform. [Explore on GitHub](https://github.com/virtee/sevctl).

For AMD SEV-SNP technology:

* **snpguest**: A command line interface utility for managing an AMD SEV-SNP enabled guest. This tool allows users to interact with the AMD SEV-SNP guest firmware device enabling various operations such as: attestation, certificate management, derived key fetching, and more. [Explore on GitHub](https://github.com/virtee/snpguest).
* **snphost**: A command line interface utility for managing and interacting with the AMD SEV-SNP firmware device of a host system. [Explore on GitHub](https://github.com/virtee/snphost).

---

## Configuration and Usage of the SEV and SEV-ES

This section covers SEV and SEV-ES configuration, tools, and API. The SEV-SNP description is presented next.

### Building sevctl

The primary tool for managing SEV platform and guests is `sevctl`. First of all let's build `sevctl`:

```bash
# Install rust and cargo
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Clone
git clone https://github.com/virtee/sevctl
cd sevctl

# Install musl-glibc on the host (can be optional)
sudo apt install musl-dev musl-tools

# Build as a static lib with musl instead of glibc to make
# resulting binary portable and small
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl

# Copy binary to a server
scp target/x86_64-unknown-linux-musl/release/sevctl user@server
```

### System probing for SEV support

`sevctl` comes with the handy feature, which verifies (probes) the system for the SEV support:

```bash
sevctl ok
```

Generates the following output:

```text
[ PASS ] - AMD CPU
[ PASS ]   - Microcode support
[ PASS ]   - Secure Memory Encryption (SME)
[ PASS ]   - Secure Encrypted Virtualization (SEV)
[ PASS ]     - Encrypted State (SEV-ES)
[ PASS ]     - Secure Nested Paging (SEV-SNP)
[ PASS ]       - VM Permission Levels
[ PASS ]         - Number of VMPLs: 4
[ PASS ]     - Physical address bit reduction: 5
[ PASS ]     - C-bit location: 51
[ PASS ]     - Number of encrypted guests supported simultaneously: 509
[ PASS ]     - Minimum ASID value for SEV-enabled, SEV-ES disabled guest: 509
[ PASS ]     - SEV enabled in KVM: enabled
[ PASS ]     - SEV-ES enabled in KVM: enabled
[ PASS ]     - Reading /dev/sev: /dev/sev readable
[ PASS ]     - Writing /dev/sev: /dev/sev writable
[ PASS ]   - Page flush MSR: ENABLED
[ PASS ] - KVM supported: API version: 12
[ PASS ] - Memlock resource limit: Soft: 201692852224 | Hard: 201692852224
```

All tests should pass.

### Platform Ownership

The firmware includes a method to confirm it's running on genuine AMD hardware that supports SEV (Secure Encrypted Virtualization). Each platform has a unique key called the Chip Endorsement Key (CEK), certified by the AMD SEV Signing Key (ASK), which in turn is validated by the AMD Root Key (ARK). Private keys for ARK, ASK or CEK are never exposed by the firmware or AMD.

To verify the hardware's authenticity, a guest owner or remote system can access the public part of AMD's signing keys. The CEK also signs another key called the Platform Endorsement Key (PEK), connecting it back to the ARK as the root of trust.

PEK is generated by the firmware and signed by the CEK (PEK can be regenerated on demand by the `PEK_GEN` firmware call).

A platform owner can confirm ownership by retrieving the PEK, signing it with a newly generated Owner Certificate Authority (OCA) private key, and uploading the signed PEK back to the firmware using the `PEK_CERT_IMPORT` firmware call.

After platform ownership is established the certificates tree will look like the following:

```text
PDH - every time generated by the firmware
 ⬑ PEK - every time generated by the firmware
    ⬑ OCA - generated and self-signed by platform owner
    ⬑ CEK - once generated and stored in the CPU
       ⬑ ASK - once generated by AMD
          ⬑ ARK - once generated and self-signed by AMD

   ⬑ = signs
```

Where ASK and ARK are publicly available certificates and simply [hardcoded](https://github.com/virtee/sev/tree/main/src/certs/sev/builtin) by the `sev` library for each generation of the EPYC AMD processor: "Naples", "Rome", "Milan", "Genoa", "Turin".

As was mentioned earlier, PDH and PEK are generated by the firmware on demand and provided to the platform owner for the signature with OCA private key.

#### Taking Ownership of the SEV Platform

Taking platform ownership includes a few phases:

1.  Generating Owner Certificate Authority (OCA) certificate and private key pair:

    ```bash
    sevctl generate oca.cert oca.key
    ```
    Once generated, the OCA certificate and private key can be used across the entire fleet as long as the servers belong to the same owner. The OCA private key should not be exposed and must be kept secret.

2.  Reset the platform to its default state by clearing all persistent data, including platform certificates and signing keys:

    ```bash
    sevctl reset
    ```

3.  Use `provision` command to request a PEK certificate, sign it with the private key, and export the signed PEK along with the OCA certificate:

    ```bash
    sevctl provision oca.cert oca.key
    ```
    Once completed without any errors, the platform is owned by the platform owner.

4.  Verification of the platform ownership:

    ```bash
    sevctl verify --oca oca.cert
    ```
    Generates the following output:
    ```text
    PDH EP384 D256 a1083a8a8c2ad36bdf29c018f605dd26617d1a8ddaf1fe86b043bc514b6348a3
     ⬑ PEK EP384 E256 210c911303a5f64f852c5d5c390778cad15f4fe947646257a0564c7b5a6b9927
       •⬑ OCA EP384 E256 fc3fbe741c6695d7e63cbf1aa5e21c5952ddbfb08d154367e12cf10a613511f4
        ⬑ CEK EP384 E256 3357016ec3a83b601a8780ce8bfc429f2a998460c8246ea4fc5b5d2c5b434082
           ⬑ ASK R4096 R384 95cba79ba3c77daea79f741bade8156a50b1c59f6d6fda104d16dd264729f5ee8989522f3711fc7c84719921ceb31bc0
             •⬑ ARK R4096 R384 569da618dfe64015c343db6d975e77b72fdeacd16edd02d9d09b889b8f0f1d91ffa5dfbd86f7ac574a1a7883b7a1e737

     • = self signed, ⬑ = signs, •̷ = invalid self sign, ⬑̸ = invalid signs
    ```

5.  Shows ownership status (optional step):

    ```bash
    sevctl show flags
    ```
    Generates the following output:
    ```text
    owned
    es
    ```
    Where "owned" means the platform ownership has been established, and "es" means the encrypted state functionality is supported.

### Guest Ownership

Guest owners wish to protect the confidentiality of the data running within their guests. While they trust the platform owner to provide the infrastructure to run their guest, they benefit from reducing their risk exposure to vulnerabilities within that infrastructure. The SEV feature encrypts the contents of the memory of the guest and provides assurances that it was encrypted properly.

When a guest is launched, its memory must first be encrypted before SEV can be enabled in hardware for this guest. The firmware provides interfaces to bootstrap the memory encryption for this purpose, which ABI is used by the kernel.

1.  The first thing a guest owner may request from the platform owner is the chain of platform certificates. This chain can be obtained by the following command:

    ```bash
    sevctl export --full chain.cert
    ```

    Full chain includes the following certificates in the order:
    * **ASK**: AMD SEV Signing Key, hardcoded in the lib for each CPU generation.
    * **ARK**: AMD Root Key, hardcoded in the lib for each CPU generation.
    * **PDH**: Platform Diffie-Hellman key, generated by the firmware.
    * **PEK**: Platform Endorsement Key, generated by the firmware.
    * **OCA**: Owner Certificate Authority, platform owner root of trust certificate.
    * **CEK**: Chip Endorsement Key, downloaded from the AMD site by the following URL `https://kdsintf.amd.com/cek/id/$ID`, where `$ID` is the system unique ID, returned by the firmware on calling the `GET_ID` command.

    Refer to the "Verification of platform ownership" command for a clearer understanding of the certificates relationship.

2.  Before starting a VM, guest owner needs to generate a Guest Owner Diffie-Hellman (GODH) certificate, VM session blob and Guest TEK (Transport Encryption Key) and TIK (Transport Integrity Key) keys by calling:

    ```bash
    sevctl session chain.cert <policy>
    ```

    Where `chain.cert` is a chain of certificates exported in the previous step, and `<policy>` is an integer number representing a guest policy. The firmware enforces the policy, which restricts the configuration and operational commands that the hypervisor can perform on this guest. The policy number can be constructed using the following policy bits (see the "Chapter 3 Guest Policy" [2]):

    * **0 NODBG**: Debugging of the guest is disallowed when set.
    * **1 NOKS**: Sharing keys with other guests is disallowed when set.
    * **2 ES**: SEV-ES is required when set.
    * **3 NOSEND**: Sending the guest to another platform is disallowed when set.
    * **4 DOMAIN**: The guest must not be transmitted to another platform that is not in the domain when set.
    * **5 SEV**: The guest must not be transmitted to another.

    Once executed, `sevctl` generates the following 4 files in the current folder: `vm_godh.b64`, `vm_session.b64`, `vm_tek.bin`, `vm_tik.bin`.

    The following are the actual steps taken by the `sev` library to generate these files:
    * Generate new pair: GODH (Guest Owner Diffie-Hellman) certificate and private key.
    * Derive `Z` key (shared secret) from the PDH certificate and previously generated private key using the ECDH key agreement protocol [4]. PDH certificate is extracted from the certificate chain, provided in the command line.
    * Generate a random `nonce` and `IV` (initialization vector) of 16 bytes in size each to ensure uniqueness and add randomness to encryption.
    * Derive master secret from the `Z` key and `nonce`.
    * Derive KEK (Key Encryption Key) from the master key.
    * Derive KIK (Key Integrity Key) from the master key.
    * Generate a random TEK (Transport Encryption Key) of 16 bytes in size.
    * Generate a random TIK (Transport Integrity Key) of 16 bytes in size.
    * Encrypt TEK and TIK with KEK key, and store the resulting encrypted blob into the `SESSION.WRAP_TK` field.
    * Generate HMAC SHA-256 hash from the `SESSION.WRAP_TK` field with the KIK key and store the resulting hash into the `SESSION.WRAP_MAC` field.
    * Generate HMAC SHA-256 hash from the integer policy value and store the resulting hash into the `SESSION.POLICY_MAC` field.

    Session fields and encryption steps are covered in the "6.2 LAUNCH_START" or "6.9 SEND_START" sections of the original AMD API spec [2].

    From the first glance, all these steps look quite complicated, but in reality, everything boils down to a few simpler steps:
    * The guest owner generates secrets: TEK and TIK, which are then encrypted with the encryption key (see the ECDH key agreement protocol [4]). Afterwards, the encryption key is forgotten.
    * Session blob is created and sent to the firmware.
    * The firmware repeats the same cryptographic procedure to derive the same encryption key to decrypt TEK and TIK.
    * The platform owner can't decrypt TEK and TIK.

    For more details, please refer to the AMD API, specifically the "2.1 Keys" section.

    The policy, `vm_godh.b64` and `vm_session.b64` files are safe to share with the platform owner.

3.  The platform owner can start a VM with the received policy, `vm_godh.b64` and `vm_session.b64` blobs from the guest owner, using the following QEMU command line:

    ```bash
    -object sev-guest,id=sev0,cbitpos=$CBIT,reduced-phys-bits=$RBIT,policy=$POLICY,dh-cert-file=vm_godh.b64,session-file=vm_session.b64
    ```

    Where `$CBIT` and `$RBIT` can be retrieved from the `sevctl ok` output from the following lines:

    ```text
    [ PASS ] - Physical address bit reduction: 5
    [ PASS ] - C-bit location: 51
    ```

### Guest Verification by the Guest Owner

Once the guest is started, the guest owner may wait to provide the guest with confidential information until they can verify the attestation measurement, to ensure the guest was not tampered with by the hypervisor. Since the guest owner knows the initial contents of the guest at boot, the attestation measurement can be verified by comparing it to what the guest owner expects.

SEV firmware supports two quite similar mechanisms for verifications, which can be obtained by the `LAUNCH_MEASURE` and `ATTESTATION` commands, see "6.5 LAUNCH_MEASURE" and "6.8 ATTESTATION" in [2].

#### Launch Measure

`LAUNCH_MEASURE` firmware command returns a signed measurement blob, which should be shared with the guest owner. The measurement blob contains the SHA-256 digest of the initial guest state, which includes:

* firmware binary
* kernel and initrd binaries (optional)
* kernel cmdline (optional)
* CPU state (only if `ES` is enabled by the policy)

The resulting blob is signed with the HMAC-SHA-256 (keyed hash function) using the TIK (Transport Integrity Key) key, which is securely kept by the guest owner, but is also known to the firmware, as it was encrypted and shared within the session blob during the VM start phase.

The guest owner can re-create the same measurement blob binary and compare it with the one shared by the platform owner. If the binaries are equal, then the guest owner can be sure that the guest state has not been tampered with.

Once guest is started, platform owner should retrieve the measurement blob by calling the QEMU QMP API:

```bash
echo '{ "execute": "qmp_capabilities" }' \
     '{ "execute": "query-sev-launch-measure" }' \
     | sudo nc -q 0 -U $MON \
     | jq -r '.return.data | select(. != null)' \
     | base64 -d > launch-measurement.bin
```

Where `$MON` is path to the QEMU monitor unix-domain socket file.

The resulting `launch-measurement.bin` blob should be shared with the guest owner.

Further steps should be executed solely by the guest owner once the `launch-measurement.bin` blob is received.

If `ES` is enabled by the policy, CPU initial states ("VM Save Area" - VMSA) should be generated. If `ES` is not enabled, first 2 steps can be skipped.

1.  Generate initial state for the vCPU0 and store it in the `vmsa0.bin` file:

    ```bash
    sevctl vmsa build vmsa0.bin \
      --userspace qemu \
      --cpu 0 \
      --family $VCPU_FAMILY \
      --stepping $VCPU_STEPPING \
      --model $VCPU_MODEL \
      --firmware AmdSev-OVMF.fd
    ```

    Where `$VCPU_FAMILY`, `$VCPU_STEPPING`, and `$VCPU_MODEL` are vCPU characteristics, set by the QEMU. These values should be shared with the guest owner in advance. For example, for the AMD EPYC CPU, these characteristics are defined as follows:

    ```text
    VCPU_FAMILY   = 23
    VCPU_MODEL    = 1
    VCPU_STEPPING = 2
    ```

    QEMU defines the table of vCPU characteristics in the `target/i386/cpu.c` file.
    The `AmdSev-OVMF.fd` is the firmware binary, shared by the platform owner in advance.

2.  Generate initial state for other vCPUs and store it in the `vmsa1.bin` file:

    ```bash
    sevctl vmsa build vmsa1.bin \
      --userspace qemu \
      --cpu 1 \
      --family $VCPU_FAMILY \
      --stepping $VCPU_STEPPING \
      --model $VCPU_MODEL \
      --firmware AmdSev-OVMF.fd
    ```

    All the parameters are the same as for generating `vmsa0.bin` file, except the `--cpu 1` option, which is set to 1 indicating that this is not the vCPU0.

    Actually, the differences between the VMSA state for vCPU0 and other vCPUs are in the initial register states: registers `RSP`, `RBP`, `RSI` are cleared, but `RIP` and `CS` are set to the `OVMF` reset address.

3.  Generate resulting measurement blob and store it in the `expected-launch-measurement.bin` file:

    ```bash
    sevctl measurement build \
      --api-major $FIR_API_MAJOR \
      --api-minor $FIR_API_MINOR \
      --build-id $FIR_BUILD \
      --policy $POLICY \
      --num-cpus $NUM_VCPUS \
      --vmsa-cpu0 vmsa0.bin \
      --vmsa-cpu1 vmsa1.bin \
      --tik vm_tik.bin \
      --firmware AmdSev-OVMF.fd \
      --launch-measure-blob launch-measurement.bin \
      | base64 -d \
      > expected-launch-measurement.bin
    ```

    Where `$FIR_API_MAJOR`, `$FIR_API_MINOR` and `$FIR_BUILD` are parts of the firmware version, which can be retrieved by the `sevctl show version` command and provided to the guest owner in advance:

    ```bash
    IFS='.' read -r FIR_API_MAJOR FIR_API_MINOR FIR_BUILD < <(sevctl show version)
    ```

    The `$POLICY` should already be known to the guest owner, as it was initiated by them.
    The `launch-measurement.bin` is the measurement blob provided by the platform owner just right after the VM was successfully started by the QEMU.
    The `$NUM_VCPUS` should equal the number of vCPUs configured for the VM, which measurement will be verified.

    If `ES` was not initially enabled by the policy, the `--vmsa-cpu0`, `--vmsa-cpu1` and `--num-cpus` options can be skipped.

4.  Verify measurement blobs are equal:

    ```bash
    cmp launch-measurement.bin expected-launch-measurement.bin
    ```

#### Attestation

There is another firmware command similar to the `LAUNCH_MEASURE` for guest verification: `ATTESTATION`. Command returns an attestation blob signed with the PEK (Platform Endorsement Key). The attestation blob also includes a SHA-256 digest of the initial guest state, with the measurement procedure being similar to what the `LAUNCH_MEASURE` command does.

The main difference between these commands is the signature used: the measurement blob retrieved by the `LAUNCH_MEASURE` command is signed with the TIK (Transport Integrity Key), which is known to the firmware and guest owner. In contrast, the blob returned by the `ATTESTATION` command is signed with the PEK (Platform Endorsement Key), which verifies that the guest was started on a known platform.

Once guest is started, platform owner should retrieve the attestation blob by calling the QEMU QMP API:

```bash
echo '{ "execute": "qmp_capabilities" }' \
     '{ "execute" : "query-sev-attestation-report", ' \
     '    "arguments": { "mnonce": "AAAAAAAAAAAAAAAAAAAAAA==" } }' \
     | sudo nc -q 0 -U $MON \
     | jq -r '.return.data | select(. != null)' \
     | base64 -d \
     > attestation-report.bin
```

Where `$MON` is path to the QEMU monitor unix-domain socket file.

For simplicity, 16 zeros are passed in the base64 encoding as the `mnonce` parameter. However, to ensure uniqueness, the nonce must be a random 16-byte value.

Unfortunately, the `sev` and `sevctl` code responsible for the attestation verification is broken. To fix verification, I made a few PRs which target the problem:

* https://github.com/virtee/sev/pull/293
* https://github.com/virtee/sevctl/pull/205

So once patches are applied, attestation report can be verified in the following one step on the guest owner side:

```bash
sevctl validate chain.cert attestation-report.bin
```

Where `chain.cert` is a chain of certificates, and `attestation-report.bin` is an attestation blob exported in the previous step. Both are provided by the platform owner.

**Be aware**: the attestation report verification procedure includes **only** signature validation. The guest state digest is not verified. This security flaw might be addressed in the future. Stay tuned.

### Launch Secret

To provide the guest with secret data after the measurement is validated, the guest owner encrypts a secret with the TEK (Transport Endorsement Key), formats a blob with the encrypted secret, flags, and measurement. The resulting blob is signed with the HMAC-SHA-256 (keyed hash function) using the TIK (Transport Integrity Key) and sent to the platform owner. The platform owner forwards the secret blob to the firmware using the `LAUNCH_SECRET` command. See section "6.6 LAUNCH_SECRET" in [2].

**Note**: the `LAUNCH_SECRET` command can be executed only when the guest is in the `LSECRET` state. This means that QEMU should be started with the `-S` flag, which freezes the vCPUs at startup. This allows the guest owner to launch a secret and then continue execution.

1.  Generate secret header and payload:

    ```bash
    sevctl secret build \
        --tik vm_tik.bin \
        --tek vm_tek.bin \
        --launch-measure-blob launch-measurement.bin \
        --secret $UUID:<(echo $SECRET) \
        secret-header.bin \
        secret-payload.bin
    ```

    Where `$UUID` is any UUID that identify a secret on the guest side, and `$SECRET` is the actual secret passed as a string. The `--secret` option accepts a file path to a secret, so the command line option can be rewritten to `--secret $UUID:$FILE_WITH_SECRET` if a secret has to be passed as a file.

    The `launch-measurement.bin` is the measurement blob provided by the platform owner just right after the VM was successfully started by the QEMU.
    Resulting header and payload blobs should be shared with the platform owner.

2.  Platform owner injects a secret to the QEMU:

    ```bash
    echo '{ "execute": "qmp_capabilities" }' \
         '{ "execute": "sev-inject-launch-secret", ' \
         '  "arguments": { "packet-header": "'$(cat secret-header.bin | base64)'", ' \
         '  "secret": "'$(cat secret-payload.bin | base64)'"}}' \
         | sudo nc -q 0 -U $MON
    ```

    Where `$MON` is path to the QEMU monitor unix-domain socket file.

    The `secret-header.bin` and `secret-payload.bin` are blobs generated by the guest owner on the previous step.

    **Note**: command will fail if the guest is already running, so ensure the QEMU is started with vCPUs frozen, which can be achieved with the `-S` QEMU flag.

3.  Once a secret is injected, QEMU vCPUs may continue execution:

    ```bash
    echo '{ "execute": "qmp_capabilities" }' \
         '{ "execute": "cont" }' \
         | sudo nc -q 0 -U $MON
    ```

    Where `$MON` is path to the QEMU monitor unix-domain socket file.

4.  Once the guest is booted, the guest owner can safely read the secret in the VM:

    ```bash
    cat /sys/kernel/security/secrets/coco/$UUID
    ```

    Troubleshooting: if there is no `secrets/coco` folder:
    * Ensure OVMF was built with AMD SEV support.
    * Ensure the guest kernel is built with `EFI_SECRET` and the modules are loaded:

    ```bash
    cat /boot/config-$(uname -r) | grep EFI_SECRET
    lsmod | grep efi_secret
    ```

    Ubuntu Noble 24.04 has a kernel with `EFI_SECRET` enabled, but don't forget to install the correct package:

    ```bash
    apt install linux-modules-extra-6.8.0-55-generic
    modprobe efi_secret
    ```

---

## Configuration and Usage of the SEV-SNP

As previously mentioned [here](#variations-of-amd-sev-technology), AMD **SEV-ES** encrypt both the memory and the CPU register state of the guest. However, it lacks integrity protection, which means a compromised hypervisor could still manipulate memory mappings or inject malicious data. **SEV-SNP** addresses this by adding memory integrity checks and protections against replay and aliasing attacks. **SEV-SNP** is a complete redesign with new firmware, API, and ABI [3], and is not backward compatible with **SEV/SEV-ES**.

The main difference with AMD **SEV/SEV-ES** is that the guest can now talk directly to the firmware without needing help from the cloud provider. This means the guest owner can perform attestation and get measurement reports on their own, from inside the VM. There’s no need to involve the hypervisor or any external service, all guest needs is the proper driver (`sev-guest`) loaded.

The main tool is `snpguest`, which lets the guest securely request information from the firmware. As a result, **SEV-SNP** gives more control to the guest owner and simplifies trust verification.

However, from my perspective, the "cuckoo" attack still seems possible (a type of attack where a malicious host redirects a guest’s attestation request to a different, trusted VM to fake a valid response, with the fake VM being fully controlled by the malicious hypervisor). I have not found any clear defense against it.

### Building snphost and snpguest

The main tool for managing the SEV-SNP platform is `snphost`. For attestation and measurements, there is a `snpguest` tool, which should be called directly from a guest. First, let's build `snphost` (for `snpguest`, follow the same instructions, just replace `snphost` with `snpguest`):

```bash
# Install rust and cargo
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Clone
git clone https://github.com/virtee/snphost
cd snphost

# Install musl-glibc on the host (can be optional)
sudo apt install musl-dev musl-tools

# Build as a static lib with musl instead of glibc to make
# resulting binary portable and small
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl

# Copy binary to a server
scp target/x86_64-unknown-linux-musl/release/snphost user@server
```

### System probing for SEV-SNP support

`snphost` comes with the handy feature, which verifies (probes) the system for the SEV-SNP support:

```bash
snphost ok
```

Generates the following output:

```text
[ PASS ] - AMD CPU
[ PASS ]   - Microcode support
[ PASS ]   - Secure Memory Encryption (SME)
[ PASS ]     - SME: Enabled in MSR
[ PASS ]   - Secure Encrypted Virtualization (SEV)
[ PASS ]     - SEV firmware version: 1.55
[ PASS ]     - Encrypted State (SEV-ES)
[ PASS ]       - SEV-ES initialized
[ PASS ]     - SEV initialized: Initialized, no guests running
[ PASS ]     - Secure Nested Paging (SEV-SNP)
[ PASS ]       - VM Permission Levels
[ PASS ]         - Number of VMPLs: 4
[ PASS ]       - SNP: Enabled in MSR
[ PASS ]       - SNP initialized
[ PASS ]         - RMP table addresses: 0x1fdff100000 - 0x1ffff1fffff
[ PASS ]         - RMP table initialized
[ PASS ]         - Alias check: Completed since last system update, no aliasing addresses
[ PASS ]     - Physical address bit reduction: 5
[ PASS ]     - C-bit location: 51
[ PASS ]     - Number of encrypted guests supported simultaneously: 509
[ PASS ]     - Minimum ASID value for SEV-enabled, SEV-ES disabled guest: 509
[ PASS ]     - /dev/sev readable
[ PASS ]     - /dev/sev writable
[ PASS ]   - Page flush MSR: ENABLED
[ PASS ] - KVM supported: API version: 12
[ PASS ]   - SEV enabled in KVM
[ PASS ]   - SEV-ES enabled in KVM
[ PASS ]   - SEV-SNP enabled in KVM
[ PASS ] - Memlock resource limit: Soft: 201692848128 | Hard: 201692848128
[ PASS ] - Comparing TCB values: TCB versions match 
```

All tests should pass.

Same verification procedure can be processed on the guest:

```bash
snpguest ok
```

Should be all green.

### QEMU

The only existing requirement for SEV-SNP is the QEMU version, which should be at least 9.1, where the `sev-snp-guest` feature was implemented.

However, there are certain things to be done:

* Add `-cpu EPYC` to the QEMU command line as a vCPU type to provide a hint for guest drivers and tools about possible SEV-SNP support. The list of expected vCPU types can be obtained by looking into the QEMU source code: `target/i386/cpu.c`.
* The Q35 machine type should be enabled. If a QEMU compatibility version is provided, make sure it is at least '9.1', like this: `-M pc-q35-9.1`.
* Make sure UEFI (OVMF) firmware is used, instead of legacy BIOS.
* In order to start a guest with SEV-SNP support, change `-drive if=pflash,file=OVMF.fd` to `-bios OVMF.fd` because it is not allowed to start `sev-snp-guest` with `pflash`, and the following error appears: `pflash with kvm requires KVM readonly memory support`.

**Be aware**: one of the major limitations of SEV-SNP is that SNP vCPUs are not resettable, so guests can't be rebooted, and QEMU powers off instead of rebooting.

To start QEMU with confidential computing enabled, use the following command line:

```bash
-machine memory-encryption=sev0 \
-object sev-snp-guest,id=sev0,policy=<NUMBER>[,id-block=<BASE64>[,id-auth=<BASE64>]]
```

Options such as `policy`, `id-block`, or `id-auth` for the `sev-snp-guest` object are described below.

The official QEMU documentation also provides information on these parameters; see [6] for details.

### VCEK and VLEK public signing keys

VLEK and VCEK are both public signing keys used in the attestation and validation process, but they serve different purposes and are issued by different entities:

* **VCEK - Versioned Chip Endorsement Key**, issued by AMD, bound to a specific physical CPU.
* **VLEK - Versioned Launch Endorsement Key**, issued by a LAA (Launch Attestation Authority), which is also AMD. It appears that the VLEK can be explicitly requested from AMD, and Amazon used that opportunity [5]. The default option is to use VCEK, which will be further covered in this guide.

### Guest Policy

Each guest has a guest policy set by the Guest Owner, which tells the firmware what the hypervisor is allowed to do. The policy also sets the minimum firmware version required for the guest. The firmware uses this policy to limit host actions.

The Guest Owner sends the policy to the firmware when launching the guest. Once set, the policy is permanently linked to the guest and can’t be changed while it's running. If the guest is moved to another system, the policy moves with it and is enforced by the new platform's firmware.

The policy is provided as an integer directly to the hypervisor. In the case of QEMU, the policy is specified on the command line and is a required parameter:

```bash
-object sev-snp-guest,policy=<NUMBER>
```

where `<NUMBER>` can be constructed using the following policy bits table (see [3], page 31):

* **bit 24, CIPHERTEXT_HIDING_DRAM**: \
    0: Ciphertext hiding for the DRAM may be enabled or disabled. \
    1: Ciphertext hiding for the DRAM must be enabled.
* **bit 23, RAPL_DIS**: \
    0: Allow Running Average Power Limit (RAPL). \
    1: RAPL must be disabled.
* **bit 22, MEM_AES_256_XTS**: \
    0: Allow either AES 128 XEX or AES 256 XTS for memory encryption. \
    1: Require AES 256 XTS for memory encryption.
* **bit 21, CXL_ALLOW**: \
    0: CXL cannot be populated with devices or memory. \
    1: CXL can be populated with devices or memory.
* **bit 20, SINGLE_SOCKET**: \
    0: Guest can be activated on multiple sockets. \
    1: Guest can be activated only on one socket.
* **bit 19, DEBUG**: \
    0: Debugging is disallowed. \
    1: Debugging is allowed.
* **bit 18, MIGRATE_MA**: \
    0: Association with a migration agent is disallowed. \
    1: Association with a migration agent is allowed.
* **bit 17, Reserved**: \
    Must be one.
* **bit 16, SMT**: \
    0: SMT is disallowed. \
    1: SMT is allowed.

The default policy can include settings for both the 17th and 16th bits, represented by **0x30000** integer number.

### Attestation

Attestation is a security process that allows a virtual machine to prove to an external party that it is running in a trusted environment. It involves generating a cryptographic report that includes measurements of the system's state, which can be verified.

#### Generate Attestation Report

The following example creates a random request string and uses it as the request file `request-file.txt`. When the command returns the attestation report, it is stored in the specified file path `report.bin`. In this case, the utility stores the report in the current directory:

```bash
snpguest report report.bin request-file.txt --random
```

The `request-file.txt` will be filled with 64 bytes of random data in hexadecimal format. The same 64-byte sequence should be included in the attestation report `report.bin` at the 0x50 offset (in the field named `REPORT_DATA`, see [3], page 56).

#### Fetch the AMD certificate chain (ARK & ASK) from KDS

In order to verify the report's signature, it is required to fetch the public certificates that establish the chain of trust back to AMD. **AMD Root Key (ARK)** and **AMD SEV Key (ASK)** certificates are fetched directly from AMD's official **Key Distribution Service (KDS)**. This will happen over the network.

```bash
snpguest fetch ca pem milan ./certs
```

Where `pem` is a certificate's encoding and `milan` is the CPU family type. The CPU family type can be obtained on the guest by calling:

```bash
cpuid | grep -o '\(Milan\|Rome\|Genoa\)' | head -n 1
```

#### Fetch the VCEK certificate from KDS

The attestation report that was previously generated is signed by the **VCEK**, which is unique to the chip and its current TCB version. We will need to use information from the generated report (specifically the Chip ID and TCB version) to request the correct **VCEK** certificate from **AMD KDS**. This is once again done over the network.

```bash
snpguest fetch vcek pem milan ./certs ./report.bin
```

Requested `vcek.pem` will be stored in the `certs` folder.

#### Verify the certificate chain

Before verifying the report itself, let's confirm that the certificates form a valid chain: the **ARK** should be self-signed (as it's the root), the **ASK** should also be signed by the **ARK**, and the **VCEK** should be signed by the **ASK**. We can verify this with the following command:

```bash
snpguest verify certs ./certs
```

If everything is set up correctly, the expected output is:

```text
The AMD ARK was self-signed!
The AMD ASK was signed by the AMD ARK!
The VCEK was signed by the AMD ASK!
```

#### Verify the attestation report

Now the attestation report can be verified using already verified certificates:

```bash
snpguest verify attestation ./certs ./report.bin
```

The expected output is:

```text
Reported TCB Boot Loader from certificate matches the attestation report.
Reported TCB TEE from certificate matches the attestation report.
Reported TCB SNP from certificate matches the attestation report.
Reported TCB Microcode from certificate matches the attestation report.
VEK signed the Attestation Report!
```

### Measurement

A measurement is a cryptographic hash of code or data (such as firmware, bootloaders, or configurations) taken at specific points during system startup. It's used in attestation to prove that the system started with known, trusted components.

#### Generate Measurement

Measurement can be generated by the Guest Owner:

```bash
snpguest generate measurement \
      --ovmf ./OVMF.fd \
      --vcpus 4 \
      --vcpu-type EPYC \
      | xxd -r -p \
      > expected-launch-measurement.bin
```

Where `--ovmf` specifies the path to the `OVMF` file, `--vcpus` is the number of vCPUs, and `--vcpus-type` specifies the type of the vCPU.

The list of expected vCPU types can be obtained by looking into the QEMU source code: `target/i386/cpu.c`, where the possible types are:

* "EPYC" - Naples
* "EPYC-Milan" - Milan
* "EPYC-Rome" - Rome
* "EPYC-Genoa" - Genoa

You can obtain the list of available AMD EPYC families from here: https://en.wikichip.org/wiki/amd/cpuid

#### Verify Launch Measurement Digest

Get launch measurement digest from `report.bin` at the 0x90 offset (the field named `MEASUREMENT`, see [3], page 56):

```bash
dd if=report.bin of=launch-measurement.bin skip=$((0x90)) bs=1 count=48 status=none
```

Compare what the firmware has returned with what the Guest Owner expects, they should be identical.

```bash
cmp launch-measurement.bin expected-launch-measurement.bin
```

### Launch Guest With ID-Block and Auth-Block

In **AMD SEV-SNP**, the **ID block** and **AUTH block** are critical components in the attestation process:

* **ID Block** is a signed data structure that uniquely identifies a Virtual Machine (VM) and is associated with it after the launch process. This block contains information like the expected launch digest and other relevant VM details, and is used for attestation purposes.
* **Auth block** is a cryptographic signature over the ID block, generated by the guest owner. It authorizes the guest launch by verifying that the configuration (from the ID block) is approved and hasn't been tampered with.

Together, they ensure that only a VM with a trusted, pre-approved configuration can be launched securely on the **SEV-SNP** platform.

#### Generate ID and Auth Blocks

First two private EC (Elliptic Curve) keys should be generated. The output of the following two commands is in `PKCS#8 PEM` format.

Generate a private key for ID block:

```bash
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp384r1 -out id-block-key.pem
```

Generate a private key for Auth block:

```bash
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp384r1 -out id-auth-key.pem
```

Generate ID and Auth blocks by a single `generate id-block` command:

```bash
snpguest --quiet generate id-block id-block-key.pem id-auth-key.pem \
     $(snpguest generate measurement --ovmf ./OVMF.fd --vcpus 4 --vcpu-type EPYC) \
     --id-file id-block.base64 \
     --auth-file id-auth.base64
```

where the `$(snpguest generate measurement ...)` command is generated measurement (see details in the previous [section](#measurement)), `--id-file` specifies the path to a file for the ID block output in `base64` encoding, and `--auth-file` specifies the path to a file for the authorization block output in `base64` encoding.

When using QEMU options `-kernel`, `-append`, and `-initrd`, OVMF must be built as AmdSevX64 (see the "Guest `OVMF` Firmware" [section](#guest-ovmf-firmware)). This is necessary because the OVMF binary must include `SNP_KERNEL_HASHES`, which is achieved by the AmdSevX64 build.

For the kernel and initrd case, the ID and AUTH blocks can be generated, for example, as follows:

```bash
snpguest --quiet generate id-block id-block-key.pem id-auth-key.pem \
     $(snpguest generate measurement \
         --ovmf AmdSev-OVMF.fd \
         --kernel vmlinuz-6.8.0-55-generic \
         --append "root=UUID=a191e346-9ec3-4394-b034-b1545dbf4eaf ro console=tty1 console=ttyS0" \
         --initrd initrd.img-6.8.0-55-generic \
         --vcpus 4
         --vcpu-type EPYC) \
     --id-file id-block.base64 \
     --auth-file id-auth.base64
```

and `-object sev-snp-guest` object must include `kernel-hashes=on`, which is described in the next section.

#### Run QEMU

The QEMU documentation [6] provides information about the necessary parameters for the ID and Auth blocks. Modify the `-object sev-snp-guest,...` command line with the following arguments:

```bash
-object sev-snp-guest,id-block=$(cat id-block.base64),id-auth=$(cat id-auth.base64),author-key-enabled=true,...
```

If measurements are done correctly, QEMU should start successfully!

**Be aware**: when QEMU is executed with the `-kernel`, `-append`, and `-initrd` options, the AmdSevX64 OVMF build should be provided, and `kernel-hashes=on` should be set for the `-object sev-snp-guest` object.

If `kernel-hashes=on` is not set, AmdSevX64 OVMF rejects booting the specified kernel, and measurements won't match when an ID block is used.

The command line should appear as follows:

```bash
-object sev-snp-guest,kernel-hashes=on,id-block=$(cat id-block.base64),id-auth=$(cat id-auth.base64),author-key-enabled=true,...
```

To provide opaque host-supplied data of 32 bytes, which will be included in the attestation report, the `host-data=<base64>` should be specified.

#### Verification

Once the guest has started with the ID and Auth block and has booted, the Guest Owner might verify the key digests retrieved from the [attestation report](#attestation).

Retrieve the digest of private keys and convert them to binary format:

```bash
# Get digest of the ID-Block key
snpguest generate key-digest id-block-key.pem --key-digest-file expected-id-block-key.hex-digest
# Convert digest from hex to binary
cat expected-id-block-key.hex-digest | xxd -r -p > expected-id-block-key.digest

# Get digest of the ID-Auth key
snpguest generate key-digest id-auth-key.pem --key-digest-file expected-id-auth-key.hex-digest
# Convert digest from hex to binary
cat expected-id-auth-key.hex-digest | xxd -r -p > expected-id-auth-key.digest
```

Get attestation report (see details in the previous [section](#attestation)):

```bash
snpguest report report.bin request-file.txt --random
```

Extract keys digests from the report at the 0xE0 and 0x110 offsets (the fields named `ID_KEY_DIGEST` and `AUTHOR_KEY_DIGEST`, see [3], pages 56, 57).

```bash
# Extract ID-Block key digest
dd if=report.bin of=id-block-key.digest skip=$((0xE0)) bs=1 count=48 status=none

# Extract ID-Auth key digest
dd if=report.bin of=id-auth-key.digest skip=$((0x110)) bs=1 count=48 status=none
```

Compare digests of both keys:

```bash
cmp id-block-key.digest expected-id-block-key.digest
cmp id-auth-key.digest expected-id-auth-key.digest
```

If the digests match, it means the Auth block is tied to the same platform as the ID block, confirming it's authorized for that specific hardware, so the guest owner can trust the attestation measurement.

---

## AMD-SNP Attestation Procedure

The following sequence ensures that the guest boots into a **genuine, untampered environment** , validated at runtime using AMD-SNP hardware-based attestation.

1.  **Customer Provides**
    * OVMF binary
    * Kernel
    * Initrd
    * ID-block blob
    * Auth-block blob
    * Encrypted disk image with customer secrets

2.  **Launch VM with QEMU**
    QEMU is started with the ID-block (which includes checksums of the OVMF, kernel, and initrd) and the Auth-block provided by the customer. (see details in the "Launch Guest With ID-Block and Auth-Block" [section](#launch-guest-with-id-block-and-auth-block))
    * The CPU halts execution if the digest in the ID-block does not match the actual binary checksums.
    * The ID-block and Auth-block are embedded into the attestation report.

    **Note**: the kernel, initrd, ID-block, and Auth-block may be tampered with by the cloud provider in a way that still allows the VM to boot. The further described attestation mechanism ensures that such tampering is detectable.

3.  **Run Attestation Application from Initrd**
    Once the guest boots into the initrd (either genuine or tampered), the initrd should invoke an **attestation application** during the final stage of the boot process. This application must be part of the checksummed initrd image.

4.  **Connect to Customer Attestation Service**
    The attestation application:
    * Establishes a **TLS connection** with the customer's attestation service (running in a trusted cloud environment)
    * Receives a **random NONCE** from the attestation service

5.  **Generate AMD-SNP Attestation Report**
    The attestation application performs AMD-SNP attestation, providing the received NONCE. It requests an attestation report which:
    * Is signed by AMD
    * Contains checksums of the OVMF, kernel, and initrd
    * Includes the ID-block and Auth-block originally provided by the customer
    * Includes the random NONCE received from the attestation service

    If any of the supplied binaries (OVMF, kernel, initrd, ID-block, Auth-block) were tampered with, the attestation report will reflect that.

    Importantly, even if the data is tampered with, the report itself is still signed with AMD's hardware key, proving its origin from a genuine AMD platform.

6.  **Send Report to Attestation Service**
    The attestation application sends the report to the customer's attestation service, which verifies:
    * The AMD signature on the report
    * The Auth-block digest (must match the customer's Auth key, known to the attestation service)
    * The ID-block digest (must match the customer's ID key, known to the attestation service)
    * The checksums of the OVMF, kernel, and initrd
    * The NONCE (must match the one previously issued)

    If any of the checks fail, the attestation service closes the connection.
    If all checks pass, the attestation service returns the **DISK-SECRET**.

    **Q: Why Does NONCE Matter?** \
    **A:** The NONCE ensures the attestation report was generated in response to this specific request, preventing reuse of older reports (replay attacks).

    **Q: What is the role of the Auth-block and ID-block key digests?** \
    **A:** Auth-block and ID-block keys are known only to the guest owner and the application service. A genuine attestation report that includes digests of these keys proves that a VM was originated by this guest owner and belongs to them. An attestation report generated inside another guest (or from tampered images) won't contain the ID-block previously provided by a customer, resulting in a mismatched ID-block key digest. The correct ID-block and Auth-block digests authenticate secret request from a guest VM to the attestation service.

7.  **Unseal the Disk and Continue Booting**
    Once the attestation application receives the DISK-SECRET:
    * It uses the secret to unseal the encrypted disk
    * The boot process continues into the fully provisioned system

---

## References

[1] [AMD SEV-SNP Firmware ABI Specification](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/56860.pdf) \
[2] [AMD SEV API Specification](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/programmer-references/55766_SEV-KM_API_Specification.pdf) \
[3] [SEV Secure Nested Paging Firmware ABI Specification](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/56860.pdf) \
[4] [Elliptic-curve Diffie-Hellman](https://en.wikipedia.org/wiki/Elliptic-curve_Diffie-Hellman) \
[5] [AWS EC2 SEV-SNP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snp-attestation.html) \
[6] [QEMU AMD Memory Encryption](https://www.qemu.org/docs/master/system/i386/amd-memory-encryption.html)

## Other Resources

* [https://www.amd.com/de/developer/sev.html](https://www.amd.com/de/developer/sev.html)
* [https://sys.cs.fau.de/extern/lehre/ws22/akss/material/amd-sev-intel-tdx.pdf](https://sys.cs.fau.de/extern/lehre/ws22/akss/material/amd-sev-intel-tdx.pdf)
* [https://www.vpsbg.eu/docs/how-to-perform-amd-sev-snp-attestation-inside-a-guest-virtual-machine](https://www.vpsbg.eu/docs/how-to-perform-amd-sev-snp-attestation-inside-a-guest-virtual-machine)
