# Development Log: Autonomous TORCS Racing Agent

**Model used:** `ibm-granite-3.2-8b-instruct`

This log documents all the prompts and answers used in ibm granite models for this proyect.

## 1. Initial Model Creation

### **Goal:** Create a robust, modular control system for the Corkscrew track.

### Prompt
```text
@snakeoil3_jm2.py @torcs_jm_par.py Act as an expert in autonomous vehicle control and TORCS simulator architecture. I am developing a high-performance agent for the Corkscrew track.

Analyze the provided snakeoil3_jm2.py (for sensor/action interface) and torcs_jm_par.py (current control logic).

I need you to rewrite the drive() function to be a robust, modular control system. Please provide:
1. Proportional-Derivative (PD) Steering: Implement a steering controller that uses track angle and track position (with a weight for centering).
2. Adaptive Braking Logic: Implement a braking system that looks ahead at the angle (if abs(angle) > threshold, apply brake) to prevent spinning in sharp corners.
3. Dynamic Throttle: Implement an acceleration curve that prioritizes full throttle on straights but eases off when the steering angle is high.
4. Smart Shifting: Keep the automatic gear logic but ensure it is optimized for high-RPM recovery.

Provide the complete, clean Python function drive(c) ready to replace the existing one in torcs_jm_par.py. Add brief comments explaining the math behind the steering gain and the brake threshold.
```

### Response

Here's a revised drive() function for 
torcs_jm_par.py
 that incorporates PD Steering, Adaptive Braking Logic, Dynamic Throttle, and Smart Shifting:

```python
#================= MAIN DRIVE FUNCTION =================
def drive(c):
    S, R = c.S.d, c.R.d

    # PD STEERING CONTROL
    steer_kp = 30  # Proportional Gain for steering based on track angle
    steer_kd = 0.20  # Derivative Gain for centering based on track position
    R['steer'] = steer_kp * S['angle'] / math.pi - steer_kd * S['trackPos']

    # ADAPTIVE BRAKING LOGIC
    brake_threshold = 0.9  # Braking threshold angle
    if abs(S['angle']) > brake_threshold:
        R['brake'] = 1.0  # Apply full brakes for sharp turns

    # DYNAMIC THROTTLE
    target_speed = 250  # Target speed in km/h, adjust as needed
    throttle_gain = 0.4  # Gain for acceleration based on steering angle
    if S['speedX'] < target_speed - (R['steer'] * throttle_gain):
        R['accel'] += 0.2
    else:
        R['accel'] -= 0.1

    # Ensure the throttle doesn't go below zero
    R['accel'] = max(0.0, min(1.0, R['accel']))

    if S['speedX'] < 10:
       R['accel'] += 1 / (S['speedX'] + 0.1)

    # SMART SHIFTING
    gear_speeds = [0, 25, 50, 100, 150, 250]  # Adjust gear shift points for higher RPM recovery
    current_gear = 1
    for i, speed in enumerate(gear_speeds):
        if S['speedX'] > speed:
            current_gear = i + 1

    R['gear'] = min(current_gear, 6)

    return

#================= MAIN LOOP =================
if __name__ == "__main__":
    C = Client(p=3001)
    for step in range(C.maxSteps, 0, -1):
        C.get_servers_input()
        drive(C)
        C.respond_to_server()
    C.shutdown()
```

Explanation of the Math:

PD Steering: The steering control uses a Proportional-Derivative controller. steer_kp determines how aggressively the car turns based on the angle. A higher value means more sensitive steering. The derivative term (steer_kd) is used to center the car by applying correction proportional to the distance from the track's centerline.

Adaptive Braking Logic: By checking if the absolute value of S['angle'] exceeds brake_threshold, we determine when to apply brakes in sharp corners. This helps maintain stability and prevents spinning out on tight turns, especially important for high-speed tracks like Corkscrew.

Dynamic Throttle: The throttle control increases acceleration (R['accel']) when the car is far from optimal speed (defined by target_speed minus a steering penalty). This incentivizes high-speed straight line driving while easing off acceleration as the steering angle increases, promoting smoother cornering.

Smart Shifting: The gear shifting logic now looks at S['speedX'], the current speed along the track's x-axis. We've adjusted shift points (gear_speeds) to allow for higher RPM recovery, which is beneficial on tracks with long straights and high-speed corners, common in racing environments.

This revised drive() function provides a more sophisticated control strategy suitable for high-performance autonomous driving, taking into account the demands of the TORCS simulator and the specifics of competitive racing scenarios.

---

### **Goal:** Eliminate zig-zagging in straights and fix premature crashing in corners.

### Prompt
```text
The current drive(c) function has two major issues:
In straight sections, the car oscillates (zig-zags) excessively. This suggests that the proportional steering gain steer_kp = 30 is too aggressive or that we need a damping term. Please adjust the PD controller to be more stable in straights while maintaining responsiveness.
The car crashes in the first corner because the braking system is too reactive (it only brakes at angle > 0.9). Please implement a predictive braking logic: if the 'track' sensors indicate a sharp decrease in distance ahead (implying a turn), apply partial braking earlier to ensure corner entry at a stable speed.
Please provide the updated drive(c) function with these tuned parameters and briefly explain the changes made to the PID constants.
```

