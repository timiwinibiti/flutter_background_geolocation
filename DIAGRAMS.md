# Background Location Tracking - Visual Diagrams

This document contains visual diagrams to help understand how the flutter_background_geolocation plugin works.

## State Machine Diagram

```mermaid
stateDiagram-v2
    [*] --> Stopped: Initial State
    Stopped --> Stationary: start()
    
    state Stationary {
        [*] --> GPSoff
        GPSoff --> MotionSensorsActive
        note right of MotionSensorsActive
            Android: Activity Recognition API
            iOS: Stationary Geofence
        end note
    }
    
    state Moving {
        [*] --> GPSon
        GPSon --> RecordingLocations
        RecordingLocations --> ApplyingDistanceFilter
        note right of RecordingLocations
            GPS actively tracking
            Recording based on distanceFilter
            Elastic filtering by speed
        end note
    }
    
    Stationary --> Moving: Motion Detected
    Moving --> Stationary: No Movement Detected
    Stationary --> Stopped: stop()
    Moving --> Stopped: stop()
```

## Motion Detection Flow

```mermaid
flowchart TD
    A[Device is Stationary] --> B{Motion Sensors Monitoring}
    B --> C{Movement Detected?}
    C -->|No| B
    C -->|Yes| D[Fire onMotionChange Event]
    D --> E[Switch to MOVING State]
    E --> F[Turn ON GPS]
    F --> G[Start Recording Locations]
    G --> H{Device Moving?}
    H -->|Yes| I[Record Location]
    I --> J{Distance Filter Met?}
    J -->|Yes| K[Fire onLocation Event]
    J -->|No| H
    K --> H
    H -->|No - Stationary| L[Fire onMotionChange Event]
    L --> M[Switch to STATIONARY State]
    M --> N[Turn OFF GPS]
    N --> B
```

## Plugin Architecture Layers

```mermaid
flowchart TB
    subgraph Flutter["Flutter Application Layer"]
        A1[Your Flutter App]
        A2[Event Listeners<br/>onLocation, onMotionChange, etc.]
        A3[API Calls<br/>start, stop, changePace, etc.]
    end
    
    subgraph Dart["Dart Plugin Layer"]
        B1[BackgroundGeolocation Class]
        B2[Config & State Models]
        B3[EventChannel Streams]
        B4[MethodChannel]
    end
    
    subgraph Platform["Platform Channel Layer"]
        C1[Method Handlers]
        C2[Event Stream Handlers]
    end
    
    subgraph iOS["iOS Native Layer"]
        D1[TSBackgroundGeolocationPlugin]
        D2[CoreLocation Framework]
        D3[CMMotionActivityManager]
        D4[Geofencing]
    end
    
    subgraph Android["Android Native Layer"]
        E1[FLTBackgroundGeolocationPlugin]
        E2[FusedLocationProvider]
        E3[Activity Recognition API]
        E4[Geofencing]
    end
    
    A1 --> A2
    A1 --> A3
    A2 --> B3
    A3 --> B1
    B1 --> B4
    B1 --> B2
    B3 --> C2
    B4 --> C1
    
    C1 --> D1
    C1 --> E1
    C2 --> D1
    C2 --> E1
    
    D1 --> D2
    D1 --> D3
    D1 --> D4
    
    E1 --> E2
    E1 --> E3
    E1 --> E4
```

## Event Flow Sequence

```mermaid
sequenceDiagram
    participant App as Flutter App
    participant Plugin as BackgroundGeolocation
    participant Native as Native Platform
    participant Sensors as Motion Sensors
    participant GPS as GPS Hardware
    
    App->>Plugin: ready(config)
    Plugin->>Native: Configure
    Native-->>Plugin: State
    Plugin-->>App: State
    
    App->>Plugin: start()
    Plugin->>Native: Start tracking
    Native->>Sensors: Enable motion detection
    Native->>GPS: Get initial location
    GPS-->>Native: Location
    Native-->>Plugin: Location event
    Plugin-->>App: onLocation(location)
    Native->>GPS: Turn OFF GPS
    
    Note over Native,Sensors: STATIONARY State
    
    Sensors->>Native: Motion detected!
    Native-->>Plugin: Motion change event
    Plugin-->>App: onMotionChange(isMoving: true)
    Native->>GPS: Turn ON GPS
    
    Note over Native,GPS: MOVING State
    
    loop While Moving
        GPS->>Native: Location update
        Native->>Native: Apply distance filter
        Native-->>Plugin: Location event
        Plugin-->>App: onLocation(location)
    end
    
    Sensors->>Native: No movement
    Native-->>Plugin: Motion change event
    Plugin-->>App: onMotionChange(isMoving: false)
    Native->>GPS: Turn OFF GPS
    
    Note over Native,Sensors: Back to STATIONARY
```

## Elastic Distance Filtering

```mermaid
graph LR
    A[Current Speed] --> B{Calculate Multiplier}
    B --> C[Round to nearest 5 m/s]
    C --> D[Divide by 5]
    D --> E[Multiply by distanceFilter]
    E --> F[Adjusted Distance Filter]
    
    style A fill:#e1f5ff
    style F fill:#c8e6c9
    
    subgraph Examples
        G["Walking: 2 m/s<br/>distanceFilter: 30m<br/>Result: ~30m"] 
        H["Biking: 10 m/s<br/>distanceFilter: 30m<br/>Result: 60m"]
        I["Highway: 30 m/s<br/>distanceFilter: 50m<br/>Result: 300m"]
    end
```

## Location Persistence & Sync Flow

