This hoverboard firmware is based upon the FOC firmware.   It has GPIO modifications to match a motherboard that was taken from a Hover 1 H1 hoverboard.  The GPIO changes are shown here: https://github.com/NiklasFauth/hoverboard-firmware-hack/issues/162#issuecomment-757374880.   So you need to restore the GPIO changes to your board before running.   
My repository uses a modified FOC ADC Variant that includes a closed loop PID controller that uses HALL effect sensor motor position to as feedback.  The inputs are from ADC1 and ADC2.  PID is parameterized to include  KP,KD,KI, deadzone and KI integerator limit.  Since the position is based upon HALL sensor feedback, the resolution is 4 degs mechanical position.   A separate encoder is required for more precision. The default software only uses KP = 2... all other PID gains are 0.  All input filters and limiters are effectively removed by setting their paramaters at extreme limits.

Since the main loop includes an input rate limit and filter I decided to regulate the main loop to be periodic at exactly the DELAY_IN_MAIN_LOOP.   This is done with a main loop timer function that regulates the main period based upon the bldc in terrupt based buzzerTimer.   If less than 32 characters are debug printed, then the print frequency can be every 5m.  I have also replaced steering/speed variables with speed1 /speed2 variables.   
 
 The  PID inputs and motor feedbacks are scaled to 2000 counts per rotation or +_1000 cnts.  ie 1000 counts moves the motor 180 mechanical degrees.   Varying the ADC from 0 to 4096 will cause one rotation.  If using type2 ADC, starting the program with ADC at mid position will create a zero startup transient.   If there is an uncenterd ADC, the motors will move to the commanded position slowly at 100 pwm speed for during first 5 seconds to slow fade the PID loop action. After 5 seconds the pwm command limit is restored to 1000.   
Notice that these #Defines are required to run the PID.
In config.h  uncomment
#Define VARIANT_ADC   
and PID_CONTROL here:
// ############################# PID CONTROL SETTINGS ################################
  #define PID_CONTROL   //uncomment to use PID control
	
	#ifdef PID_CONTROL 
	  
	 #define KP  2.
	 #define KI  0. 
	 #define PIDDZ 0
	 #define PIDLIM  360
	 typedef struct 
     {
		 float Kp;
		 float Ki;
		 int dz ;
		 int limit;
		 int error;
		 int cum_error;
		 int input;
		 int feedback;
		 int output;
		 
		 }PID_DATA ;
		 void PID(PID_DATA *p);
		 void print_PID(PID_DATA p);
	#endif

Good luck...
The motors are very responsive....Motor response to a full rotation step input to the PID controller takes less than an 1/8 second reach a full 360 deg mechanical rotation.    See plots and video in this issue... https://github.com/EmanuelFeru/hoverboard-firmware-hack-FOC/issues/139

