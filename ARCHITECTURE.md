# Background Geolocation Architecture

## Overview

The `flutter_background_geolocation` plugin is a sophisticated, battery-conscious location tracking system that uses motion detection intelligence to automatically manage location services on iOS and Android devices. This document explains how the plugin works internally.

## Table of Contents

- [Core Philosophy](#core-philosophy)
- [Architecture Overview](#architecture-overview)
- [Motion Detection System](#motion-detection-system)
- [Tracking Lifecycle](#tracking-lifecycle)
- [Key Components](#key-components)
- [Platform-Specific Implementations](#platform-specific-implementations)
- [Event System](#event-system)
- [Location Sampling and Filtering](#location-sampling-and-filtering)
- [Power Management](#power-management)

## Core Philosophy

The plugin's fundamental approach (detailed in the [Philosophy of Operation](https://github.com/transistorsoft/flutter_background_geolocation/wiki/Philosophy-of-Operation)) is based on **motion detection** using device sensors:

### Two-State System

The plugin operates in one of two states:

1. **STATIONARY State**: Device is detected as not moving
   - Location services are **OFF** to conserve battery
   - Motion detection sensors are active (accelerometer, gyroscope, magnetometer)
   - **iOS**: Creates a circular geofence (stationaryRadius) around the current location
   - **Android**: Monitors the Activity Recognition API

2. **MOVING State**: Device is detected as moving
   - Location services are **ON** and actively tracking
   - Records locations according to `distanceFilter` configuration
   - Automatically adjusts tracking based on speed (elastic distance filtering)

### The Key Insight

The plugin doesn't keep GPS running constantly. Instead, it:
- Uses low-power motion sensors to detect when movement starts
- Only activates GPS when movement is detected
- Automatically turns GPS off when the device becomes stationary
- This approach can save **massive amounts of battery** compared to continuous GPS tracking

## Architecture Overview

The plugin uses a three-layer architecture:

```
┌─────────────────────────────────────────────────┐
│          Flutter/Dart Layer                     │
│  ┌───────────────────────────────────────────┐  │
│  │  BackgroundGeolocation API                │  │
│  │  - Event listeners (onLocation, etc.)     │  │
│  │  - Control methods (start, stop, etc.)    │  │
│  │  - Configuration (Config)                 │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
                      ↕ MethodChannel / EventChannel
┌─────────────────────────────────────────────────┐
│          Platform Channel Layer                 │
│  ┌───────────────────────────────────────────┐  │
│  │  FLTBackgroundGeolocationPlugin           │  │
│  │  - Method handlers                        │  │
│  │  - Event stream handlers                  │  │
│  │  - Platform message routing               │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
                      ↕
┌─────────────────────────────────────────────────┐
│       Native Platform Layer                     │
│  ┌──────────────────┐  ┌──────────────────────┐ │
│  │  iOS (Obj-C)     │  │  Android (Java)      │ │
│  │                  │  │                      │ │
│  │  CoreLocation    │  │  FusedLocation API   │ │
│  │  CMMotionManager │  │  ActivityRecognition │ │
│  │  Geofencing      │  │  Geofencing          │ │
│  └──────────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## Motion Detection System

### How Motion is Detected

#### Android
Uses Google's **Activity Recognition API**:
- Monitors device sensors continuously in the background
- Classifies user activity: `still`, `on_foot`, `in_vehicle`, `on_bicycle`, `running`
- Low battery impact (~1% per day)
- Fires `onActivityChange` events with confidence levels

#### iOS
Uses a combination of techniques:
- **Stationary Geofence**: Creates a circular geofence around the current location when stationary
- **Significant Location Change API**: Monitors for significant location changes (500-1000m)
- **CMMotionActivityManager**: Detects motion activity types
- Exit from stationary geofence triggers transition to MOVING state

### Motion State Transitions

```
                    ┌──────────────┐
                    │   Initial    │
                    │    State     │
                    └──────┬───────┘
                           │
                    start() called
                           │
                           ▼
                    ┌──────────────┐
          ┌─────────│  STATIONARY  │◄─────────┐
          │         │    State     │          │
          │         └──────┬───────┘          │
          │                │                  │
          │      Motion Detected              │
          │      (auto or manual)             │
          │                │                  │
          │                ▼                  │
          │         ┌──────────────┐          │
          │         │    MOVING    │          │
          └────────►│    State     ├──────────┘
                    └──────────────┘
                    Device Stationary
                    (auto detected)
```

## Tracking Lifecycle

### 1. Initialization Phase

```dart
// Configure the plugin with desired settings
bg.BackgroundGeolocation.ready(bg.Config(
  desiredAccuracy: bg.Config.DESIRED_ACCURACY_HIGH,
  distanceFilter: 10.0,
  stopOnTerminate: false,
  startOnBoot: true
));
```

**What happens:**
- Plugin loads persisted configuration from native storage
- Applies configuration (only defaults on first install)
- Initializes native location managers
- Does NOT start tracking yet

### 2. Starting Tracking

```dart
bg.BackgroundGeolocation.start();
```

**What happens:**
- Plugin enters STATIONARY state
- Fetches initial location
- Turns OFF location services
- **iOS**: Creates stationary geofence at current location
- **Android**: Begins monitoring Activity Recognition
- Waits for motion to be detected

### 3. Motion Detection → MOVING State

**Automatic Transition:**
- Motion sensors detect movement
- `onMotionChange` event fires with the location where motion was detected
- Location services turn ON
- Begins recording locations according to `distanceFilter`

**Manual Transition:**
```dart
// Immediately engage location tracking (e.g., "Start Workout" button)
bg.BackgroundGeolocation.changePace(true);
```

### 4. Active Tracking (MOVING State)

- GPS actively recording locations
- Locations recorded when device moves >= `distanceFilter` meters
- **Elastic Distance Filtering**: `distanceFilter` automatically scales with speed
  - Biking speed: filter increases
  - Highway speed: filter increases significantly
  - Walking speed: filter remains low
- Each location fires `onLocation` event
- Locations stored in local SQLite database
- Optional automatic HTTP sync to server

### 5. Detecting Stationary → STATIONARY State

**Automatic Detection:**
- No significant movement for a period
- Plugin automatically transitions back to STATIONARY
- `onMotionChange` event fires (with `isMoving: false`)
- Location services turn OFF
- Returns to low-power motion monitoring

**Manual Transition:**
```dart
// Force stationary state (e.g., "End Workout" button)
bg.BackgroundGeolocation.changePace(false);
```

### 6. Stopping Tracking

```dart
bg.BackgroundGeolocation.stop();
```

**What happens:**
- Disables all location tracking
- Stops motion detection
- Plugin becomes completely idle
- Persists current state to storage

## Key Components

### 1. BackgroundGeolocation API (Dart)

Main public API with key methods:

- **`ready(Config)`**: Initialize plugin with configuration
- **`start()`**: Enable tracking (enters STATIONARY state)
- **`stop()`**: Disable all tracking
- **`changePace(bool)`**: Manually toggle MOVING/STATIONARY
- **`getCurrentPosition()`**: One-time location fetch
- **`addGeofence(Geofence)`**: Add geofence for monitoring

### 2. Event System

The plugin is **event-driven** with 14 different event types:

#### Primary Events:
- **`onLocation`**: Fired when location is recorded
- **`onMotionChange`**: Fired when transitioning between MOVING ↔ STATIONARY
- **`onActivityChange`**: Fired when device activity changes (walking, driving, etc.)
- **`onProviderChange`**: Fired when location services settings change
- **`onGeofence`**: Fired on geofence ENTER/EXIT/DWELL transitions

#### Supporting Events:
- **`onHttp`**: HTTP response from server sync
- **`onHeartbeat`**: Periodic events (configurable interval)
- **`onSchedule`**: Schedule-based start/stop events
- **`onConnectivityChange`**: Network connectivity changes
- **`onPowerSaveChange`**: Device power-save mode changes
- **`onEnabledChange`**: Plugin enabled/disabled events
- **`onGeofencesChange`**: Active geofence list changes
- **`onNotificationAction`**: (Android) Notification button clicks
- **`onAuthorization`**: Authorization token events

### 3. Configuration System (Config)

Extensive configuration options control every aspect of behavior:

#### Location Tracking:
- `desiredAccuracy`: GPS accuracy level (NAVIGATION, HIGH, MEDIUM, LOW, etc.)
- `distanceFilter`: Minimum meters before recording location (elastically adjusted)
- `stationaryRadius`: (iOS) Radius of stationary geofence
- `disableElasticity`: Disable automatic distance filter scaling
- `useSignificantChangesOnly`: Periodic-only tracking (500-1000m intervals)

#### Lifecycle:
- `stopOnTerminate`: Stop tracking when app terminates
- `startOnBoot`: Auto-start tracking on device boot
- `stopAfterElapsedMinutes`: Auto-stop after specified duration

#### HTTP Sync:
- `url`: Server endpoint for location upload
- `autoSync`: Automatically POST locations to server
- `batchSync`: Send all locations in single request
- `headers`: Custom HTTP headers
- `params`: Extra parameters with each request

#### Power Management:
- `heartbeatInterval`: Periodic event interval (seconds)
- `preventSuspend`: (iOS) Prevent app suspension
- `locationTimeout`: Timeout for location requests

### 4. Location Storage

- All locations persisted to **native SQLite database**
- Survives app termination and device reboot
- Accessible via: `locations`, `count`, `destroyLocations()`, etc.
- Automatic sync to server (if configured)
- Records metadata: timestamp, accuracy, speed, heading, battery level, etc.

### 5. Geofencing System

- Store unlimited geofences in database
- Plugin activates only nearby geofences:
  - **iOS**: Max 20 simultaneously monitored
  - **Android**: Max 100 simultaneously monitored
- Automatic proximity queries using `geofenceProximityRadius`
- Events: ENTER, EXIT, DWELL (loitering)
- Can track in geofence-only mode: `startGeofences()`

## Platform-Specific Implementations

### iOS Implementation

**Core Technologies:**
- `CoreLocation` framework for location services
- `CMMotionActivityManager` for motion detection
- Stationary geofence for motion-state changes
- Background execution modes

**Key Files:**
- `TSBackgroundGeolocationPlugin.m`: Main plugin coordinator
- `TSLocationStreamHandler.m`: Location event handling
- Various stream handlers for each event type

**iOS Specifics:**
- Requires `UIBackgroundMode: "location"` permission
- `stationaryRadius` typically requires ~200m movement to exit
- `preventSuspend` keeps app alive during stationary state
- Superior battery management compared to continuous GPS

### Android Implementation

**Core Technologies:**
- Google Play Services `FusedLocationProviderClient`
- Activity Recognition API for motion detection
- Foreground Service for reliable background execution
- JobScheduler for periodic tasks

**Key Files:**
- `FLTBackgroundGeolocationPlugin.java`: Flutter plugin registration
- `BackgroundGeolocationModule.java`: Core plugin logic
- Various `StreamHandler` classes for event channels

**Android Specifics:**
- Requires foreground notification when tracking
- Activity Recognition provides activity confidence levels
- More aggressive in maintaining tracking (less auto-suspend)
- Requires multiple permissions (location, activity recognition)

## Event System

### How Events Flow

```
Native Platform Detection
         │
         ▼
Native Event Broadcast
         │
         ▼
Platform Channel Bridge (EventChannel)
         │
         ▼
Flutter EventChannel Stream
         │
         ▼
Dart Event Listener Callback
         │
         ▼
Your Application Code
```

### Event Registration

Events use Dart streams internally:

```dart
// Event registration creates a stream subscription
bg.BackgroundGeolocation.onLocation((bg.Location location) {
  // Your handler code
  print('[onLocation] ${location.coords.latitude}, ${location.coords.longitude}');
});
```

**Behind the scenes:**
1. First subscription creates `EventChannel` stream
2. Stream broadcasts events from native platform
3. Multiple listeners can subscribe to same event
4. Subscriptions are managed automatically
5. Use `removeListeners()` to unsubscribe

## Location Sampling and Filtering

### Elastic Distance Filtering

The plugin implements intelligent **elastic distance filtering**:

**Algorithm:**
```
rounded_speed = round(current_speed, 5)  // Round to nearest 5 m/s
multiplier = rounded_speed / 5
adjusted_filter = multiplier × distanceFilter × elasticityMultiplier
```

**Examples:**

1. **Walking (2 m/s)** with `distanceFilter: 30`:
   - Adjusted filter: ~30 meters
   - Records locations frequently

2. **Biking (7.7 m/s)** with `distanceFilter: 30`:
   - Rounded speed: 10 m/s
   - Multiplier: 2
   - Adjusted filter: 60 meters
   - Reduces sampling frequency

3. **Highway (27 m/s)** with `distanceFilter: 50`:
   - Rounded speed: 30 m/s
   - Multiplier: 6
   - Adjusted filter: 300 meters
   - Much lower sampling frequency

**Benefits:**
- Automatically optimizes for speed
- Reduces battery drain at high speeds
- Maintains detail at low speeds
- Can be disabled with `disableElasticity: true`

### Location Sampling

When requesting a location (via `getCurrentPosition` or motion change):

1. Plugin requests **multiple samples** (configurable via `samples` parameter)
2. Samples arrive over several seconds
3. Each sample fires `onLocation` event with `sample: true`
4. Best (most accurate) location is selected
5. Final location fires with `sample: false` and is persisted

**Why multiple samples?**
- GPS accuracy improves over first few seconds
- Ensures highest quality location
- Allows progressive UI updates (e.g., map marker refinement)

## Power Management

### Battery Conservation Strategies

1. **Motion-Based Tracking**
   - GPS only active during movement
   - Low-power sensors used when stationary
   - Typical power usage: 1-3% per hour while moving

2. **Elastic Distance Filtering**
   - Fewer samples at high speeds
   - Reduces GPS usage without losing track fidelity

3. **Significant Changes Mode** (`useSignificantChangesOnly`)
   - Minimum power mode
   - Location every 500-1000 meters
   - No continuous GPS or foreground service
   - Typical power usage: <1% per day

4. **Configurable Accuracy**
   - `DESIRED_ACCURACY_HIGH`: GPS + WiFi + Cellular (highest power)
   - `DESIRED_ACCURACY_MEDIUM`: WiFi + Cellular (medium power)
   - `DESIRED_ACCURACY_LOW`: WiFi only (low power)
   - `DESIRED_ACCURACY_VERY_LOW`: Cellular only (lowest power)

5. **iOS Preventive Suspension**
   - By default, iOS aggressively suspends apps
   - `preventSuspend: true` keeps app active
   - Necessary for `heartbeatInterval` events
   - Trade-off: slight battery increase for reliability

### Monitoring Battery Impact

Location objects include `battery` field:
- Current battery level (0.0 - 1.0)
- Is charging state
- Allows tracking power consumption
- Can adjust behavior based on battery level

## Best Practices

### 1. Configuration

```dart
// For most applications - balanced approach
bg.BackgroundGeolocation.ready(bg.Config(
  desiredAccuracy: bg.Config.DESIRED_ACCURACY_HIGH,
  distanceFilter: 10.0,              // Meters
  stopOnTerminate: false,             // Continue after app close
  startOnBoot: true,                  // Auto-start on device boot
  debug: false,                       // Disable for production
  logLevel: bg.Config.LOG_LEVEL_VERBOSE  // Detailed logs for dev
));
```

### 2. Event Handling

```dart
// Always register event listeners BEFORE calling ready()
void initState() {
  super.initState();
  
  // 1. Register event listeners
  bg.BackgroundGeolocation.onLocation(_onLocation);
  bg.BackgroundGeolocation.onMotionChange(_onMotionChange);
  bg.BackgroundGeolocation.onActivityChange(_onActivityChange);
  
  // 2. Configure the plugin
  bg.BackgroundGeolocation.ready(config).then((state) {
    // 3. Start tracking if not already enabled
    if (!state.enabled) {
      bg.BackgroundGeolocation.start();
    }
  });
}
```

### 3. Testing

- Use `debug: true` and `logLevel: LOG_LEVEL_VERBOSE` during development
- Monitor native logs:
  - **iOS**: Xcode Console
  - **Android**: `adb logcat` or Android Studio Logcat
- Test motion state changes by actually moving (walking, driving)
- Emulator testing is limited - motion detection requires physical device

### 4. Production Checklist

- [ ] Set `debug: false`
- [ ] Reduce `logLevel` (ERROR or WARNING)
- [ ] Test on physical devices (not emulator)
- [ ] Verify background permissions
- [ ] Test app termination scenarios
- [ ] Verify server sync (if using HTTP)
- [ ] Test battery impact over extended periods
- [ ] Handle all error cases in event listeners
- [ ] Test with various motion scenarios (walking, driving, stationary)

## Troubleshooting

### Common Issues

**1. Tracking doesn't start**
- Ensure location permissions are granted
- Call `ready()` before `start()`
- Check `State.enabled` after `ready()`
- Verify background location permission (iOS: "Always", Android: "Allow all the time")

**2. Motion state doesn't change**
- Motion detection requires actual physical movement
- iOS: May require 200+ meters to exit stationary
- Android: Activity Recognition may take 1-2 minutes to detect changes
- Try manual override: `changePace(true)`

**3. Locations not recording**
- Check `distanceFilter` - may be set too high
- Verify GPS is available (go outdoors)
- Check `desiredAccuracy` settings
- Review logs for errors

**4. High battery usage**
- Reduce `desiredAccuracy` if possible
- Increase `distanceFilter`
- Consider `useSignificantChangesOnly: true`
- Review `preventSuspend` usage on iOS
- Check for infinite location requests

**5. App doesn't track in background**
- **iOS**: Verify `UIBackgroundMode: "location"` in Info.plist
- **Android**: Ensure foreground notification is configured
- Check `stopOnTerminate` setting
- Verify background location permissions

## Summary

The `flutter_background_geolocation` plugin provides enterprise-grade location tracking through:

1. **Intelligent Motion Detection**: Automatically manages GPS based on device movement
2. **Two-State System**: STATIONARY (GPS off) ↔ MOVING (GPS on)
3. **Battery Optimization**: Multiple strategies to minimize power consumption
4. **Rich Event System**: 14 events for complete tracking lifecycle visibility
5. **Cross-Platform**: Consistent API across iOS and Android with platform-specific optimizations
6. **Flexible Configuration**: Extensive options to tune behavior for any use case
7. **Persistence**: SQLite storage survives app termination
8. **HTTP Integration**: Automatic server synchronization built-in

By understanding these architectural concepts, you can effectively implement sophisticated location tracking features while maintaining excellent battery life and user experience.

## Additional Resources

- [API Documentation](https://pub.dartlang.org/documentation/flutter_background_geolocation/latest/flt_background_geolocation/flt_background_geolocation-library.html)
- [Philosophy of Operation](https://github.com/transistorsoft/flutter_background_geolocation/wiki/Philosophy-of-Operation)
- [Debugging Guide](https://github.com/transistorsoft/flutter_background_geolocation/wiki/Debugging)
- [iOS Setup Guide](./help/INSTALL-IOS.md)
- [Android Setup Guide](./help/INSTALL-ANDROID.md)
- [Example Application](./example)
