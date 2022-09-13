---
title:  "Deloitte Hackazon Writeup Phase-3"
date:   2021-02-03T00:48:00-0400
categories:
  - writeup
tags:
  - ctf
---

## PHASE 3 QUESTIONS
### Happy New Maldoc (Reversing) 125 point 
 As seen in challenge name it is about document with malware. I opened this slide on my windows machine. (I tried something on ubuntu libre office but something went wrong.)<br> This document contains only one slide and button on that. I opened the macro section (view -> macro) and start enumerating the vba script code. This code contains lots of trash functions and at the bottom of them ı found the funtion that run when we click the button.<br>
 ![](/assets/images/maldoc.png)
 <br>
 This function have 2 variable, one of them is cap and another one is sHN. cap get the values from left function and sHN gets the value from enviroment variable.<br>
 We can add variables and functions calls to "Add watch" section with right click -> add watch. I added some variebles and set breakpoint(f9) on if block. <br>
 In if block it checks the sHN and if its correct, it prints success message and the sHN is our pc name :D As seen on watches section is checks is our pc name is "SANTA-PC" or not. You can change your pc name to "SANTA-PC" or change the sHN befora the if check.
 ![](/assets/images/maldoc1.png)

I replaced sHN with the requested value and run teh code and flag appeared on the watches section :)
Flag is :
```
CTF{im_a_maldoc_pro}
```
