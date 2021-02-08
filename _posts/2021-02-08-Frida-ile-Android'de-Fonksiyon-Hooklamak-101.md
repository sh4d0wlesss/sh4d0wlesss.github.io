---
layout: post
title:  "Frida ile Android'de Fonksiyon Hooklamak 101"
date:   2021-02-08 00:48:00 +0300
categories: Mobile Application Security
---
# Frida ile Android'de Fonksiyon Hooklamak 101
Merhaba!<br>
Bu yazıda başlıktan da anlaşıldığı üzere frida kullanarak giriş seviyesinde function hook nasıl yapılır adım adım anlatmaya çalışacağım. Ben de frida öğrenme aşamasındayım, bu yüzden yanlış yazıda bir şey var ise bana mesaj vs. atarsanız çok sevinirim.   Kurban uygulamamız OWASP Uncrackable1 olacak.
> Uygulama Link: https://github.com/OWASP/owasp-mstg/blob/master/Crackmes/Android/Level_01/UnCrackable-Level1.apk

Apk dosyasını genymotiona sürükleyip bırakıyoruz ve uygulama yükleniyo. Uygulama başladığında 
```
Root detected!
This is unacceptable. The app is now going to exit.
```
diye bize kızıyor. Bu kontrolü atlatmak için frida devreye girecek. Önce apkyı jadx ile açıp koda bakalım.
```java
//sg.vantagepoint.uncrackable1.MainActivity
public void onCreate(Bundle bundle) {
        if (c.a() || c.b() || c.c()) {
            a("Root detected!");
        }
        if (b.a(getApplicationContext())) {
            a("App is debuggable!");
        }
        super.onCreate(bundle);
        setContentView(R.layout.activity_main);
    }
```
Root detected yazısının geçtiği onCreate metodunu görüyoruz hemen. onCreate metodu nedir derseniz kaynak:<br>
- https://developer.android.com/guide/components/activities/activity-lifecycle
- https://omerates760.medium.com/android-activity-lifecycle-aktive-ya%C5%9Fam-d%C3%B6ng%C3%BCs%C3%BC-nedir-66aae3b905b6 

Şimdi bu fonksiyona baktığımızda c.a, c.b ve c.c methodlarından dönen değere bakılarak root detection işlemi yapılıyor. Bu metodlara bakalım
```java
package sg.vantagepoint.a;

import android.os.Build;
import java.io.File;

public class c {
    public static boolean a() {
        for (String str : System.getenv("PATH").split(":")) {
            if (new File(str, "su").exists()) {
                return true;
            }
        }
        return false;
    }

    public static boolean b() {
        String str = Build.TAGS;
        return str != null && str.contains("test-keys");
    }

    public static boolean c() {
        for (String str : new String[]{"/system/app/Superuser.apk", "/system/xbin/daemonsu", "/system/etc/init.d/99SuperSUDaemon", "/system/bin/.ext/.su", "/system/etc/.has_su_daemon", "/system/etc/.installed_su_daemon", "/dev/com.koushikdutta.superuser.daemon/"}) {
            if (new File(str).exists()) {
                return true;
            }
        }
        return false;
    }
}

```
- a metodunda PATH enviroment variable'ı alınıyor ve içerisinde "su" var mı diye bakılıyor.
- b metodunda ne yapıyor derseniz şurda sormuşlar: https://stackoverflow.com/questions/18808705/android-root-detection-using-build-tags
- c metodunda superuser vs tarzı root yetkilendirme uygulamaları yüklü mü diye dosyalara bakıyor.

Şimdi biz bu fonksiyonları tek tek false döndürerek bu root detection işini bypasslayabiliriz. Bunun kodunu da isterseniz yazıyı okuduktan sonra yazabilirsiniz.

MainActivity'den aldığımız koda bakarsak eğer a b c şartları sağlanmazsa a("...") şeklinde bir method çağırıyor.
```java
//sg.vantagepoint.uncrackable1.MainActivity
private void a(String str) {
        AlertDialog create = new AlertDialog.Builder(this).create();
        create.setTitle(str);
        create.setMessage("This is unacceptable. The app is now going to exit.");
        create.setButton(-3, "OK", new DialogInterface.OnClickListener() {
            /* class sg.vantagepoint.uncrackable1.MainActivity.AnonymousClass1 */

            public void onClick(DialogInterface dialogInterface, int i) {
                System.exit(0);
            }
        });
        create.setCancelable(false);
        create.show();
    }
``` 
Burada verilen string ifade ile bir alert oluşturuluyor(yani ben öyle anladım) ve ardından biz bu uyarıda "ok" dediğimizde System.exit(0) ile uygulama kapatılıyor. Eğer biz bu System.exit'i hooklarsak çıkış yapmasını engelleyebiliriz. Bunun için google amcaya "frida system exit hook" yazarsanız hazır kodu da bulabilirsiniz. Şimdi adım adım yazalım kodu:
- Hooklanacak metod: System.exit() 
- Hooklayacağımız metodun bulunduğu class: java.lang.System https://developer.android.com/reference/java/lang/System#exit(int)

