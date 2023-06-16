# TheLastDance
Challenge Name: The Last Dance
------------------------------

*   **Status:** Active
*   **Category:** Crypto
*   **Difficulty:** Very Easy
*   **Date Owned:** 6/16/2023
*   Description:

> To be accepted into the upper class of the Berford Empire, you had to attend the annual Cha-Cha Ball at the High Court. Little did you know that among the many aristocrats invited, you would find a burned enemy spy. Your goal quickly became to capture him, which you succeeded in doing after putting something in his drink. Many hours passed in your agency's interrogation room, and you eventually learned important information about the enemy agency's secret communications. Can you use what you learned to decrypt the rest of the messages?

<br>

File Review
-----------

First lets look at the files that are provided for this challenge. The password protected zip contains the code used to invoke the encryption algorithm, and a text file containing the encrypted output.

**Provided encryption program:**

```text-plain
from Crypto.Cipher import ChaCha20
from secret import FLAG
import os

def encryptMessage(message, key, nonce):
   cipher = ChaCha20.new(key=key, nonce=iv)
   ciphertext = cipher.encrypt(message)
   return ciphertext

def writeData(data):
   with open("out.txt", "w") as f:
       f.write(data)

if __name__ == "__main__":
   message = b"Our counter agencies have intercepted your messages and a lot "
   message += b"of your agent's identities have been exposed. In a matter of "
   message += b"days all of them will be captured"
   key, iv = os.urandom(32), os.urandom(12)
   encrypted_message = encryptMessage(message, key, iv)
   encrypted_flag = encryptMessage(FLAG, key, iv)
   data = iv.hex() + "\n" + encrypted_message.hex() + "\n" + encrypted_flag.hex()
   writeData(data)
```

Encrypted output of this code:

```text-plain
c4a66edfe80227b4fa24d431
7aa34395a258f5893e3db1822139b8c1f04cfab9d757b9b9cca57e1df33d093f07c7f06e06bb6293676f9060a838ea138b6bc9f20b08afeb73120506e2ce7b9b9dcd9e4a421584cfaba2481132dfbdf4216e98e3facec9ba199ca3a97641e9ca9782868d0222a1d7c0d3119b867edaf2e72e2a6f7d344df39a14edc39cb6f960944ddac2aaef324827c36cba67dcb76b22119b43881a3f1262752990
7d8273ceb459e4d4386df4e32e1aecc1aa7aaafda50cb982f6c62623cf6b29693d86b15457aa76ac7e2eef6cf814ae3a8d39c7
```

Based on the code provided we can tell that the first string of output represents the hex value of the iv variable (the nonce). The second string represents the hex value of the encrypted message, and the third string is the hex value of the encrypted flag.

```text-plain
data = iv.hex() + "\n" + encrypted_message.hex() + "\n" + encrypted_flag.hex()
   writeData(data)
```

Starting from the top we can see that the encryption algorithm being used is ChaCha20, which is a stream cypher. Next an encryption function is defined which creates a new ChaCha20 object and passes a key and a nonce value (note the lack of nonce counter). Then the cyphertext is generated by passing the plaintext message to the ChaCha20 cypher, and is returned as a byte string.

In the main method we can see the plaintext message, which will come in handy. Then the key and nonce are generated using python's urandom() function which returns a byte string of a specified length. The key is generated with a length of 32 bytes (256bits) and the nonce with a length of 12 bytes (96bits) as required by the ChaCha20 algorithm.

Next the message and flag are both encrypted with the same encryption function and key/nonce pair.

Nonce Re-Use Attack
-------------------

To understand how this attack works we'll need to look under the hood of ChaCha20. These are some good resources that explain the basics of exploiting stream cyphers and ChaCha20 in particular.

*   [https://loup-vaillant.fr/tutorials/chacha20-design](https://loup-vaillant.fr/tutorials/chacha20-design)
*   [https://news.ycombinator.com/item?id=9561816](https://news.ycombinator.com/item?id=9561816)
*   [https://xilinx.github.io/Vitis\_Libraries/security/2019.2/guide\_L1/internals/chacha20.html](https://xilinx.github.io/Vitis_Libraries/security/2019.2/guide_L1/internals/chacha20.html)

Basically if the keystream is reused without incrementing the nonce, and one of the plaintext values is known, then we can solve for the keystream by comparing the plaintext and cyphertext strings with the XOR operator. The extracted keystream can then be used to decrypt anything else that was encrypted the same way. This attack compromises the C (confidentiality) of the CIA Triad.

First we'll start by converting the encrypted message and flag values from hex back to byte strings.

```text-plain
byte_msg = bytes.fromhex(hex_msg)
byte_flag = bytes.fromhex(hex_flag)
```

Then compare each bit of the cyphertext message with each bit of the plaintext message using the XOR (^) operator. To do this I will use python's zip() function which will compare each set of tuples of the iterator objects passed to it. The loop will return a list containing the XOR values of each bit comparison and cast the list back to a byte string giving us the keystream value.

```text-plain
keystream = bytes([a ^ b for a, b in zip(cypher_msg, plain_msg)])
```

Knowing that the ChaCha20 algorithm encrypts messages by comparing the plaintext with the keystream using the XOR operator, we can decrypt the flag the same way by comparing its cyphertext value to the keystream using XOR. You can define the decryption loop as a function if you'd like, I chose to simply copy and paste since I wont be using this code again.

```text-plain
flag = bytes([a ^ b for a, b in zip(cypher_flag, keystream)])
```

Decryption
----------

Putting it all together we end up with a script that looks like this.

```text-plain
hex_msg = '7aa34395a258f5893e3db1822139b8c1f04cfab9d757b9b9cca57e1df33d093f07c7f06e06bb6293676f9060a838ea138b6bc9f20b08afeb73120506e2ce7b9b9dcd9e4a421584cfaba2481132dfbdf4216e98e3facec9ba199ca3a97641e9ca9782868d0222a1d7c0d3119b867edaf2e72e2a6f7d344df39a14edc39cb6f960944ddac2aaef324827c36cba67dcb76b22119b43881a3f1262752990'
hex_flag = '7d8273ceb459e4d4386df4e32e1aecc1aa7aaafda50cb982f6c62623cf6b29693d86b15457aa76ac7e2eef6cf814ae3a8d39c7'

cypher_msg = bytes.fromhex(hex_msg)
cypher_flag = bytes.fromhex(hex_flag)

message = b"Our counter agencies have intercepted your messages and a lot "
message += b"of your agent's identities have been exposed. In a matter of "
message += b"days all of them will be captured"
plain_msg = message

keystream = bytes([a ^ b for a, b in zip(cypher_msg, plain_msg)])
flag = bytes([a ^ b for a, b in zip(cypher_flag, keystream)])
print(flag)
```

The plaintext flag is printed to the console. Congrats, you cracked the code!

```text-plain
HTB{********************************}
```