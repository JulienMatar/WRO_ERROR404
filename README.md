# Engineering materials
This repository contains engineering materials of a self-driven vehicle's model participating in the WRO Future Engineers competition in the season 2022.

# Content
# t-photos 
contains 2 photos of the team (an official one and one funny photo with all team members)
# v-photos 
contains 6 photos of the vehicle (from every side, from top and bottom)
# video
contains the video.md file with the link to a video where driving demonstration exists
# schemes 
contains one or several schematic diagrams in form of JPEG, PNG or PDF of the electromechanical components illustrating all the elements (electronic components and motors) used in the vehicle and how they connect to each other.
# src 
contains code of control software for all components which were programmed to participate in the competition
# models
is for the files for models used by 3D printers, laser cutting machines and CNC machines to produce the vehicle elements. If there is nothing to add to this location, the directory can be removed.
# other 
is for other files which can be used to understand how to prepare the vehicle for the competition. It may include documentation how to connect to a SBC/SBM and upload files there, datasets, hardware specifications, communication protocols descriptions etc. If there is nothing to add to this location, the directory can be removed.
# Introduction
This part must be filled by participants with the technical clarifications about the code: which modules the code consists of, how they are related to the electromechanical components of the vehicle, and what is the process to build/compile/upload the code to the vehicle’s controllers.
# WRO Future Engineers 2026 — Team ERROR404

**Team members:** Julien Matar, Ali Kholindy
**Category:** WRO Future Engineers — Self-Driving Cars Challenge, Season 2026

This repository contains the engineering documentation, source code, electromechanical
schematics, and vehicle media for our self-driving vehicle competing in the WRO Future
Engineers category.

---

## Content

| Folder | Contents |
|---|---|
| `t-photos/` | Team photos (official + informal) |
| `v-photos/` | Vehicle photos (all sides, top, bottom) |
| `video/` | `video.md` with links to autonomous driving demonstration videos |
| `schemes/` | Wiring/electrical diagrams of all electromechanical components |
| `src/` | Control software for the Raspberry Pi and Arduino Uno |
| `models/` | 3D-printable / CNC files for chassis, steering, and gearbox components |
| `other/` | Supporting documentation, datasets, and setup instructions |

---

## 1. Mobility and Mechanical Design

Our vehicle is a 4-wheeled car with **Ackermann steering** at the front axle and a single
**driving motor** connected to the rear axle through a gearbox, satisfying the WRO
requirement that driving wheels be mechanically linked (no independent per-side motors /
electronic differential).

**Drivetrain:**
- Drive motor: **25GA-370 DC gear motor**, driving the rear axle through a gearbox.
- Steering: a dedicated **servo motor** actuates the front axle through an Ackermann
  linkage, giving each front wheel a slightly different turning radius — closer to how a
  real car steers than a simple single-pivot mechanism, which reduces tire scrub in tight
  corners.

**Why Ackermann + single rear drive motor:**
We chose a single drive motor over a dual-motor differential-style setup because the WRO
rules explicitly disqualify differential wheeled bases with one motor per side. Beyond
compliance, a single motor through a gearbox is mechanically simpler to keep aligned and
reduces the number of things that can desync during a run (e.g. two motors drifting out
of sync under slightly different loads).


`[FILL IN: gear ratio you used on the 25GA-370 gearbox, and why — e.g. did you test more
than one ratio? What tradeoff did you observe between top speed and torque/turning
control? Include any measured lap-time or stability differences.]`

**Chassis:**
The chassis and steering linkage are 3D-printed (STL files in `models/chassis/` and
`models/Steering/`), keeping the vehicle within the WRO limits of 300×200 mm footprint,
300 mm height, and 1.5 kg total weight.

The 25GA-370 drive motor (200 RPM variant) came bundled with our chassis kit rather than being independently selected from a range of gear ratios. During testing, we found the vehicle's top speed to be a limiting factor rather than torque — the car did not struggle with load or cornering, but felt slower than we'd like on the straight sections of the track. This suggests that a higher-RPM variant (e.g. 300 RPM, which uses a lower internal gear ratio and sacrifices some torque for speed) could improve lap times, since our track sections do not appear to demand more torque than the 200 RPM motor already provides. We are noting this as a potential upgrade for future iterations rather than a change made this season

