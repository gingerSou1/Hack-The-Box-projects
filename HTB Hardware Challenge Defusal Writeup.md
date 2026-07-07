# HTB Hardware Challenge - Defusal Writeup

**Category:** Hardware / Reverse Engineering  
**Difficulty:** Medium  
**Tools Used:** Ghidra, strings, avr-objdump (optional)

---

# Challenge Description

> "BOMB HAS BEEN PLANTED"

The provided firmware belongs to an Arduino Mega-based bomb simulation. The LCD prompts for a password, but the standard defusal equipment fails. The challenge provides the hardware schematic, indicating that the only path to success is through firmware analysis.

---

# Enumeration

## Hardware

The provided schematic reveals:

- Arduino Mega 2560
- 16x2 LCD
- 4x4 Matrix Keypad
- MAX7219 8x8 LED Matrix

There is:

- no EEPROM
- no SD card
- no network connectivity

This immediately suggests the password must exist somewhere inside the firmware.

---

# Initial Static Analysis

Running `strings` against the firmware reveals several interesting artifacts.

```bash
strings Defusal | less
```

Interesting output:

```text
735560
7355608

Bomb has been
DEFUSED!

BOOOOOOOOOOOM!

Enter Password:
```

The firmware also contains symbols such as:

```text
correctPassword
print_flag
xorValue
setup
loop
```

Because the ELF still contains symbols, reverse engineering becomes significantly easier.

---

# Loading into Ghidra

Import the ELF into Ghidra.

Run Auto Analysis.

Navigate to:

```
Symbol Tree
```

Interesting functions:

```
setup()
loop()
print_flag()
```

---

# Password Discovery

Searching for references to `correctPassword` shows that the firmware stores the value:

```text
7355608
```

This number is also a well-known reference from the movie *The Rock*, making it recognizable but still requiring firmware analysis to verify.

The keypad password is therefore:

```
7355608
```

---

# The Twist

Entering the password only displays:

```
Bomb has been
    DEFUSED!
```

At first glance it appears the challenge ends here.

However...

The challenge description specifically mentions:

> "...something about the device's output seems unusual."

That hints there is additional logic after successful authentication.

---

# Reverse Engineering print_flag()

The `print_flag()` function performs significantly more work than simply printing "DEFUSED."

It:

1. Clears the LCD
2. Displays the success message
3. Reads a large blob from `.data`
4. XORs the blob with the password
5. Sends the decoded data to the MAX7219 LED matrix

Relevant decompiled code:

```cpp
byte value = encoded[i];
value ^= correctPassword[i];
LedControl::setLed(...);
```

This immediately identifies the password as an XOR key rather than merely an unlock code.

---

# Encoded Data

The firmware copies a **296-byte** array from program memory.

```
296 bytes
```

Each frame consists of:

```
8 bytes
```

Therefore:

```
296 / 8 = 37 frames
```

Each frame represents one 8x8 bitmap rendered on the LED matrix.

The decode process is simply:

```
decoded_byte = encoded_byte XOR password_byte
```

where the repeating key is:

```
7355608\0
```

---

# Recovering the Flag

After decoding every frame, the LED matrix animation spells out:

```
HTB{B1NGO_B4NGO_B0NGO_B1SH_B4SH_B0SH}
```

---

# Final Flag

```
HTB{B1NGO_B4NGO_B0NGO_B1SH_B4SH_B0SH}
```

---

# Lessons Learned

This challenge demonstrates several useful reverse engineering concepts:

- Hardware schematics reveal attack surface.
- Embedded firmware often contains symbols if compiled without stripping.
- `strings` can provide valuable reconnaissance before opening a disassembler.
- Looking beyond the obvious success path is important—successful authentication may trigger additional hidden functionality.
- XOR "encryption" is common in embedded firmware for lightweight obfuscation.
- LED matrices frequently store characters as 8-byte bitmap frames, making them straightforward to decode once the rendering routine is understood.

---

# Key Takeaways

- Always inspect firmware after authentication succeeds.
- Reverse engineering the rendering routine can reveal hidden outputs.
- Hardware documentation can drastically reduce reverse engineering time.
- Embedded CTF challenges often hide flags in display logic rather than password verification.
