---
title:  "Mobile Hacking Labs Strings Writeup"
date:   2024-01-01T00:48:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for [Strings](https://www.mobilehackinglab.com/course/lab-strings) challenge from Mobile Hacking Labs. 

## Static Analysis
Mobile Hacking Labs give you 120 min online lab as Corellium instance but i want to solve the challenge on my machine and download base.apk from corellium instance. 

When we open the app in emulator, it opens blank page with "Hello from c++" text. Let's open it with jadx to understand the app.

![](/assets/images_mhl_strings/activity2manifest.png)

As seen in the AndroidManifest.xml file, we have another exported activity with intent filter. To properly triger this activity we need to send intent with uri with "mhl" scheme and "labs" host.
```bash
adb shell am start -d "mhl://labs" -n com.mobilehackinglab.challenge/.Activity2
```
But we need to know what happened when this activity triggered. To undestand this, lets analyze Activity2

![](/assets/images_mhl_strings/activity2part1.png)
 
On part 1:
- app try to read DAD4.xml file from shared preferences and reads a string value associated with the key "UUU0133"
- compare the action of retrieved intent with "android.intent.action.VIEW"
- compare the retrieved value from shared preferences with result of "cd" function

The cd function in the activity2:

```java
private final String cd() {
        String str;
        SimpleDateFormat sdf = new SimpleDateFormat("dd/MM/yyyy", Locale.getDefault());
        String format = sdf.format(new Date());
        Intrinsics.checkNotNullExpressionValue(format, "format(...)");
        Activity2Kt.cu_d = format;
        str = Activity2Kt.cu_d;
        if (str == null) {
            Intrinsics.throwUninitializedPropertyAccessException("cu_d");
            return null;
        }
        return str;
    }
```
This function returns the today's date as string with dd/mm/yyyy format. To bypass the `isActionView && isU1Matching` checks we need to create proper shared preferences file and send the intent with VIEW action. While I was thinking about how to create the shared preferences file, I saw the KLOW function in MainActivity.

```java
  public final void KLOW() {
        SharedPreferences sharedPreferences = getSharedPreferences("DAD4", 0);
        SharedPreferences.Editor editor = sharedPreferences.edit();
        Intrinsics.checkNotNullExpressionValue(editor, "edit(...)");
        SimpleDateFormat sdf = new SimpleDateFormat("dd/MM/yyyy", Locale.getDefault());
        String cu_d = sdf.format(new Date());
        editor.putString("UUU0133", cu_d);
        editor.apply();
    }
```
This function creates the file we need. I will use frida to call this function in nest steps. Lets continue on activity2.

Insıde the if block,application checks the uri and scheme as we saw in the manifest file. After that it gets last segment of path and compare the base64 decoded value of string with "str" variable. We can decrypt given string with hardcoded key:

```
Enc string -> "bqGrDKdQ8zo26HflRsGvVA=="
Key -> "your_secret_key_1234567890123456"
IV -> "1234567890123456" (comes from Activity2Kt.fixedIV variable)
Mode -> CBC
```
![](/assets/images_mhl_strings/cyberchef.png)

Our secret value is "mhl_secret_1337". If we send the base64 encoded version of this secret string, we can pass all checks and app will load "flag" library and call getflag function.(And dont forget to call KLOW function before this steps :D)
```sh
sh4d0wless@paradise:~$ echo -n "mhl_secret_1337" | base64
bWhsX3NlY3JldF8xMzM3

```


## Testing solution with emulator

I wrote this meaningless frida script to call KLOW fucntion and see some other functions return values.
```js
Java.perform(function () {


  setTimeout(function () {

    Java.choose("com.mobilehackinglab.challenge.MainActivity" , {
      onMatch : function(instance){ 
        console.log("Found instance: "+instance);
        console.log("call KLOW func: " + instance.KLOW());
      },
      onComplete:function(){}
    
    });
  }, 1000);

  setTimeout(function () {
    Java.choose("com.mobilehackinglab.challenge.Activity2" , {
        onMatch : function(instance){ 
          console.log("Found instance: "+instance);
          console.log("cd func: " + instance.cd());
          console.log("native func: " + instance.getflag());
        },
        onComplete:function(){}
      
      });
    }, 10000);
  
});
```
And for sending intent we can use adb:
```sh
adb shell am start -a android.intent.action.VIEW -W -d "mhl://labs/bWhsX3NlY3JldF8xMzM3" -n com.mobilehackinglab.challenge/.Activity
```
![](/assets/images_mhl_strings/result1.png)

Oh, we get Success message but where is the flag? I tried to understand what happened in the getflag function from libflag.so but its obfuscated. It contains tons of functions and they call tons of memcpy and mhl gives us a hint about frida memory dump. Because of that first of all i tried fridump tool to dump memory of application and grepped the flag format because i dont know how to dump app memory inside of my frida script.

![](/assets/images_mhl_strings/result2.png)

Flag owned :)
But i searched frida memory scan in frida docs and [found a useful code snippet](https://frida.re/docs/javascript-api/#memory) and changed it a little bit.

```js
Java.perform(function () {

  //---------------------------------------------HOOK MAIN ACTIVITY FUNCTIONS------------------
  setTimeout(function () {

    Java.choose("com.mobilehackinglab.challenge.MainActivity" , {
      onMatch : function(instance){ 
        console.log("Found instance: "+instance);
        console.log("call KLOW func: " + instance.KLOW());
      },
      onComplete:function(){}

    });
  }, 1000);

  //---------------------------------------------HOOK ACTIVITY2 FUNCTIONS------------------
  setTimeout(function () {
    Java.choose("com.mobilehackinglab.challenge.Activity2" , {
        onMatch : function(instance){ 
          console.log("Found instance: "+instance);
          console.log("cd func: " + instance.cd());
          console.log("native func: " + instance.getflag());
        },
        onComplete:function(){}
      
      });
    }, 5000);

    //---------------------------------------------MEMORY SCAN FOR FLAG------------------
    setTimeout(function () {
      //find libflag.so from modules
      const m = Process.getModuleByName("libflag.so")
      console.log(JSON.stringify(m));
      //console.log(hexdump(m.base,{length:m.size}));

      // scan module memory for flag format 
      const pattern = '4d 48 4c 7b'; // MHL{

      Memory.scan(m.base, m.size, pattern, {
        onMatch(address, size) {
          console.log('Memory.scan() found match at', address,
              'with size', size);
        },
        onComplete() {
          console.log('Memory.scan() complete');
        }
      });
      
      const results = Memory.scanSync(m.base, m.size, pattern);// [{"address":"0x7025edb83fc0","size":4}] 
      console.log("Result:" + JSON.stringify(results));
      const flag_addr = results[0].address;
      console.log(hexdump(flag_addr,{length: 30}));

    }, 10000);
  
});
```
And here is the flag again :

![](/assets/images_mhl_strings/frida_memscan.png)

Thanks for reading :)
