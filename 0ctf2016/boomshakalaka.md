# boomshakalaka (plane)
> play the game, get the highest score  
boomshakalaka  

(*mobile*)

This was an Android reverse engineering challenge. We're given an `apk`, *plane.apk*. First thing to do is check out the apk by 
launching an emulator, or using your phone. I do some Android development right now, so I already had a couple of Android VMs 
ready made in [Android Studio](http://developer.android.com/sdk/index.html). 

```
$ adb devices
List of devices attached
emulator-5556	device
$ adb install plane.apk # Note: The emulator must have an ARM ABI, not x86 for this apk
```

![](/blog/content/images/2016/03/plane.png)
Turns out it's a pretty fun game! Nothing really sticks out from a security and puzzle perspective.
Well, let's take a look into the source code. We're given an `apk` file, which is just a fancy Android `jar`, and a `jar` file is 
just a fancy Java `zip` file, so all the contents that form the app are contained inside the `apk` and just need to be unpacked. 
We can either unpack the `dex` files (fancy Android .class files), or `smali` assembly. I did the latter first, by using the tool, 
[apktool](http://ibotpeaches.github.io/Apktool/).

```
$ apktool d plane.apk
$ ls
AndroidManifest.xml  apktool.yml  assets  lib  original  res  smali
$ ls smali/com/example/plane/
a.smali  BuildConfig.smali  FirstTest.smali  R$attr.smali  R$drawable.smali  R.smali  R$string.smali
```

We have every file inside of the `apk`, you can browse around the directories and figure out how the application works. The 
interesting parts are in [a.smali](https://gist.github.com/eugenekolo/f125dee08320b9b5d43c#file-a-smali), and 
[FirstTest.smali](https://gist.github.com/eugenekolo/f125dee08320b9b5d43c#file-firsttest-smali). The first thing you'll notice is 
the base64 string: `const-string v0, "YmF6aW5nYWFhYQ=="`, decoded to: "bazingaaaa" - funny. Without diving into how to read Java 
bytecode, it looks like the programs don't really do too much interesting except set some shared preferences (persistent global 
variables), put some weird strings (`DATA`, `N0`, and `MG`) into them, and call the init functions. There's no actual game logic 
anywhere though - it must be implemented elsewhere. Looking back into the directories that `apktool` told us, we can find 
`lib/armeabi/libcocos2dcpp.so`. So the actual logic and everything for the game must be written in "native code" inside of the 
`libcocos2dcpp.so`. We have to take a peak at the code inside of that compiled native binary... but before that, let's backtrack 
and make sure we understand the Java side of the code.

I know of a couple more tool for decompiling and reversing Android programs, [dex2jar](https://github.com/pxb1988/dex2jar) and 
[jd-gui](http://jd.benow.ca/). All these great tools used are included in my [sec-tools 
collection](https://github.com/eugenekolo/sec-tools).

```
$ dex2jar plane.apk
dex2jar plane.apk -> ./plane-dex2jar.jar
$ jdgui plane-dex2jar.jar
```

Produces us these decompiled Java files: [a.java](https://gist.github.com/eugenekolo/f125dee08320b9b5d43c#file-a-java) and 
[FirstTest.java](https://gist.github.com/eugenekolo/f125dee08320b9b5d43c#file-firsttest-java). Pretty much what we understood in 
the earlier `smali` files, but much clearer. This is a good technique to keep in mind when reversing future Android apps.

Back to the `libcocos2dcpp.so`, let's look for those strings we saw on the Java side of the app, specifically `DATA` (that's a 
juicy one).

![IDA Strings](/blog/content/images/2016/03/idaboomshaka.png)

Interesting other strings near it.

![IDA Cross References](/blog/content/images/2016/03/idaboom-shaka.png)

`setStringForKey` from `cocos2d` seems to use this string in some sort of way. Looking up what the function does leads me to 
[cocos2d reference manual](http://www.cocos2d-x.org/reference/native-cpp/V3.0alpha0/db/d94/classcocos2d_1_1_user_default.html). It 
looks like the function sets a key to be equal to a string. This sounds like persistent memory storage, *for things like the game 
score!*

Remember the description of this challenge... our goal is to get the highest score. Remembering my teenage years, I used to cheat 
in some games that stored things like the game score in a simple text file. I wonder if we can do the same thing here, and just 
change some number stored in a text file, get the highest score, and get our flag.

Let's do that.

```shell
$ adb shell
$ cd /data/data/com.example.plane/shared_prefs
$ cat Cocos2dxPrefsFile.xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="HighestScore" value="14000" />
    <string 
name="DATA">MGN0ZntDMGNvUzJkX0FuRHJvdz99ZntDMGNvUzJkX0FuRHJvMWRzdz99ZntDMGNvUzJkX0FuRHJvRfdz99ZntDMGNvUzJkX0FuRHJvMWRzdz99ZntDMGNvUzJkX0FuRHJvMWdz99ZntDMGNvUzJkX0FuRHJvRfRz</string>
    <boolean name="isHaveSaveFileXml" value="true" />
</map>
$ echo 
'MGN0ZntDMGNvUzJkX0FuRHJvdz99ZntDMGNvUzJkX0FuRHJvMWRzdz99ZntDMGNvUzJkX0FuRHJvRfdz99ZntDMGNvUzJkX0FuRHJvMWRzdz99ZntDMGNvUzJkX0FuRHJvMWdz99ZntDMGNvUzJkX0FuRHJvRfRz' 
| base64 --decode
0ctf{C0coS2d_AnDrow?}f{C0coS2d_AnDro1dsw?}f{C0coS2d_AnDroE�s��g�3
6�3&E�
�G&�
G7s��g�3
6�3&E�
�G&�
w?}f{C0coS2d_AnDroE�s
```

Trying out, `0ctf{C0coS2d_AnDrow?}` won't do us any good, it's not the flag. I tried a bunch of combinations such as 
`0ctf{0ctf{C0coS2d_AnDro1d?}` as well, it's none of that. Okay. Let's think. We seem to have a repeating pattern in the base64. 
Some examination of the base64 compared to the ASCII output reveals that `MGN0` is actually '0ct', and `ZntD` is 'f{C', and `dz99` 
is 'w?}', and some stuff inbetween. It looks like we have to use the pieces of data we saw in the `.so` binary to form some sort 
of base64 that'll decode into the flag. Editing the `HighestScore` value to `1000000` and playing the game, and then checking back 
the `xml` file doesn't seem to do anything. 

If you look for all the tiny strings near the ones we found around `DATA` that are "base64-esque", you'll get this list: `Jv Uz X0 
Zn tD dz99 MG Nv Fu Jk RH 4w S2 Vf b1 9z RV Bt Rz Rf MW`. We have to probably use them in the right order, we already know the 
placement of most of those. We really only need to basically place ~8 strings in the correct order that'll decode correctly into 
ASCII. You can bruteforce, or you can examine the binary .so some more.

The characters seem to get added to the string, `DATA`, inside of the `Cocos2dxPrefsFile.xml` when the game is initialized, when 
the game layers are loaded, when certain high scores are achieved, and when the game is ending. By following the logical 
progression that would happen, e.g. you init first, load layers, get high score in linear order, and end the game, you'll form the 
correct base64 string.

```
echo 'MGN0ZntDMGNvUzJkX0FuRHJvMWRfRzBtRV9Zb1VfS24wdz99' | base64 --decode
0ctf{C0coS2d_AnDro1d_G0mE_YoU_Kn0w?}
```
