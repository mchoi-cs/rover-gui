# MSERVO Arm GUIs

Two operator front-ends for the rover's 6-DOF manual arm — one that runs in a
terminal over SSH, one that runs in a browser — driving the same arm through the
same command grammar.

## Why it exists

The arm is controlled by typing `!`-terminated frames at an Arduino over serial.
That works, but the operator gets no feedback: no view of which joints are
moving, which stepper drivers are enabled, or whether the arm is e-stopped, and a
typo can command a joint at full speed. These GUIs put a single readable control
and situational-awareness surface in front of that link, so an operator can jog
the arm, kill it instantly, and see exactly what went on the wire — from the
rover PC over SSH, or from a base-station laptop in a browser.

## How it works

Both GUIs are ROS 2 (Humble, rclpy) nodes. They publish `std_msgs/String`
frames on `/arm_cmd` and subscribe to that same topic, so the display is driven
by bus traffic rather than by local key presses. That means they mirror their
own commands *and* commands from any other publisher (joystick, keyboard node),
and the two GUIs stay consistent when run side by side.

    GUI (term | web) ──publish──> /arm_cmd ──> m_router ──serial──> Arduino Mega
            ^                        |
            └──── subscribe (mirror) ┘

`mservo_protocol.py` is the single source of truth for the wire grammar — pure
functions, no ROS and no serial imports — so the terminal GUI, the web GUI, and
the firmware can't drift apart. The grammar:

    S;<TW>;<WP>;<WR>;<EE>;<SL>;<EL>;!   per-joint velocity multipliers
    set0;!  zero    stop;!  e-stop    v;!  toggle verbose feedback
    stepper1..4;!   toggle a stepper driver enable
    svu;! / svd;! / svs;!   camera servo up / down / stop

**Terminal GUI** (`mservo_gui_term.py`): full-redraw box-drawing dashboard at
30 Hz with signed per-joint velocity bars, a stepper-enable diamond, e-stop
banner, scrolling bus log, and a `:` prompt for raw frames. Reads keys straight
from the tty in cbreak mode, so it works unchanged over SSH.

**Web GUI** (`mservo_gui_web.py`): NiceGUI/Quasar dashboard on port 8080 with the
same hotkeys plus buttons, sliders, and a serial box. It also has a `--demo`
mode that swaps the ROS publisher for a local loopback bus, so the exact
interface can be previewed on any laptop with no ROS and no arm attached.

Both publish a zero command on shutdown so the arm never keeps moving.

## Embedded stack

Frames land on an Arduino Mega 2560 (ATmega2560) running `Arm_V2_2_Controls_SPI`:

- 4 stepper joints — tower, wrist pitch, wrist roll, end effector — driven
  non-blocking via AccelStepper, stepped on a TimerThree interrupt
- 2 linear actuators — shoulder, elbow — via Pololu Jrk G2 controllers over I2C
- 1 camera servo
- AMT22 12-bit absolute encoders read over SPI (currently populated on tower and
  wrist pitch), reported back as `f;` position and `g;` GPIO/e-stop frames when
  verbose is on

A parallel refactor moves that sketch into layered C++ (driver / task / app)
under PlatformIO for the same board.

## Tech

Python 3, ROS 2 Humble (rclpy, std_msgs), NiceGUI + Quasar/Tailwind, ANSI
terminal rendering with termios/tty, pyserial, Arduino/C++ on AVR with
PlatformIO, AccelStepper, TimerThree, Pololu JrkG2, Servo.

## Status

v1 is mirror-only: the display reflects commanded state, not measured hardware
state. `m_router` currently logs the arm's `f;`/`g;` feedback instead of
republishing it, so live encoder positions, the collision-watch check (a
deliberate stub today), and any 3D view are blocked on that change.
