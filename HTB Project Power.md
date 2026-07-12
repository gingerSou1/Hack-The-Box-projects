# HTB - Project Power

> Recovering an AES-128 encryption key using Correlation Power Analysis (CPA) against an embedded device.

---

## Challenge Overview

Project Power is a hardware side-channel challenge where the goal is to recover the secret AES-128 encryption key from an embedded device by analyzing its power consumption during encryption.

Unlike traditional cryptography challenges, AES itself is not vulnerable. Instead, the physical implementation leaks information through power consumption, allowing the encryption key to be recovered statistically.

---

## Skills Demonstrated

- Side-Channel Analysis (SCA)
- Correlation Power Analysis (CPA)
- AES Internals
- Statistical Analysis
- Python Automation
- NumPy
- Socket Programming

---

## Challenge Interface

The provided client exposes two options:

```text
1. Encrypt plaintext (Returns power trace)
2. Verify key
```

Option 1 accepts a chosen 16-byte plaintext and returns a Base64 encoded NumPy array representing the captured power trace.

Option 2 accepts a hexadecimal AES key and returns the flag if the recovered key is correct.

---

## Attack Overview

The attack consists of four major phases:

1. Collect power traces
2. Build a plaintext/trace dataset
3. Perform Correlation Power Analysis
4. Recover and verify the AES key

---

## Attack Workflow

```text
             Random Plaintexts
                    │
                    ▼
        Embedded AES Encryption
                    │
          Power Consumption
                    │
                    ▼
          Oscilloscope Capture
                    │
                    ▼
            Power Trace Dataset
                    │
                    ▼
      Correlation Power Analysis
                    │
                    ▼
          Recover AES-128 Key
                    │
                    ▼
             Submit to Server
                    │
                    ▼
                   Flag
```

---

## Trace Collection

The supplied Python client was modified to repeatedly request encryptions using randomly generated plaintexts.

Each request returned a power trace consisting of 1042 samples.

Approximately 985 traces were collected.

Dataset:

```text
Plaintexts : (985,16)
Traces     : (985,1042)
```

---

## Leakage Model

The attack assumes the device leaks information proportional to the Hamming Weight of the first AES S-Box output.

For every key hypothesis:

```text
HW(
    SBOX(
        PlaintextByte XOR KeyGuess
    )
)
```

where:

- HW = Hamming Weight
- SBOX = AES substitution box

Example:

```text
0x3C

00111100

HW = 4
```

---

## Correlation Power Analysis

For each byte of the AES key:

1. Guess every possible key value (0–255)
2. Compute the hypothetical leakage
3. Correlate the predicted leakage with every sample of every measured trace
4. Select the key guess with the highest Pearson correlation

Pseudo-code:

```python
for key_guess in range(256):

    hypothetical = HW(
        SBOX(
            plaintext ^ key_guess
        )
    )

    correlation = corr(
        hypothetical,
        measured_traces
    )

best_guess = max(correlation)
```

---

## Results

Recovered AES-128 Key:

```text
35425203F4BF23C7F93444BF772F2E1F
```

Correlation Scores:

| Byte | Correlation |
|------|------------:|
|00|0.8505|
|01|0.8610|
|02|0.8444|
|03|0.8399|
|04|0.8461|
|05|0.8627|
|06|0.8590|
|07|0.8343|
|08|0.8441|
|09|0.8562|
|10|0.8352|
|11|0.8322|
|12|0.8383|
|13|0.8389|
|14|0.8532|
|15|0.8009|

Each byte showed a clear statistical separation from the remaining 255 hypotheses, indicating successful key recovery.

---

## Verification

The recovered key was submitted using the verification interface provided by the challenge.

```text
Recovered Key
        │
        ▼
Verify Key
        │
        ▼
Correct
        │
        ▼
Flag
```

---

## Files

```text
collect_traces.py
```

Collects random plaintexts and corresponding power traces from the remote device.

```text
cpa.py
```

Performs first-round Correlation Power Analysis using a Hamming Weight leakage model.

```text
verify_key.py
```

Submits the recovered AES key to the challenge server for verification.

---

## Key Concepts

- Side-Channel Analysis
- Power Analysis
- Correlation Power Analysis
- Hamming Weight Leakage
- Pearson Correlation
- AES First Round Leakage
- Embedded Security

---

## Lessons Learned

This challenge demonstrates that a secure cryptographic algorithm can still be compromised through implementation weaknesses. Although AES itself remains mathematically secure, measurable variations in power consumption during computation leaked enough information to statistically recover the entire 128-bit secret key.

Collecting fewer than one thousand traces was sufficient to recover every key byte with high confidence using a standard first-round Hamming Weight CPA attack.

---

## References

- Kocher, P., Jaffe, J., & Jun, B. (1999). *Differential Power Analysis*
- Mangard, Oswald & Popp. *Power Analysis Attacks*
- NIST FIPS 197 – Advanced Encryption Standard (AES)

---

**Category:** Hardware Security / Side-Channel Analysis  
**Technique:** Correlation Power Analysis (CPA)  
**Language:** Python  
**Difficulty:** Medium
