## コード品質とビルド設定のテスト

### アプリが正しく署名されていることの検証

#### 概要

Android requires that all APKs be digitally signed with a certificate before they can be installed. The digital signature is required by the Android system before installing/running an application, and it's also used to verify the identity of the owner for future updates of the application. This process can prevents an app from being tampered with, or modified to include malicious code.

When an APK is signed, a public-key certificate is attached to the APK. This certificate uniquely associates the APK to the developer and their corresponding private key. When building an app in debug mode, the Android SDK signs the app with a debug key specifically created for debugging purposes. An app signed with a debug key is not be meant for distribution and won't be accepted in most app stores, including the Google Play Store. To prepare the app for final release, the app must be signed with a release key belonging to the developer.

When testing the final release build of an app, you should verify that the APK has been signed with a valid certificate. Note that Android expects any updates to the app to be signed with the same certificate, so a validity period of 25 years or more is recommended. Apps published on Google Play must be signed with a certificate that is valid at least until October 22th, 2033.

#### 静的解析

Unsigned APK:

```
jarsigner -verify -verbose -certs Unsigned.apk

  s = signature was verified 
  m = entry is listed in manifest
  k = at least one certificate was found in keystore
  i = at least one certificate was found in identity scope

no manifest.
jar is unsigned. (signatures missing or not parsable)
```

Signed with debug certificate:

```
$ jarsigner -verify -verbose -certs example-debug.apk 

sm     11116 Fri Nov 11 12:07:48 ICT 2016 AndroidManifest.xml

      X.509, CN=Android Debug, O=Android, C=US
      [certificate is valid from 3/24/16 9:18 AM to 8/10/43 9:18 AM]
      [CertPath not validated: Path does not chain with any of the trust anchors]

```

Signed with release certificate:

```
$ jarsigner -verify -verbose -certs example-release.apk 

sm     11116 Fri Nov 11 12:07:48 ICT 2016 AndroidManifest.xml

      X.509, CN=Awesome Corporation, OU=Awesome, O=Awesome Mobile, L=Palo Alto, ST=CA, C=US
      [certificate is valid from 9/1/09 4:52 AM to 9/26/50 4:52 AM]
      [CertPath not validated: Path does not chain with any of the trust anchors]

```

#### 動的解析

Static analysis should be used to verify the APK signature. If you don't have the APK available locally, pull it from the device first:

```bash
$ adb shell pm list packages
(...)
package:com.awesomeproject
(...)
$ adb shell pm path com.awesomeproject
package:/data/app/com.awesomeproject-1/base.apk
$ adb pull /data/app/com.awesomeproject-1/base.apk
```

#### 改善方法

Developers need to make sure that release builds are signed with the appropriate certificate from the release keystore. In Android Studio, this can be done manually or by configuring creating a signing configuration and assigning it to the release build type [2].

#### 参考情報

##### OWASP Mobile Top 10 2014

N/A

##### OWASP MASVS

- V7.1: "アプリは有効な証明書で署名およびプロビジョニングされている。"

##### CWE

N/A

##### その他

- [1] Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
- [2] Sign your App - https://developer.android.com/studio/publish/app-signing.html

##### ツール

- jarsigner - http://docs.oracle.com/javase/7/docs/technotes/tools/windows/jarsigner.html

### アプリがデバッグ可能であるかのテスト

#### 概要

The <code>android:debuggable</code> attiribute in the <code>Application</code> tag in the Manifest determines whether or not the app can be debugged when running on a user mode build of Android. In a release build, this attribute should always be set to "false" (the default value).

#### 静的解析

Check in <code>AndroidManifest.xml</code> whether the <code>android:debuggable</code> attribute is set:

```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.android.owasp">

    ...

    <application android:allowBackup="true" android:debuggable="true" android:icon="@drawable/ic_launcher" android:label="@string/app_name" android:theme="@style/AppTheme">
        <meta-data android:name="com.owasp.main" android:value=".Hook"/>
    </application>
</manifest>
```

#### 動的解析

Set the <code>android:debuggable</code> to false, or simply leave omit it from the <code>Application</code> tag.

#### 改善方法

For production releases, the attribute android:debuggable must be set to false within the application element. This ensures that a debugger cannot attach to the process of the application.

#### 参考情報

##### OWASP Mobile Top 10 2014

* MX - Title - Link
* M3 - Insufficient Transport Layer Protection - https://www.owasp.org/index.php/Mobile_Top_10_2014-M3

##### OWASP MASVS

-- TODO [Update reference "VX.Y" below for "Testing If the App is Debuggable"] --
- VX.Y: ""

##### CWE