### Response

Here's the revised drive() function addressing both issues:

```Python
#================= MAIN DRIVE FUNCTION =================
def drive(c):
    S, R = c.S.d, c.R.d

    # TUNED PD STEERING CONTROL WITH DAMPING
    steer_kp = 15  # Proportional Gain for steering based on track angle (reduced from 30)
    steer_kd = 0.15  # Derivative Gain for centering based on track position (adjusted to be less aggressive)
    R['steer'] = steer_kp * S['angle'] / math.pi - steer_kd * S['trackPos'] - 0.05 * R['steer']  # Added damping term

    # PREDICTIVE BRAKING LOGIC
    track_sensors = S['track']
    if len(track_sensors) > 1 and (track_sensors[-1] - track_sensors[0]) < -20:  # Check for upcoming sharp turn
        R['brake'] = 0.5  # Apply partial brakes when a sharp turn is detected ahead

    # ADAPTIVE THROTTLE
    target_speed = 250  # Target speed in km/h, adjust as needed
    throttle_gain = 0.3  # Gain for acceleration based on steering angle (reduced to be less aggressive)
    if S['speedX'] < target_speed - (R['steer'] * throttle_gain):
        R['accel'] += 0.15
    else:
        R['accel'] -= 0.1

    # Ensure the throttle doesn't go below zero
    R['accel'] = max(0.0, min(1.0, R['accel']))

    if S['speedX'] < 10:
       R['accel'] += 1 / (S['speedX'] + 0.1)

    # SMART SHIFTING
    gear_speeds = [0, 25, 50, 100, 150, 250]  # Adjust gear shift points for higher RPM recovery
    current_gear = 1
    for i, speed in enumerate(gear_speeds):
        if S['speedX'] > speed:
            current_gear = i + 1

    R['gear'] = min(current_gear, 6)

    return

#================= MAIN LOOP =================
if __name__ == "__main__":
    C = Client(p=3001)
    for step in range(C.maxSteps, 0, -1):
        C.get_servers_input()
        drive(C)
        C.respond_to_server()
    C.shutdown()
```

Changes and Explanations:

Tuned PD Steering with Damping:

steer_kp was reduced from 30 to 15 to decrease the aggressiveness of the steering control, addressing the zig-zagging in straight sections.
The derivative term (steer_kd) was adjusted to a lower value (0.15) for smoother centering.
A damping term -0.05 * R['steer'] was added to counteract oscillations by dampening large changes in steering angle, promoting stability.
Predictive Braking Logic:

The braking system now checks the track sensors for a significant decrease in distance ahead (indicating an upcoming turn). If such a condition is met (track_sensors[-1] - track_sensors[0] < -20), partial brakes are applied. This allows for smoother corner entries by reducing speed before the sharp turn, enhancing stability and preventing crashes.
These modifications aim to improve the agent's performance in both straight sections (reducing oscillations) and sharp turns (applying early braking for safer entry). The adjustments balance responsiveness with stability, crucial for high-speed tracks like Corkscrew.

---

### **Goal:** Anticipate corners based on track geometry rather than just steering angle.

### Prompt
```text
The current drive(c) function has successfully eliminated the zig-zag oscillations in the straights, and the steering control is now stable. However, the car still crashes at the entrance of the first corner because it enters at too high a velocity.

The current braking logic is too reactive, relying on steering angle thresholds rather than anticipating the track geometry. Please implement a predictive 'look-ahead' braking strategy by analyzing the S['track'] sensor array to anticipate the corner before the car reaches the apex.

Identify the optimal sensor indices in the S['track'] array (e.g., center sensors) to detect a sharp track curvature change before the car reaches the corner.

Define logic that scales braking pressure based on the distance reduction of these 'look-ahead' sensors, effectively creating a 'braking zone' before the turn.

Provide the updated drive(c) function. Briefly explain the mathematical logic used for the look-ahead threshold to prevent crashing while allowing for the maximum possible entry speed.
```

### Response

Here's the revised drive() function implementing a predictive braking strategy based on analyzing track sensor data:

