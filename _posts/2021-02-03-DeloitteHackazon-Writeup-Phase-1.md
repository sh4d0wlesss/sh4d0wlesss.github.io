---
title:  "Deloitte Hackazon Writeup Phase-1"
date:   2021-02-03T00:48:00-0400
categories:
  - ctf writeup
tags:
  - ctf
---

## PHASE 1 QUESTIONS
### Holiday Snack (Babycrypto) 150 point
İts beginner level crypto challenge. I generally use Cyberchef fot easy crypto challenges, its "magic" option is very good :)

#### Snack 1 (25 point)
```
69 110 106 111 121 32 116 104 101 32 103 97 109 101 33 32 67 84 70 123 55 57 50 101 55 52 54 101 56 56 53 48 101 57 53 97 54 97 51 50 56 51 52 51 56 56 100 102 57 102 52 101 125
```
This deciamls looks like ascii numbers of characters. Decode with cyberchef
```
Enjoy the game! CTF{792e746e8850e95a6a32834388df9f4e}


Link: https://gchq.github.io/CyberChef/#recipe=From_Decimal('Space',false)&input=NjkgMTEwIDEwNiAxMTEgMTIxIDMyIDExNiAxMDQgMTAxIDMyIDEwMyA5NyAxMDkgMTAxIDMzIDMyIDY3IDg0IDcwIDEyMyA1NSA1NyA1MCAxMDEgNTUgNTIgNTQgMTAxIDU2IDU2IDUzIDQ4IDEwMSA1NyA1MyA5NyA1NCA5NyA1MSA1MCA1NiA1MSA1MiA1MSA1NiA1NiAxMDAgMTAyIDU3IDEwMiA1MiAxMDEgMTI1
```
#### Snack 2 (25 point)
```
ppaHaH ykkun !harreMhC ytsir!sampaH N ypY we!raevaH  a egalfFTC d10{fbf6f6257784459adb3abbcccf22 .}6 uoYesed evr!!ti
```
I tried some shfit operations and some methods but cant get result. After that i realize that we can get meaningfull string by getting 4 chars and reverse them. Finally ı got this result and the flag:(Also you can write small python code to make it faster.)
```
Happy Hanukkah! Merry Christmas! Happy New Year! Have a flag CTF{01d6fbf526f4877a954a3bdccbb22fc6}. You deserve it!!
``` 
#### Snack 3 (25 point)
```
789c0bc9485548cb494c57c82c56700e71ab3631b34c354e3133b734364e49b130374b344bb348b24c33b1484d4e334f3635a9e5020083360ec2
```
I gave this string to cyberchef and result:
```
The flag is CTF{469e3d67933dd876a6f8b9f48ecf7c54}


Link: https://gchq.github.io/CyberChef/#recipe=From_Hex('None')Zlib_Inflate(0,0,'Adaptive',false,false)&input=Nzg5YzBiYzk0ODU1NDhjYjQ5NGM1N2M4MmM1NjcwMGU3MWFiMzYzMWIzNGMzNTRlMzEzM2I3MzQzNjRlNDliMTMwMzc0YjM0NGJiMzQ4YjI0YzMzYjE0ODRkNGUzMzRmMzYzNWE5ZTUwMjAwODMzNjBlYzI
```
#### Snack 4 (25 point)
```
4gw22559BUW013u2PVCWzWRLZCw6K1SPP6RQ7
w31IVPVXw04SWREVd0qeJ5jx8qAPsW7f5PdeJ
```
I tried a lot methods like rot, shift etc. and finally i thouhgt that it can be xor :D Xor this two string with cyberchef:
```
CTF{deca5eccfa0d4f220b84b26f8fd6ef64}

Link: https://gchq.github.io/CyberChef/#recipe=XOR(%7B'option':'UTF8','string':'w31IVPVXw04SWREVd0qeJ5jx8qAPsW7f5PdeJ'%7D,'Standard',false)&input=NGd3MjI1NTlCVVcwMTN1MlBWQ1d6V1JMWkN3NksxU1BQNlJRNw
```
#### Snack 5 (25 point)
```
9352663 86 8447 2278873 843 3524 38368 93 4673 968 9455 4283 2 47328 8463 843 62442 9673 47 44643727323
```
On this level we have a hint: have a look at your phone. All words are part of standard English dictionaries. I solved some crypto challenges like that and i tried T9 keyboard decode :)
```
https://www.dcode.fr/t9-cipher

Result: WELCOME TO THIS CAPTURE THE FLAG EVENT WE HOPE YOU WİLL HAVE A GREAT TIME THE MAGIC WORD IS GINGERBREAD

Flag : GINGERBREAD
```



