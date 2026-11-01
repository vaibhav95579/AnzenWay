# AnzenWay (ModeTracker) - Project Documentation

## 1. Project Overview
**AnzenWay** (internally known as ModeTracker) is an Android application designed to ensure user safety while in motion—specifically while walking or driving. The core philosophy is "Safety in Motion." It minimizes distractions by automatically screening incoming calls when the user is in a "dangerous state" (e.g., walking in a traffic zone or driving above a certain speed). It also includes a highly reliable emergency SOS feature triggered by a violent shake of the device.

## 2. Key Features
* **Walk Mode (Geofenced Distraction Blocking):** Monitors the user's walking activity using the accelerometer and checks if they are inside a predefined dangerous geofence using GPS. If both conditions are met, it blocks incoming calls and displays a warning overlay.
* **Drive Mode (Speed-based Distraction Blocking):** Monitors the user's speed via GPS. If the speed exceeds a user-defined threshold, it activates a safety shield, drops active calls, and blocks incoming calls.
* **Smart Auto-Reply (Dual SIM Support):** When a call is blocked, the app automatically sends a customized SMS to the caller. The user can explicitly choose which SIM card (SIM 1 or SIM 2) to use for sending these texts.
* **Shake-to-SOS (Emergency Alert System):** If the user violently shakes the phone, the app fetches their live GPS location, sends an emergency SMS with a Google Maps link to two predefined contacts, and automatically places a phone call to the primary contact.

---

## 3. Core Architecture & Components

The project is built using Kotlin and relies heavily on Android Background Foreground Services, Sensor APIs, Location APIs, and Telephony APIs.

### 3.1 Background Services
* **`WalkTrackingService.kt`**: A Foreground Service that keeps the Walk Mode alive when the app is minimized. It integrates the `WalkAccelerometerTracker` and the `ShakeDetector`. It handles displaying the UI overlay (Shield) when danger is detected.
* **`DriveTrackingService.kt`**: A Foreground Service that tracks the user's driving speed using `FusedLocationProviderClient`. If the speed threshold is crossed, it triggers the danger state and shows the UI overlay.
* **`AnzenCallScreeningService.kt`**: Extends Android's native `CallScreeningService`. It intercepts incoming calls before they ring. It checks `WalkTrackingService.isDangerousState` and `DriveTrackingService.isDangerousState`. If either is true, it silently rejects the call and dispatches the auto-reply SMS.

### 3.2 Sensors & Trackers
* **`WalkAccelerometerTracker.kt`**: Combines Accelerometer data and GPS data. 
    * Calculates walking magnitude (threshold > 11.2f).
    * Checks distance against a hardcoded danger zone (Currently: `30°22'34.7"N 78°00'57.9"E`) using a radius defined in Settings.
* **`ShakeDetector.kt`**: A `SensorEventListener` that calculates 3D G-Force. 
    * If G-Force > `3.2G`, it triggers a callback. 
    * Includes a `5000ms` debounce/cooldown to prevent spamming multiple alerts.

### 3.3 User Interface (UI)
* **`MainActivity.kt`**: The main dashboard. Features prominent cards to toggle Walk Mode and Drive Mode on/off. Updates UI states to show "Active" or "Standby".
* **`SettingsActivity.kt`**: Allows the user to configure preferences. Saves data via `SharedPreferences`.

---

## 4. How It Works (Technical Workflows)

### A. The Distraction Blocking Workflow (Walk / Drive)
1. User activates a mode via `MainActivity`.
2. A **Foreground Service** starts, showing a persistent notification to prevent the OS from killing it.
3. **Detection**:
   * *Drive*: `FusedLocationProviderClient` updates location every 2 seconds. Calculates speed. If `Speed > Settings.DriveThreshold`, state = DANGEROUS.
   * *Walk*: Accelerometer detects walking motion. GPS verifies if the user is inside the Geofence. If both true, state = DANGEROUS.
4. **UI Overlay**: A `WindowManager` overlay (Shield) is drawn over the screen to deter phone usage.
5. **Call Screening**: If a call comes in, `AnzenCallScreeningService` evaluates the state. If dangerous, it tells the Telecom system to reject the call (`setRejectCall(true)`).
6. **SMS Dispatch**: It queries `SharedPreferences` for the user's preferred SIM slot. It uses `SubscriptionManager` to get the correct `SmsManager` and sends the custom text message.

### B. The Emergency SOS Workflow
1. User violently shakes the phone.
2. `ShakeDetector` calculates G-force > 3.2G and triggers the `onShake` callback inside `WalkTrackingService` (or `DriveTrackingService`).
3. **Location Retrieval**: App requests `lastLocation` from `FusedLocationProviderClient`.
4. **Message Construction**: Creates a string: `"EMERGENCY! I need help. My current location is: https://maps.google.com/?q={lat},{long}"`.
5. **SMS Dispatch**: Uses `SmsManager` to send this text to both `Emergency Contact 1` and `Emergency Contact 2` (loaded from SharedPreferences).
6. **Auto-Call**: Immediately fires an `Intent.ACTION_CALL` using `Intent.FLAG_ACTIVITY_NEW_TASK` to dial `Emergency Contact 1`.

---

## 5. SharedPreferences (Data Storage)
The app uses a shared preference file named `AnzenWaySettings` with the following keys:
* `custom_sms`: The custom text sent to rejected callers.
* `drive_speed`: The km/h threshold for Drive Mode.
* `walk_radius`: The geofence radius (in meters) for Walk Mode.
* `default_sms_sim_slot`: Integer (0 for SIM 1, 1 for SIM 2).
* `sos_contact_1`: Phone number for the primary emergency contact.
* `sos_contact_2`: Phone number for the secondary emergency contact.

---

## 6. Required Permissions
Because of its robust background, telecommunication, and location features, AnzenWay requires extensive permissions in `AndroidManifest.xml`:

* **Location**: `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `ACCESS_BACKGROUND_LOCATION` (for foreground service tracking).
* **Telephony & SMS**: `SEND_SMS`, `READ_PHONE_STATE` (to read Dual SIM info), `CALL_PHONE` (for SOS auto-call).
* **Call Screening**: `ANSWER_PHONE_CALLS`, `READ_CALL_LOG`, `READ_CONTACTS` (to find who called and log it).
* **System**: `FOREGROUND_SERVICE` / `FOREGROUND_SERVICE_LOCATION` (to run services), `SYSTEM_ALERT_WINDOW` (to draw the warning shield over other apps).