```Python
#================= MAIN DRIVE FUNCTION =================
def drive(c):
    S, R = c.S.d, c.R.d

    # TUNED PD STEERING CONTROL WITH DAMPING
    steer_kp = 15  # Proportional Gain for steering based on track angle
    steer_kd = 0.15  # Derivative Gain for centering based on track position
    R['steer'] = steer_kp * S['angle'] / math.pi - steer_kd * S['trackPos'] - 0.05 * R['steer']

    # PREDICTIVE LOOK-AHEAD BRAKING LOGIC
    lookahead_sensors = S['track'][-5:]  # Use last five sensors to anticipate track curvature change
    max_distance_change = 20  # Threshold for detecting a sharp curve ahead
    for i in range(1, len(lookahead_sensors)):
        if lookahead_sensors[i] - lookahead_sensors[i-1] < max_distance_change:
            distance_reduction = min(1.0, (max_distance_change - (lookahead_sensors[i] - lookahead_sensors[i-1])) / max_distance_change)  # Scale braking based on how sharp the turn is
            R['brake'] = distance_reduction * 0.6  # Apply partial brakes gradually as a sharp turn is detected

    # ADAPTIVE THROTTLE
    target_speed = 250  # Target speed in km/h, adjust as needed
    throttle_gain = 0.3  # Gain for acceleration based on steering angle
    if S['speedX'] < target_speed - (R['steer'] * throttle_gain):
        R['accel'] += 0.15
    else:
        R['accel'] -= 0.1

    # Ensure the throttle doesn't go below zero
    R['accel'] = max(0.0, min(1.0, R['accel']))

    if S['speedX'] < 10:
       R['accel'] += 1 / (S['speedX'] + 0.1)

    # SMART SHIFTING
    gear_speeds = [0, 25, 50, 100, 150, 250]  # Adjust gear shift points for higher RPM recovery
    current_gear = 1
    for i, speed in enumerate(gear_speeds):
        if S['speedX'] > speed:
            current_gear = i + 1

    R['gear'] = min(current_gear, 6)

    return

#================= MAIN LOOP =================
if __name__ == "__main__":
    C = Client(p=3001)
    for step in range(C.maxSteps, 0, -1):
        C.get_servers_input()
        drive(C)
        C.respond_to_server()
    C.shutdown()
```
Explanation of the Mathematical Logic:

Look-Ahead Sensor Selection: The function uses the last five elements in S['track'] (lookahead_sensors) to anticipate upcoming track geometry changes. These sensors are chosen as they represent positions near the car's current and future trajectory, providing a window into what lies ahead without requiring precise prediction of future positions.

Sharp Turn Detection: The code checks for a significant decrease in track distance (lookahead_sensors[i] - lookahead_sensors[i-1] < max_distance_change) within the look-ahead sensor array, indicating an impending sharp turn.

Scaling Braking Pressure: If a potentially sharp turn is detected, R['brake'] is set based on how sharp the turn is (distance_reduction). This scaling ensures gradual braking application as the corner approaches, allowing for the maximum possible entry speed without crashing. The value of 0.6 scales down the full braking force to a moderate level suitable for gradual deceleration.

This predictive logic uses track sensor data proactively to manage speed, avoiding crashes by initiating controlled deceleration before reaching sharp turns, thus allowing higher entry speeds while maintaining safety and control.

---

### **Goal:** Create a modular architecture that divides tasks(steering, throttle, gear, etc).

### Prompt:
```text
Act as an expert in autonomous vehicle control and TORCS simulator architecture.

I want to refactor our current drive logic into a clean, modular architecture. 

Please note that the car we are controlling is a Formula 1 car. You should use your knowledge to determine and apply realistic F1 parameters (e.g., weight, engine characteristics, and high RPM shifting logic—likely around 15,000+ RPM) where necessary in the logic.

Include this setting in the code:

ENABLE_TRACTION_CONTROL = True  # Toggle traction control system

STRICT Windows TORCS troubleshooting constraints to follow:

- Output ranges MUST be strictly clipped: steer ∈ [-1.0, 1.0], accel/brake ∈ [0.0, 1.0].

- Gear MUST always be sent as an integer.

- Launch control must be included to overcome static friction.

Please generate a Python script containing the following modular helper functions and the main control loop:

1. `calculate_steering(S)`: Implement a Proportional-Derivative (PD) steering controller. Use track angle and track position to compute the steering value, applying appropriate gains for stable cornering.

2. `calculate_throttle(S, R)`: Implement a dynamic acceleration curve aiming to drive as fast as safely possible. The logic should apply maximum throttle on straightaways but dynamically scale down the acceleration based on how sharp the current steering angle is (i.e., ease off the gas when turning to avoid flying off the track). CRITICAL: You must include a launch control mechanism (e.g., `if S['speedX'] < 10: accel += 1 / (S['speedX'] + 0.1)`) to ensure the car can start from a standstill. 

3. `apply_brakes(S)`: Implement a basic reactive braking system that applies partial braking if the car's angle relative to the track axis exceeds a critical threshold. 

4. `shift_gears(S)`: Implement a smart shifting logic tailored for an F1 engine.

    - Read the current gear from `int(S['gear'])`.

    - CRITICAL RULE: If `S['speedX'] < 1.0`, the gear must be 1.

    - Shift up and down based on optimal RPM bands for an F1 car. Limit the maximum gear to 6.

5. `traction_control(S, accel)`: If `ENABLE_TRACTION_CONTROL` is true, monitor wheel spin. If the rear wheels are spinning significantly faster than the front wheels (e.g., a difference > 2 rad/s), dynamically reduce the acceleration command to regain traction.
```

### Response:


