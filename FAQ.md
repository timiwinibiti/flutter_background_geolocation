# Background Geolocation - Frequently Asked Questions

Common questions about how `flutter_background_geolocation` works and how to use it effectively.

## General Understanding

### Q: What makes this plugin different from other location plugins?

**A:** Most location plugins keep GPS running continuously, which drains battery quickly. This plugin uses **motion detection** to intelligently turn GPS on only when the device is actually moving. When stationary, it turns GPS off and uses low-power motion sensors (accelerometer, gyroscope) to detect when movement starts again.

**Result:** You can track all day long with minimal battery impact (~1-3% per hour when moving vs. 8-15% per hour with continuous GPS).

### Q: How does the plugin know when I'm moving?

**A:** 
- **Android:** Uses Google's Activity Recognition API which analyzes motion sensor patterns to classify activity (still, walking, driving, etc.)
- **iOS:** Creates an invisible circular "geofence" around your location when stationary. When you move outside this boundary, it detects motion and turns GPS back on

Both methods use very little battery (~1% per day) compared to continuous GPS.

### Q: What are the two states and when does the plugin switch between them?

**A:** The plugin operates in two states:

1. **STATIONARY State:**
   - GPS is OFF (saving battery)
   - Motion sensors are ON (listening for movement)
   - Fires when you call `start()` initially or when you stop moving

2. **MOVING State:**
   - GPS is ON (actively tracking)
   - Recording locations based on your `distanceFilter` setting
   - Fires when motion is detected

The plugin automatically switches between these states based on detected motion. You'll receive an `onMotionChange` event each time it switches.

## Setup & Configuration

### Q: What's the minimum code needed to start tracking?

**A:** Just three essential parts:

```dart
// 1. Listen to location events
bg.BackgroundGeolocation.onLocation((location) {
  print('${location.coords.latitude}, ${location.coords.longitude}');
});

// 2. Configure and initialize (call once)
await bg.BackgroundGeolocation.ready(bg.Config(
  desiredAccuracy: bg.Config.DESIRED_ACCURACY_HIGH,
  distanceFilter: 10.0
));

// 3. Start tracking
await bg.BackgroundGeolocation.start();
```

### Q: When should I call `ready()`?

**A:** Call `ready()` **once and only once** when your app starts, typically in `initState()`. Do NOT hide it behind a button or conditional logic. The plugin needs to be initialized even if you're not actively tracking, especially on iOS where the OS may launch your app in the background.

### Q: What's the difference between `ready()` and `start()`?

**A:**
- **`ready(config)`**: Initializes the plugin with your configuration. Does NOT start tracking. Safe to call on every app launch.
- **`start()`**: Actually begins location tracking. Only call this when you want to start tracking.

Think of `ready()` as "configure the engine" and `start()` as "turn on the engine."

### Q: Do I need to call `ready()` every time the app starts?

**A:** Yes! Always call `ready()` when your app initializes. The plugin automatically remembers your configuration from the last session, so you only need to provide a full `Config` the first time. After that, `ready()` just signals "I'm ready to use the plugin now."

## Location Tracking

### Q: Why isn't my app tracking locations?

**A:** Check these common issues:

1. **Location permissions not granted**
   - iOS: Need "Always" permission
   - Android: Need "Allow all the time" permission

2. **Forgot to call `start()`**
   - `ready()` alone doesn't start tracking
   
3. **Not actually moving**
   - The plugin won't record locations if you're stationary
   - Try `changePace(true)` to force MOVING state
   
4. **Testing indoors**
   - GPS needs clear sky view
   - Try testing outdoors

5. **Distance filter too large**
   - If `distanceFilter: 1000`, you need to move 1km before next location
   - Try lower values like `10.0` for testing

### Q: How often does the plugin record locations?

**A:** It depends on your `distanceFilter` setting and your speed:

- **Base rate:** One location every `distanceFilter` meters (e.g., every 10 meters)
- **Elastic adjustment:** Automatically increases at higher speeds
  - Walking: ~every 10-30 meters
  - Biking: ~every 30-60 meters  
  - Driving: ~every 100-300 meters

This elastic adjustment reduces battery use at high speeds without losing track quality.

### Q: Can I get the location right now without waiting?

**A:** Yes! Use `getCurrentPosition()`:

```dart
Location location = await bg.BackgroundGeolocation.getCurrentPosition(
  timeout: 30,
  desiredAccuracy: 10,
  samples: 3
);
```

This gets a one-time location immediately. It doesn't require `start()` to be called first.

### Q: What's the difference between `onLocation` and `onMotionChange`?

**A:**
- **`onLocation`**: Fires for EVERY location recorded (could be hundreds per trip)
- **`onMotionChange`**: Fires only when transitioning between MOVING ↔ STATIONARY (twice per trip)

