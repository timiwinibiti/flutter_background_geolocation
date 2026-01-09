# How Background Location Tracking Works - Quick Guide

A simplified explanation of how `flutter_background_geolocation` works under the hood.

## The Problem

Traditional location tracking keeps GPS running continuously, which **drains the battery rapidly**. If you've ever used a navigation app for an extended period, you know how quickly it can deplete your battery.

## The Solution: Motion-Aware Tracking

This plugin uses a smart approach: **only turn on GPS when the device is actually moving**.

### The Magic: Two States

The plugin operates like a light switch with two positions:

#### 🔴 STATIONARY State (GPS OFF)
- The device isn't moving
- GPS is **turned off** to save battery
- Motion sensors (accelerometer, gyroscope) are listening
- Waiting for movement to be detected

#### 🟢 MOVING State (GPS ON)  
- Motion detected! Device is moving
- GPS **turns on** and actively tracks location
- Records your path as you move
- Automatically goes back to STATIONARY when you stop

### Visual Representation

```
You're sitting at home
         ↓
    [STATIONARY]
    GPS: OFF ⚫
    Motion sensors: ON ✓
         ↓
    (You start walking)
         ↓
    Motion Detected!
         ↓
    [MOVING]
    GPS: ON 🟢
    Recording locations...
         ↓
    (You arrive and stop)
         ↓
    No movement detected
         ↓
    [STATIONARY]
    GPS: OFF ⚫
    Motion sensors: ON ✓
```

## How Motion is Detected

### Android
- Uses **Activity Recognition API**
- Continuously analyzes motion patterns
- Knows if you're walking, driving, biking, or still
- Very low battery impact (~1% per day)

### iOS
- Creates an invisible "**geofence**" around your location when stopped
- When you move outside this fence → GPS turns on
- Uses **motion activity** sensors for classification
- Also very battery efficient

## Real-World Example: Food Delivery App

Imagine a food delivery driver using your app:

**Morning: Driver is at home**
- App state: `STATIONARY`
- GPS: OFF
- Battery drain: Minimal

**9:00 AM: Driver starts their car**
- Motion sensors detect movement
- App automatically switches to `MOVING`
- GPS turns ON
- Starts recording delivery route

**9:15 AM: Arrives at restaurant**
- Stops moving for pickup
- App detects stationary state
- Automatically switches to `STATIONARY`
- GPS turns OFF while waiting

**9:25 AM: Leaves with food**
- Movement detected again
- Switches to `MOVING`
- GPS ON - tracking delivery route

**9:40 AM: Delivers food, waits for next order**
- Becomes stationary
- GPS OFF

**Result:** GPS only runs during actual movement. Instead of 8+ hours of continuous GPS (battery killer), you only use GPS during the actual driving time (maybe 4-5 hours), saving massive battery.

## Smart Features

### 1. Elastic Distance Filtering

The plugin is smart about **when** to record locations:

- **Walking (slow)**: Records location every ~30 meters
- **Biking (medium)**: Records every ~60 meters  
- **Driving (fast)**: Records every ~300 meters

**Why?** At highway speeds, you don't need a point every 30 meters - that's excessive and wastes battery. The plugin automatically adjusts the sampling rate based on your speed.

### 2. Location Sampling

When you request a location:
- Plugin takes **multiple samples** over a few seconds
- GPS accuracy improves in the first few seconds
- Returns the **best/most accurate** location
- Your app can show progressive updates (smooth map animations)

### 3. Automatic Server Sync

If configured:
- Locations are saved to local database
- Automatically uploaded to your server
- Retry on network failure
- Batch mode available (send many at once vs one-by-one)

## What You Control

### Configuration Example

```dart
bg.BackgroundGeolocation.ready(bg.Config(
  // How accurate? (HIGH uses GPS, MEDIUM uses WiFi/Cell)
  desiredAccuracy: bg.Config.DESIRED_ACCURACY_HIGH,
  
  // Minimum meters between location recordings
  distanceFilter: 10.0,
  
  // Keep tracking even after app is closed?
  stopOnTerminate: false,
  
  // Auto-start tracking after device reboot?
  startOnBoot: true,
  
  // Your server URL (optional)
  url: 'https://your-server.com/locations',
  
  // Automatically send to server?
  autoSync: true
));
```

### Event Listeners

Know exactly what's happening:

```dart
// Fired whenever location is recorded
bg.BackgroundGeolocation.onLocation((location) {
  print('New location: ${location.coords.latitude}, ${location.coords.longitude}');
});

// Fired when device starts/stops moving
bg.BackgroundGeolocation.onMotionChange((location) {
  if (location.isMoving) {
    print('Started moving!');
  } else {
    print('Stopped moving!');
  }
});

// Fired when activity changes (walking → driving)
bg.BackgroundGeolocation.onActivityChange((event) {
  print('Activity: ${event.activity}'); // "in_vehicle", "on_foot", etc.
});
```

## The Complete Flow

### 1. Setup (One Time)
```dart
// Register your event listeners
bg.BackgroundGeolocation.onLocation(_onLocation);
bg.BackgroundGeolocation.onMotionChange(_onMotionChange);

// Configure the plugin
await bg.BackgroundGeolocation.ready(config);
```

### 2. Start Tracking
```dart
// Starts in STATIONARY state
await bg.BackgroundGeolocation.start();
```