---

### Snowfall (Stego) 75 Point

For this challenge we have been given an image file and this is stego challenge :) Lets open this image in stegsolve.
>Stegsolve is a tool that it try some filters on given image to find the hidden messages. <br>Installation script: https://github.com/zardus/ctf-tools/blob/master/stegsolve/install<br>
Start with: java -jar stegsolve.jar

Also i tried online stego website aperisolve.fr and some gimp edits but couldnt get the useful results. On stegsolve we can get the flag with green plane 3 filter.<br>
![](/assets/images/snowfall.png)<br>
After some guessing the flag is:
```
CTF{stegho_ho_ho}
```
---
### Smart Light System (Reversing) 100 Point

For this challenge we have been given an firmware file. Lets check it with "file" command
```
shad0w@faruk:~/Documents/ctf/hackazon/phase1/rev$ file firmware
firmware: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 3.2.0, BuildID[sha1]=1bcfc5a147ad758289dcd713652e84a40657554e, stripped
```
It's linux 64 bit executable file. When we run the file its get a license key and check this key.
```bash
shad0w@faruk:~/Documents/ctf/hackazon/phase1/rev$ ./firmware 
💽 Initializing firmware 💽
🔑 Reading key 🔑
hellooo
Checking license
⭕ Key is invalid ⭕
```
Open the given file in decompiler to find the key checking mechanism. I use Ghidra but you can use any decompiler like ida, r2 etc.
> Ghidra is a software reverse engineering and decompiling tool developed by NSA.<br>Download: https://ghidra-sre.org/ 

In ghidra we saw lots of functions but their names are not meaningful. To find the checking key function i searched for a "Checking license" string from Search -> Program Text -> All fields. This search returned 2 result. One of them is from data segment(string address) and another from  FUN_00400d98 function.<br> 
![](/assets/images/smartlightsystem.png)<br>
In this function, on line 16 it goes to checking the input that we enter and goes to if else block to print the result.<br>
![](/assets/images/Smartlightsystem1.png)<br>

Here we can see 2 defined arrays. After the definition it make loop while local_9c(used for index) is smaller than 0x3f (63) and this number is lenght of our arrays. It gets our input and make XOR with local_98 array and check is it equal local_58.
<br>
In XOR operation we have simple rule:
```
if A ^ B = C then C ^ B = A
Here A is our input
```
To get the true input ı wrote small python script to xor this arrays:
```python
a = [71,0,60,45,88,13,87,10,83,93,12,84,81,28,55,41,44,56,60,53,58,47,92,26,28,75,74,8,2,70,69,40,68,18,50,41,13,69,62,29,100,95,94,54,33,73,33,1,25,3,64,36,95,100,6,92,32,71,17,21,43,61,48,47]

b = [4,84,122,86,104,60,101,58,50,63,62,50,104,126,6,26,20,15,89,84,13,30,110,120,126,121,125,108,50,116,33,78,114,113,1,31,108,125,90,40,87,111,107,15,71,40,18,96,43,53,116,70,103,85,54,108,65,114,117,112,73,11,1,82]
i=0
for i in range(64):
	#print a[i],b[i],
	print str(chr(a[i]^b[i])),
```
And run it :)
```bash
shad0w@faruk:~/Documents/ctf/hackazon/phase1/rev$ python deneme.py 
C T F { 0 1 2 0 a b 2 f 9 b 1 3 8 7 e a 7 1 2 b b 2 7 d 0 2 d f 6 c 3 6 a 8 d 5 3 0 5 9 f a 3 a 2 6 4 b 8 1 0 0 a 5 d e b 6 1 }
```
Flag is :
```
CTF{0120ab2f9b1387ea712bb27d02df6c36a8d53059fa3a264b8100a5deb61}
```
---

### Happy Buckets (Reversing, Cloud) 200 Point
This challenge have 3 steps.
#### 1- HEAPS, BUNCHES, LOADS, MYRIADS, TONS (50 point)