# hoverboard-firmware-hack-FOC
[![Build Status](https://travis-ci.com/EmanuelFeru/hoverboard-firmware-hack-FOC.svg?branch=master)](https://travis-ci.com/EmanuelFeru/hoverboard-firmware-hack-FOC)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donate_SM.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=CU2SWN2XV9SCY&currency_code=EUR&source=url)

This repository implements Field Oriented Control (FOC) for stock hoverboards. Compared to the commutation method, this new FOC control method offers superior performance featuring:
 - reduced noise and vibrations 	
 - smooth torque output and improved motor efficiency. Thus, lower energy consumption
 - field weakening to increase maximum speed range


Table of Contents
=======================

* [Hardware](#hardware)
* [FOC Firmware](#foc-firmware)
* [Example Variants](#example-variants)
* [Dual Inputs](#dual-inputs)
* [Flashing](#flashing)
* [Troubleshooting](#troubleshooting)j
* [Diagnostics](#diagnostics)
* [Projects and Links](#projects-and-links)
* [Contributions](#contributions)

#### For the hoverboard sideboard firmware, see the following repositories:
 - [hoverboard-sideboard-hack-GD](https://github.com/EmanuelFeru/hoverboard-sideboard-hack-GD)
 - [hoverboard-sideboard-hack-STM](https://github.com/EmanuelFeru/hoverboard-sideboard-hack-STM)
 
#### For the FOC controller design, see the following repository:
 - [bldc-motor-control-FOC](https://github.com/EmanuelFeru/bldc-motor-control-FOC)

#### Videos:
<table>
  <tr>
    <td><a href="https://youtu.be/IgHCcj0NgWQ" title="Hovercar" rel="noopener"><img src="/docs/pictures/videos_preview/hovercar_intro.png"></a></td>
    <td><a href="https://youtu.be/gtyqtc37r10" title="Cruise Control functionality" rel="noopener"><img src="/docs/pictures/videos_preview/cruise_control.png"></a></td>
    <td><a href="https://youtu.be/jadD0M1VBoc" title="Hovercar pedal functionality" rel="noopener"><img src="/docs/pictures/videos_preview/hovercar_pedals.png"></a></td>
  </tr>
  <tr>
    <td><a href="https://youtu.be/UnlbMrCkjnE" title="Commutation vs. FOC (constant speed)" rel="noopener"><img src="/docs/pictures/videos_preview/com_foc_const.png"></a></td> 
    <td><a href="https://youtu.be/V-_L2w10wZk" title="Commutation vs. FOC (variable speed)" rel="noopener"><img src="/docs/pictures/videos_preview/com_foc_var.png"></a></td>       
    <td><a href="https://youtu.be/tVj_lpsRirA" title="Reliable Serial Communication" rel="noopener"><img src="/docs/pictures/videos_preview/serial_com.png"></a></td>
  </tr>
</table>


---
## Hardware
 
![mainboard_pinout](/docs/pictures/mainboard_pinout.png)

The original Hardware supports two 4-pin cables that originally were connected to the two sideboards. They break out GND, 12/15V and USART2&3 of the Hoverboard mainboard. Both USART2&3 support UART, PWM, PPM, and iBUS input. Additionally, the USART2 can be used as 12bit ADC, while USART3 can be used for I2C. Note that while USART3 (right sideboard cable) is 5V tolerant, USART2 (left sideboard cable) is **not** 5V tolerant.

Typically, the mainboard brain is an [STM32F103RCT6](/docs/literature/[10]_STM32F103xC_datasheet.pdf), however some mainboards feature a [GD32F103RCT6](/docs/literature/[11]_GD32F103xx-Datasheet-Rev-2.7.pdf) which is also supported by this firmware.

For the reverse-engineered schematics of the mainboard, see [20150722_hoverboard_sch.pdf](/docs/20150722_hoverboard_sch.pdf)

 
---
## FOC Firmware
 
In this firmware 3 control types are available:
- Commutation
- SIN (Sinusoidal)
- FOC (Field Oriented Control) with the following 3 control modes:
  - **VOLTAGE MODE**: in this mode the controller applies a constant Voltage to the motors. Recommended for robotics applications or applications where a fast motor response is required.
  - **SPEED MODE**: in this mode a closed-loop controller realizes the input speed target by rejecting any of the disturbance (resistive load) applied to the motor. Recommended for robotics applications or constant speed applications.
  - **TORQUE MODE**: in this mode the input torque target is realized. This mode enables motor "freewheeling" when the torque target is `0`. Recommended for most applications with a sitting human driver.
  
#### Comparison between different control methods

|Control method| Complexity | Efficiency | Smoothness | Field Weakening | Freewheeling | Standstill hold |
|--|--|--|--|--|--|--|
|Commutation| - | - | ++ | n.a. | n.a. | + |
|Sinusoidal| + | ++ | ++ | +++ | n.a. | + |
|FOC VOLTAGE| ++ | +++ | ++ | ++ | n.a. | +<sup>(2)</sup> |
|FOC SPEED| +++ | +++ | + | ++ | n.a. | +++ |
|FOC TORQUE| +++ | +++ | +++ | ++ | +++<sup>(1)</sup> | n.a<sup>(2)</sup> |

<sup>(1)</sup> By enabling `ELECTRIC_BRAKE_ENABLE` in `config.h`, the freewheeling amount can be adjusted using the `ELECTRIC_BRAKE_MAX` parameter.<br/>
<sup>(2)</sup> The standstill hold functionality can be forced by enabling `STANDSTILL_HOLD_ENABLE` in `config.h`. 

In all FOC control modes, the controller features maximum motor speed and maximum motor current protection. This brings great advantages to fulfil the needs of many robotic applications while maintaining safe operation.


### Field Weakening / Phase Advance

 - By default the Field weakening is disabled. You can enable it in config.h file by setting the FIELD_WEAK_ENA = 1 
 - The Field Weakening is a linear interpolation from 0 to FIELD_WEAK_MAX or PHASE_ADV_MAX (depeding if FOC or SIN is selected, respectively)
 - The Field Weakening starts engaging at FIELD_WEAK_LO and reaches the maximum value at FIELD_WEAK_HI
 - The figure below shows different possible calibrations for Field Weakening / Phase Advance
 ![Field Weakening](/docs/pictures/FieldWeakening.png) 
 - If you re-calibrate the Field Weakening please take all the safety measures! The motors can spin very fast!


### Parameters
 - All the calibratable motor parameters can be found in the 'BLDC_controller_data.c'. I provided you with an already calibrated controller, but if you feel like fine tuning it feel free to do so 
 - The parameters are represented in Fixed-point data type for a more efficient code execution
 - For calibrating the fixed-point parameters use the [Fixed-Point Viewer](https://github.com/EmanuelFeru/FixedPointViewer) tool
 - The controller parameters are given in [this table](https://github.com/EmanuelFeru/bldc-motor-control-FOC/blob/master/02_Figures/paramTable.png)


---
## Example Variants

This firmware offers currently these variants (selectable in [platformio.ini](/platformio.ini) or [config.h](/Inc/config.h)):
- **VARIANT_ADC**: The motors are controlled by two potentiometers connected to the Left sensor cable (long wired)
- **VARIANT_USART**: The motors are controlled via serial protocol (e.g. on USART3 right sensor cable, the short wired cable). The commands can be sent from an Arduino. Check out the [hoverserial.ino](/Arduino/hoverserial) as an example sketch.
- **VARIANT_NUNCHUK**: Wii Nunchuk offers one hand control for throttle, braking and steering. This was one of the first input device used for electric armchairs or bottle crates.
- **VARIANT_PPM**: RC remote control with PPM Sum signal.
- **VARIANT_PWM**: RC remote control with PWM signal.
- **VARIANT_IBUS**: RC remote control with Flysky iBUS protocol connected to the Left sensor cable.
- **VARIANT_HOVERCAR**: The motors are controlled by two pedals brake and throttle. Reverse is engaged by double tapping on the brake pedal at standstill. See [HOVERCAR video](https://www.youtube.com/watch?v=IgHCcj0NgWQ&t=).
- **VARIANT_HOVERBOARD**: The mainboard reads the two sideboards data. The sideboards need to be flashed with the hacked version. The balancing controller is **not** yet implemented.
- **VARIANT_TRANSPOTTER**: This is for transpotter build, which is a hoverboard based transportation system. For more details on how to build it check [here](https://github.com/NiklasFauth/hoverboard-firmware-hack/wiki/Build-Instruction:-TranspOtter) and [here](https://hackaday.io/project/161891-transpotter-ng).
- **VARIANT_SKATEBOARD**: This is for skateboard build, controlled using an RC remote with PWM signal connected to the right sensor cable.

Of course the firmware can be further customized for other needs or projects.


---
## Dual Inputs

The firmware supports the input to be provided from two different sources connected to the Left and Right cable, respectively. To enable dual-inputs functionality uncomment `#define DUAL_INPUTS` in config.h for the respective variant. Various dual-inputs combinations can be realized as illustrated in the following table:
| Left | Right | Availability |
| --- | --- | --- |
| ADC<sup>(0)</sup> | UART<sup>(1)</sup> | VARIANT_ADC |
| ADC<sup>(0)</sup> | {PPM,PWM,iBUS}<sup>(1)</sup> | VARIANT_{PPM,PWM,IBUS} |
| ADC<sup>(0)</sup> | Sideboard<sup></sup><sup>(1*)</sup> | VARIANT_HOVERCAR |
| UART<sup>(0)</sup> | Sideboard<sup>(1*)</sup> | VARIANT_UART |
| UART<sup>(1)</sup> | Nunchuk<sup>(0)</sup> | VARIANT_NUNCHUK |

<sup>(0)</sup> Primary input: this input is used when the Auxilliary input is not available or not connected.<br/>
<sup>(1)</sup> Auxilliary input: this inputs is used when connected or enabled by a switch<sup>(*)</sup>. If the Auxilliary input is disconnected, the firmware will automatically switch to the Primary input. Timeout is reported **only** on the Primary input.

With slight modifications in config.h, other dual-inputs combinations can be realized as:
| Left | Right | Possibility |
| --- | --- | --- |
| Sideboard<sup>(1*)</sup> | UART<sup>(0)</sup> | VARIANT_UART |
| UART<sup>(0)</sup> | {PPM,PWM,iBUS}<sup>(1)</sup> | VARIANT_{PPM,PWM,IBUS} |
| {PPM,PWM,iBUS}<sup>(1)</sup> | Nunchuk<sup>(0)</sup> | VARIANT_{PPM,PWM,IBUS} |


---
## Flashing

Right to the STM32, there is a debugging header with GND, 3V3, SWDIO and SWCLK. Connect GND, SWDIO and SWCLK to your SWD programmer, like the ST-Link found on many STM devboards.

If you have never flashed your sideboard before, the MCU is probably locked. To unlock the flash, check-out the wiki page [How to Unlock MCU flash](https://github.com/EmanuelFeru/hoverboard-firmware-hack-FOC/wiki/How-to-Unlock-MCU-flash).

Do not power the mainboard from the 3.3V of your programmer! This has already killed multiple mainboards.

Make sure you hold the powerbutton or connect a jumper to the power button pins while flashing the firmware, as the STM might release the power latch and switches itself off during flashing. Battery > 36V have to be connected while flashing.

To build and flash choose one of the following methods:

### Method 1: Using Platformio IDE

- open the folder in the IDE of choice (vscode or Atom)
- press the 'PlatformIO:Build' or the 'PlatformIO:Upload' button (bottom left in vscode).

### Method 2: Using Keil uVision

- in [Keil uVision](https://www.keil.com/download/product/), open the [mainboard-hack.uvproj](/MDK-ARM/)
- if you are asked to install missing packages, click Yes
- click Build Target (or press F7) to build the firmware
- click Load Code (or press F8) to flash the firmware.

### Method 3: Using Linux CLI

- prerequisites: install [ST-Flash utility](https://github.com/texane/stlink).
- open a terminal in the repo check-out folder and if you have definded the variant in [config.h](/Inc/config.h) type:
```
make
```
or you can set the variant like this
```
make -e VARIANT=VARIANT_####
```
- flash the firmware by typing:
```
make flash
```
- or
```
openocd -f interface/stlink-v2.cfg -f target/stm32f1x.cfg -c flash "write_image erase build/hover.bin 0x8000000"
```

### Method 4: MacOS CLI
- prerequisites:  first get brew https://brew.sh
- then install stlink ST-Flash utility

#### Using Make
```
brew install stlink
```
- open a terminal in the repo check-out folder and if you have definded the variant in [config.h](/Inc/config.h) type:
```
make
```
or you can set the variant like this
```
make -e VARIANT=VARIANT_####
```
If compiling fails because something is missing just install it with brew AND leave a comment to improve this howto or pull request ;-)

- flash the firmware by typing:
```
make flash
```
- if unlock is needed
```
make unlock
```

#### Using platformio CLI

```
brew install platformio
platformio run -e VARIANT_####
platformio run –target upload -e VARIANT_####
```
If you have set default_envs in [platformio.ini](/platformio.ini) you can ommit -e parameter


---
## Troubleshooting
First, check that power is connected and voltage is >36V while flashing.
If the board draws more than 100mA in idle, it's probably broken.

If the motors do something, but don't rotate smooth and quietly, try to use an alternative phase mapping. Usually, color-correct mapping (blue to blue, green to green, yellow to yellow) works fine. However, some hoverboards have a different layout then others, and this might be the reason your motor isn't spinning.

Nunchuk not working: Use the right one of the 2 types of nunchuks. Use i2c pullups.

Nunchuk or PPM working bad: The i2c bus and PPM signal are very sensitive to emv distortions of the motor controller. They get stronger the faster you are. Keep cables short, use shielded cable, use ferrits, stabilize voltage in nunchuk or reviever, add i2c pullups. To many errors leads to very high accelerations which triggers the protection board within the battery to shut everything down.

Recommendation: Nunchuk Breakout Board https://github.com/Jan--Henrik/hoverboard-breakout

Most robust way for input is to use the ADC and potis. It works well even on 1m unshielded cable. Solder ~100k Ohm resistors between ADC-inputs and gnd directly on the mainboard. Use potis as pullups to 3.3V.


---
## Diagnostics
The errors reported by the board are in the form of audible beeps:
- **1 beep  (low pitch)**: Motor error (see [possible causes](https://github.com/EmanuelFeru/bldc-motor-control-FOC#diagnostics))
- **2 beeps (low pitch)**: ADC timeout
- **3 beeps (low pitch)**: Serial communication timeout
- **4 beeps (low pitch)**: General timeout (PPM, PWM, Nunchuk)
- **5 beeps (low pitch)**: Mainboard temperature warning
- **1 beep slow (medium pitch)**: Low battery voltage < 36V
- **1 beep fast (medium pitch)**: Low battery voltage < 35V
- **1 beep fast (high pitch)**: Backward spinning motors

For a more detailed troubleshooting connect an [FTDI Serial adapter](https://s.click.aliexpress.com/e/_AqPOBr) or a [Bluetooth module](https://s.click.aliexpress.com/e/_A4gkMD) to the DEBUG_SERIAL cable (Left or Right) and monitor the output data using the [Hoverboard Web Serial Control](https://candas1.github.io/Hoverboard-Web-Serial-Control/) tool developed by [Candas](https://github.com/Candas1/).

---
## Projects and Links

- **Original firmware:** [https://github.com/NiklasFauth/hoverboard-firmware-hack](https://github.com/NiklasFauth/hoverboard-firmware-hack)
- **[Candas](https://github.com/Candas1/) Hoverboard Web Serial Control:** [https://candas1.github.io/Hoverboard-Web-Serial-Control/](https://candas1.github.io/Hoverboard-Web-Serial-Control/)
- **[RoboDurden's](https://github.com/RoboDurden) online compiler:** [https://pionierland.de/hoverhack/](https://pionierland.de/hoverhack/) 
- **Hoverboard hack for AT32F403RCT6 mainboards:** [https://github.com/cloidnerux/hoverboard-firmware-hack](https://github.com/cloidnerux/hoverboard-firmware-hack)
- **Hoverboard hack for split mainboards:** [https://github.com/flo199213/Hoverboard-Firmware-Hack-Gen2](https://github.com/flo199213/Hoverboard-Firmware-Hack-Gen2)
- **Hoverboard hack from BiPropellant:** [https://github.com/bipropellant](https://github.com/bipropellant)
- **Hoverboard breakout boards:** [https://github.com/Jan--Henrik/hoverboard-breakout](https://github.com/Jan--Henrik/hoverboard-breakout)

<a/>

- **Bobbycar** [https://github.com/larsmm/hoverboard-firmware-hack-bbcar](https://github.com/larsmm/hoverboard-firmware-hack-bbcar)
- **Wheel chair:** [https://github.com/Lahorde/steer_speed_ctrl](https://github.com/Lahorde/steer_speed_ctrl)
- **TranspOtterNG:** [https://github.com/Jan--Henrik/transpOtterNG](https://github.com/Jan--Henrik/transpOtterNG)
- **ST Community:** [Custom FOC motor control](https://community.st.com/s/question/0D50X0000B28qTDSQY/custom-foc-control-current-measurement-dma-timer-interrupt-needs-review)

<a/>

- **Telegram Community:** If you are an enthusiast join our [Hooover Telegram Group](https://t.me/joinchat/BHWO_RKu2LT5ZxEkvUB8uw)


---
## Contributions

Every contribution to this repository is highly appreciated! Feel free to create pull requests to improve this firmware as ultimately you are going to help everyone. 

If you want to donate to keep this firmware updated, please use the link below:

[![paypal](https://www.paypalobjects.com/en_US/NL/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=CU2SWN2XV9SCY&currency_code=EUR&source=url)


---

