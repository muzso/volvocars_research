# Volvo EX30 vs. privacy

First published (on [Reddit](https://www.reddit.com/r/ex30/comments/1wp9fio/volvo_ex30_vs_privacy/)): 2026-09-24

Last updated: 2026-10-01

## Abstract

Volvo Cars/Geely tracks your EX30's approximate location even if you didn't give any consent in the vehicle's privacy settings. The proof for this lies in every vehicle's logs and anybody with some expertise can verify it. Also, they collect a lot of privacy sensitive information under the pretext of intrusion detection.

The [Volvo Cars Privacy Notice](https://www.volvocars.com/uk/legal/privacy/privacy-customer-privacy-policy/) claims it's for fulfilling "legal obligations", but I doubt that this is really warranted. And I am pretty sure that (at least in the EU) most people expect that a vehicle manufacturer doesn't (by default and almost secretly) track the location of any of the vehicles it sells and such tracking should be an opt-in (i.e. an explicit user consent should be asked for). This might even conflict with the EU's GDPR rules since owners of a vehicle cannot even opt out from this data collection and the range of collected data doesn't seem to be in line with the targeted goal (intrusion detection, etc.).

In practice this probably only affects a very small percentage of EX30 owners/drivers. My guess is that 99% of vehicle users give their voluntary consent for location tracking (of course only for the purposes that are described in the given setting's description).

Later on I'll show you that this is not necessarily just for the EX30. The possibility of collecting location/position data even without user consent is in the publicly declared privacy policy of Volvo Cars for all of their vehicles.

Note: this document is not for the faint of heart, it's about 30k characters long.

I've split it into six parts:

1. Abstract
2. IDS (Intrusion Detection System) in the EX30 head unit
3. IDS in the EX90 head unit
4. Privacy settings in the EX30
5. Volvo Cars Privacy Notice
6. Conclusions

Note: an automotive [head unit](https://en.wikipedia.org/wiki/Automotive_head_unit) (it's sometimes called the infotainment system) is *a vehicle audio component providing a unified hardware interface for the system, including screens, buttons and system controls for numerous integrated information and entertainment functions*. In modern vehicles it's also a sort of control center for vehicle functions/settings and diagnostics.

Let's cut the chase by looking at the EX30's IDS data collection.

## IDS in the EX30 head unit

The EX30's IDS probably has multiple components, each responsible for different aspects of the vehicle. In the head unit it is implemented by a system app (`/system/app/IDS_GAS/IDS_GAS.apk`) that first appeared in the 1.4.* OTA updates (second half of 2024). The core app code is in the `com.geely.ivisecassistant` package (so the app itself probably came from Geely), but it heavily relies on Baidu's vehicle security SDK (`com.baidu.vehiclesec`, `com.baidu.ivisecdataservicelibrary`). In the UI (Settings / System / Applications / Show all apps / Show system) the app appears as `iviSecAssistant`. It collects all sorts of information which it regularly transmits to an Azure IoT Hub via MQTT. Records are cached locally and transmission occurs once every minute if the cache is not empty.

So far the `iviSecAssistant` app (at least up to sw. version 1.8.0 of the vehicle) has only detection capabilities. It does have the API laid out for active prevention/intervention capabilities, but nothing is implemented yet. Thus in the first two years of the vehicle the app was used as an information gathering tool probably for threat analysis. There was no instant response capability, probably because there was no need for it. E.g. it was just a few months ago (in August) that the news reported about the [first (?) malware that specifically targeted a head unit](https://web.archive.org/web/20260821082641/https://securelist.com/android-head-unit-malware/121106/). So it's totally understandable that so far they focused on detection capabilities.

There's a geofence that limits the data exfiltration of the app: data is only transmitted while the vehicle is within a bounding box (of geolocation coordinates). The app contains bounding boxes for four regions, but only one is used in every vehicle (the applied bbox is hardwired into the compiled code, which also confirms that the vehicle's software is slightly different for each region).

The regional bounding boxes are a bit strange though.

- Europe: https://bboxfinder.com/#37.000000,-9.520000,71.200000,66.200000
- China: https://bboxfinder.com/#3.510000,73.330000,53.330000,135.050000
- Middle-East: https://bboxfinder.com/#12.000000,25.000000,42.000000,65.000000
- USA: https://bboxfinder.com/#24.000000,-125.000000,50.000000,-67.000000

The EX30 is sold in more areas than what's covered by these.

Also, none of the bboxes seem to cover all of their respective regions. E.g. the Europe bbox doesn't properly cover all of Greece. Crete is not covered by the European bbox either, and only the eastern half of it is covered by the Middle-East bbox (but I think Crete probably gets the European sw. version, thus IDS probably doesn't work at all for vehicles on that island). Malta is not covered by any of the bboxes either.

Here're a couple of places that fall outside of all four bboxes:

- all of Australia
- southern half of Mexico
- most of Canada
- approx. 70% of Japan (Kyōtango and everything east of it)
- southern half of Malaysia
- southern coastline of Peloponnese (peninsula in southern Greece)
- western half of Crete (Greece)
- southern coastline of Spain
- south of Sicily
- parts of the western coastline of Ireland
- south border of Turkey
- (etc.)

The IDS functionality is split into modules (they are called "monitors"). Each monitor hands a JSON record to a service that caches it and flushes the cache every minute, batching up to 10 records into messages (<= 32268 bytes each) published to Azure IoT Hub over MQTT. Sending only happens when the vehicle's GPS is inside a bounding box, otherwise the records stay cached. Every message is wrapped in an envelope carrying the vehicle/device identity, the collected records sit in its `content[]` array.

Example for an envelope (wrapper) sent with every message:

```json
{
  "brand": "VolvoCars", "OEM": "Volvo",
  "chanNo": "V216", "vehConf": "V216", "upModu": "DHU",
  "dVer": "2.2.1", "dataMode": "1.5",
  "dID": "SN-0a1b2c3d", // the `ro.serialno` system property
  "VIN": "q1W2e3R4t5Y6u7I8o9P0aSdFgHjKlZxCvBnM12==", // white-box-AES encrypted
  "lat": "50.11", "lon": "8.68", // GPS coordinates rounded to 2 dp
  "sIP": "0.0.0.0", // WiFi IP
  "period": "60-s",
  "rpTime": "2026-05-01T09:15:00.000+02:00", "timeStamp": 1780000100000, // time of upload
  "sec": "",
  "content": [ /* one or more records, see below */ ]
}
```

Each uploaded MQTT message contains a `lat` and `lon` coordinate pair, both are rounded to 2 decimal places. This makes the precision/accuracy of a single location into an approx. 0.75 km x 1.11 km (i.e. 0.47 miles x 0.69 miles) area. These messages are sent at least every minute, sometimes even more often, so even though the coordinates are not very precise, the frequency paints a pretty good picture on the comings and goings of the vehicle, and by extension its driver. I think that this location tracking would be unwarranted even if it recorded coordinates with a 50 km accuracy and only once a day.

Each `content[]` record has the shape `{ agent, agentVer, OS, dID, collTime, idsType, op, cate, info: { leakType, leakID, ext, type, secLog } }`.

### What is collected & uploaded

The examples below show only the `secLog` payload, the part that differs per data type.

| Data type | Module (config name) | Trigger / interval | Seen uploaded |
| --- | --- | --- | --- |
| System / HW / OS asset info | `systemInfoMonitor` | every 3 h, only if changed | yes |
| Installed-app inventory | `appInfoMonitor` | once at boot + on app add/remove/replace; one message per app | yes |
| App behaviour (install/uninstall, APK download, crash/ANR) | `appInfoMonitor` | event | yes |
| Running-process list | `processInfoMonitor` | sampled every 10 min, reported 30 min, only if changed | yes |
| Security posture (root/SELinux/adb/TEE) | `secConfigMonitor` | every minute, only if changed | yes |
| Network traffic-speed warning | `networkInfoMonitor` | sampled 10 s; sent only when Rx>5120 KB/s or Tx>1024 KB/s | yes |
| Network data-usage rollup | `networkInfoMonitor` | every 3 h | yes |
| Active socket/TCP connections | `networkConnectionMonitor` | sampled 10 min, reported 30 min | yes |
| WiFi state / connection info | `wifiMonitor` | event (connect/disconnect, radio on/off, hotspot on/off) | yes |
| SELinux `avc: denied` log lines | `logInfoMonitor` | 100 line batches, every minute | yes |
| Location-permission usage (which app) | `locationMonitor` | event (GNSS active) | yes |
| File-integrity change on watched paths | `fileAccessMonitor` | event (inotify) | yes |
| Heartbeat (CPU/mem/disk) | *(not a monitor)* | every 10 min | yes |
| Per-app usage stats | `appUsageInfoMonitor` | daily | yes |
| Camera-permission usage (which apps) | `cameraMonitor` | event (camera in use) | no (active, but not seen) |
| Mic-permission usage (which apps) | `micMonitor` | event (recording / call) | no (active, but not seen) |
| Geely framework logs | `geelyFrameworkLogCollector` | 100 line batches, every minute | no (active, but not seen) |
| Geely QNX security logs | `geelyQnxLogCollector` | hourly | no (active, but not seen) |

"(active, but not seen)" = the monitor is enabled, but I didn't see it in the logs

---

### Fictional examples per data type

I did thoroughly go through the logs of my own car, but I won't post them for privacy reasons. Instead I provide actual logs, but with made up device/app/IP/location/etc. details. I kept the data from the original logs where it cannot have any privacy implications.

#### System / HW / OS asset - `systemInfoMonitor`

Every 3h, only when the fingerprint changes (normally once per boot). MACs and VIN are encrypted.

```json
"secLog": {
  "androidos": {
    "type": "x-android-os",
    "android_version": "12",
    "os_fp": "VolvoCars/gecko_gas/habanero:12/SAG3.230621.002/xxxxxxxxxxxx:user/release-keys",
    "kernel_version": "5.4.274-qgki-gxxxxxxxxxxxx",
    "build_id": "SAG3.230621.002",
    "security_patch_date": "2025-07-05",
    "baseband": "VCC",
    "tee": 1,
    "sn": "0a1b2c3d"
  },
  "hardware": {
    "type": "x-hardware-info",
    "eth0_mac": "Kx8Lp2Qm5Rn7Ts0Vw3Yb99==\n",
    "wlan0_mac": "Kx8Lp2Qm5Rn7Ts0Vw3Yb99==\n",
    "mem_size": "7346MB",
    "disk_size": "2655MB",
    "bluetooth_info": {
      "bluetooth": true,
      "BLE": true
    },
    "cpu_info": {
      "cpu_core_num": 8,
      "cpu_detail": [
        "..."
      ]
    }
  },
  "manufacturer": {
    "type": "x-manufacturer-info",
    "manufacture": "Volvo",
    "brand": "VolvoCars",
    "model": "EX30",
    "product": "gecko_gas",
    "device": "habanero",
    "board": "msmnile",
    "chip_manufacturer": "qcom",
    "screen": {
      "resolution": "1200*1472",
      "density": "180dpi"
    }
  },
  "vehicle": {
    "type": "x-vehicle-model",
    "vehicle_model": "V216",
    "vehicle_vin": "q1W2e3R4t5Y6u7I8o9P0aSdFgHjKlZxCvBnM12=="
  }
}
```

#### Installed-app inventory - `appInfoMonitor`

Once at boot and on any package add/remove/replace; one message per app. Launcher apps only.

```json
"secLog": {
  "dataArray": [
    {
      "name": "com.example.player",
      "appName": "Example Player",
      "versionName": "3.1.0",
      "versionCode": 310,
      "isSystemApp": false,
      "uid": 10142,
      "minSdkVersion": 29,
      "targetSdkVersion": 34,
      "sourceDir": "/data/app/~~aBcD==/com.example.player-XyZ==/base.apk",
      "apkSize": 41231234,
      "nativeLibraryDir": "/data/app/~~aBcD==/com.example.player-XyZ==/lib/arm64",
      "dataDir": "/data/user/0/com.example.player",
      "apkMD5": "0f1e2d3c4b5a69788796a5b4c3d2e1f0",
      "mD5": "AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99",
      "sHA1": "00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33",
      "sHA256": "00:11:...:FF",
      "mainActivity": "com.example.player.MainActivity",
      "mainApplication": "com.example.player.App",
      "primaryCpuAbi": "arm64-v8a",
      "firstInstallTime": 1735689600000,
      "lastUpdateTime": 1746057600000,
      "permissions": "android.permission.INTERNET;android.permission.ACCESS_NETWORK_STATE;..."
    }
  ]
}
```

#### App behaviour - `appInfoMonitor`

Event: package added/removed/replaced, an `.apk` download completing, or an app crash / ANR / system-not-responding.

```json
"secLog": {
  "type": "x-android-app-action",
  "packageName": "com.example.player",
  "actionType": "package_action",
  "packageAction": "package_added",
  "activityAction": "",
  "version": "3.1.0",
  "apkMD5": "0f1e2d3c4b5a69788796a5b4c3d2e1f0",
  "sHA1": "00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33"
}
```

(Crash/ANR variant: `"actionType": "activity_action", "activityAction": "app_crashed"` - or `app_not_responding` / `system_not_responding` - with `packageName` set.)

#### Running processes - `processInfoMonitor`

Sampled every 10 min, reported every 30 min, only when the process set changes. `tracepid` <> 0 signals a debugger/ptrace.

```json
"secLog": {
  "dataArray": [
    {
      "pid": 3340,
      "ppid": 505,
      "uid": 1000,
      "processName": "com.geely.ivisecassistant:SecMonitorService",
      "tracepid": 0,
      "vmrss": 109780,
      "vmsize": 15255260
    },
    {
      "pid": 999,
      "ppid": 1,
      "uid": 0,
      "processName": "/system/bin/IdsNativeInfoReader_GAS",
      "tracepid": 0,
      "vmrss": 3152,
      "vmsize": 10826092
    }
  ]
}
```

So far the process list contains only the processes of the `iviSecAssistant` app, but this is more likely a bug than a planned feature (the latter would be useless). The app could most definitely access the full process list if implemented correctly.

#### Security posture - `secConfigMonitor`

Every minute, but only if something changed.

```json
"secLog": {
  "selinux_status": 2,
  "debug_status": 0,
  "root_status": 0
}
```

`selinux_status`:

- `0`: disabled
- `1`: permissive
- `2`: enforcing

`debug_status`: `0` / `1` (shows whether ADB is enabled)

`root_status`: `0` / `1`  (shows whether device is rooted, i.e. `which su` test)

#### Network traffic-speed warning - `networkInfoMonitor`

Sampled every 10 s, emitted only when download > 5120 KB/s or upload > 1024 KB/s. Names the heaviest app.

```json
"secLog": {
  "type": "x-traffic-speed-warning",
  "x-rxSpeed": 8000,
  "x-txSpeed": 30,
  "x-most-packagaes": {
    "rx": "com.example.player",
    "tx": ""
  }
}
```

#### Network data-usage rollup - `networkInfoMonitor`

Every 3 h (per-app / per-interface byte totals over rolling 1 h windows).

```json
"secLog": {
  "type": "x-traffic-usage",
  "duration_num": 12,
  "apps": [
    {
      "name": "com.example.player",
      "rxBytes": 123456789,
      "txBytes": 2345678
    }
  ]
}
```

#### Active connections - `networkConnectionMonitor`

Sampled every 10 min, reported every 30 min. IPs are encrypted, ports/state are not.

```json
"secLog": {
  "total": 2,
  "conn_info": [
    {
      "lIp": "Kx8Lp2Qm5Rn7Ts0Vw3Yb99==",
      "lPort": 49152,
      "rIp": "Vw3Yb99Kx8Lp2Qm5Rn7Ts0==",
      "rPort": 443,
      "state": "ESTABLISHED",
      "type": "tcp"
    },
    {
      "lIp": "Kx8Lp2Qm5Rn7Ts0Vw3Yb99==",
      "lPort": 0,
      "rIp": "",
      "rPort": 0,
      "state": "LISTEN",
      "type": "tcp"
    }
  ]
}
```

#### WiFi info - `wifiMonitor`

Events:

- on connect: full detail
- radio on/off
- hotspot on/off

BSSID and IP are encrypted.

```json
"secLog": {
  "wifi_conn_state": 1,
  "wifi_detail": {
    "ssid": "ExampleWiFi",
    "bssid": "Rn7Ts0Vw3Yb99Kx8Lp2Qm5==",
    "ip": "Qm5Rn7Ts0Vw3Yb99Kx8Lp2==",
    "enc_type": "WPA/WPA2",
    "rssi": -54,
    "state": "COMPLETED",
    "link_speed": 433,
    "apFreBand": "5 GHz",
    "dns1": "192.168.1.1",
    "dns2": "8.8.8.8"
  }
}
```

#### SELinux denials - `logInfoMonitor`

`logcat -b events` lines containing `avc: denied`.

Lines are transmitted in batches (max. 100 at a time) every minute and are base64-encoded.

```json
"secLog": {
  "data": "MDYtMjIgYXZjOiBkZW5pZWQgeyByZWFkIH0gZm9yIC4uLg=="
}
```

#### Location-permission usage - `locationMonitor`

Collected when GNSS becomes active, reports which app used the location permission.

```json
"secLog": {
  "name": "com.example.maps",
  "version": "25.1.0",
  "lastTimeUsed": "2026-05-01T09:14:58.000+02:00"
}
```

#### Camera-permission usage - `cameraMonitor`

Collected when a camera is activated, reports which app used the camera permission.

```json
"secLog": {
  "name": "com.example.dashcam",
  "version": "2.0.0",
  "lastTimeUsed": "2026-05-01T09:10:00.000+02:00",
  "cameraId": "0",
  "cameraList": [
    "0",
    "1"
  ]
}
```

#### Mic-permission usage - `micMonitor`

Collected when a microphone is activated, reports which app used the microphone permission.

```json
"secLog": {
  "name": "com.example.voice",
  "version": "1.4.2",
  "lastTimeUsed": "2026-05-01T09:11:30.000+02:00"
}
```

#### File-integrity change - `fileAccessMonitor`

Watches a specific set of paths for changes (using `inotify`). The intent is probably host-integrity and rootkit detection.

```json
"secLog": {
  "path": "/system/bin/example",
  "name": "example",
  "op": "MODIFY",
  "mask": 2,
  "md5": "0f1e2d3c4b5a69788796a5b4c3d2e1f0",
  "attrib": {
    "size": 4096,
    "modiDate": "2026-05-01T09:00:00+02:00",
    "accDate": "",
    "permission": "r-x"
  }
}
```

This only watches the following paths:

- `/data/app`
- `/system/app`
- `/system/bin`
- `/system/etc`
- `/system/lib`
- `/system/lib64`
- `/system/priv-app`
- `/system/xbin`

And the following if the IDS has root permissions (which at the time of writing it does not have):

- `/init.rc`
- `/init.usb.configfs.rc`
- `/init.usb.rc`
- `/init.zygote32.rc`
- `/init.zygote64_32.rc`
- `/sbin/su`
- `/system/etc/selinux`

In actual logs only paths with the following two patterns were seen:

- `/data/app/~~jfEhikU3Sth7FHGd4dfh4c==`
- `/data/app/vmdl1947567326.tmp`

#### Per-app usage stats - `appUsageInfoMonitor`

Once every day.

```json
"secLog": {
  "dataArray": [
    {
      "name": "com.example.player",
      "lastTimeUsed": "2026-05-01T08:00:00.000+02:00",
      "launchCount": 12,
      "totalTimeInForeground": 3600000
    }
  ]
}
```

#### Geely framework logs - `geelyFrameworkLogCollector`

Output of `logcat | grep ESAL:`, batches of 100 records or every minute (whichever comes first), base64-encoded.

```json
"secLog": {
  "data": "WyJFU0FMIGV4YW1wbGUgc2VjdXJpdHkgbG9nIGxpbmUiXQ=="
}
```

#### Geely QNX security logs - `geelyQnxLogCollector`

Hourly, reads the QNX-domain `DIM_Slog.log` `ESAL:` lines (JSON array), base64-encoded.

```json
"secLog": {
  "data": "WyJFU0FMIGV4YW1wbGUgc2VjdXJpdHkgbG9nIGxpbmUiXQ=="
}
```

#### Heartbeat - (not a monitor)

Every 10 min.

```json
"secLog": {
  "memory_info": {
    "mem_usage_rate": "55.0%",
    "mem_avail": "3200MB",
    "mem_total": "7483MB"
  },
  "cpu_info": {
    "cpu_usage_rate": "unknown"
  },
  "disk_info": {
    "internal_storage_avail": "20000MB",
    "disk_usage_rate": "66.0%",
    "internal_storage_total": "58839MB"
  }
}
```

## IDS in the EX90 head unit

I had the opportunity to look into the EX90 as well and its IDS app seems to be a lot less privacy invasive. It's actually two apps (sitting surrounded by a framework, i.e. relying on other apps/components):

- `com.volvocars.intrusiondetection` (app label is "IntrusionDetection")
- `com.volvocars.intrusiondetectiongateway`

The version I looked at (from 2025) only looks for two kinds of events:

- SELinux denials
- app installed/removed/replaced

No device identifiers, no network info (IDs, connections), no running processes, etc. And possibly no geolocation. I'm not 100% sure about the latter, because I couldn't look at all layers of the upload process (the actual HTTP requests go through the HAL and I've only seen the apps).

It has the nuts and bolts for preventive actions too, but (same as in the EX30) nothing was implemented yet (at least in the version of the app that I saw). E.g. actions like retrieve logs on command, remove an app/package, disable the internet access, do a factory reset and "limp home". I wonder what the last one means. Maybe the car should drive itself (very slowly) home? :o That would be truly wild.

## Privacy settings in the EX30

The EX30 has a "Privacy" screen with two toggles:

1. Vehicle location sharing

Description:

> "By disabling vehicle location sharing, location data will not leave the vehicle, except when required by law or to support third parties' apps and services which you have independently agreed to. When driving a Volvo Cars vehicle, location data is being used by some of the functions and services. We also give our customers an easy means of control when location data is being processed by us. By disabling vehicle location sharing, location data will not leave the vehicle, except when required by law or to support third parties' apps and services which you have independently agreed to. This means that some vehicle functions and services will not work as intended, such as Volvo Cars app, connected safety, vehicle data analytics etc."

2. Vehicle Analytics & Improvements

This is a longer text and Markdown's support for formatted content inside quotes isn't the best, so I'll just put it between two markers.

=== description start ===

"If you provide your consent by enabling Vehicle Analytics & Improvements, we will collect and use data (listed below) to get information about your vehicle and how it is being used. We use this information for fault tracing, product research and development purposes, such as to monitor and improve the sustainability and quality of our vehicles, their safety features and other functions. If we think it is needed, we will also contact and inform you of potential problems or faults.

The data we collect and use:

- Information related to the vehicle: model, production year, Vehicle Identification Number (VIN), hardware and software information.
- Information generated by the vehicle, its sensors and systems: odometer, service indicators, alarms, warnings, lights, faults, diagnostics, connectivity and network information, certifications and authorisation data, security logs, calibrations, temperature, weather and road conditions, energy and fuel consumption, timestamps, state of health, status of onboard systems and functions, behaviour of onboard systems and functions and output of onboard systems and functions.
- Information related to how the vehicle is used and operated: configurations, settings, device connections, trailer connected, usage mode (charge, park, drive), charging information, timestamps, speed, passenger occupancy, interactions and use of the infotainment head unit (the screen in the car), use of remote services, use of car functions such as accelerator, brakes, steering, seat belts and doors.
- Location data: Location is collected in the following cases.
  - incidents (near accident) and accidents;
  - if the eCall function has been activated; and
  - in case of connectivity issues.
- Location is also continuously collected from safety functions, autonomous drive and advanced driving assistance systems.
- Camera data: outward-facing camera images are captured
  - in connection to incidents and accidents; and
  - continuously from safety functions, autonomous drive and advanced driving assistance systems.

The legal basis for the collection and use of the data mentioned above is your consent. We keep the data for five years, other than the information on the car’s high-voltage battery, which is retained for the life of the battery. At the five-year mark, we will delete the data or, if we see a need to keep the data, we make sure it’s completely anonymised."

=== description end ===

Both of these toggles can grant or revoke consent regarding certain kinds of data collection. Both involve location as well.

The Privacy screen has a third item too: the "Privacy Notice". It shows you a QR-code that contains this URL: https://www.volvocars.com/intl/legal?path=privacy/privacy-car

It leads to the Volvo Cars General Privacy Notice. E.g. for UK customers it's at https://www.volvocars.com/uk/legal/privacy/privacy-customer-privacy-policy/

Note: the content is pretty much the same for all countries/markets.

## Volvo Cars Privacy Notice

The Privacy Notice lists a lot of data (some personal, some perhaps not) that are collected through various means (email, phone calls, website, smartphone app, vehicle, etc.). Each data collection has a "legal basis" category, one of the following four:

1. Contract
2. Legitimate (business) interest
3. Consent
4. Legal obligation (i.e. required by law)

In the Privacy Notice look for the phrase "position and movement information" (or just the "position" keyword). Position is listed in 19 rows, and only 2 have "consent" as the sole legal basis, one has both "contract" and "consent" (in which case contract probably trumps consent) and 16 have one of the other three justifications (contract, legitimate interest or legal obligation).

The toggles in the vehicle most likely only affect data collections where "consent" is the sole reason.

If I am optimistic, the following (from the 17 that are not just consent-based) might apply to the vehicle all the time while it's being used:

- Remote vehicle services (legal basis: contract)
- Connect Plus (legal basis: contract and legal obligation)
- Intrusion Detection System (legal basis: legal obligation)
- Intelligent Speed Assist (legal basis: legal obligation)

I've not included those that match any of the following:

- the data collection is assumably bound to specific events / services that the user might decide to use (or not to use)
- the data collection can be opted out (i.e. consent revoked)
- the specific feature doesn't seem to be available in the EX30 (e.g. Connected Safety)

So despite the user's settings in the vehicle, there are plenty of other "reasons" for Volvo Cars/Geely to continuously record your vehicle's location. I understand that switching that location consent toggle to "off" does disable some kind of location tracking, but it doesn't make a difference from a privacy PoV if location is still being tracked by other means (component/feature).

Also, I understand that "data collection" and "retention period" don't necessarily mean that the given data leaves the vehicle. But honestly: if this was the case (i.e. the Notice listed also location tracking where the data stays in the vehicle), the Privacy Notice would be very incomplete since there's a lot more data in the vehicle than what's listed in the Notice. It seems to be a reasonable assumption that the Privacy Notice is about data that Volvo Cars/Geely collects and preserves outside the vehicle.

Also, the "position" term doesn't reveal the accuracy. It is possible that "position" means different things for different vehicle features/services (as seen in the case of the EX30 IDS, which collects an approximate location).

I've a couple of concerns regarding some of the location tracking justifications.

1. Connect Plus: why would anyone need the vehicle's position to provide internet connectivity? What might the "legal obligation" mean in this case? Does Volvo Cars sell the EX30 in countries where internet service providers are mandated (by law) to track the location of their users? The retention column is pretty vague too: "For as long as needed to provide you with connectivity services. Your data will be retained for as long as required under applicable law." Btw. there's a "Specific Terms for Connect Plus EX30" section on the [Volvo Cars General Terms and Conditions for Services](https://www.volvocars.com/uk/legal/terms/terms-services/) page that starts with this: "Volvo Cars is not a provider of internet or telecommunication services". And this is just for the EX30! The "Specific Terms for Connect Plus" section (for all other models) doesn't have this sentence in it.

2. Intelligent Speed Assist (ISA): again, the justification is "legal obligation". Where does EU Reg. 2019/2144 (GSR, aka. General Safety Regulation) or 2021/1958 (technical requirements and test procedures for the type-approval of ISA systems) say that any of the ISA reporting requirements need location data? Reg. 2021/1958 requires automakers to report statistics and none of it requires vehicle location. I am no lawyer or expert in these regulations, but from what I could gather, location tracking is not necessary for any of it. It wouldn't even make sense.

3. Intrusion Detection System (IDS): the Privacy Notice says the justification for the data collection is "legal obligation". What law/regulation requires location data for an IDS?

An IDS is part of the solution to the following requirement of UN Reg. 155 (which is a vehicle type-approval standard that mandates robust cybersecurity risk management for connected vehicles):

> 7.3.7. The vehicle manufacturer shall implement measures for the vehicle type to:
> (a) Detect and prevent cyber-attacks against vehicles of the vehicle type;
> (b) Support the monitoring capability of the vehicle manufacturer with regards to detecting threats, vulnerabilities and cyber-attacks relevant to the vehicle type;
> (c) Provide data forensic capability to enable analysis of attempted or successful cyber-attacks.

The regulation does not tell manufacturers how to do this, it is up to Volvo Cars/Geely to make decisions about the implementation. It seems that their decision includes the collection of vehicle location data.

Unconsented location tracking is a severe privacy violation in the EU and the mere requirement to "detect and prevent cyber-attacks" (which is nowadays a very generic requirement in many industries and areas of work, etc.) has never before required the knowledge of the geographical location of the given system. So I don't understand the reasoning here.

Let's take a deeper look at the data collection of the IDS (i.e. Intrusion Detection System).

First: what exactly does the Privacy Notice say about it?

Purpose of data collection:

> To detect and prevent anomalies and violations that could be possible cybersecurity threats for the vehicle systems. To monitor how software and apps behave to ensure they are not violating what they are permitted to do.

The data that is collected:

- Vehicle details
- Vehicle information and status
- Vehicle usage and user behaviour
- Position and movement information

Retention period:

> Your data will be retained for up to one (1) year after the investigation has been concluded.

This is interesting. The data collection is continuous and doesn't occur just at a moment, when somebody/something decides that an investigation should take place. How does this retention definition translate for the continuously recorded data? The existence of the collected data is likely a prerequisite for any investigation. So the maximum data retention interval must consist of two time periods:

1. An unknown time period the collected data is retained for by default.
2. Up to one additional year (as per Notice) if the data is involved in an investigation.

The Notice doesn't say anything about the first part.

About the purpose of the data collection ...

> To monitor how software and apps behave to ensure they are not violating what they are permitted to do.

This sounds very much aimed at the head unit. No other part of the vehicle allows software or "apps" (from third party sources).

I wonder: who gets to decide what software and apps running on these vehicles (i.e. vehicle owners' properties) "are permitted to do"? Where are these rules published? I mean if you buy any other device that allows you to run apps, the rules are usually very well defined. In case of Volvo Cars they made a bet on the Android Automotive OS so it'd make sense that unless Volvo Cars publishes its own ruleset (that owners can get familiar with before making a purchase), the rules made by the vendor of AAOS (Google) apply.

E.g. rooting an Android device does not automatically void any warranties, it depends on the manufacturer's Terms and Services. Google's own Pixel devices allow rooting. Volvo Cars did not announce any such rules, thus (by default) rooting shouldn't allowed here as well. Not that I'm counting on it, I expect them (especially seeing how IDS works) to consider rooting a warranty voiding action.

People have been getting emails if they charged their EX30 beyond 70% SoC with a car that is involved in the battery recall (which imho was handled very badly, some people have been waiting for 9+ months for a battery replacement/fix). I wonder if the owner of a vehicle would get an email too if he/she rooted the head unit. They certainly have the monitoring in place to tell.

## Conclusions

All of my investigations were conducted using non-intrusive methods. The vehicles were not hacked/rooted or otherwise modified. I wouldn't do anything like that till my warranty expires. I only used methods that are completely legitimate in all Android ecosystems that I know of (i.e. read operations in the file system, access to public interfaces of apps, etc.). If someone would go deeper, probably even more data collection practices could be discovered. My guess is that the TCAM module does some data (perhaps location) exfil as well.

So far the head unit of the EX30 seems to be the only one Volvo Cars had its parent company Geely build through Chinese suppliers (ECARX in this case). You cannot really see the difference on the surface (e.g. the EX30 and the EX90 UI are pretty similar), but under the surface it's a pretty different world.

Also, I suspect this is not just about the head unit. My hypothesis is that the EX30 was designed by Volvo Cars (they wrote the specs for it), but the engineering/implementation part was mostly done by Geely and its suppliers.

The privacy invasive, 24/7 data collection in the EX30 head unit is most certainly not something I'd expect from a respected European automaker. And although primarily Geely is to blame, the brand belongs to Volvo Cars.
