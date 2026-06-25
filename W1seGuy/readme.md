<div align="center">

# 🏴 W1seGuy — TryHackMe
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-212C42?style=for-the-badge&logo=tryhackme)

</div>

# W1seGuy — TryHackMe Writeup

A walkthrough and exploit script for the **W1seGuy** TryHackMe room — a known-plaintext XOR attack against a TCP server.

## Challenge Summary

The target runs a TCP server that:
- XOR-encrypts a known flag format (`THM{........}`) using a random 5-character key (letters + digits)
- Sends you the encrypted result as hex
- Asks you to recover the key to unlock a second flag

Because the plaintext prefix `THM{` is always known, this is a classic **known-plaintext XOR attack**.

## Attack Strategy

1. Connect to the server and capture one hex-encoded ciphertext.
2. XOR the first 4 bytes of the ciphertext with the known plaintext `THM{` to recover the first 4 characters of the key.
3. Brute-force the 5th character (only 62 possibilities: `a-z`, `A-Z`, `0-9`).
4. Decrypt the full ciphertext and confirm it starts with `THM{` and ends with `}`.
5. Submit the recovered key back to the server to receive Flag 2.

## Files

- `exploit.py` — Python script that automates key recovery and flag decryption.

## Usage

### 1. Connect to the target

```bash
nc MACHINE_IP 1337
```

Example output: 1f0e3b1f047a27...

### 2. Paste the hex string into the script

Edit `exploit.py` and set:

```python
cipher_hex = "PASTE_HEX_HERE"
```



### 3. Run the script

```bash
python3 exploit.py
```

<img width="861" height="232" alt="image" src="https://github.com/user-attachments/assets/c5456672-b297-4bd0-b3f6-88afdb04570d" />

### 4. Submit the key

Paste the recovered key (e.g. `KFvdp`) back into the open `nc` session to receive **Flag 2**.

## Exploit Script

```python
import binascii
import string

# Paste the XOR hex string you get from nc HERE
cipher_hex = "PASTE_HEX_HERE"
cipher = binascii.unhexlify(cipher_hex)
known_plain = "THM{"

# Step 1: Recover first 4 key characters
key_prefix = ""
for i in range(len(known_plain)):
    key_prefix += chr(cipher[i] ^ ord(known_plain[i]))
print("[+] Key prefix found:", key_prefix)

# Step 2: Brute-force last character
charset = string.ascii_letters + string.digits
for c in charset:
    key = key_prefix + c
    decrypted = "".join(
        chr(cipher[i] ^ ord(key[i % len(key)]))
        for i in range(len(cipher))
    )
    if decrypted.startswith("THM{") and decrypted.endswith("}"):
        print("\n[+] FULL KEY FOUND:", key)
        print("[+] FLAG 1:", decrypted)
        break
```

## Key Concepts

- **XOR encryption**: reversible bitwise operation; if you know any part of the plaintext, you can recover the corresponding part of the key.
- **Known-plaintext attack**: exploiting a predictable plaintext format (`THM{...}`) to break weak encryption.
- **Brute-forcing limited keyspace**: only the unknown 5th key character needs guessing — drastically reducing search space.

## Disclaimer

This writeup is for educational purposes as part of a TryHackMe lab environment. Do not use these techniques against systems you don't own or have explicit permission to test.