-- TODO [Add relevant CWE for "Testing If the App is Debuggable"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### その他

* [1] Application element - https://developer.android.com/guide/topics/manifest/application-element.html


### デバッグシンボルに関するテスト

#### 概要

-- TODO [Give an overview about the functionality and it's potential weaknesses] --

For native binaries, use a standard tool like nm or objdump to inspect the symbol table. A release build should generally not contain any debugging symbols. If the goal is to obfuscate the library, removing unneeded dynamic symbols is also recommended.

#### 静的解析

Symbols  are usually stripped during the build process, so you need the compiled bytecode and libraries to verify whether the any unnecessary metadata has been discarded. 

To display debug symbols:

```bash
export $NM = $ANDROID_NDK_DIR/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-nm
```

```bash
$ $NM -a libfoo.so 
/tmp/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi-nm: libfoo.so: no symbols
```
To display dynamic symbols:

```bash
$ $NM -D libfoo.so 
```

Alternatively, open the file in your favorite disassembler and check the symbol tables manually. 

#### 動的解析

#### 改善方法

Dynamic symbols can be stripped using the <code>visibility</code> compiler flag. Adding this flag causes gcc to discard the function names while still preserving the names of functions declared as <code>JNIEXPORT</code>.

Add the following to build.gradle:

```
        externalNativeBuild {
            cmake {
                cppFlags "-fvisibility=hidden"
            }
        }
```

#### 参考情報

##### OWASP Mobile Top 10 2014

* MX - Title - Link
* M3 - Insufficient Transport Layer Protection - https://www.owasp.org/index.php/Mobile_Top_10_2014-M3

##### OWASP MASVS

-- TODO [Update reference "VX.Y" below for "Testing for Debugging Symbols"] --
- VX.Y: ""

##### CWE

-- TODO [Add relevant CWE for "Testing for Debugging Symbols"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### その他

* Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
* Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### ツール

-- TODO [Add relevant tools for "Testing for Debugging Symbols"] --
* Enjarify - https://github.com/google/enjarify


### デバッグコードや詳細エラーログに関するテスト

#### 概要

-- TODO [Give an overview about the functionality and it's potential weaknesses] --

#### ホワイトボックステスト

-- TODO [Add content on white-box testing for "Testing for Debugging Code and Verbose Error Logging"] --

#### ブラックボックステスト

-- TODO [Add content on black-box testing for "Testing for Debugging Code and Verbose Error Logging"] --

#### 改善方法

-- TODO [Describe the best practices that developers should follow to prevent this issue "Testing for Debugging Code and Verbose Error Logging"] --

#### 参考情報

##### OWASP Mobile Top 10 2014

* MX - Title - Link
* M3 - Insufficient Transport Layer Protection - https://www.owasp.org/index.php/Mobile_Top_10_2014-M3

##### OWASP MASVS

-- TODO [Update reference "VX.Y" below for "Testing for Debugging Code and Verbose Error Logging"] --
- VX.Y: ""

##### CWE

-- TODO [Add relevant CWE for "Testing for Debugging Code and Verbose Error Logging"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### その他

* Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
* Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### ツール

-- TODO [Add relevant tools for "Testing for Debugging Code and Verbose Error Logging"] --
* Enjarify - https://github.com/google/enjarify


### 例外処理のテスト

#### 概要

-- TODO [Give an overview about the functionality and it's potential weaknesses] --

#### ホワイトボックステスト

Review the source code to understand/identify who the application handle various types of errors (IPC communications, remote services invokation, etc). Here are some examples of the checks to be performed at this stage :

* Verify that the application use a [well-designed] (https://www.securecoding.cert.org/confluence/pages/viewpage.action?pageId=18581047) (an unified) scheme to handle exceptions.
* Verify that the application doesn't expose sensitive information while handeling exceptions, but are still verbose enough to explain the issue to the user.
* C3

#### ブラックボックステスト

-- TODO [Describe how to test for this issue using static and dynamic analysis techniques. This can include everything from simply monitoring aspects of the app’s behavior to code injection, debugging, instrumentation, etc. ] --

#### 改善方法

-- TODO [Describe the best practices that developers should follow to prevent this issue "Testing Exception Handling"] --

#### 参考情報

##### OWASP Mobile Top 10 2014

* MX - Title - Link
* M3 - Insufficient Transport Layer Protection - https://www.owasp.org/index.php/Mobile_Top_10_2014-M3

##### OWASP MASVS

-- TODO [Update reference "VX.Y" below for "Testing Exception Handling"] --
- VX.Y: ""

##### CWE

-- TODO [Add relevant CWE for "Testing Exception Handling"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### その他

* Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
* Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### ツール

-- TODO [Add relevant tools for "Testing Exception Handling"] --
* Enjarify - https://github.com/google/enjarify


### コンパイラ設定の検証

#### 概要

Since most Android applications are Java based, they are [immunue](https://www.owasp.org/index.php/Reviewing_Code_for_Buffer_Overruns_and_Overflows#.NET_.26_Java) to buffer overflow vulnerabilities.

#### ホワイトボックステスト

-- TODO [Describe how to assess this with access to the source code and build configuration] --

#### ブラックボックステスト

-- TODO [Describe how to test for this issue using static and dynamic analysis techniques. This can include everything from simply monitoring aspects of the app’s behavior to code injection, debugging, instrumentation, etc. ] --

#### 改善方法

-- TODO [Describe the best practices that developers should follow to prevent this issue "Verifying Compiler Settings"] --

#### 参考情報

##### OWASP Mobile Top 10 2014

* MX - Title - Link
* M3 - Insufficient Transport Layer Protection - https://www.owasp.org/index.php/Mobile_Top_10_2014-M3

##### OWASP MASVS

-- TODO [Update reference "VX.Y" below for "Verifying Compiler Settings"] --
- VX.Y: ""

##### CWE

-- TODO [Add relevant CWE for "Verifying Compiler Settings"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### Info

* Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
* Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### Tools

-- TODO [Add relevant tools for "Verifying Compiler Settings"] --
* Enjarify - https://github.com/google/enjarify


### Testing for Memory Management Bugs

#### 概要

-- TODO [Give an overview about the functionality and it's potential weaknesses] --

#### ホワイトボックステスト

-- TODO [Add content for white-box testing "Testing for Memory Management Bugs"] --

#### ブラックボックステスト

-- TODO [Add content for black-box testing "Testing for Memory Management Bugs"] --

#### 改善方法

-- TODO [Add remediations for "Testing for Memory Management Bugs"] --

#### 参考情報

##### OWASP Mobile Top 10 2014

* MX - Title - Link
* M3 - Insufficient Transport Layer Protection - https://www.owasp.org/index.php/Mobile_Top_10_2014-M3

##### OWASP MASVS

- V7.7: "アンマネージドコードでは、メモリは安全に割り当て、解放、使用されている。"

##### CWE

-- TODO [Add relevant CWE for "Testing for Memory Management Bugs"] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### その他

* Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
* Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### ツール

-- TODO [Add relevant tools for "Testing for Memory Management Bugs"] --
* Enjarify - https://github.com/google/enjarify


### JavaバイトコードがMinifyされていることの検証

#### 概要

Because Java classes are trivial to decompile, applying some basic obfuscation to the release bytecode is recommended. For Java apps on Android, ProGuard offers an easy way to shrink and obfuscate code. It replaces identifiers such as  class names, method names and variable names with meaningless character combinations. This is a form of layout obfuscation, which is “free” in that it doesn't impact the performance of the program.

#### ホワイトボックステスト

If source code is provided, build.gradle file can be check to see if obfuscation settings are set. From the example below, we can see that minifyEnabled and proguardFiles are set. It is common to see application exempts some class from obfuscation with "-keepclassmembers" and "-keep class", so it is important to audit proguard configuration file to see what class are exempted. The getDefaultProguardFile('proguard-android.txt') method gets the default ProGuard settings from the Android SDK tools/proguard/ folder and proguard-rules.pro is where you defined custom proguard rules. From our sample proguard-rules.pro file, we can see that many classes that extend common android classes are exempted, which should be done more granular on exempting specific classes or library.

build.gradle
```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```

proguard-rules.pro
```
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Application
-keep public class * extends android.app.Service
```

#### ブラックボックステスト

If source code is not provided, apk can be decompile to verify if codebase have been obfuscated. dex2jar can be used to convert dex code to jar file. Tools like JD-GUI can be used to check if class, method and variable name is human readable.

Sample obfuscated code block
```
package com.a.a.a;

import com.a.a.b.a;
import java.util.List;

class a$b
  extends a
{
  public a$b(List paramList)
  {
    super(paramList);
  }

  public boolean areAllItemsEnabled()
  {
    return true;
  }

  public boolean isEnabled(int paramInt)
  {
    return true;
  }
}
```

#### 改善方法

ProGuard should be used to strip unneeded debugging information from the Java bytecode. By default, ProGuard removes attributes that are useful for debugging, including line numbers, source file names and variable names. ProGuard is a free Java class file shrinker, optimizer, obfuscator, and preverifier. It is shipped with Android’s SDK tools. To activate shrinking for the release build, add the following to build.gradle:

~~~~
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile(‘proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
~~~~

#### 参考情報

##### OWASP Mobile Top 10 2014

* MX - Title - Link
* M3 - Insufficient Transport Layer Protection - https://www.owasp.org/index.php/Mobile_Top_10_2014-M3

##### OWASP MASVS

-- TODO [Update reference below "VX.Y" for "Verifying that Java Bytecode Has Been Minified"] --
- VX.Y: ""

##### CWE

-- TODO [Add relevant CWE for Verifying that Java Bytecode Has Been Minified] --
- CWE-312 - Cleartext Storage of Sensitive Information

##### その他

* Configuring your application for release - http://developer.android.com/tools/publishing/preparing.html#publishing-configure
* Debugging with Android Studio - http://developer.android.com/tools/debugging/debugging-studio.html

##### ツール

-- TODO [Add relevant tools for Verifying that Java Bytecode Has Been Minified] --
* Enjarify - https://github.com/google/enjarify

