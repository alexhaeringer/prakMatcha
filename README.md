# Automated Matcha Preparation with CPEE and UR5

## Overview

This project automates the preparation of Matcha using:

* a **UR5 cobot**
* the **CPEE Process Execution Engine**
* **MQTT** for kettle monitoring
* several individual **URP robot programs**
* a Robotiq gripper

CPEE controls the complete process and starts the required robot programs through HTTP requests.

Here is the video link to watch the entire process for two matchas: https://youtu.be/gr00s2_kUkw 

## Process Flow

At the beginning, two actions run in parallel:

1. The robot moves to its starting position.
2. CPEE monitors the kettle through MQTT.

After the kettle has finished heating, the following steps are repeated for every mug:

1. Pick up the kettle.
2. Pour hot water into the mug.
3. Return the kettle.
4. Move to the spoon holder.
5. Select the correct spoon position.
6. Pick up the spoon.
7. Pour the Matcha powder into the mug.
8. Return the spoon.
9. Whisk the Matcha with the milk frother.

The number of prepared mugs is defined by the CPEE variable `num_of_mugs`.

## MQTT Kettle Monitoring

The kettle is connected to a smart plug that publishes its power consumption through MQTT.

CPEE uses the service:

```text
https-get://lab.bpm.in.tum.de/mqtt/read/record_start_stop/
```

The following topic is monitored:

```text
/lab-power/socket-4/Power
```

The kettle is considered started when the power value reaches `5` and finished when the value returns to `0`.

This allows CPEE to wait until the water is heated before continuing the robot process.

## Spoon Selection with Registers

The spoons are stored at different positions in the holder.

CPEE stores the current spoon number in:

```text
data.spoon
```

Before the robot grabs a spoon, CPEE writes this value into the UR5 integer input register `0`:

```text
https://lab.bpm.in.tum.de/ur5/kanne/registers/input/int/0
```

The `Grab_spoon.urp` program reads this register and moves to the corresponding spoon position.

After a spoon is used, the value is increased:

```ruby
data.spoon += 1
```

The process continues until `data.spoon` reaches `num_of_mugs`.

## Robot Programs

| Program                   | Function                                                         |
| ------------------------- | ---------------------------------------------------------------- |
| `Pick_up_position.urp`    | Moves the robot to the starting position in front of the kettle. |
| `Pick_up_kettle.urp`      | Picks up the kettle.                                             |
| `Pour_water.urp`          | Pours hot water into the mug.                                    |
| `Return_kettle.urp`       | Returns the kettle.                                              |
| `From_kettle_to_home.urp` | Moves back from the kettle area.                                 |
| `From_home_to_spoon.urp`  | Moves to the spoon holder.                                       |
| `Grab_spoon.urp`          | Grabs the spoon selected through the register.                   |
| `Pour_spoon.urp`          | Pours the Matcha powder into the mug.                            |
| `Return_spoon.urp`        | Returns the used spoon.                                          |
| `From_spoon_to_home.urp`  | Moves back from the spoon area.                                  |
| `whisk_matcha.urp`        | Picks up the frother and whisks the Matcha.                      |

## CPEE Integration

The complete process is stored in:

```text
matchav3.xml
```

CPEE starts each URP program through an HTTP `PUT` request.

Example:

```text
https://lab.bpm.in.tum.de/ur5/kanne/programs/Matcha/Pour_water.urp/wait/
```

The `/wait/` endpoint ensures that CPEE waits until the robot program has finished before starting the next step.

## How to Start the Program

1. Set the required number of Matchas in the CPEE variable:

```
num_of_mugs
```

2. Fill the required number of spoons with Matcha powder and place them in the spoon holder.
3. Place the mugs in the predefined position.
4. Fill the kettle with enough water and place it in its predefined position.
5. Manually start the kettle.
6. Start the CPEE process.