Use `onLocation` to track the user's path. Use `onMotionChange` to know when they start/stop moving.

### Q: What does `location.isMoving` mean?

**A:** It indicates whether the plugin is in MOVING or STATIONARY state when that location was recorded:
- `isMoving: true` = Plugin was in MOVING state (GPS was on)
- `isMoving: false` = Plugin was in STATIONARY state (usually the location where motion was detected)

### Q: Why do I get multiple locations with `sample: true`?

**A:** When the plugin requests a location (during motion change or `getCurrentPosition`), it takes multiple GPS samples over a few seconds. This is because:

1. GPS accuracy improves over the first few seconds
2. The plugin wants the most accurate location possible
3. You can use samples for progressive UI updates (e.g., refining a map marker)

The final, best location will have `sample: false` and is the one that gets persisted to the database.

## Battery & Performance

### Q: How much battery does this plugin use?

**A:** It depends on how much you're actually moving:

- **When stationary:** ~0% per hour (GPS is off)
- **When moving:** ~1-3% per hour (GPS is on)
- **Significant Changes Only mode:** <1% per day

For a typical use case (e.g., delivery driver moving 4 hours, stationary 4 hours in an 8-hour shift):
- This plugin: ~4-12% battery used
- Continuous GPS: ~64-120% (would need charging)

### Q: How can I reduce battery usage further?

**A:**

1. **Lower accuracy:**
   ```dart
   desiredAccuracy: bg.Config.DESIRED_ACCURACY_MEDIUM  // WiFi/Cell instead of GPS
   ```

2. **Increase distance filter:**
   ```dart
   distanceFilter: 50.0  // Only record every 50 meters instead of 10
   ```

3. **Use Significant Changes Only:**
   ```dart
   useSignificantChangesOnly: true  // Only track every 500-1000 meters
   ```

4. **Disable elastic filtering:**
   ```dart
   disableElasticity: true  // Prevent distance filter from auto-adjusting (may increase samples)
   ```

### Q: Does the plugin work when the app is closed?

**A:** Yes, if configured correctly:

```dart
bg.BackgroundGeolocation.ready(bg.Config(
  stopOnTerminate: false,  // Keep running after app is closed
  startOnBoot: true        // Auto-restart after device reboot
));
```

**Important:** 
- You need appropriate background location permissions
- iOS requires "Always" permission
- Android requires "Allow all the time" permission

## Platform-Specific

### Q: Why does iOS require 200+ meters to detect I've started moving?

**A:** This is an iOS limitation, not a plugin limitation. iOS uses a "stationary geofence" to detect motion. The native iOS API typically requires ~200 meters of movement before triggering the geofence exit event.

**Workaround:** Use `changePace(true)` to manually force MOVING state if you need immediate tracking (e.g., "Start Workout" button).

### Q: Why does Android show a persistent notification?

**A:** Android requires a foreground notification when an app uses location in the background. This is an Android OS requirement, not a plugin choice. You can customize the notification:

```dart
bg.BackgroundGeolocation.ready(bg.Config(
  notification: bg.Notification(
    title: "Tracking your delivery",
    text: "Route recording in progress",
    color: "#ff0000"
  )
));
```

### Q: Can I track on iOS without the "Always" permission?

**A:** No, not for true background tracking. iOS requires "Always" permission for background location. 

