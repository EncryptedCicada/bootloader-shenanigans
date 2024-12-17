# About the Android Bootloader and Boot Process

## Reading

1. An article about the qualcomm firmware and boot process: [[Link](https://lineageos.org/engineering/Qualcomm-Firmware/)]
2. A blog series by Inoki mentioning the difference between Aboot and XBL/PBL: [[Link](https://blog.inoki.cc/2021/10/17/android-bootloader-analysis-aboot-en/)]
3. A blog series by Inoki following up on the XBL/PBL analysis: [[1](https://blog.inoki.cc/2021/10/18/android-bootloader-analysis-abl-1-en/index.html)] [[2](https://blog.inoki.cc/2024/04/20/android-bootloader-analysis-abl-2-en/)] [[3](https://blog.inoki.cc/2024/04/20/android-bootloader-analysis-abl-3-en/)]

## Resources

1. ABL [[Source code](https://git.codelinaro.org/clo/la/abl/tianocore/edk2)]
2. Qualcomm Flash Loader V2 [[Link](https://xdaforums.com/t/qualcomm-flash-loader-v2.4668538/)]

## A Conversation with ChatGPT

> [!CAUTION]
> This section is a conversation with ChatGPT about my bootloader queries. Information here might not be factually correct.
> 
> _You have been warned._

I have got a crude understanding of XBL and ABL in android now. I did this to understand the contemporary android boot process of modern 2018- qcom devices. I have a few doubts. Can you help me clear them out? They are as follows:

### Question 1

What is the secure-boot state in bootloader and what implications does it have with respect to modifications 
possible to the XBL or ABL?

<details>
	<summary>Answer</summary>

Secure Boot is a mechanism implemented to ensure that only cryptographically signed and verified bootloaders and firmware components are executed during the boot process. It is typically enforced in hardware through the Qualcomm Secure Boot Chain and involves the following key elements:

- **Root of Trust**: A hardware-fused public key (known as the OEM Public Key or Root Key) is used to verify the signature of the next boot component.
- **Bootloader Stages**: Secure Boot verifies every bootloader stage sequentially:
	* Primary Bootloader (PBL) verifies the Secondary Bootloader (like XBL).
	* XBL verifies subsequent components like ABL and others.
	* ABL then hands off to the Linux kernel.

**Implications of Secure Boot:**

- **Signed Images**: Bootloader components (like XBL and ABL) must be signed with a valid key recognized by the device’s hardware root of trust. If the signature does not match, the device will not boot.
- **Modifications Are Restricted**: You cannot modify or replace XBL/ABL directly unless you have access to:
	
	* The manufacturer’s private key (used for signing).
	* A bootloader unlocked device, in which secure boot might be disabled or allow booting unsigned components.
- **Anti-Rollback Protection**: Many modern devices include rollback counters to prevent downgrading to older, possibly vulnerable bootloader versions.

In short, with secure boot enabled, XBL and ABL cannot be modified or replaced unless you bypass the verification (e.g., exploit vulnerabilities, unlock bootloader).
</details>


### Question 2

I have a manufacturer provided OTA zip which has abl.elf and a patch file named xbl.elf.p. If my hunch is correct about secure-boot requiring proper signing to boot a new abl or xbl, this could mean that when the manufacturer updates the abl, it has to patch the xbl with the new checksum or key to somehow verify the abl. Is this assumption correct? If yes, how can I see what changes are there in the xbl.elf.p file?

<details>
	<summary>Answer</summary>

Your hunch about secure boot requiring proper signing is correct. The relationship between `abl.elf` and `xbl.elf.p` can be understood as follows:

- `abl.elf`: This is the updated ABL (Applications Bootloader), which is verified and signed by the manufacturer.
- `xbl.elf.p`: This is a patch file for the XBL (Extensible Bootloader), likely containing updates or adjustments needed to verify the new `abl.elf`.

**Why would XBL need a patch?**

XBL has to verify ABL as part of the secure boot chain. If the ABL version or signature changes, XBL may need updated verification metadata, such as:

- A new hash or checksum of the ABL image.
- Updated public key or signature metadata used for ABL verification.

This ensures that XBL can validate and allow the execution of the new ABL during the boot process.

**How Can You Inspect Changes in xbl.elf.p?**

Analyzing a patch file like `xbl.elf.p` is tricky but possible using reverse engineering tools. Here’s how you can approach it:

1. **Understand the File Format**:
	* `xbl.elf.p` might be in a proprietary format, but you can first check if it’s a standard ELF or binary patch file.
	* Use tools like `file` on Linux to identify the file type:
	```
	file xbl.elf.p
	```
2. **Hexdump Analysis**:
	* View the raw binary contents with a hex editor like `xxd` or `HxD`:
	```
	xxd xbl.elf.p | less
	```
	* Look for ASCII strings, offsets, and patterns that resemble hash values, keys, or signatures.
3. **Compare with Original XBL**:
    * If you have the original `xbl.elf` binary, you can diff the two versions.
    * Tools like `bindiff` or `binwalk` can help analyze binary changes.
	
	For example: 
	```
	binwalk xbl.elf.p
	```
4. **Check for Hashes or Keys**:
    * If the `xbl.elf.p` file contains new checksums or signatures, you might spot SHA-256 hashes or public key structures embedded within the file.
    * Compare these with known hashes of `abl.elf` to confirm relationships.
5. **Reverse Engineering**:
    * Tools like IDA Pro, Ghidra, or Radare2 can help you disassemble and inspect the patch file if it has executable components or data structures. 
6. **OTA Update Scripts**:
    * Sometimes, OTA zips include an `update-script` or metadata that references how the patch is applied. Inspect `META-INF/com/google/android/updater-script` or similar files for clues.
</details>

### Question 3

So, if I am to flash a self built unsigned abl, the device will get bricked right? Even if I am to sign it with a private key of my own, it will get bricked because it does not match the Root Key, am I right?

<details>
	<summary>Answer</summary>

Yes, your understanding is absolutely correct.

If **Secure Boot** is enabled on the device:

1. **Flashing an unsigned ABL**:
	* The device will detect that the ABL is not cryptographically signed.
	* Since the signature verification (based on the Root of Trust) will fail, the bootloader will refuse to load the new ABL, and the device will not boot.
2. **Signing ABL with Your Own Private Key**:
	* Even if you sign the ABL with your own private key, the device will still refuse to boot it because the verification process relies on the public key fused into the hardware as part of the Root of Trust.
	* Your custom key will not match the OEM Public Key that the device expects.

In this case:
- The bootloader detects a signature mismatch and aborts the boot process.
- The result is usually a brick or a soft-brick, depending on whether recovery or bootloader modes are still accessible.

**Why Does This Happen?**

Secure Boot on Qualcomm devices relies on a chain of trust:

1. **PBL (Primary Bootloader)** is burned into ROM and is immutable. It verifies the **XBL (Extensible Bootloader)**.
2. **XBL** then verifies the **ABL**.
3. If the ABL signature fails verification, the boot process stops to prevent unsigned or tampered code from executing.

The signature must be:
- Cryptographically generated using the manufacturer’s private key.
- Verified using the manufacturer’s public key fused into the hardware.

**Possible Outcomes of Flashing an Unsigned ABL:**

1. **If Secure Boot is Enforced**:
	- The device will fail to boot and may enter a recovery mode, bootloader mode, or remain hard-bricked if no fallback mechanism exists.

2. **If Secure Boot is Disabled** (e.g., unlocked bootloader or test devices):
	- You could flash and boot unsigned ABL images.
	- Some devices allow disabling secure boot for development or testing purposes.

**Is There Any Way Around This?**

- **Bootloader Unlock**: If your device allows unlocking the bootloader, secure boot enforcement might be relaxed, enabling you to boot unsigned components.
- **Exploit Vulnerabilities**: In rare cases, security vulnerabilities in the bootloader (e.g., in XBL or ABL) can allow bypassing Secure Boot or flashing unsigned firmware.
- **Access to OEM Signing Key**: This is practically impossible unless you are the manufacturer or have access to their signing infrastructure.

Without these conditions, modifying or replacing ABL or XBL on a secure device is not feasible.
</details>

### Question 4

From your explanation it seems that secure-boot can be turned off by unlocking the bootloader right?

But, I have an HTC U12 Life device, of which I have already unlocked a bootloader and am able to boot custom recoveries, but the bootloader information displayed when entering it with the keycombination Vol Down + PWR still shows Secure boot:- yes. Does this mean the secure boot is not relaxed?

<details>
	<summary>Answer</summary>
**Bootloader Unlock vs. Secure Boot**

Unlocking the bootloader and disabling Secure Boot are two different things:

1. **Bootloader Unlock**:
	- Allows you to flash custom images (e.g., recovery, kernel, or boot images) to the device.
	- It relaxes some checks that prevent modifications to partitions like boot, recovery, or system.
	- It enables booting unsigned or modified kernels and recoveries.

2. **Secure Boot**:
	- Secure Boot remains in place as part of the **Qualcomm Hardware Root of Trust**.
	- Even with an unlocked bootloader, certain critical bootloader components (like **XBL** and **ABL**) are still verified using the fused Root of Trust public key.
	- Secure Boot cannot typically be disabled unless explicitly supported or bypassed (e.g., through an exploit).

**Why Secure Boot Still Shows as "Yes"?**

On devices like your **HTC U12 Life**:

- Unlocking the bootloader **does not disable Secure Boot**. Instead, it only allows you to boot unsigned kernels and recoveries.
- **XBL and ABL** remain protected under Secure Boot because they are part of the **immutable bootloader chain** verified by the hardware root of trust.
- The bootloader information displays `Secure Boot: Yes` because Secure Boot is still enforced for the lower-level bootloader stages (PBL, XBL, and ABL).

**Implications for Modifications**

1. **You Can**:
	- Boot unsigned or self-signed kernels and recoveries since the unlocked bootloader relaxes the checks for the `boot` and `recovery` partitions.
	- Replace the boot image (kernel + ramdisk) and recovery image.

2. **You Cannot**:
	- Modify or replace XBL or ABL because these components are still verified against the hardware Root of Trust.
	- Flash unsigned or self-signed versions of critical bootloader components.

</details>