When i open the given link music started. I looked at the source code of the page and found the url of this music.
```html
<!--audio player-->
		<audio controls autoplay>
			<source src="https://happybucketscontent.s3.eu-west-2.amazonaws.com/media/audio.ogg" type="audio/ogg" />
			<source src="https://happybucketscontent.s3.eu-west-2.amazonaws.com/media/audio.mp3" type="audio/mpeg" />
			Your browser does not support the audio element.
		</audio>
		<!--//audio player-->
```
(During the ctf i can access the files in this aws s3 bucket but now ı cant access the files.) When i opened this s3 bucket from browser ı found first flag and a jar file 
```
flag.txt = (I cant access now and ı didnt get any screenshot on this step :/ )
Santa-Letters-jar-with-dependencies.jar
```
#### 2- REVERSING (75 point)

Jar files are archive files contains java code and some another files like resource and manifest. I try to get the java files with Jadx but cant get and i tried online java decompiler http://www.javadecompilers.com/. This website use Procyon on the background for decompiling the jar files to java.
> Procyon is a code generation and analysis framework and it contains java decompiler.<br>
Source: https://github.com/mstrobel/procyon

I got java files from the online decompiler and start enumarating on them. After few minutes i found second flag and aws credentials on BucketsGui.java file.
```java
// /Santa-Letters-jar-with-dependencies_source_from_Procyon/com/happybuckets/mavenproject/BucketsGui.java
...
...
public static void main(final String[] args) {
        final String sneakyflag = "CTF{1ab1103d8248633dafa64f535d310037}";
        final BasicAWSCredentials awsCreds = new BasicAWSCredentials("AKIA5ZG7KIXZPIQMEA6R", "RzqFX+nnW/Ut9GurorpPr3ya6zOfjw5CvQclBYxs");
        final AmazonSQS sqsClient = ((AwsSyncClientBuilder<Subclass, AmazonSQS>)((AwsClientBuilder<AmazonSQSClientBuilder, TypeToBuild>)((AwsClientBuilder<AmazonSQSClientBuilder, TypeToBuild>)AmazonSQSClientBuilder.standard()).withCredentials(new AWSStaticCredentialsProvider(awsCreds))).withRegion("eu-west-2")).build();
        final String url = "https://sqs.eu-west-2.amazonaws.com/947508626930/HappyBucketsApp";
        final JFrame frame = new JFrame("HappyBuckets");
        frame.setDefaultCloseOperation(3);
        frame.setSize(400, 400);
        frame.setLocationRelativeTo(null);
        final JMenuBar mb = new JMenuBar();
        final JMenu m1 = new JMenu("Help");
        mb.add(m1);
        final JMenuItem m2 = new JMenuItem("About");
        m2.addActionListener(arg0 -> createFrame());
        m1.add(m2);
        final JTextArea ta = new JTextArea();
        ta.setEditable(false);
        final JButton reload = new JButton("Load message");
        reload.addActionListener(arg0 -> ta.setText(getSQSMessage(url, sqsClient)));
        final JPanel panel = new JPanel();
        panel.add(reload);
        frame.getContentPane().add("South", panel);
        frame.getContentPane().add("Center", ta);
        frame.getContentPane().add("North", mb);
        frame.setVisible(true);
    }
```
The second flag is:
```
CTF{1ab1103d8248633dafa64f535d310037}
```
#### 3- QUEUEING (75 point)
The credentials from previous step:
```
BasicAWSCredentials("AKIA5ZG7KIXZPIQMEA6R", "RzqFX+nnW/Ut9GurorpPr3ya6zOfjw5CvQclBYxs")

withRegion("eu-west-2")

https://sqs.eu-west-2.amazonaws.com/947508626930/HappyBucketsApp
```
I installed aws cli tool and set configurations with this credentials.
Guide: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html
```
$ aws configure
AWS Access Key ID : AKIA5ZG7KIXZPIQMEA6R
AWS Secret Access Key : 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
Default region name [eu-west-2]: 
Default output format [None]: 
```
After that i searched for mean of "queue" on aws services and found this documentation: https://docs.aws.amazon.com/cli/latest/reference/sqs/
I tried to get the list of queues:
```bash
shad0w@faruk:~$ aws sqs list-queues
{
    "QueueUrls": [
        "https://sqs.eu-west-2.amazonaws.com/947508626930/FlagQueue",
        "https://sqs.eu-west-2.amazonaws.com/947508626930/HappyBucketsApp"
    ]
}

```
After that i tried to get message from the Flag queue:
```bash
shad0w@faruk:~$ aws sqs receive-message --queue-url https://sqs.eu-west-2.amazonaws.com/947508626930/HappyBucketsApp

...
...
...
```
The last flag is:
```
CTF{}
// I cant copy the flag because aws queue is not accesable now. (28 Jan 2021)
```
---
### Santas Smart Home (Network) 200 Point
This challenge have 2 steps.
#### 1- Santas Smart Home (100 point)
On the webpage we can see only login screen and we dont have any password. After some enumeration we found that home asistant have logs files on error logs and we download the log file.
![](/assets/images/santasmarthome4.png)<br>