If Apple rejected your app for this, you can:
1. Use `useSignificantChangesOnly: true` (doesn't require "Always" permission but very coarse tracking)
2. Explain to Apple why your app needs background location
3. Consider using only foreground tracking (when app is open)

## Data & HTTP Sync

### Q: Where are locations stored?

**A:** In a native SQLite database on the device. Locations persist even if:
- App is closed
- Device is rebooted
- App is uninstalled and reinstalled (unless you clear app data)

Access them with: `await bg.BackgroundGeolocation.locations`

### Q: How do I send locations to my server?

**A:** Configure HTTP sync:

```dart
bg.BackgroundGeolocation.ready(bg.Config(
  url: 'https://your-server.com/locations',
  autoSync: true,
  batchSync: false,  // false = one request per location, true = batch multiple
  headers: {
    'Authorization': 'Bearer your-token'
  }
));
```

The plugin will automatically POST locations to your server and delete them from the database upon successful response (200 OK).

### Q: What if my server is down or there's no network?

**A:** Locations remain in the local database and the plugin will retry later when:
- Network becomes available (via `onConnectivityChange` event)
- You manually call `sync()`
- More locations are recorded (triggers retry)

### Q: Can I upload locations manually instead of automatically?

**A:** Yes:

```dart
bg.BackgroundGeolocation.ready(bg.Config(
  url: 'https://your-server.com/locations',
  autoSync: false  // Don't automatically sync
));

// Later, when you want to upload:
List locations = await bg.BackgroundGeolocation.sync();
```

## Geofencing

### Q: How do geofences work with this plugin?

**A:** The plugin can monitor unlimited geofences, but platforms have limitations:
- iOS: Max 20 simultaneously monitored
- Android: Max 100 simultaneously monitored

**Solution:** The plugin uses proximity queries. It:
1. Stores all your geofences in the database
2. Activates only the nearest N geofences based on your current location
3. Automatically swaps them as you move

This allows thousands of geofences with no performance impact.

### Q: Can I track only geofences without location tracking?

**A:** Yes! Use geofence-only mode:

```dart
// Add your geofences
await bg.BackgroundGeolocation.addGeofence(bg.Geofence(
  identifier: 'Home',
  latitude: 37.234,
  longitude: -122.345,
  radius: 200,
  notifyOnEntry: true,
  notifyOnExit: true
));

// Start geofence-only tracking (no location tracking)
await bg.BackgroundGeolocation.startGeofences();

// Listen for events
bg.BackgroundGeolocation.onGeofence((event) {
  print('Geofence ${event.action}: ${event.identifier}');
});
```

## Testing & Debugging

### Q: How do I debug location tracking issues?

**A:**

1. **Enable verbose logging:**
   ```dart
   bg.BackgroundGeolocation.ready(bg.Config(
     debug: true,
     logLevel: bg.Config.LOG_LEVEL_VERBOSE
   ));
   ```

2. **Check native logs:**
   - iOS: Xcode Console
   - Android: `adb logcat` or Android Studio Logcat
   - Filter for "TSLocationManager" (iOS) or "BackgroundGeolocation" (Android)

3. **Verify state:**
   ```dart
   bg.State state = await bg.BackgroundGeolocation.state;
   print('Enabled: ${state.enabled}');
   print('isMoving: ${state.isMoving}');
   print('trackingMode: ${state.trackingMode}');
   ```

### Q: Can I test in an emulator/simulator?

**A:** Limited. You can test basic functionality but:
- Motion detection won't work (no motion sensors)
- GPS simulation is basic
- Won't reflect real-world performance

**Recommendation:** Always test on physical devices for motion detection and battery testing.

### Q: Why isn't motion detection working?

**A:**

1. **Not actually moving:** Motion detection requires real physical movement (walking, driving)
2. **Takes time:** Android Activity Recognition may need 1-2 minutes to detect motion
3. **iOS stationary radius:** May need 200+ meters of movement
4. **Testing indoors:** May have poor GPS signal

**Quick test:** Use `changePace(true)` to manually force MOVING state and verify location tracking works.

## Advanced

### Q: What's `preventSuspend` on iOS?

**A:** By default, iOS suspends apps aggressively to save battery. Setting `preventSuspend: true` keeps your app alive:

```dart
bg.BackgroundGeolocation.ready(bg.Config(
  preventSuspend: true  // iOS won't suspend the app
));
```

**Use cases:**
- Required for `heartbeatInterval` events to fire while stationary
- Needed for apps that must respond immediately to location changes

**Trade-off:** Slightly higher battery usage in exchange for guaranteed responsiveness.

### Q: What is `elasticityMultiplier`?

**A:** Controls how aggressively the `distanceFilter` scales with speed:

```dart
elasticityMultiplier: 2.0  // More aggressive (doubles the scaling)
```

- Higher value = fewer locations at high speeds (more battery saving)
- Lower value = more locations at high speeds (less battery saving, more detail)
- `0` = same as `disableElasticity: true`

### Q: What's the difference between `stop()` and `stopSchedule()`?

**A:**
- **`stop()`**: Completely stops all location tracking immediately
- **`stopSchedule()`**: Only stops the schedule timer but doesn't stop tracking if currently enabled

If you're using a schedule (auto-start/stop at specific times), you need to:
```dart
await bg.BackgroundGeolocation.stopSchedule();  // Stop the schedule
if (state.enabled) {
  await bg.BackgroundGeolocation.stop();  // Also stop tracking
}
```

### Q: How do I implement "Start Workout" / "End Workout" functionality?

**A:** Use `changePace()`:

```dart
// "Start Workout" button pressed
void startWorkout() async {
  await bg.BackgroundGeolocation.changePace(true);  // Force MOVING state
  print('Started tracking workout');
}

// "End Workout" button pressed  
void endWorkout() async {
  await bg.BackgroundGeolocation.changePace(false);  // Force STATIONARY state
  // Or completely stop: await bg.BackgroundGeolocation.stop();
  print('Stopped tracking workout');
}
```

### Q: Can I customize what data is included with each location?

**A:** Yes, use `extras`:

```dart
// When recording locations automatically:
bg.BackgroundGeolocation.ready(bg.Config(
  extras: {
    'user_id': 123,
    'session_id': 'abc-def'
  }
));

// Or for a one-time location:
Location location = await bg.BackgroundGeolocation.getCurrentPosition(
  extras: {
    'event_type': 'checkin',
    'location_name': 'Coffee Shop'
  }
);
```

This metadata is stored with the location and sent to your server.

## Troubleshooting

### Q: My app keeps stopping tracking after a while

**A:** Check:

1. **Battery optimization:** Android may be killing your app
   - Go to Settings → Battery → Battery Optimization
   - Set your app to "Not Optimized"

2. **Background restrictions:** Android may restrict background activity
   - Settings → Apps → Your App → Battery → "Allow background activity"

3. **Config settings:**
   ```dart
   stopOnTerminate: false,  // Should be false
   stopAfterElapsedMinutes: 0  // Should be 0 (or don't set it)
   ```

### Q: Locations are inaccurate or jumping around

**A:**

1. **Poor GPS signal:** Go outdoors with clear sky view
2. **Low accuracy setting:** Try `DESIRED_ACCURACY_HIGH`
3. **Filter inaccurate locations:**
   ```dart
   bg.BackgroundGeolocation.onLocation((location) {
     if (location.coords.accuracy <= 50) {  // Only accept accuracy <= 50 meters
       // Use this location
     }
   });
   ```

### Q: Getting error "Location permission denied"

**A:**

1. **Check manifest/Info.plist:** Ensure location permissions are declared
2. **Request permissions:** The plugin attempts to request permissions, but you may need to use a separate permission plugin first
3. **Check system settings:** User may have denied permission in device settings

## Best Practices

### Q: What's the recommended configuration for most apps?

**A:**

```dart
bg.BackgroundGeolocation.ready(bg.Config(
  // Accuracy
  desiredAccuracy: bg.Config.DESIRED_ACCURACY_HIGH,
  distanceFilter: 10.0,
  
  // Lifecycle  
  stopOnTerminate: false,
  startOnBoot: true,
  
  // Logging (disable for production)
  debug: false,
  logLevel: bg.Config.LOG_LEVEL_ERROR,
  
  // HTTP (if needed)
  url: 'https://your-server.com/locations',
  autoSync: true,
  batchSync: false,
  
  // Headers (if using HTTP)
  headers: {
    'Authorization': 'Bearer ${yourAuthToken}'
  }
));
```

### Q: Should I use `reset: false`?

**A:** **NO**, unless you know exactly what you're doing. 

With `reset: false`, your `Config` in `ready()` is only applied on first install. Every subsequent launch ignores your config. This is useful for the demo app but confusing for normal apps.

**Rule of thumb:** Don't use `reset: false` unless you have a runtime settings screen and don't want `ready()` to overwrite user preferences.

### Q: How do I handle user logout?

**A:**

```dart
Future<void> logout() async {
  // Stop tracking
  await bg.BackgroundGeolocation.stop();
  
  // Clear stored locations (optional)
  await bg.BackgroundGeolocation.destroyLocations();
  
  // Remove geofences (optional)
  await bg.BackgroundGeolocation.removeGeofences();
  
  // Your logout logic...
}
```

## Getting Help

### Q: Where can I get more help?

**A:**

1. **Documentation:**
   - [HOW_IT_WORKS.md](./HOW_IT_WORKS.md) - Simple explanation
   - [ARCHITECTURE.md](./ARCHITECTURE.md) - Technical deep dive
   - [DIAGRAMS.md](./DIAGRAMS.md) - Visual flowcharts
   - [API Docs](https://pub.dartlang.org/documentation/flutter_background_geolocation/latest/)

2. **Setup Guides:**
   - [iOS Setup](./help/INSTALL-IOS.md)
   - [Android Setup](./help/INSTALL-ANDROID.md)

3. **Example Code:**
   - [Example App](./example)

4. **Community:**
   - [GitHub Issues](https://github.com/transistorsoft/flutter_background_geolocation/issues)
   - [Wiki](https://github.com/transistorsoft/flutter_background_geolocation/wiki)

### Q: Is there commercial support available?

**A:** Yes, this is a commercial plugin with full-time support. The plugin has been actively maintained and field-tested since 2013. For licensing and support inquiries, visit [transistorsoft.com](https://www.transistorsoft.com).

---

**Didn't find your question?** Check the [GitHub Issues](https://github.com/transistorsoft/flutter_background_geolocation/issues) or [Wiki](https://github.com/transistorsoft/flutter_background_geolocation/wiki) for more information.
