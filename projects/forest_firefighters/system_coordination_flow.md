# Forest Firefighters: System Coordination Flow

This document details the system-wide control flow, illustrating how the centralized Fire Supervisor coordinates with the Mavic and Spot robots. It highlights the use of the shared `fire_targets.json` file for navigation and the `customData` field for executing water drops.

## System Coordination Diagram

```mermaid
graph TD
    %% Shared Resources
    JSON[(fire_targets.json)]
    
    %% Supervisor Control
    subgraph Fire Supervisor (fire.py)
        FS[Initialize Environment] --> PubInit[Publish Initial Targets]
        PubInit --> FL{Simulation Loop}
        FL --> CheckWater[Clean Water & Read customData]
        CheckWater --> WaitAlt{Mavic Alt > 40m?}
        WaitAlt -- No --> FL
        WaitAlt -- Yes --> Ignite[Burn Trees & Propagate Fire]
        Ignite --> Extinct[Check Extinction & Reignite]
        Extinct --> Pub[Publish Targets to JSON]
        Pub --> FL
    end
    
    %% Mavic Drone Control
    subgraph Mavic 2 PRO Drone (autonomous_mavic.py)
        MC[Start Drone] --> ML{Simulation Loop}
        ML --> ReadJSON[Read JSON targets]
        ReadJSON --> FireExist{Target Exists?}
        FireExist -- No --> Patrol[Patrol default Waypoints]
        FireExist -- Yes --> NavTarget[Navigate straight to fire coordinates]
        Patrol --> CamScan[Downward Camera: OpenCV HSV mask]
        NavTarget --> CamScan
        CamScan --> FireDet{Fire/Smoke Detected?}
        FireDet -- Yes --> DropWater1[Set customData: Drop Water]
        FireDet -- No --> ML
        DropWater1 --> ML
    end
    
    %% Spot Robot Control
    subgraph Spot Ground Robot (spot.py)
        SC[Stand Up] --> SL{Simulation Loop}
        SL --> Walk[Execute Walk Step / Patrol Line]
        Walk --> KeyCheck{Key 'D' pressed?}
        KeyCheck -- Yes --> DropWater2[Set customData: Drop Water]
        KeyCheck -- No --> SL
        DropWater2 --> SL
    end
    
    %% Inter-Process Coordination Links
    Pub -.->|Writes| JSON
    JSON -.->|Reads| ReadJSON
    DropWater1 -.->|Updates robot customData| CheckWater
    DropWater2 -.->|Updates robot customData| CheckWater
```

## How the Coordination Works

1. **Takeoff and Altitude Synchronization**
   - At the beginning of the simulation, the Fire Supervisor does not immediately start the fire. It waits, allowing the Mavic drone time to take off.
   - Only when the drone reaches a safe patrol altitude of $> 40\text{m}$ does the supervisor ignite the first tree and begin the dynamic fire propagation.

2. **Target Broadcasting via File I/O (`fire_targets.json`)**
   - The **Fire Supervisor** tracks which `Sassafras` trees are currently on fire.
   - It continuously serializes the $(X, Y)$ world coordinates of these burning trees into a shared JSON file.
   - The **Mavic** drone reads this JSON file during its control loop. If coordinates are present, the drone aborts its standard patrol waypoints and flies a direct vector to the fire.

3. **Visual Verification (Moves and Camera)**
   - Once the Mavic reaches the target coordinates, it doesn't drop water blindly. It uses a downward-facing camera and OpenCV (HSV thresholding and morphological filtering) to locate the exact center of the smoke/flames.
   - The drone fine-tunes its pitch and yaw to hover directly above the detected fire centroid.

4. **Water Dropping via Webots API (`customData`)**
   - When the drone is aligned (or when the 'D' key is pressed for Spot), the robot updates its `customData` field in the Webots API with an integer representing the quantity of water to drop.
   - On the next simulation step, the **Fire Supervisor** reads the `customData` field of every registered robot. If the value is greater than 0, the supervisor physically spawns water nodes beneath the robot.
   - The supervisor tracks the water collisions with the ground. If water falls within the extinction radius of a burning tree, the fire is put out, and the JSON file is updated on the next tick.