In log files we found  password but its not work on webpage:
```
Log file:
...
..
.
2020-12-05 22:30:43 ERROR (MainThread) [homeassistant.components.system_log.external] Could not connect to MQTT server with username 'santa' and password 'l1TTleH3lP3R'. Unknown connection error.
.
..
...
```
```
user: santa
pass: l1TTleH3lP3R
```
After that we searched the given ip on shodan.io to find a service to use this credentials and found some information about that:
> shodan.io is search engine for internet connected devices and useful for enumerating ip adresses :)

![](/assets/images/santasmarthome.png)<br>
As seen in image mqtt works on port 1883 and upnp works on 1900. After this enumeration i made some searches about mqtt.
>A lightweight messaging protocol for small sensors and ıot devices.
After some Google searches again and i found mosquito.
>Mosquito is a client that will subscribe to topics and print the messages that it receives.
I installed it on my ubuntu machine. <br>
For usage ı used this guide:http://www.steves-internet-guide.com/mosquitto_pub-sub-clients/

We have username and password but we dont know the device names that we want to subscribe and receive message from them. I found that we can subscribe all devices with # character :)<br>
https://stackoverflow.com/questions/32904012/how-do-i-subscribe-to-all-topics-of-a-mqtt-broker

Lets try:
```bash
shad0w@faruk:~$ mosquitto_sub -h 3.64.12.74 -v -t '#' -u santa -P l1TTleH3lP3R -d 
Client (null) sending CONNECT
Client (null) received CONNACK (0)
Client (null) sending SUBSCRIBE (Mid: 1, Topic: #, QoS: 0, Options: 0x00)
Client (null) received SUBACK
Subscribed (mid: 1): 0
Client (null) received PUBLISH (d0, q0, r0, m0, 'santa/smarthome/forgotpassword', ... (110 bytes))
santa/smarthome/forgotpassword {"name": "Password", "value": "I knew you forgot your password again! Here you go: S4nt4_cL4us3=c0m1ng2T0Wn!"}
Client (null) received PUBLISH (d0, q0, r0, m0, 'santa/smarthome/northpole_temperature/east', ... (58 bytes))
```
And yes we found another password:
```
S4nt4_cL4us3=c0m1ng2T0Wn!
```
I logged in  with this password on webpage and after opened up the lights i got the flag :)
![](/assets/images/santasmarthome1.png)<br>
Flag is:
```
CTF{S4nt4s_Sh0ulD_S3cur3_t0o!}
```
#### 2- Santa's Smart Sleigh (100 point)
On this step upnp is our hint. On the previous step ı opened up shodan and find some results. On port 1900 upnp running.
```
HTTP/1.1 200 OK
CACHE-CONTROL: max-age=3600
DATE: Mon, 25 Jan 2021 19:59:48 GMT
ST: upnp:rootdevice
USN: N0RTHP0LE-C0LD-1337-RUD0LPH::
EXT: 
SERVER: Linux UPnP/1.0 Santa-UPnP/0.1
LOCATION: http://169.254.172.42:8080/descr.xml
BOOTID.UPNP.ORG: 11688039
CONFIGID.UPNP.ORG: 24015

UPnP Device:
  Device Type: urn:schemas-upnp-org:device:SmartSleigh:1
  Friendly Name: Santa's Smart Sleigh
  Manufacturer: ElfCorp.
  Manufacturer URL: https://www.youtube.com/watch?v=C41q5YLnF10
  UDN: uuid:N0RTHP0LE-C0LD-1337-RUD0LPH
```
On location line we can see a link. I tried this link with our ip address:
```
http://3.64.12.74:8080/descr.xml
```

