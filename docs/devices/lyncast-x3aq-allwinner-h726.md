# LYNCAST X3AQ / Allwinner H726 (LuminaOS), Android 14

Hardware specification, vendor ADB sabotage analysis, root bypass, and debloat notes for the **LYNCAST X3AQ** projector.

| Field | Observed value |
|---|---|
| Retail model | LYNCAST X3AQ Android TV Projector |
| SoC / Board | Allwinner H726 (`sun50iw15p1`), board reports `model: h726-2` / `H726 Focus` |
| Reported Model | `product: rk3562_projector`, `device: rk3562_projector` |
| Fingerprint | `google/redfin/redfin:13/TQ2A.230305.008.F1/7679548:user/release-keys__2025-10-05` (faked as Pixel 5 Android 13, but actually Android 14 / API 34) |
| Android / API | Android 14 / API 34 (`ro.build.version.sdk=34`) |
| Kernel | `5.15.167-g3b33e70e1aa6-dirty #249 SMP PREEMPT armv7l Toybox` |
| SELinux | `Permissive` (`ro.boot.selinux=permissive`) |
| Stock Home | `com.htc.luminaos` (`.activity.MainActivity`) |
| Autofocus / Keystone | `com.hysd.vafocus` (ChuangFei Vision / `tpjz_allwinner` camera stepper) + `com.htc.luminaos` |
| Video Inputs | `com.softwinner.awsource`, `com.softwinner.vis` (VideoInputService) |
| Web Remote API | `com.htc.btremote.receiver` ("Lumina Go" REST API listening on port `8899`) |

---

## 1. Vendor ADB Sabotage: Why the ADB Toggle Reverts to Off

On this firmware, enabling **USB Debugging** in Developer Options, LuminaOS Developer Mode, or Softwinner Factory Menu (`*#*#3646633#*#*` / `AwTvFactoryMenu`) immediately flips back to **Disabled / Off** within 1 second. When connecting over the network on port 5555, the device reports `unauthorized` and **no authorization dialog ever appears on screen**.

### Root Cause in `SettingsProvider.apk`
Decompilation of `/system/priv-app/SettingsProvider/SettingsProvider.apk` revealed custom vendor code in `com/android/providers/settings/SettingsProvider.java`:

```java
private static boolean shouldForceDisableDebugSetting(String str) {
    return "adb_enabled".equals(str) || "adb_wifi_enabled".equals(str) || "development_settings_enabled".equals(str);
}

// In mutateGlobalSetting (write hook):
if (!Build.IS_DEBUGGABLE && shouldForceDisableDebugSetting(str) && (i2 == 1 || i2 == 3)) {
    str4 = "0";
}

// In packageValueForCallResult (read hook):
return (Build.IS_DEBUGGABLE || !shouldForceDisableDebugSetting(setting.getName())) ? setting.getValue() : "0";
```

Because `Build.IS_DEBUGGABLE` checks `ro.debuggable == 1` (which is `0` on user builds):
1. Any write to `adb_enabled` or `adb_wifi_enabled` is actively overwritten to `"0"`.
2. Any query to `Settings.Global.getInt("adb_enabled")` returns `"0"`, causing the UI to animate the switch back to Disabled.
3. In `system_server`, `AdbDebuggingManager` observes `adb_enabled == 0` and stays dormant—it never listens on `/dev/socket/adbd` and never triggers the on-screen key confirmation dialog.

---

## 2. Root Privilege Escalation Vector (SELinux Permissive)

Because kernel SELinux is `Permissive`, arbitrary root command execution is possible via the factory burn-in test service defined in `/system/etc/init/runin.rc`:

```rc
on property:debug.runin.restart=1
    setprop debug.runin.restart 0
    start runin_test_restart

service runin_test_restart /system/bin/runin.sh restart
    oneshot
    disabled
    user root
    group root shell system media_rw wakelock
    seclabel u:r:shell:s0
```

Inside `/system/bin/runin.sh`:
```bash
DEFAULT_CONFIG_FILE=/system/etc/runin_config
CUSTOM_CONFIG_FILE=/sdcard/runin_config
if [ -f $CUSTOM_CONFIG_FILE ]; then
    config=$CUSTOM_CONFIG_FILE
fi

while read line; do
    if [[ $line == "loop="* ]]; then
        eval $line
```

Any unprivileged local process (e.g., SimpleSSHD on port 2222) can write to `/sdcard/runin_config` and trigger `setprop debug.runin.restart 1` to execute arbitrary bash commands as `root` (uid 0).

---

## 3. Bypassing Sabotage: Direct ADB Key Injection

Rather than relying on `AdbDebuggingManager` or the sabotaged `SettingsProvider`, inject the workstation's ADB public key (`~/.android/adbkey.pub`) directly into the target's auth keystore:

