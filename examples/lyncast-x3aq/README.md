# LYNCAST X3AQ (Allwinner H726, Android TV 14)

Android TV projector running LuminaOS on an Allwinner H726 SoC (`sun50iw15p1`).

The hardware specification, vendor `SettingsProvider` ADB sabotage analysis, root bypass via the factory `runin` service, and safe debloat notes are in the [device note](../../docs/devices/lyncast-x3aq-allwinner-h726.md).

- **Launcher**: CleanYourTV replaces LuminaOS (`com.htc.luminaos`) as default Home via `cmd package set-home-activity com.gathin.tvhome/.HomeActivity`. LuminaOS must remain enabled to preserve keystone correction, remote pairing, and factory hardware handlers.
- **Autofocus**: Managed by `com.hysd.vafocus` (ChuangFei Vision camera focus motor). Must be kept.
- **HDMI Input**: Managed by `com.softwinner.awsource`. Must be kept.
- **App Store**: Droid-ify (`com.looker.droidify`) works with full Android TV Leanback D-pad navigation.
