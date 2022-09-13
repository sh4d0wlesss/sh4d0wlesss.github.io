---
title:  "Deloitte Hackazon Writeup Phase-2"
date:   2021-02-03T00:48:00-0400
categories:
  - writeup
tags:
  - ctf
---

## PHASE 2 QUESTIONS
### SANTA'S GIFTSHOPPER (PPC) 100 point
When i connected the given link it prints some ascii arts and says somethings about santas notebook. After that it print 1000 time gifts and ask number of gifts. We need to keep the number of gifts and send them as a answer to questions. Every execution it pritns randomly gifts so the numbers change every connection. I wrote a python script for making all the things automatically.
```python
from pwn import *
# tcp://portal.hackazon.org:17001
conn = remote('portal.hackazon.org',17001)
cevaplar = ""
i = 1
while(i<1001):
	cevap = conn.recvuntil('..>')
	print(cevap)
	cevaplar = cevaplar + cevap
	print(conn.sendline('n'))
	print i
	i += 1


Books = cevaplar.count('book') - 1
Racecar = cevaplar.count('racecar')
Crayons = cevaplar.count('crayons')
Puzzle = cevaplar.count('puzzle')
Doll = cevaplar.count('doll')
mouse = cevaplar.count('mouse')
Binocular = cevaplar.count('binoculars')
Kitten = cevaplar.count('kitten')
Guitar = cevaplar.count('guitar')
console = cevaplar.count('console')
print 'Books' ,Books, '\nRacecar' ,Racecar, '\nCrayons' ,Crayons, '\nPuzzle' ,Puzzle, '\nDoll' ,Doll, '\nmouse' ,mouse, '\nBinocular' ,Binocular, '\nKitten' ,Kitten, '\nGuitar' ,Guitar, '\nconsole' ,console


for i in range(10):
	soru = conn.recvuntil('>')
	print soru

	if soru.count('book'):
		conn.sendline(str(Books))
	elif soru.count('racecar'):
		conn.sendline(str(Racecar))
	elif soru.count('crayons'):
		conn.sendline(str(Crayons))
	elif soru.count('puzzle'):
		conn.sendline(str(Puzzle))
	elif soru.count('doll'):
		conn.sendline(str(Doll))
	elif soru.count('mouse'):
		conn.sendline(str(mouse))
	elif soru.count('binoculars'):
		conn.sendline(str(Binocular))
	elif soru.count('kitten'):
		conn.sendline(str(Kitten))
	elif soru.count('guitar'):
		conn.sendline(str(Guitar))
	elif soru.count('console'):
		conn.sendline(str(console))
	else:
		print 'upps!'

print(conn.recvline())
print(conn.recvline())
print(conn.recvline())
print(conn.recvline())
print(conn.recvline())
print(conn.recvline())
```
To briefly explain the code first of all im making connection to given service with pwntools. I used pwntools for connection because its simple :) After connection  i stored all the answers from server on array and count the gift names with .count function. 
>Warning: On the "book" count i subtracted one from count because after the ascii art the sentence contain "book" substring because of "notebook".


After that send the numbers to server on every question and receive lines.
![](/assets/images/santasgiftshop.png)<br>
...<br>
...<br>
...<br>
![](/assets/images/santasgiftshop1.png)

Flag is:
```
CTF{S4nt4-H4s-4-B4d-M3m0ry}
```

---
### How the Grinch Stole Time (Reversing) 200 point
This challenge we have another exe file and now its not an .net  :) When we run the exe it print some characters and wait seconds and print again but the waiting time  increasinging after every char. We need to bypass this wait time.Lets open the exe with Ghidra.
![](/assets/images/time.png)<br>
I found the sleep function from symbol tree on imports section. Its imported from kernel32.dll and used in this function.(you can find the cross references with right click menu.) <br>
In this function sleep function called with dwmilisecomd parameter. This parameter comes from another function(line 79).<br>
On the number 1, the FUN_00401000 function is called and it returns value to uvar6 and on the number 2 same function called again and it return value to uVar7. After that,on the number 3, this values subtracted and sended as parameter to another function.
![](/assets/images/time1.png)<br>
 On function FUN_00401000 , QueryPerformanceCounter function used. This function is generally used on anti-debug techniques.
 > QueryPerformanceCounter: This is the time-related
API of kernel32.dll. This API returns the current data value
of the hardware performance counter to the specified
memory. Because DBI execution is slower than the real
execution, we can judge that the program is being analyzed
when the difference between the two data values exceeds a
predetermined threshold value after this API is called twice<br>
Source: https://www.researchgate.net/publication/333577831_Automatic_Detection_and_Bypassing_of_Anti-Debugging_Techniques_for_Microsoft_Windows_Environments