## 2. Power and Sensor Architecture

**Power system:**
Two 11.1V LiPo batteries are wired in **parallel** (not series) behind a single main
power switch on the combined positive lead — satisfying the WRO requirement for one
switch to power on the vehicle. From there, power splits into two independent branches:

1. A DC-DC buck converter (rated for up to 36V input, well above our battery voltage,
   giving headroom against sag under load) steps the pack voltage down to a **regulated
   5V** logic rail, powering the Raspberry Pi, Arduino Uno, and servo.
2. The **L9110 dual-channel H-bridge** motor driver is fed directly from the 11.1V battery
   rail to drive the 25GA-370 drive motor at full torque, independent of the logic supply.

Separating the motor supply from the logic supply protects the Pi and Arduino from voltage
dips caused by motor stall current or back-EMF, which is a common failure mode if a motor
driver and logic share a single unregulated rail.

The voltage overall from the batteries drops approximately 0.3-0.8V when the motor is spinning, which is normal because one battery is supplying the whole robot at a time, while supplying the buck converter for the stable 5V for the microcontrollers and sensors, even with this voltage drop, its not significant to a point where the robot will slow down/ slug/ or even lag when driving,nor is affecting the output of the buck converter, one 12V LiPo battery is enough to supply for the whole robot, because we also have a backup we can switch freely and easily by simply switching the T plug from a battery to another,

**Start button:**
A separate momentary push button acts as the WRO-compliant Start button (distinct from
the main power switch), matching the required "one switch to power on, one button to
start the program" sequence.

**Sensors:**

| Sensor | Purpose |
|---|---|
| Raspberry Pi Camera v2 (5MP) | Primary vision input — lane detection, traffic sign (pillar) color and position detection |
| TCS3200 color sensor | Auxiliary color detection (e.g. surface/line color cues, or close-range pillar color confirmation) |
| 4–6 ultrasonic sensors | Distance/wall/obstacle sensing around the vehicle perimeter for wall-following and collision avoidance |

we have used 3 ultrasonic sensors for the front side of the robot, to detect distance from all sides while driving in order to avoid all obstacles and not hit one single pillar or wall, and one ultrasonic sensor from the back to assist with the parallel parking in order to prevent the car from hitting the back wall,
We also used the camera to only detect the pillars color with a specific code uploaded to the rpi to let the camera know if the pillar is green or red, and the color sensor module is for the lines on the ground ( orange and blue) in order to count the lines to know how many laps were already accomplished because the rpi that we chose is budget friendly, but not powerful enough to handle a powerful AI to do all of this, we priorotized sensor integration and color detection.
Lastly, we tuned the color detection detection code using the CMYK and RGB values given in the PDF

---

## 3. Software Architecture and Obstacle Strategy

Control is split across two controllers by responsibility:

- **Raspberry Pi 3B+** — runs the vision pipeline (camera capture and image processing)
  and high-level decision-making: detecting the lane boundaries, detecting and
  classifying red/green traffic pillars, and deciding which side to pass them on per the
  WRO rule (red pillar → pass on the right, green pillar → pass on the left).
- **Arduino Uno** — handles real-time, low-latency execution: driving the L9110 motor
  driver, actuating the steering servo, and polling the ultrasonic sensors and TCS3200
  color sensor. It receives high-level commands (e.g. target heading, speed) from the Pi
  over a serial link and executes them continuously between updates.

This split exists because vision processing on the Pi is not guaranteed to run at a fixed,
low-latency interval, while motor and servo control benefit from a tight, predictable loop
that only a dedicated microcontroller can reliably provide.

Two pillars appearing in the same section is not yet handled by our current logic, this is planned as future work rather than something implemented this season.
The Arduino determines it has cleared a pillar once the relevant ultrasonic sensor no longer detects it within range, at which point the steering offset resets back to center (wall-equidistant) driving.