```js
//console.log ile ekrana log bastırabiliyoruz
console.log("Hook islemine basliyoruz!");
Java.perform(function(){
    //hooklayacağımız class için bir wrapper tanımlıyoruz
    var my_system = Java.use("java.lang.System");
    // class.metod.implementation şeklinde fonksiyonu hooklayıp içerisine yapacaklarımızı yazıyoruz
    my_system.exit.implementation = function(x){
        console.log("Kendi exit metodumuz calisti, root detect bypasslandı!");
    }
    // tanımladığımız fucntionda sadece log basıyoruz. Normalde bu fonksiyon çıkış yapacak iken şimdi sadece log a string basacak :)
});
```
Kodu çalıştıralım:
```java
$ frida -U -f owasp.mstg.uncrackable1 -l script.js --no-pause
// -U, --usb                   connect to USB device
// -f FILE, --file=FILE        spawn FILE
// -l SCRIPT, --load=SCRIPT    load SCRIPT
// --no-pause                  automatically start main thread after startup
```

![](/assets/images/root_bypass1.png)

Resimde de görüldüğü üzere uygulamamız açıldı, çıkan uyarıda ok dediğimizde uygulamadan çıkmak yerine log a stringi bastırdı. Şimdi bizden bir secret string bulmamızı istiyor. Koda bakalım:
```java
//sg.vantagepoint.uncrackable1.MainActivity
public void verify(View view) {
        String str;
        String obj = ((EditText) findViewById(R.id.edit_text)).getText().toString();
        AlertDialog create = new AlertDialog.Builder(this).create();
        if (a.a(obj)) {
            create.setTitle("Success!");
            str = "This is the correct secret.";
        } else {
            create.setTitle("Nope...");
            str = "That's not it. Try again.";
        }
        create.setMessage(str);
        create.setButton(-3, "OK", new DialogInterface.OnClickListener() {
            /* class sg.vantagepoint.uncrackable1.MainActivity.AnonymousClass2 */

            public void onClick(DialogInterface dialogInterface, int i) {
                dialogInterface.dismiss();
            }
        });
        create.show();
    }
```
Verify fonksiyonunda bizim input olarak verdiğimiz "obj" isimli stringi a.a(obj) şeklinde bir kontrole veriyor. Eğer bu kontrolden true dönerse başarı mesajını ekrana basıyor. a.a() fonksiyonuna bakalım:
```java
//sg.vantagepoint.uncrackable1.a
public static boolean a(String str) {
        byte[] bArr;
        byte[] bArr2 = new byte[0];
        try {
            bArr = sg.vantagepoint.a.a.a(b("8d127684cbc37c17616d806cf50473cc"), Base64.decode("5UJiFctbmgbDoLXmpL12mkno8HT4Lv8dlat8FxR2GOc=", 0));
        } catch (Exception e) {
            Log.d("CodeCheck", "AES error:" + e.getMessage());
            bArr = bArr2;
        }
        return str.equals(new String(bArr));
    }
```
Bu kodda da "sg.vantagepoint.a.a" classından a fonksiyonunu çağırıyor. Bu fonksiyona verdiği 2 parametrenin ardından fonksiyondan dönen değer ile bizim verdiğimiz inputu (str) karşılaştırıp true veya false dönüyor. bArr byte arrayının oluşturulduğu fonksiyona bakarsak:
```java
//sg.vantagepoint.a.a
public static byte[] a(byte[] bArr, byte[] bArr2) {
        SecretKeySpec secretKeySpec = new SecretKeySpec(bArr, "AES/ECB/PKCS7Padding");
        Cipher instance = Cipher.getInstance("AES");
        instance.init(2, secretKeySpec);
        return instance.doFinal(bArr2);
    }
```
Bu kodda da aldığı iki parametre ile AES decryption yapıyor gibi (kriptodan hiç anlamıyorum :) ). Buradan dönecek olan değer bizim aradığımız secret stringin byteları çünkü bunları bizim input ile karşılaştırıyordu hatırlarsanız. Şimdi bu değeri frida ile okumaya çalışalım:
```js
console.log("Hook islemine basliyoruz!");
Java.perform(function(){
    
    var my_system = Java.use("java.lang.System");

    my_system.exit.implementation = function(x){
        console.log("Kendi exit metodumuz calisti, root detect bypasslandı!");
    }
    // buraya kadar olan kısım zaten root detection bypass 

    //aynı şekilde yine lazım olan class için wrapper tanımlıyoruz
    var my_decrypt = Java.use("sg.vantagepoint.a.a");

    // a fonksiyonu için implementation tanımlıyoruz
    // burada x ve y şeklinde iki adet parametre veriyoruz çünkü orjinalinde de 2 parametre alıyordu
    // fonksiyon tetiklendiğinnde o parametreler x ve y oluyor yani
    my_decrypt.a.implementation = function(x,y){
        //fonksiyonu orijinal imlplementasyonu ile çağırıyoruz ve aradığımız değeri alıyoruz
        var ret = this.a(x,y);
        // bu dönen değerin string değil de byte array olduğunu java kodunda görmüştük
        // https://stackoverflow.com/questions/25965573/printing-the-contents-of-a-javascript-uint8array-as-raw-bytes
        // Byte arrayı ekrana bastıralım
        console.log("Aranan byte array:"+ Array.apply([], ret).join(","));
        //program düzgünce çalışmaya devam etsin diye değeri, return ediyorum
        return ret;
    }
});
```
![](/assets/images/aes_bypass1.png)
<br>
Scripti çalıştırdıktan sonra root uyarısına ok dedik, ardından AES fonksiyonunun tetiklenmesi için yanlış bir input verdik ve byte array geldi.
```
73,32,119,97,110,116,32,116,111,32,98,101,108,105,101,118,101
```
Bunu decimal to text ile ascii değerlerinden texte dönüştürürsek:
- Link: [Cyber Chef Recipe](https://gchq.github.io/CyberChef/#recipe=From_Decimal('Comma',false)&input=NzMsMzIsMTE5LDk3LDExMCwxMTYsMzIsMTE2LDExMSwzMiw5OCwxMDEsMTA4LDEwNSwxMDEsMTE4LDEwMQ)
```
I want to believe
```
stringini elde ediyoruz. Test ettiğimizde doğru olduğunu anlıyoruz :) Ha ben elimle bunları çevirmekle uğraşmak istemem frida scriptinin içinde çevirsek olmaz mı derseniz o da şöyle bir for döngüsü ile olabilir tabii:

