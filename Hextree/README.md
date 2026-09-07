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
