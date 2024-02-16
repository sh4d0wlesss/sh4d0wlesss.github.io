---
title:  "Mobile Hacking Lab Food Store Writeup"
date:   2024-01-22T16:27:00-0400
categories:
  - writeup
tags:
  - android
  - ctf
---


Hello everyone!
In this blog post , I will try to explain my solution steps for [Food Store](https://www.mobilehackinglab.com/course/lab-food-store) challenge from Mobile Hacking Lab. 

## Static Analysis
When we open the app in emulator, it opens a login page with login and signup buttons and below there is a note that says the users created with signup will be a regular users, for pro user privilages we need to contact with administrator. We can understand that our aim is getting pro account on this app👀. Let's open apk with jadx to understand the app.

![](./assets/images_mhl_foodstore/manifest.png)

When we examine the manifest file, we see that we have three activity defined and two of them are exported. Lets start with LoginActivity:

### Login Activity

![](./assets/images_mhl_foodstore/loginactivity.png)

As seen in screenshot, there are two different method setted as onClick listener for login adn signup buttons. When we press the login button, app checks the given password with username. If password is true, it sends intent to MainActivity with username, credit, address and isPro value. If user is pro, user credit will be 10k but for regular users it will be only 100🥺 If we press signup button, it start signup activity with intent without extras.

### Signup Activity

![](./assets/images_mhl_foodstore/signup.png)

In signup activity, app checks the input boxes is null or not and creates user object with given informations. After that it sends this object to addUser function of dbHelper class.

### dbHelper

On Android, you can create your database as sqlite database [by extending SQLiteOpenHelper class](https://developer.android.com/training/data-storage/sqlite#DbHelper) and define your methods for adding or deleting data to your database.

![](./assets/images_mhl_foodstore/dbhelper.png)

In this application, we have a dbhelper class that extends SQLiteOpenHelper and some methods defined inside it. getUserByUsername method called during login and returns user object with given username. Also addUser method called during signup and creates new user with given user object. Before adding informations to database, password and address strings are encoded with Base64. But we have a problem on this code. Username value is concatenating to sql query without any validation, which can lead to a SQL injection attack. Maybe we can get pro user with this vulnerability.

Here is my sample payload💀:
```sql
sh4d0wless','MTMzNw==','MTMzNw==',1)--

+ pass and addr = 1337  -> base64 -> MTMzNw==
+ 1 -> pro user value
```
if i give this payload as username, the sql query inside adduser method will become this:

```sql
INSERT INTO users (username, password, address, isPro) VALUES ('"sh4d0wless','MTMzNw==','MTMzNw==',1)-- + "', '" + encodedPassword + "', '" + encodedAddress + "', 0)
```
With double dash(`--`) we can make comment rest of query and we cant inject our user as pro user😈

![](./assets/images_mhl_foodstore/error1.png)

But i got an error with this payload :/ To understand what the hack is happening, i used [this frida script.](https://codeshare.frida.re/@ninjadiary/sqlite-database/)

![](./assets/images_mhl_foodstore/error2.png)

I got an error after `--`. I think it gives error because we cant add whitespace after `--` because before creating user object, app used trim function to remove whitespaces from given strings and (as i know) we have to add at least one whitespace after `--` chars in our query if commenting start at end of the query. Also as seen in frida output, address and isPro values are printed after newlines and our double dash can only comment the first line of query and maybe other lines can cause this error(But i tried commenting without spaces and it worked on online sqlite website :/ I'm confusing and if you know the real problem here, please dm me on twitter or linkedin with solution👉👈)

But wait, we can also use `/*` to commenting(multiline) on sqlite. Lets try it:
```sql
sh4d0wless','MTMzNw==','MTMzNw==',1)/*
```
![](./assets/images_mhl_foodstore/success.png)

It works🎉🎉 We have 10k credit.

And here is poc video:

[![](https://img.youtube.com/vi/P9AT2L-slcg/0.jpg)](https://www.youtube.com/watch?v=P9AT2L-slcg)

Thanks for reading! See you in next writeups 👋🏻👋🏻