```bash
# 1. Stage the payload on /sdcard
cat << 'EOF' > /sdcard/do_root.sh
mkdir -p /data/misc/adb
echo "<YOUR_ADB_PUBLIC_KEY>" > /data/misc/adb/adb_keys
chmod 2750 /data/misc/adb
chmod 0640 /data/misc/adb/adb_keys
chown -R system:shell /data/misc/adb
setprop persist.adb.tcp.port 5555
setprop service.adb.tcp.port 5555
setprop persist.sys.usb.config adb
setprop sys.usb.config adb
setprop persist.adb.tls_server.enable 0
setprop ctl.restart adbd
EOF

# 2. Trigger execution via runin service
echo "loop=1; sh /sdcard/do_root.sh; rm -f /sdcard/runin_config; exit 0" > /sdcard/runin_config
setprop debug.runin.restart 1
```

Once executed, `adbd` restarts on port `5555` with the authorized key preloaded. `adb connect <projector-ip>:5555` immediately authenticates with `device` status.

---

## 4. CRITICAL GOTCHA: The `runin.sh` State File Reboot Trap

When `/system/bin/runin.sh restart` is executed, it creates a persistent state tracking directory at `/data/local/tmp/.runin/`:
```bash
echo 0 > /data/local/tmp/.runin/loop_index
```

On every subsequent cold boot, `init` triggers:
```rc
on property:sys.boot_completed=1
    start runin_test
```
If `/data/local/tmp/.runin/loop_index` exists, `runin.sh` believes an endurance test is active and runs `__runin_reboot_test`:
```bash
function __runin_reboot_test() {
    ...
    sleep 15
    reboot
    exit 0
}
```
**This causes a 15-second reboot loop on every boot.**

### Prevention & Fix
Before rebooting after root execution, **always remove `/data/local/tmp/.runin`**:
```bash
rm -rf /data/local/tmp/.runin /sdcard/runin_config
```
If the device enters the reboot loop:
- The script is programmed with `loop=1`, so it completes after 1 full reboot cycle and cleans itself up automatically.
- Alternatively, connect during the 15-second window and run `rm -rf /data/local/tmp/.runin`.

---

## 5. CleanYourTV Launcher Takeover

1. **Build**: CleanYourTV (`com.gathin.tvhome`) compiles cleanly for API 34.
2. **Install**:
   ```bash
   adb push app-release.apk /sdcard/cleanyourtv.apk
   adb shell pm install -r /sdcard/cleanyourtv.apk
   adb shell rm -f /sdcard/cleanyourtv.apk
   ```
3. **Set Default Home**:
   ```bash
   adb shell cmd package set-home-activity com.gathin.tvhome/.MainActivity
   ```
4. **DO NOT DISABLE `com.htc.luminaos` COMPLETELY**:
   Unlike standard Android TV devices where the stock launcher can be safely disabled, `com.htc.luminaos` also hosts the projector's keystone adjustment receivers (`com.htc.intent.action.OPEN_MANUAL_KEYSTONE`), projection orientation modes, and Bluetooth remote bindings. 
   Leaving `com.htc.luminaos` enabled while pointing `set-home-activity` to `com.gathin.tvhome` works reliably across reboots without losing projector controls.

---

## 6. Debloat List

See [`examples/lyncast-x3aq/debloat.txt`](../../examples/lyncast-x3aq/debloat.txt).

### Safe to Disable (`pm disable-user --user 0`)
- `com.google.android.youtube.tv` (Stock YouTube TV)
- `com.android.ytb` (YouTube accessibility helper)
- `com.android.nfx` (Netflix accessibility helper)
- `com.android.music` (AOSP stock music player)
- `com.softwinner.TvdVideo` (AOSP stock video player)
- `com.softwinner.awlogsettings` (Vendor logging UI)
- `com.htc.hyk_test` (Factory test app)
- `com.htc.screensaver` (Vendor screensaver)
- `com.hotack.screensaversdk` (Vendor screensaver SDK)
- `com.google.android.katniss` (Voice search telemetry)
- `com.android.bips` (Built-in print service)
- `com.android.companiondevicemanager` (Companion manager)
- `com.android.dreams.basic` (AOSP screensaver)
- `com.android.traceur` (System tracing)
- `com.android.htmlviewer` (HTML viewer)
- `com.toofifi.lineserver` (Cast server)
- `com.mphotool.usbcastserver` (USB mirror server)

### Load-Bearing — Keep Enabled
- `com.hysd.vafocus` (Autofocus camera and stepper motors)
- `com.htc.luminaos` (Keystone, settings, remote bindings)
- `com.softwinner.awsource` (HDMI / AV input switcher)
- `com.softwinner.vis` (Video Input Service)
- `com.softwinner.miracastReceiver` (Miracast screen mirroring)
- `com.htc.btremote.receiver` (Remote receiver & Lumina Go Web API)
- `com.softwinner.update` (OTA Update service)

---

## 7. App Stores & Remote Navigation
- **Official F-Droid**: Touch-only UI. Does not support D-pad Leanback navigation well.
- **Droid-ify (`com.looker.droidify`)**: Fully compatible with Android TV D-pad remotes; Material You design works cleanly at 1080p.
- **Google Play Store**: Built-in Play Store remains 100% operational after debloat.
