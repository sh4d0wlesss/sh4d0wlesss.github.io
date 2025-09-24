---
title:  "8kSec FactsDroid Writeup"
date:   2025-09-12T16:32:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for FactsDroid challenge from 8kSec Android Labs. 

```
Goal:
Successfully implement a Machine-in-The-Middle (MITM) attack that allows you to manipulate the facts being displayed to the user, potentially inserting custom content or modifying the retrieved facts before they reach the application.
Response manipulation is fair game.
```

## Static Analysis

Main function of application is getting random facts from remote server on internet and shows on screen. When we open the app in emulator, it says our device is rooted and because of that "random facts" button is disabled. We need to bypass root checks. Lets start our analysis with jadx.

### Code Analysis

![](/assets/images_8ksec_FactsDroid/jadx0.png)

Manifest file have only one activity, MainActivity. Also we can see some keywords related to flutter. 

![](/assets/images_8ksec_FactsDroid/jadx1.png)

In MainActivity code, we have 2 variable. When we look references for f502g, we can find root check mechanism.

![](/assets/images_8ksec_FactsDroid/jadx2.png)

`g` function is responsible for root checking. Its check for binaries, check debugable status etc. For bypassing this checks we can use frida. 

```js
Java.perform(function(){
    let a = Java.use("B.a");
    a["g"].implementation = function (aVar, jVar) {
        console.log(`a.g is called: aVar=${aVar}, jVar=${jVar}`);
        console.log("root bypassed");
    };
});
```
With this frida script, we can hook the `g` function and just print some log to console instead of making root checks.

![](/assets/images_8ksec_FactsDroid/frida2.png)

We can succesfully bypassed root checks but we have another error.

![](/assets/images_8ksec_FactsDroid/after_root_bypass.png)

I don't know what caused this error, but I ignored it and tried to proxy outgoing requests from the application. This application is made with Flutter and we need to bypass ssl pinning. NVISO have developed great script for this job. [This frida script](https://github.com/NVISOsecurity/disable-flutter-tls-verification/blob/main/disable-flutter-tls.js) searches specific pattern for finding the function that responsible with certification validation and hook it. Let's try it(don't forget to add root bypass code to beginning of script)

![](/assets/images_8ksec_FactsDroid/frida1.png)

Script succesfully found the ssl verification function offset inside libapp.so and patched it. Now we can get random facts from server without error. The final step is intercepting and modifying the response from server.
Flutter apps not trust the system proxy setting on Android. Because of that we will use ProxyDroid app to proxying packets to burpsuite(Also you can use ip table rules).
I have set the proxy setting to my host ip and 8080 port. After that configure the burp for listen all interfaces and enable invisible proxy. Now we can capture packets.

![](/assets/images_8ksec_FactsDroid/intercept.png)

For changing the random fact comes from server, intercept request and intercept the response too.

![](/assets/images_8ksec_FactsDroid/intyercept2.png)

![](/assets/images_8ksec_FactsDroid/intercept3.png)

Thats all, see you!

PoC Video: [Youtube](https://youtu.be/grgNCHF4Edc)