![](/assets/images/santasmarthome2.png)<br>

On the service section we can see another path for flag :)
```
http://3.64.12.74:8080/SmartSleigh/api/GetFlag
```
![](/assets/images/santasmarthome3.png)<br>

The second flag is:
```
CTF{N0t_s0_SM4rT_N0w,Sl31gh!}
```

---
### Choises Choises (Reversing) 150 point
On this challenge we have an exe file. I opened it on my windows machine.<br>
![](/assets/images/choises.png)<br>
We need to set true color values to circles on christmas tree to get the flag.I checked the file with Die(Detect it easy).
>Detect it easy is a packer identifier and it can deetrmine type of files.<br>
https://github.com/horsicq/Detect-It-Easy


![](/assets/images/choises2.png)<br>

This exe file written by .net, we can decompile .net applications with dnSpy.
> dnSpy is a debugger and .NET assembly editor. You can use it to edit and debug assemblies even if you don't have any source code available.<br>https://github.com/dnSpy/dnSpy

![](/assets/images/choises1.png)<br>
On the left side we can use explorer to find the useful functions and files. In main window section ı found the DecryptFlag function.
```cs
// ChoicesChoices.MainWindow
// Token: 0x0600000C RID: 12 RVA: 0x000025B8 File Offset: 0x000007B8
private void DecryptFlag()
{
  byte[] array = new byte[25];
  for (int i = 0; i < 25; i++)
  {
    if (this.allBalls[i].Tag != null)
    {
      array[i] = Convert.ToByte(this.allBalls[i].Tag);
    }
  }
  if ((array[20] ^ 60) == array[7] && array[21] - array[3] == 0 && array[2] >> 2 == 24 && array[13] == 95 && 2 * array[11] == 190 && array[7] >> (int)(array[17] % 4) == 11 && (array[6] ^ 4) == array[14] && array[8] == 115 && (int)array[5] << (int)(array[9] % 2) == 190 && array[16] - array[7] == 16 && (int)array[7] << (int)(array[23] % 3) == 380 && array[2] - array[7] == 2 && array[21] == 95 && (array[2] ^ 62) == array[3] && array[0] == 66 && array[13] == 95 && (array[20] & 240) == 96 && (array[8] & 171) == 35 && array[12] == 67 && array[4] >> 4 == 7 && array[13] == 95 && array[0] >> (int)(array[0] % 4) == 16 && (array[8] & 172) == 32 && array[16] == 111 && (array[22] & 61) == 37 && array[9] == array[5] && array[5] == 95 && array[20] >> 4 == 6 && array[7] >> 1 == 47 && array[1] == array[9] && array[3] >> 4 == 5 && (array[19] & 73) == 73 && array[4] == 119 && (array[2] & array[11]) == 65 && array[0] == 66 && array[4] + array[5] == 214 && (int)array[15] << 1 == (int)(2 * array[1]) && (array[10] ^ 38) == array[17] && (array[12] ^ 52) == array[4] && array[19] - array[21] == 0 && array[12] - 10 == 57 && array[15] >> 1 == 47 && array[19] == 95 && array[17] + array[18] == 200 && array[22] == 101 && (array[23] & array[21]) == 95 && (int)array[6] << (int)(array[19] % 2) == 216 && (array[22] & 237) == 101 && (array[12] & 220) == 64 && (array[18] ^ 54) == array[15] && (array[16] & 170) == 42 && (array[0] & 221) == 64 && (array[6] ^ 51) == array[21] && array[20] + 156 == 255 && array[19] + 156 == 251 && array[12] == 67 && array[2] + 119 == 216 && array[23] == 95 && array[10] + 26 == 147 && (array[22] & array[9]) == 69 && array[3] + array[2] == 192 && array[14] - array[17] == 9 && array[21] == 95 && (array[19] ^ 38) == array[10] && (array[6] & 255) >= array[1] && (array[12] & array[22]) == 65 && array[7] >> (int)(array[19] % 4) == 11 && (array[20] ^ 6) == array[22] && array[12] - 16 == 51 && array[19] - array[13] == 0 && array[14] >> 4 == 6 && (array[12] & 105) == 65 && array[8] >> (int)(array[10] % 4) == 57 && array[6] >> (int)(array[22] % 8) == 3 && array[22] - array[5] == 6 && array[7] >> (int)(array[22] % 7) == 11 && array[16] + array[22] == 212 && (array[23] ^ 54) == array[18] && array[23] + array[14] == 199 && (array[5] & array[2]) == 65 && (array[15] & 187) == 27 && (array[23] ^ 29) == array[0] && (array[6] ^ 51) == array[11] && (int)array[24] << 1 == 230)
  {
    this.ShowThoughBalloon("HO HO HO \r\nMerry X-MAS: CTF{" + Encoding.ASCII.GetString(array) + "}", 10);
    return;
  }
  this.ShowThoughBalloon("❌❌❌ Jeee, that's an ugly tree. No iPhone 12 Pro Max and no flag for you this year buddy ❌❌❌", 5);
}
```
This funtion gets all values from balls on the christmas tree and check the values with huge if block. If all this values are true it gives the flag with this array, else it return "ugly tree" :) <br>
I copied this if block and move to text editor and start finding values.
![](/assets/images/choises3.png)<br>

