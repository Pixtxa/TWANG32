# TWANG32 Hack
An ESP32 based, 1D, LED strip, dungeon crawler. inspired by Line Wobbler by Robin B

The [ESP32 version](https://github.com/bdring/TWANG32) was ported from the [TWANG fork](https://github.com/bdring/TWANG) by bdring of [Buildlog.net Blog](http://www.buildlog.net/blog?s=twang)
Pixtxa forked it again, added some extra leds to the controller and added some other features and bugfixes.

**Current State**

- All of the Arduino version game features are functional.
- The game now has a WiFi access port to get game stats. Connect a smartphone or computer to see them.
  - **SSID:** TWANG_AP
  - **Password:** 12345666
  - **URL:** 192.168.4.1
- You can update these settings over WiFi
  - WS2812B LED type
  - LED Count
  - LED Brightness
  - Audio Volume
  - Joystick direction
  - Joystick Deadzone (removes drift)
  - Attack Threshold (twang sensitivity)
  - Lives Per Level

## TO DO List:
- Wireless features
  - level/score/life count/... live output
  - stream to another stripe/make controller wireless
  - support a brightness sensor for automatic dimming at night
  - reset highscore
- Add an extra animation if new highscore is set
- Add a level display
- Add a score display
- Add a scoreboard/highscore list?
- Config for adding a little glow to the LEDs, so people don't hit unlit parts of the LED chain at night or oversee it during build up/teardown
- Save energy on idle for longer powerbank run time

## Required libraries:
* [FastLED](http://fastled.io/)
* [RunningMedian](http://playground.arduino.cc/Main/RunningMedian)

## Hardware used:
* ESP32, I use the D1-mini module
* I use cheap WS2812B ones, the maximum I used is 300 LEDs, which is a 5 m LED stripe with 60 LED/m, more might be bad because of the latency
  * If you want it longer, I reccomend using more spacing between the LEDs to keep it cheap. I've combined two LED chains with 10 led/m, where the LEDs are visible from all sides, playing the game on 20 meters is really fun
  * The game supports 1000 LEDs maximum, but then APA102C should be used. HD107S might also work, but I haven't tested this. Anything compatible with the FastLED library should work.
* MPU6050 accelerometer
* Originally a spring doorstop, is used, I used a random spring from the local hardware store
* Speaker and amplifier: A PAM8403 module is recomended, because the ESP32 cannot drive a speaker as loudly as an Arduino. But I've connected a piezoelectric speaker which is good enough for me
* Some level converter because the LEDs want 5 V on the data lines. Somehow putting an 1 kOhm resistor in series works, found this hack somewhere

See description of [ESP32 version](https://github.com/bdring/TWANG32) for more details.

## Enclosure
[ESP32 version](https://github.com/bdring/TWANG32) has STL files, but I've used some e-waste parts.

## Overview
The following is a quick overview of the code to help you understand and tweak the game to your needs.

The game is played on a 1000 unit line, the position of enemies, the player, lava etc range from 0 to 1000 and the LEDs that represent them are derived using the `getLED()` function. You don't need to worry about this but it's good to know for things like the width of the attack and player max move speed. Regardless of the number of LEDs, everything takes place in this 1000 unit wide line.

**LED SETUP** Defines the quantity of LEDs as well as the data and clock pins used. I've tested several WS2812B strips and the color order sometimes changes. In my setup the player is green, the exit blue and the enemies are red. Brightness should range from 50 to 255, use a lower number if playing at night or wanting to use a smaller power supply. the joystick direction can be set to 0 or 1 to flip the game orientation. In `setup()` there is a `FastLED.addLeds()` line, in there you could change it to another brand of LED strip.

I've added 13 other neopixels: A circle of 12 for life-display (that also output the level number by counting flashes) and one RGBW on top of the controller that lights up when wobble or on game start/end. They're set in `drawLifebar()` function.

**JOYSTICK SETUP** All parameters are commented in the code, you can set it to work in both forward/backward as well as side-to-side mode by changing `JOYSTICK_ORIENTATION`. Adjust the `ATTACK_THRESHOLD` if the "Twanging" is overly sensitive and the `JOYSTICK_DEADZONE` if the player slowly drifts when there is no input (because it's hard to get the MPU6050 dead level).

**WOBBLE ATTACK** Sets the width, duration (ms) of the attack.

**POOLS** These are the object pools for enemies, particles, lava, conveyors etc. You can modify the quantity of any of them if your levels use more or if you want to save some memory, just remember to update the respective counts to avoid errors.

**USE_GRAVITY** 0/1 to set if particles created by the player getting killed should fall towards the start point, the `BEND_POINT` variable can be set to mark the point at which the strip of LEDs goes from being horizontal to vertical. The game is 1000 units wide (regardless of number of LED's) so 500 would be the mid point. If this is confusing just set `USE_GRAVITY` to 0.

## Modifying / Creating levels
Find the `loadLevel()` function, in there you can see a switch statement with the 10 levels I created.
They all call different functions and variables to setup the level. Each one is described below:

**playerPosition;** Where the player starts on the 0 to 1000 line. If not set it defaults to 0. I set it to 200 in the first level so the player can see movement even if the first action they take is to push the joystick left

**spawnEnemy(position, direction, speed, wobble);** (10 enemies max)
* position: 0 to 1000
* direction: 0/1, initial direction of travel
* speed: >=0, speed of the enemy, remember the game is 1000 wide and runs at 60fps. I recommend between 1 and 4
* wobble: 0=regular moving enemy, 1=sine wave enemy, in this case speed sets the width of the wave

**spawnPool[poolNumber].Spawn(position, rate, speed, direction);** (2 spawners max)
* A spawn pool is a point which spawns enemies forever
* position: 0 to 1000
* rate: milliseconds between spawns, 1000 = 1 second
* speed: speed of the enemis it spawns
* direction: 0=towards start, 1=away from start

**spawnLava(startPoint, endPoint, ontime, offtime, offset);** (4 lava pools max)
* startPoint: 0 to 1000
* endPoint: 0 to 1000, combined with startPoint this sets the location and size of the lava
* ontime: How long (ms) the lava is ON for
* offtime: How long the lava is ON for
* offset: How long (ms) after the level starts before the lava turns on, use this to create patterns with multiple lavas
* grow: This specifies the rate of growth. Use 0 for no growth. Reasonable growth is 0.1 to 0.5
* flow: This specifies the rate/direction of flow. Reasonable numbers are 0.2 to 0.8 

**spawnConveyor(startPoint, endPoint, speed);** (2 conveyors max)
* startPoint, endPoint: Same as lava
* speed: The direction and speed of the travel. Negative moves to base and positive moves towards exit. Must be less than +/- max player speed.

**spawnBoss();** (only one, don't edit boss level)
* There are no parameters for a boss, they always spawn in the same place and have 3 lives. Tweak the values of Boss.h to modify
