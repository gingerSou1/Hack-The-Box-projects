## Overview

This challenge focused on analyzing an ESP32 firmware image from the Hack The Box Hardware challenge **Espresso**. Rather than immediately relying on an existing walkthrough, I approached the firmware as I would an embedded device assessment by identifying the firmware format, validating the target architecture, examining the partition layout, and searching for indicators of how the application functioned.

This exercise reinforced the firmware analysis workflow I have been developing through previous hardware hacking projects involving routers, IoT cameras, and embedded Linux devices.

---

# Objectives

* Identify the firmware type and target architecture.
* Determine the flash layout.
* Extract useful metadata from the firmware.
* Locate potential flag generation routines.
* Understand the anti-cloning mechanism used by the application.
* Determine the intended solution path.

---

# Initial Firmware Identification

The provided firmware image was 4 MB in size.

Using standard firmware triage techniques, I identified the image as an **ESP32 firmware built using ESP-IDF**.

Important metadata recovered included:

* Flash Size: 4 MB
* Architecture: Xtensa (ESP32)
* Framework: ESP-IDF
* Factory application beginning at **0x10000**
* Standard ESP-IDF bootloader present

The firmware also contained the expected ESP32 partition table beginning at address **0x8000**, confirming that this was a complete flash image rather than only an application binary.

---

# Partition Analysis

The partition table revealed a straightforward firmware layout:

| Partition | Offset  | Purpose                |
| --------- | ------- | ---------------------- |
| NVS       | 0x9000  | Non-volatile storage   |
| PHY Init  | 0xF000  | Wi-Fi calibration data |
| Factory   | 0x10000 | Main application       |

Unlike many ESP32 firmware images that contain OTA partitions, this challenge used only a single factory application.

---

# Firmware Metadata

Additional information extracted from the application included:

* Project Name: **espresso**
* Build Date:

  * February 28, 2026
* Version:

  * `2c1ec8fd-dirty`

This metadata confirmed that the firmware had been compiled specifically for the challenge rather than representing production firmware.

---

# Static Analysis

Searching through the firmware strings revealed one particularly interesting message:

```text
flag did not generate correctly.
```

Additional nearby strings included:

```text
It seems you are running the firmware on cloned hardware.
Buy the real hardware, or perhaps try to emulate it. ;)
```

These strings immediately suggested that:

* the flag was **not stored as plaintext** inside the firmware
* the firmware attempted to generate the flag dynamically
* hardware-specific values were likely involved
* the challenge intentionally prevented execution on incorrect hardware

Rather than searching for a hardcoded flag, the objective became understanding the hardware validation mechanism.

---

# Reverse Engineering Observations

The firmware was identified as an ESP32 Xtensa application suitable for loading into Ghidra using the Xtensa processor module.

The discovered strings indicated that the application likely checked one or more hardware-dependent values before allowing successful flag generation.

Possible values included:

* eFuse contents
* MAC address
* Flash identification
* Chip revision
* Device identification values

The presence of the "perhaps try to emulate it" message strongly hinted that hardware emulation was the intended solution rather than binary patching.

---

# Intended Solution

Research confirmed that the challenge was designed to execute correctly under the Espressif ESP32 QEMU emulator.

When executed inside the emulator, the emulated hardware satisfied the firmware's validation logic, allowing the application to generate and print the flag over the serial console.

This demonstrated that the challenge focused less on defeating the protection mechanism and more on recognizing the correct execution environment.

---

# Skills Demonstrated

During this challenge I successfully:

* Identified an unknown firmware image.
* Recognized an ESP-IDF firmware layout.
* Parsed and interpreted the ESP32 partition table.
* Distinguished bootloader, NVS, PHY, and application regions.
* Extracted application metadata.
* Located meaningful strings related to program logic.
* Inferred that the flag was generated dynamically rather than statically embedded.
* Identified the anti-cloning mechanism through string analysis.
* Determined the appropriate execution strategy using hardware emulation.

---

# Lessons Learned

This challenge reinforced several important embedded reverse engineering concepts.

Unlike traditional firmware challenges where secrets may exist as hardcoded strings, embedded firmware frequently derives sensitive values from hardware-specific information. Identifying these behaviors early can significantly reduce unnecessary reverse engineering effort.

The challenge also demonstrated the importance of understanding the target platform. Recognizing the firmware as an ESP32 image immediately suggested the appropriate tooling, including ESP-IDF utilities, Ghidra with Xtensa support, and Espressif's QEMU emulator.

---

# Conclusion

The Espresso challenge provided practical experience analyzing ESP32 firmware using a structured firmware analysis methodology. Beginning with firmware identification, progressing through partition analysis and static inspection, and ultimately recognizing that hardware emulation was the intended execution environment mirrored the workflow used during real embedded security assessments.

Although the final flag was obtained through emulation rather than deep code modification, the exercise strengthened my understanding of ESP32 firmware structure, embedded reverse engineering techniques, and the value of identifying environmental assumptions before beginning extensive static analysis.
