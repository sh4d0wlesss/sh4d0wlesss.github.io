---
title:  "Frida ile Android Native Library Function Hooklamak - Frida 101"
date:   2021-02-26T00:48:00-0400
categories:
  - android
tags:
  - frida
  - android
---

Merhaba<br>
Bu yazıda frida ile Android uygulamalarda native kütüphane dosyalarından fonksiyon hooklama konusunu bir örnek üzerinden anlatmaya çalışacağım. Önceki yazıda direkt olarak java kodundan fonksiyon hooklamıştık, bu sefer de uygulamanın kullandığı native lib dosyasından fonksiyon "kancalayacağız". Kurban uygulamamız yine owasp'ın uncrackable serisinden UnCrackable 2.
> Uygulama: [Uncrackable2](https://github.com/OWASP/owasp-mstg/raw/master/Crackmes/Android/Level_02/UnCrackable-Level2.apk)

Apk dosyasını genymotiona sürükleyip bırakıyoruz ve uygulama yükleniyo. Uygulama başladığında öncekinde olduğu gibi root detection uyarısı verip kapanıyor. Bunu önceki yazıda yaptığım kodu verip geçiyorum isteyen bir önceki yazıyı da okuyabilir.
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
<br>

```java
$ frida -U -f owasp.mstg.uncrackable2 -l script.js --no-pause
// -U, --usb                   connect to USB device
// -f FILE, --file=FILE        spawn FILE
// -l SCRIPT, --load=SCRIPT    load SCRIPT
// --no-pause                  automatically start main thread after startup
```
Bu frida scripti ile root detectionu atlatıyoruz. Peki sonrasında ne var koda bakalım.
```java
public void verify(View view) {
        String str;
        String obj = ((EditText) findViewById(R.id.edit_text)).getText().toString();
        AlertDialog create = new AlertDialog.Builder(this).create();
        if (this.m.a(obj)) {
            create.setTitle("Success!");
            str = "This is the correct secret.";
        } else {
            create.setTitle("Nope...");
            str = "That's not it. Try again.";
        }
        create.setMessage(str);
        create.setButton(-3, "OK", new DialogInterface.OnClickListener() {
            /* class sg.vantagepoint.uncrackable2.MainActivity.AnonymousClass3 */

            public void onClick(DialogInterface dialogInterface, int i) {
                dialogInterface.dismiss();
            }
        });
        create.show();
    }
```
Main activity içerisinde bulunan verify metodu ile bizim verdiğimiz string kontrol ediliyor ve doğru ise ekrana bir kutucuk açıyor. Bu kontrolü yaparken dikkat ettiyseniz `this.m.a(obj)` metodunun True dönmesi lazım. Bu fonksiyona bakalım:
```java
//sg.vantagepoint.uncrackable2.CodeCheck

public class CodeCheck {
    private native boolean bar(byte[] bArr);

    public boolean a(String str) {
        return bar(str.getBytes());
    }
}
```
Kodda görüldüğü üzere "a" fonksiyonuna gelen string "bar" fonksiyonuna byte array olarak gönderiliyor ve oradan gelecek return değeri return ediliyor. Bu "bar" fonksiyonu ise native library dosyasından geliyor. Main activity içerisine bakarsak:
```java
static {
        System.loadLibrary("foo");
    }

...
...

private native void init();
...
...
public void onCreate(Bundle bundle) {
        init();
        ...
        ...

```
şeklinde kütüphane dosyası yüklenmiş ve ardından initialize edilmiş. Ayrıntılı bilgi için: [Link](https://developer.android.com/training/articles/perf-jni)

Load lib yaparken "foo" yazıyordu. Linke baktıysanız dosya adının libfoo.so olacağını anlamışsınızdır. Uygulamayı apktool ile açıp içerisinden libfoo.so dosyasını alalım ve ghidra ile açalım.(Ben genymotion kullandığım için x86 versiyonunu aldım.)
Fonksiyonların isimlendirmesinden de anlaşıldığı üzere bizim fonksiyonumuz "Java_sg_vantagepoint_uncrackable2_CodeCheck_bar" isimli fonksiyon. Decompiled hali, şu şekilde(ben biraz isimlendirme yaptım sadece):
```c++
undefined4
Java_sg_vantagepoint_uncrackable2_CodeCheck_bar(int *param_1,undefined4 param_2,undefined4 param_3)

{
  char *__s1;
  int len;
  undefined4 uVar1;
  int in_GS_OFFSET;
  undefined4 local_30;
  undefined4 local_2c;
  undefined4 local_28;
  undefined4 local_24;
  undefined2 local_20;
  undefined4 local_1e;
  undefined2 local_1a;
  int local_18;
  
  local_18 = *(int *)(in_GS_OFFSET + 0x14);
  if (DAT_00014008 == '\x01') {
    local_30 = 0x6e616854;
    local_2c = 0x6620736b;
    local_28 = 0x6120726f;
    local_24 = 0x74206c6c;
    local_20 = 0x6568;
    local_1e = 0x73696620;
    local_1a = 0x68;
    __s1 = (char *)(**(code **)(*param_1 + 0x2e0))(param_1,param_3,0);
    len = (**(code **)(*param_1 + 0x2ac))(param_1,param_3);
    if (len == 0x17) {
      len = strncmp(__s1,(char *)&local_30,0x17);
      if (len == 0) {
        uVar1 = 1;
        goto LAB_00011009;
      }
    }
  }
  uVar1 = 0;
LAB_00011009:
  if (*(int *)(in_GS_OFFSET + 0x14) == local_18) {
    return uVar1;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}
```
Local 18 stack canary için tanımlanıyor. Ardından if blogu içinde hex şekilde bir string tanımlaması var.Önce verdiğimiz inputun uzunluğu 0x17(23) mi diye bakılıyor ve ardından "strncmp" fonksiyonu ile beklenen string ile karşılaştırma yapılıyor. Eğer bu iki string aynı ise strncmp fonksiyonu 0 dönecek ve uVar1 değeri 1 olacak. En sonda da canary check yaptıktan sonra 1 döndürecek ve bizim java tarafındaki fonksiyona 1 dönmüş olacak ve başarılı olacağız.<br>
Burada zaten hex değerler kabak gibi ortada olduğu için flagi bulduk ancak eğer orada tek bir kontrol yerine karman çorman şeyler olsa zorlanacaktık. Bu yüzden bu işi frida ile yapacağız.<br>
Frida'da [interceptor](https://frida.re/docs/javascript-api/#interceptor) diye güzel bir özellik var.
```
Interceptor.attach(target, callbacks[, data]): intercept calls to function at target. This is a NativePointer specifying the address of the function you would like to intercept calls to. 
```
Ayrıntılarını dokümentasyonda güzelce anlatmışlar ve örnek kod da var. Bu örnek koddan faydalanarak biz de hook eyleyelim. Benim kod şu şekilde:
```js
Interceptor.attach(Module.getExportByName('libc.so', 'strncmp'), {
    onEnter(args) {
        var param1 = Memory.readUtf8String(args[0],23);
        var param2 = Memory.readUtf8String(args[1]);
        
        
		if(param1 == "11122233344455566677788"){
			console.log("Flag : " + param2);
		}
    },
    onLeave(retval) {
      
    }
```
İlk satırda müdahale etmek istediğimiz kütüphane ve fonksiyon adını veriyoruz. Ardından bu fonksiyonun ilk argümanı olan inputumuzu alıyoruz. Ardından 2. parametre olan hedef stringimizi alıyoruz. 3. parametre de uzunluk değeri ama ben onu kullanmadım. İf blogunda da verdiğimiz string ile param1 eşleşirse flagi konsola basıyor. Eğer ilk denemenizde çalışmaz ise fridayı kapatmadan tekrar tekrar verify butonuna basmayı deneyin çünkü bazen ilk denemede olmuyor ya da arkada çok fazla strncmp call yapıldığı için ve bunların hepsini if ile kontrolden geçirdiği için geç düşüyor olabilir o konudan tam emin değilim. Benim denememde flagi konsola basana kadar 64518 kere strncmp çağrısı yapılmıştı, count değişkeni koyarak siz de deneyebilirsiniz. Kodumuzun son hali de şu şekilde:
```js
  console.log("Hook islemine basliyoruz!");
  Java.perform(function(){ 
      var my_system = Java.use("java.lang.System");
      my_system.exit.implementation = function(x){
          console.log("Kendi exit metodumuz calisti, root detect bypasslandı!");
      }
  });

  var count = 0;
  Interceptor.attach(Module.getExportByName('libc.so', 'strncmp'), {
    onEnter(args) {
    var param1 = Memory.readUtf8String(args[0]);
    var param2 = Memory.readUtf8String(args[1]);    
    count += 1;
    
    if(param1 == "11122233344455566677788"){
            console.log("Flag : " + param2);
            console.log(count);
		}
    },
    onLeave(retval) {
      
    }
  });
// teşekkürler 0xabc 
```
<br>

```sh
[sh4d0wless@paradise uncrackable-lvl2]$ frida -U -f owasp.mstg.uncrackable2 -l script.js --no-pause
     ____
    / _  |   Frida 14.2.8 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://www.frida.re/docs/home/
Spawning `owasp.mstg.uncrackable2`...                                   
Hook islemine basliyoruz!
Spawned `owasp.mstg.uncrackable2`. Resuming main thread!                
[Google Nexus 6::owasp.mstg.uncrackable2]-> Kendi exit metodumuz calisti, root detect bypasslandı!
Flag : Thanks for all the fish
67658

```
Bir başka çözüm yolu olarak Enovella'nın blogunda yazdığı çözümden faydalanabilirsiniz. Frida'da bulunan backtrace özelliği ile yapılan çağrının nerelerden gelinerek yapıldığına bakılabiliyor ve burada if bloğunda string kontrolü yerine backtrace ile oluşan array içerisinde libfoo.so var mı diye bakarsak oradan gelip gelmediğimizi anlayıp ona göre parametreyi konsola basabiliriz. İlgili blog paylaşımı: [Link](https://enovella.github.io/android/reverse/2017/05/20/android-owasp-crackmes-level-2.html) <br>
Ancak bu yöntemi denediğimde emulatörde uygulama beyaz ekranda kalıyordu. Çok fazla strncmp çağrısından dolayı ağır bir işlem gerçekleşiyor ve pek de performanslı olmuyor ancak ileride daha hızlı hooklanabilecek fonksiyonlarda deneme yapmaya çalışacağım :D
<br>
Evet bu yazı bu kadardı, ileride daha komplike işlemleri öğrenip yazmaya çalışacağım. Umarım birilerine faydası oluyordur. Görüşmek üzere :)