After founding all the values ı wrote a simple python code to get the string from ascii values.
```python
array = [None]*25

array[0] = 66
array[1] = 95
array[2] = 97
array[3] = 95
array[4] = 119
array[5] = 95
array[6] = 108
array[7] = 95
array[8] = 115
array[9] = 95
array[10] = 121
array[11] = 95 
array[12] = 67
array[13] = 95
array[14] = 104
array[15] = 95 
array[16] = 111
array[17] = 95
array[18] = 105
array[19] = 95 
array[20] = 99
array[21] = 95
array[22] = 101
array[23] = 95
array[24] = 115

result = ''.join(chr(i) for i in array)
print(result)
```
With this script ı got this flag:
```
CTF{B_a_w_l_s_y_C_h_o_i_c_e_s}
```

---

### Can You Check My CV? (Misc) 150 point
In this challenge we have CV generator service. We can access this service with netcat on terminal.
```
$ nc portal.hackazon.org 17001 
What's your name: sh4d0wless
...
...
...
...
...
/Producer (pdfTeX-1.40.19)
/Creator (TeX)
/CreationDate (D:20210128220420Z)
/ModDate (D:20210128220420Z)
/Trapped /False
/PTEX.Fullbanner (This is pdfTeX, Version 3.14159265-2.6-1.40.19 (TeX Live 2019/dev/Debian) kpathsea version 6.3.1/dev)
>>
endobj
...
...
...
```
When we connect to service, it asks a name and print some non readable text. But on the last lines ı saw that this lines produced by pdfTex.
>  pdfTeX is an extension of TeX which can produce PDF directly from TeX source, as well as original DVI files.

So we can save this lines as pdf file:
```
$ nc portal.hackazon.org 17001 > deneme.pdf
sh4d0wless
$ ls
deneme.pdf
```
It saved to pdf, lets check what its contain.
<br>
![](/assets/images/cv.png)
<br>
It says something about file descriptors and add the name(our input) to last line. I searched exploits etc. for pdfTex version but cant find anything. After some Google searchs i found the LaTex injection cheetsheat.
<br>
Source: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/LaTeX%20Injection

I tried one by one and this one worked :)
```
\verbatiminput{/etc/passwd}
```
I gave this input and result:
<br>
![](/assets/images/cv1.png)
<br>
Yes we can read files! We have "file descriptor" hint. On linux systems /proc/self/fd/ contains current processes file descriptors. On linux 0,1 and 2 is pre defined file descriptors:
```
0 	Standard input 	  STDIN_FILENO 	  stdin
1 	Standard output 	STDOUT_FILENO 	stdout
2 	Standard error 	  STDERR_FILENO 	stderr
```
Lets try next one.
```
$ nc portal.hackazon.org 17001 > deneme.pdf
\verbatiminput{/proc/self/fd/3}
```

![](/assets/images/cv2.png)

<br>

We find another path, read it :D
```
$ nc portal.hackazon.org 17001 > deneme.pdf
\verbatiminput{/home/elf/file_descriptors.c}
```


![](/assets/images/cv3.png)
<br>

And the flag is:
```
CTF{NEVer_7RU757_u5ER_1nPu7}
```

---

