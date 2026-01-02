# kivy-java-alarmmanager-bridge

A working example of a Kivy (Python) Android app packaged with Buildozer that embeds Java code to schedule Android AlarmManager tasks which execute even when the app is not running.

# Overview

This project demonstrates how to reliably combine Kivy (Python) and native Java inside a single Android APK built with Buildozer, allowing Python code to schedule Android AlarmManager events that trigger even after the app is closed or killed.

The solution bridges Python UI logic with Java background execution using PyJNIus, a properly compiled Java BroadcastReceiver, and a Buildozer patch hook that injects the receiver into the Android manifest.

# Project Structure (Critical)

Java sources must be placed under a valid src/ hierarchy that matches the Java package name, otherwise Buildozer will not compile them.
<pre>
project_root/
│
├── buildozer.spec          # Buildozer configuration
├── main.py                # Kivy + PyJNIus Python logic
├── src/                   # Java source root (required)
│   └── org/
│       └── redstoon/
│           └── pushfcmdemo/
│               └── MyReceiver.java   # BroadcastReceiver
└── p4a/
    └── hook.py             # Buildozer patcher (manifest injection)
</pre>

# Important notes:

src/ is treated as a Java source directory by Buildozer

The directory tree must match the Java package declaration

Any mismatch results in Java silently not being compiled

# Buildozer Configuration

Minimal additions are required in buildozer.spec:
<pre>
- Include Java sources
android.add_src = src

- Register patcher hook
p4a.hook = ./p4a/hook.py

# Permissions required for notifications
android.permissions = POST_NOTIFICATIONS

# Python dependencies
requirements = python3,kivy,pyjnius
</pre>

- This configuration ensures:

Java sources are compiled into the APK

The patcher runs automatically during the build

Required permissions are granted

Python can access Java APIs via PyJNIus

Java BroadcastReceiver (Alarm Entry Point)

The Java side is responsible for executing code when the alarm fires.

#MyReceiver.java
<pre>
package org.redstoon.pushfcmdemo;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;
import android.app.NotificationChannel;
import android.app.NotificationManager;
import android.os.Build;
import androidx.core.app.NotificationCompat;

public class MyReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        String msg = intent.getStringExtra("message");
        Log.d("MyReceiver", "Broadcast received: " + msg);

        NotificationManager nm =
            (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
        String channelId = "alarm_channel";

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel =
                new NotificationChannel(channelId, "Alarm Channel",
                                        NotificationManager.IMPORTANCE_DEFAULT);
            nm.createNotificationChannel(channel);
        }

        NotificationCompat.Builder builder =
            new NotificationCompat.Builder(context, channelId)
                .setSmallIcon(android.R.drawable.ic_dialog_info)
                .setContentTitle("Alarm Triggered")
                .setContentText(msg)
                .setPriority(NotificationCompat.PRIORITY_DEFAULT);

        nm.notify((int) System.currentTimeMillis(), builder.build());
    }
}
</pre>

What this does:

Acts as the Android entry point when the alarm fires

Runs even if the app process is not alive

Displays a notification using data sent from Python

Python Side (Scheduling the Alarm)

On the Python side, PyJNIus is used to interact with Android APIs.

Key steps:

Obtain the current Android activity context

Create an Intent targeting MyReceiver

Wrap it in a PendingIntent

Schedule it with AlarmManager

Example logic (simplified):
<pre>
    
def schedule_alarm():
    context = PythonActivity.mActivity
    alarm = context.getSystemService(Context.ALARM_SERVICE)

    intent = Intent(context, autoclass('org.redstoon.pushfcmdemo.MyReceiver'))
    intent.setAction("com.redstoon.ALARM_ACTION")
    intent.putExtra("message", "Hello from Python!")

    pending = PendingIntent.getBroadcast(
        context, 0, intent, PendingIntent.FLAG_IMMUTABLE
    )

    trigger_time = int((time() + 10) * 1000)  # 10 seconds later
    alarm.setExact(AlarmManager.RTC_WAKEUP, trigger_time, pending)

</pre>


Once scheduled, the Android system handles execution — Python does not need to be running.

Manifest Injection (Patcher Hook)

By default, Buildozer does not register custom Java components in AndroidManifest.xml.
Without registration, Android cannot discover the BroadcastReceiver.

Patcher Location

The hook must be placed at:

p4a/hook.py


And registered in buildozer.spec:

p4a.hook = ./p4a/hook.py

What the Hook Does

The before_apk_assemble hook runs after Buildozer generates Android sources and before the APK is built.

Conceptually, it injects the following into the manifest:

<pre>
<receiver
    android:name="org.redstoon.pushfcmdemo.MyReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="com.redstoon.ALARM_ACTION"/>
    </intent-filter>
</receiver>
</pre>

Simplified hook logic:
<pre>
def before_apk_assemble(toolchain):
    # Locate AndroidManifest.xml
    # Parse it
    # Inject <receiver> entry
    # Save changes before APK assembly

</pre>
This step is mandatory.
Without it:

The alarm may fire

The receiver will never be called

No crash, no logs, no notification

Summary

Place Java sources under src/ with correct package naming

Configure Buildozer to compile Java and include PyJNIus

Implement a BroadcastReceiver to handle alarms

Schedule alarms from Python using AlarmManager

Inject the receiver into the manifest using a Buildozer hook





With these pieces in place, a reliable bridge between Kivy (Python) and Android’s native AlarmManager is established, enabling scheduled tasks that survive app termination.