### 3. Automatic Operation
- Plugin now works automatically
- Switches between STATIONARY ↔ MOVING based on motion
- Records locations when moving
- Syncs to server (if configured)
- All happens in the background, even when app is closed

### 4. Manual Override (Optional)
```dart
// Force tracking ON (e.g., "Start Workout" button)
bg.BackgroundGeolocation.changePace(true);

// Force tracking OFF (e.g., "End Workout" button)  
bg.BackgroundGeolocation.changePace(false);
```

### 5. Stop Tracking
```dart
await bg.BackgroundGeolocation.stop();
```

## Battery Impact

**With this plugin (motion-aware):**
- Stationary: ~0% per hour (GPS off)
- Moving: ~1-3% per hour (GPS on, with smart filtering)
- Average: Depends on how much you actually move

**Without this approach (continuous GPS):**
- ~8-15% per hour constantly
- Battery dead in 6-8 hours

**Extreme battery saving mode:**
```dart
// Use "Significant Changes Only" mode
bg.BackgroundGeolocation.ready(bg.Config(
  useSignificantChangesOnly: true  // Only track every 500-1000 meters
));
```
This mode uses almost no battery (< 1% per day) but gives very coarse tracking.

## Common Use Cases

### 1. Delivery/Courier Apps
- Track drivers during shifts
- Automatic route recording
- Proof of delivery locations
- Fleet management

### 2. Fitness/Running Apps  
- Record workout routes
- Track distance traveled
- Calculate pace/speed
- Map visualization

### 3. Field Service Apps
- Employee time tracking
- Service area coverage
- Mileage tracking
- Location-based clock in/out

### 4. Family Safety Apps
- Real-time location sharing
- Geofence alerts (arrived home, left school)
- Location history
- Emergency location

### 5. Asset Tracking
- Vehicle tracking
- Equipment monitoring
- Theft recovery
- Usage analytics

## Key Takeaways

1. **Motion Detection**: The core innovation - GPS only runs when you're actually moving
2. **Two States**: STATIONARY (GPS off) ↔ MOVING (GPS on)
3. **Automatic**: Works in the background without user intervention
4. **Battery Efficient**: Can run all day with minimal battery impact
5. **Configurable**: Tune every aspect for your specific use case
6. **Event-Driven**: Know exactly what's happening via event callbacks
7. **Persistent**: Survives app termination and device reboots
8. **Cross-Platform**: Same API for iOS and Android

## Next Steps

- Read the full [ARCHITECTURE.md](./ARCHITECTURE.md) for deep technical details
- Check out the [Example App](./example) to see it in action
- Review [API Documentation](https://pub.dartlang.org/documentation/flutter_background_geolocation/latest/)
- Follow setup guides:
  - [iOS Setup](./help/INSTALL-IOS.md)
  - [Android Setup](./help/INSTALL-ANDROID.md)

## Quick Comparison

| Feature | Continuous GPS | This Plugin (Motion-Aware) |
|---------|---------------|----------------------------|
| Battery drain | 8-15% per hour | 0-3% per hour (avg) |
| Works in background | Sometimes | Yes, reliably |
| Accuracy | High (always) | High when needed |
| Power when stationary | Still draining | Minimal drain |
| Complexity | Simple | Managed for you |
| Production ready | For short sessions | For all-day tracking |

## Quick Reference Card

### Three-Line Setup
```dart
bg.BackgroundGeolocation.onLocation((location) => print(location));
await bg.BackgroundGeolocation.ready(bg.Config(desiredAccuracy: bg.Config.DESIRED_ACCURACY_HIGH, distanceFilter: 10.0));
await bg.BackgroundGeolocation.start();
```

### Essential Methods
| Method | Purpose |
|--------|---------|
| `ready(config)` | Initialize with configuration (call once) |
| `start()` | Begin tracking (starts in STATIONARY state) |
| `stop()` | Stop all tracking |
| `changePace(bool)` | Manually force MOVING/STATIONARY state |
| `getCurrentPosition()` | Get location right now (one-time) |

### Essential Events
| Event | When it Fires |
|-------|---------------|
| `onLocation` | New location recorded |
| `onMotionChange` | Switched between MOVING ↔ STATIONARY |
| `onActivityChange` | Activity detected (walking, driving, etc.) |
| `onProviderChange` | Location permissions/settings changed |
| `onGeofence` | Geofence boundary crossed |

### Key Config Options
| Option | Default | Purpose |
|--------|---------|---------|
| `desiredAccuracy` | HIGH | How accurate (affects battery) |
| `distanceFilter` | 10m | Min meters between locations |
| `stopOnTerminate` | true | Keep running after app close? |
| `startOnBoot` | false | Auto-start on device reboot? |
| `url` | null | Server endpoint for auto-sync |
| `autoSync` | true | Automatically upload to server? |

### Troubleshooting Checklist
- [ ] Location permissions granted? (iOS: "Always", Android: "All the time")
- [ ] Called `ready()` before `start()`?
- [ ] Testing on physical device? (Emulator has limitations)
- [ ] Actually moving? (Motion detection needs real movement)
- [ ] GPS available? (Go outdoors if testing)
- [ ] Check logs with `debug: true` and `logLevel: LOG_LEVEL_VERBOSE`

---

**Still have questions?** The plugin has been field-tested daily since 2013 and is used in thousands of production applications. Check the repository's Wiki and Issues for more information.