```js
console.log("Hook islemine basliyoruz!");
Java.perform(function(){
    var my_system = Java.use("java.lang.System");
    my_system.exit.implementation = function(x){
        console.log("Kendi exit metodumuz calisti, root detect bypasslandı!");
    }
    
    var my_decrypt = Java.use("sg.vantagepoint.a.a");

    my_decrypt.a.implementation = function(x,y){
        
        var ret = this.a(x,y);
        var i = 0;
        var sonuc = "";
        for(i=0; i<ret.length; i++){
            sonuc += String.fromCharCode(ret[i]);
        }
        console.log("Sonuc:" + sonuc);
        return ret;
    }
});
```
<br>
```bash
[sh4d0wless@paradise uncrackable-lvl1]$ frida -U -f owasp.mstg.uncrackable1 -l script.js --no-pause
     ____
    / _  |   Frida 14.2.8 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://www.frida.re/docs/home/
Spawning `owasp.mstg.uncrackable1`...                                   
Hook islemine basliyoruz!
Spawned `owasp.mstg.uncrackable1`. Resuming main thread!                
[Google Nexus 6::owasp.mstg.uncrackable1]-> Kendi exit metodumuz calisti, root detect bypasslandı!
Sonuc:I want to believe

```
Bu şekilde sorumuzu çözmüş olduk. En temel şekilde frida kullanımı bu şekilde diyebiliriz. Frida baya işlevsel ve kapsamlı bir tool olduğundan dolayı ileride öğrendikçe daha farklı konularda yazı yazmaya çalışacağım. Bu konularda çok güzel içerikleri olan birkaç blog linkini de buraya bırakayım:
- https://alp.run/
- https://eybisi.run
- https://11x256.github.io/