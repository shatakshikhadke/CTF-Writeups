# CyberTalents – Cypher Anxiety

## Challenge Information

- **Platform:** CyberTalents
- **Challenge:** Cypher Anxiety
- **Category:** Digital Forensics
- **Difficulty:** Easy
- **Points:** 50

---

## Challenge Description

> An image was leaked from a babies store. The manager is so annoyed because he needs to identify the image to fire charges against the responsible employee. The key is the MD5 of the image.

For this challenge, I was given a PCAP file named `find the image.pcap`.

The goal was to analyze the network traffic, recover the leaked image, and calculate its MD5 hash. The MD5 hash of the recovered image is the flag.

---

# Solution

## 1. Initial PCAP Analysis

I started by checking the PCAP file for anything interesting using `strings`.

    strings "find the image.pcap"

Among the output, I found a conversation between two people:

    Hey bro
    Sup supp, are we ready
    yeah, u got the files?
    yes but i think the channel is not secured
    the UTM will block the file transfer as the DLP module is active
    ok we can use cryptcat
    ok what the password then
    let it be P@ssawordaya
    hhh, ok
    listen on 7070 and ill send you the file , bye

This immediately gave me two important pieces of information:

- The file was transferred using **Cryptcat**
- The transfer was happening on **port 7070**

The password was also exposed in the conversation:

    P@ssawordaya

---

## 2. Finding the File Transfer

I opened the PCAP in Wireshark and filtered the traffic using:

    tcp.port == 7070

There was TCP traffic on port `7070`, which matched the information from the conversation.

I then followed the TCP stream to inspect the data being transferred.

In Wireshark:

    Right click packet
    → Follow
    → TCP Stream

The stream contained encrypted/binary data rather than a normal image file.

Since the conversation explicitly mentioned Cryptcat, I knew that this was the encrypted file transfer I needed to recover.

---

## 3. Identifying Cryptcat

The conversation gave me the tool being used:

    cryptcat

Cryptcat is essentially a version of Netcat that supports encrypted communication. It uses **Twofish** encryption.

The password from the conversation was:

    P@ssawordaya

And the transfer was taking place on:

    7070

So at this point I had:

    Tool     : Cryptcat
    Password : P@ssawordaya
    Port     : 7070

---

## 4. Extracting the Encrypted Stream

I extracted the TCP stream containing the Cryptcat communication.

The transferred data was not directly recognizable as a JPEG because it was encrypted.

The stream consisted of encrypted chunks, with most of the chunks being `8192` bytes in size.

After processing the Cryptcat stream using the recovered password, the decrypted data started to reveal file information.

One of the first recognizable pieces of the decrypted output was:

    8192 3304x

This indicated that the decrypted stream was beginning to contain structured file data rather than random encrypted traffic.

---

## 5. Recovering the Image

After decrypting and reconstructing the stream, I obtained the transferred file.

The resulting file was identified as a JPEG image.

The JPEG file contained the standard JPEG/JFIF structure and had dimensions of approximately:

    1600 x 1200

At this point, the leaked image had successfully been recovered from the PCAP.

---

## 6. Calculating the MD5 Hash

The challenge description says that the **MD5 hash of the image is the key/flag**.

I calculated the MD5 hash of the recovered image using:

    md5sum decrypted.jpg

The result was:

    decrypted.jpg
Therefore, the MD5 hash of the recovered image is:

    b7db3b48587c1ea2da8aee31ef42f026

---

## Key Takeaways

The main clues in this challenge were hidden in the network conversation itself.

1. The conversation revealed that **Cryptcat** was used for the file transfer.
2. The communication was taking place on **port 7070**.
3. The Cryptcat password was exposed in the conversation as `P@ssawordaya`.
4. The TCP stream contained the encrypted file transfer.
5. After decrypting and reconstructing the stream, I recovered the JPEG image.
6. The MD5 hash of the recovered image was the final flag.

This was a good example of why examining both the **network traffic and the surrounding conversation** is important during PCAP analysis. The conversation provided the information needed to understand and decrypt the actual file transfer.