---

## 4. Systems Thinking and Engineering Decisions

Key constraints we designed around:
- **Weight and size limits** (1.5 kg, 300×200×300 mm) — influencing our choice of
  lightweight 3D-printed chassis parts over heavier off-the-shelf alternatives.
- **Compute split** — offloading real-time motor/servo control to the Arduino so that
  camera processing delays on the Pi never translate into jerky or delayed motor response.
- **Power isolation** — separating the motor supply from the logic supply to avoid
  brownouts on the Pi/Arduino during motor stall or direction changes.

A key tradeoff in our sensor architecture was splitting color-detection tasks between the camera and the TCS3200 rather than relying on the camera alone. Our Raspberry Pi 3B+ is not powerful enough to reliably run full real-time computer vision for both pillar detection and ground-line/lap counting simultaneously, so we assigned each task to the sensor best suited for it: the camera for pillar color/position (where a wider field of view matters), and the TCS3200 for ground-line detection (a simpler, close-range color read that doesn't need a full image pipeline). This let us keep both detection tasks reliable without overloading the Pi's processing budget.
One failure mode we identified: if the wall-centering steering correction is too weak, the vehicle drifts into a wall during cornering rather than actively correcting (see Section 3 for the fix — we increased correction sensitivity to resolve this).
Our design changed iteratively during testing: version 1 of the wall-centering logic used a weak steering correction, which caused the vehicle to drift into walls on corners. After identifying this, we increased the correction's sensitivity to the left/right ultrasonic distance difference, which resolved the drift without introducing oscillation

## 5. Reproducibility and Build Instructions

### Repository structure
- `src/arduino/` — Arduino Uno sketch(es) controlling the L9110 motor driver, steering
  servo, ultrasonic sensors, and TCS3200 color sensor.
- `src/pi/` — Raspberry Pi Python code for camera capture, image processing, and
  high-level driving logic.
- `schemes/` — Wiring diagrams showing how the batteries, buck converter, L9110, motor,
  servo, sensors, Pi, and Arduino are all connected.
- `models/` — STL files for 3D-printed chassis and steering components.

### Build / compile / upload process

1. **Arduino side:**
   - Open `src/arduino/<sketch_name>.ino` in the Arduino IDE (or use `arduino-cli`).
   - Select board: **Arduino Uno**.
   - Connect via USB and upload.

2. **Raspberry Pi side:**
   - Copy the contents of `src/pi/` to the Raspberry Pi (e.g. via `scp`, or clone this
     repository directly on the Pi).
   - Install dependencies: `pip install -r requirements.txt`
   - Run the main control script, e.g. `python3 main.py`.
   - Currently, the control script is launched manually via terminal/SSH rather than auto-starting on boot. Setting this up as a systemd service (or an rc.local       entry) so the vehicle powers on directly into the WRO-required "waiting for start button" state is planned as a next step before competition, since manual         terminal launching would not satisfy the WRO starting procedure requirement.

3. Confirm the serial port name and baud rate match between the Pi script and the
   Arduino sketch before running.
The Raspberry Pi code currently uses opencv-python (cv2) for camera capture and pillar color/image processing, and numpy for underlying array/image data handling. The Arduino sketch uses standard built-in libraries for servo control and digital I/O; no external Arduino libraries are required beyond the standard toolchain.
---

## Wiring Overview (see `schemes/` for full diagrams)

- 2× 11.1V LiPo batteries → parallel → single main power switch → splits to:
  - DC-DC buck converter (36V-rated input, set to 5V output) → Raspberry Pi, Arduino,
    servo logic
  - L9110 motor driver (direct 11.1V) → 25GA-370 drive motor
- Push button (separate from main switch) → WRO-compliant start trigger
- Arduino Uno ↔ Raspberry Pi 3B+ via serial (USB)
- Arduino Uno ↔ 4–6 ultrasonic sensors, TCS3200 color sensor, steering servo, L9110
- Raspberry Pi 3B+ ↔ Camera v2 (5MP) via CSI ribbon cable