```mermaid
flowchart TD
    A[Location Recorded] --> B[Store in SQLite DB]
    B --> C{autoSync enabled?}
    C -->|No| D[Stored locally]
    C -->|Yes| E{Network available?}
    E -->|No| D
    E -->|Yes| F{batchSync enabled?}
    F -->|Yes| G[Collect multiple locations]
    F -->|No| H[Single location]
    G --> I[HTTP POST to server]
    H --> I
    I --> J{Success?}
    J -->|Yes| K[Delete from DB]
    J -->|No| L[Retry later]
    K --> M[Fire onHttp event]
    L --> D
    
    D --> N[Available via<br/>getLocations API]
```

## Geofencing System

```mermaid
flowchart TB
    A[All Geofences in DB] --> B{Device Location Update}
    B --> C[Query nearby geofences<br/>within proximityRadius]
    C --> D[Activate up to N geofences]
    
    D --> E1[iOS: Max 20 active]
    D --> E2[Android: Max 100 active]
    
    E1 --> F[Platform Geofence Monitor]
    E2 --> F
    
    F --> G{Geofence Event?}
    G -->|ENTER| H[Fire onGeofence<br/>action: ENTER]
    G -->|EXIT| I[Fire onGeofence<br/>action: EXIT]
    G -->|DWELL| J[Fire onGeofence<br/>action: DWELL]
    
    H --> K[Update active geofences]
    I --> K
    J --> K
    K --> B
```

## Battery Consumption Comparison

```mermaid
graph TD
    subgraph "Continuous GPS (Traditional)"
        A1[Always ON] --> A2[8-15% per hour]
        A2 --> A3[Battery dead in 6-8 hours]
    end
    
    subgraph "Motion-Aware (This Plugin)"
        B1[Stationary: GPS OFF] --> B2[~0% per hour]
        B3[Moving: GPS ON] --> B4[1-3% per hour]
        B2 --> B5[Average depends on<br/>movement patterns]
        B4 --> B5
        B5 --> B6[Can run all day]
    end
    
    subgraph "Significant Changes Only"
        C1[Minimal tracking<br/>500-1000m] --> C2[<1% per day]
        C2 --> C3[Can run for days]
    end
    
    style A3 fill:#ffcdd2
    style B6 fill:#c8e6c9
    style C3 fill:#c8e6c9
```

## Configuration Options Impact

```mermaid
mindmap
    root((Config))
        Accuracy
            DESIRED_ACCURACY_HIGH
                GPS + WiFi + Cell
                Highest power
            DESIRED_ACCURACY_MEDIUM
                WiFi + Cell
                Medium power
            DESIRED_ACCURACY_LOW
                WiFi only
                Low power
            DESIRED_ACCURACY_VERY_LOW
                Cell only
                Lowest power
        Filtering
            distanceFilter
                Minimum meters between locations
                Lower = more locations
            disableElasticity
                Disable speed-based adjustment
            elasticityMultiplier
                Scale elastic filtering
        Lifecycle
            stopOnTerminate
                Continue after app close
            startOnBoot
                Auto-start on reboot
            stopAfterElapsedMinutes
                Auto-stop timer
        HTTP Sync
            url
                Server endpoint
            autoSync
                Automatic upload
            batchSync
                Single vs multiple requests
        Special Modes
            useSignificantChangesOnly
                Periodic tracking only
                Lowest power mode
            trackingMode
                Location vs Geofence only
```

## Common Use Case: Delivery Driver

```mermaid
gantt
    title Delivery Driver - GPS Activity During 8 Hour Shift
    dateFormat HH:mm
    axisFormat %H:%M
    
    section GPS Status
    OFF (At Home)           :done, 08:00, 09:00
    ON (Driving to store)   :active, 09:00, 09:15
    OFF (In store)          :done, 09:15, 09:30
    ON (Driving delivery 1) :active, 09:30, 09:45
    OFF (Customer stop)     :done, 09:45, 09:50
    ON (Driving delivery 2) :active, 09:50, 10:05
    OFF (Customer stop)     :done, 10:05, 10:10
    ON (Driving delivery 3) :active, 10:10, 10:30
    OFF (Lunch break)       :done, 10:30, 11:00
    
    section Battery Impact
    ~0% drain               :08:00, 09:00
    2-3% used              :09:00, 09:15
    ~0% drain              :09:15, 09:30
    2% used                :09:30, 09:45
    ~0% drain              :09:45, 09:50
    2% used                :09:50, 10:05
    ~0% drain              :10:05, 10:10
    3% used                :10:10, 10:30
    ~0% drain              :10:30, 11:00
```

---

**Note:** These diagrams use [Mermaid](https://mermaid.js.org/) syntax and will render automatically in GitHub, GitLab, and many markdown viewers. If viewing in an editor that doesn't support Mermaid, you can paste the diagram code into the [Mermaid Live Editor](https://mermaid.live/) to view them.

## How to Use These Diagrams

1. **State Machine**: Understand the core states and transitions
2. **Motion Detection Flow**: Follow how the plugin detects and responds to movement
3. **Architecture Layers**: See how the plugin is structured from Flutter to native
4. **Event Flow**: Understand the sequence of events during tracking
5. **Elastic Filtering**: Visualize how distance filtering adapts to speed
6. **Persistence & Sync**: Follow data flow from capture to server
7. **Geofencing**: Understand proximity-based geofence activation
8. **Battery Comparison**: Compare power consumption across modes
9. **Configuration Impact**: Explore how settings affect behavior
10. **Use Case Example**: See real-world GPS on/off patterns

These diagrams complement the [ARCHITECTURE.md](./ARCHITECTURE.md) and [HOW_IT_WORKS.md](./HOW_IT_WORKS.md) documentation.