All this thing mean that If we remove the uVar7 assignment, subtruction step(part 2,part 3) and sleep step we can get the flag directly without waiting the sleep call. I did it with x32dbg.
![](/assets/images/time2.png)<br>

You can find the same code on x32dbg using offset of sleep code on Ghidra. On ghidra the sleep call is on the "004011bb" and the base adress was 0040100, so the ofsett is "1bb".
On x32dbg base adress is "00bf1000" and the slepp call on "00bf11bb".<br>
To bypass the sleep function and debugger check we need to delete this selected lines. We delete "push eax" to keep the stack layout correctly. <br>
We cant delete this line directly because its broke the program and addressing but we can use nop instruction to replace them.
>NOP is no operation instruction. https://en.wikipedia.org/wiki/NOP_(code) 

On x32dbg:
```
right click the assembly code -> assemble -> select fill with nop and change instruction to nop
``` 
![](/assets/images/time3.png)<br>

After this steps this code will look like that:
![](/assets/images/time4.png)<br>

And run the code :) 
> Note: Add breakpoint to sleep function to see the flag.The program will close quickly if we do not put breakpoint.

![](/assets/images/time5.png)<br>

Flag is:
```
CTF{8aca06c44}
```



---
### Present Drop (Network) 50 point
We have a dump.pcap file. Lets open it up with wireshark.
>Pcap files are packet capture files and wireshark is a tool that can be analyze pcap files.<br>https://www.wireshark.org/

Wireshark list some tcp and http packages. 
![](/assets/images/presentdrop.png)
To understand the whole data stream between source and destination we can use "follow stream" option. <br>
Right click the first package -> Follow -> TCP Stream
![](/assets/images/presentdrop1.png)

We can see some meaningless lines but on the first line of data received from destination, PNG header is a hint for us. This stream contains PNG file and we can export files from pcap files with wireshark.<br>
To export the files: File -> Export Objects -> HTTP -> Save All.<br>
We extract the happy-holidays.png from pcap, lets open it :)<br>
![](/assets/images/happy-holidays.png)<br>
Flag is:
```
CTF{HAPPY_HOLIDAYS}
```
---
### Candle Locator (Forensics) 50 point
On this challenge we have an jpg file and we know that it's not contains stego. Get more info from pictures we can look at the exif data.
>Exif data is a metadata that contains some informations about file like resolution, encoding, photograph machine, geolocations etc.

We can get exif data with online tool but i used exiftool on my linux machine.
```
$ exiftool hanukkah.jpg 
ExifTool Version Number         : 12.00
File Name                       : hanukkah.jpg
Directory                       : .
File Size                       : 1135 kB
File Modification Date/Time     : 2021:01:28 00:15:41+03:00
File Access Date/Time           : 2021:01:28 00:16:30+03:00
File Inode Change Date/Time     : 2021:01:28 00:16:30+03:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.02
Exif Byte Order                 : Big-endian (Motorola, MM)
X Resolution                    : 100
Y Resolution                    : 100
Resolution Unit                 : None
Y Cb Cr Positioning             : Centered
GPS Version ID                  : 2.3.0.0
XMP Toolkit                     : Image::ExifTool 10.80
Quality                         : 100%
DCT Encode Version              : 100
APP14 Flags 0                   : [14], Encoded with Blend=1 downsampling
APP14 Flags 1                   : (none)
Color Transform                 : YCbCr
Image Width                     : 3000
Image Height                    : 3000
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:4:4 (1 1)
Image Size                      : 3000x3000
Megapixels                      : 9.0
GPS Latitude                    : 52 deg 4' 40.06" N
GPS Longitude                   : 4 deg 20' 4.83" E
GPS Latitude Ref                : North
GPS Longitude Ref               : East
GPS Position                    : 52 deg 4' 40.06" N, 4 deg 20' 4.83" E
```
At the bottom of result, we find some geolocation data. Lets transform them to search on Google maps.(https://www.gps-coordinates.net/gps-coordinates-converter)
```

Latitude: 52.0777944
Longitude: 4.334675
```
Result from Google Maps: https://www.google.com.tr/maps/place/52%C2%B004'40.1%22N+4%C2%B020'04.8%22E/@52.0777746,4.3342405,19z/data=!4m5!3m4!1s0x0:0x0!8m2!3d52.0777944!4d4.334675

The street is "Schenkkade". Flag is:
```
CTF{Schenkkade}
```
---
