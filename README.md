# USFSMAX MMC5983 Motion Coprocessor Module

## Introduction
This repository provides background information and practical code examples for implementing Tlera Corporation's follow-on product to the [USFSMAX motion coprocessor](https://www.tindie.com/products/onehorse/max32660-motion-co-processor/), the [USFSMAX module and carrier](https://www.tindie.com/products/onehorse/usfsmax-module-and-carrier/). In the process of introducing the USFSMAX into the marketplace, we learned several areas for improvement in the original design:
* The validity of the bench calibration can degrade from board-level strain impacting the response characteristics of the individual sensors
* This is typically results from thermal strain encountered when making solder connections to the plated through-holes on the USFSMAX PCB
* The LIS2MDL magnetometer is sufficiently noisy to cause the indicated heading to fluctuate +/- ~0.2deg. Perhaps there is a lower noise magnetometer available?
* The form factor of the original USFSMAX could stand further optimization, especially to facilitate incorporation into OEM products

Optimization efforts in these areas resulted in the USFSMAX module and carrier product shown (disassembled) below. The module has been shrunk to a 1/2" x 1/2" PCB and connects to the carrier board using two [Molex SlimStack](http://www.literature.molex.com/SQLImages/kelmscott/Molex/PDF_Images/987651-8281.pdf) micro mezzanine connectors. The connector strategy has the inherent advantage of providing both mechanical and thermal isolation of the USFSMAX module from the carrier or motherboard. As a practical matter, the module can be removed from the carrier board so it can be soldered without the risk of compromising the factory bench calibration. Furthermore, OEM products can be developed with mating Molex mezzanine connectors so that a pre-calibrated USFSMAX module can simply snapped into place to complete assembly. Please note that some care should be taken when assembling and dis-assembling the module from the carrier board. The faces of the module and carrier boards should remain as close to parallel as possible. Do not "Pry" the boards together or apart from one edge but instead [use even force on both connectors at once](https://www.molex.com/pdm_docs/as/5050660000-A06.pdf).

![alt text](https://user-images.githubusercontent.com/5760946/102443217-0b9a9580-3fdb-11eb-9c88-19f36b8adc16.jpg)

I should mention here that the orientation of the USFSMAX module on the test object or vehicle is important. The figure below shows a USFSMAX module and carrier assembly with the world coordinate system properly oriented. The USFSMAX conforms to the ["East-North-Up" (ENU)](https://en.wikipedia.org/wiki/Local_tangent_plane_coordinates) sensor coordination system convention. When installed on the test object/vehicle, the component side of the USFSMAX module should be facing up and the "Y" axis should point in the direction of travel (forward). ***It is not recommended to install the USFSMAX module with the sensor-side of the board pointing downward.***

![alt text](https://user-images.githubusercontent.com/5760946/102438031-6a0e4680-3fd0-11eb-9ebd-0b2a075cc67a.jpg)


## MMC5983-Based USFSMAX Performance
The [USFSMAX hardware](https://hackaday.io/project/160283-max32660-motion-co-processor/log/171113-final-hardware-design) is an excellent platform for enhanced sensor calibration and fusion algorithm development, as [discussed earlier](https://github.com/gregtomasch/USFSMAX). This variant of the USFSMAX product uses the [MMC5983 magnetometer](https://www.mouser.com/datasheet/2/821/Memsic_09102019_MMC5983MA_Datasheet_Rev_A-1635338.pdf). It is an [anisotropic magnetoresistance (AMR) sensor](https://www.allaboutcircuits.com/news/amr-current-sensor-roundup/) like the [ST Microelectronics LIS2MDL](https://www.st.com/resource/en/datasheet/lis2mdl.pdf) but with several key differences:
* Dynamic range better-matched to measuring the geomagnetic field (+/-8G versus +/-50G)
* Greatly improved resolution (18bit, 16G range versus 16bit, 100G range)
* Much better noise level (0.4mG RMS versus 3mG RMS)
* "Set and Reset" capability to remove offsets from high field exposure ([Similar to Honeywell AMR sensors](https://neurophysics.ucsd.edu/Manuals/Honeywell/HMC%201001%20and%20HMC%201002.pdf))

The improvements in resolution and noise delivered by the MMC5983 do indeed translate to better USFSMAX heading estimation performance. ***The heading estimate noise declined from ~0.093 to ~0.015deg RMS.*** The figure below shows heading error characterization data from two prototype MMC5983-equipped USFSMAX units plotted with similar data from [LIS2MDL-equipped units shown earlier](https://hackaday.io/project/160283-max32660-motion-co-processor/log/172109-unit-to-unit-variation-and-on-board-residual-hard-iron-error-correction).

![alt text](https://user-images.githubusercontent.com/5760946/102422856-82746600-3fbc-11eb-81e3-9b4ca88f852d.jpg)

Although the sample size is small, we can clearly see the total amplitudes of the MMC5983-based heading error curves are significantly smaller and the data sets are much less noisy than the LIS2MDL-based curves

## USFSMAX Firmware Updates/Changes
The behavior of the new USFSMAX module is quite similar to the original product but new features have been added and existing features have been refined. These include:
* Dynamic hard iron correctors 
* Gyroscope bias offset calibration
* User-selectable USFSMAX I2C slave address
* USFSMAX deep sleep mode and wake-up
* Firmware ID byte

The following sections will present relevant information for each of these general areas.

### Dynamic Hard Iron (DHI) Correctors
The DHI correctors are quite similar to those of the original USFSMAX product but are deployed slightly differently after receiving valuable user input. ***Please read this section carefully to avoid unanticipated behavior from the USFSMAX module.***
1. The DHI corrector can be enabled or disabled at startup by user command from the host MCU. For the test sketches in this repository, this is handled by definitions in the "config.h" tab
2. If there is a valid DHI correction in the USFSMAX's EEPROM, it is loaded and used at startup if the DHI corrector is enabled
3. The DHI corrector only collects new training data at startup *only* if the DHI corrector is enabled and there is no valid calibration in the EEPROM. If the 3D corrector is active, this feature can give a "better than nothing" hard iron offset estimate but the result will probably be sub-optimal. If the 2D corrector is active, the training algorithm will probably not converge unless the motion constrained to the X- Y plane
4. The recommended method for training the DHI corrector is to reset it from the host MCU and purposefully manipulate the test object/vehicle to train the corrector under optimal conditions
5. Once the training process is complete, the new DHI correction estimate is automatically stored in the USFSMAX's EEPROM
6. When the new hard iron correction estimate is complete, a quality figure-of-merit for the estimate is available as well. The user can then choose to let the estimate stand or reset the corrector and try again
7. The quality figure-of-merit ***is not*** stored in the EEPROM

***So, the user decides when to use the DHI corrector and controls the conditions under which the corrector training data set is collected.*** If the hard iron correction offset estimate becomes stale, the corrector can be reset at will to begin re-training. If the magnetic environment is stable, the USFSMAX works well by relying on the hard iron offset corrections stored in the EEPROM. The quality of the hard iron offset correction estimate is judged by the [R-squared metric](https://en.wikipedia.org/wiki/Coefficient_of_determination). When the variation in the magnetic data set NOT explained by hard iron offsets, the Mx, My and Mz correction estimates tends to zero and R-squared tends to 1.0. Ultimately it is up to the user to interpret the R-squared value and either accept the hard iron correction offset estimate or reject it and reset the corrector.

There are actually two user-selectable versions of the DHI corrector available: 3-D and 2-D. They are both capable of yielding excellent hard iron correction offset estimates but are intended for different use cases. If the test object to which the USFSMAX is attached is small/light and can easily be tumbled in 3-Space, the 3-D corrector is the best choice. If the test object is unwieldy or its motion is largely constrained to the X-Y (horizontal) plane the 2-D corrector may be a better choice. Both correctors collect a pre-determined number of data points at enforced spatial separation before calculating the final correction estimate. When the estimate is complete, the hard iron corrector status bit in the calibration status register toggles true (0->128) and the R-squared value populates the appropriate data registers. The DHI corrector version is chosen by user command from the host MCU (defined in the "config.h" tab of the example sketches in this repository). To get the best hard iron correction estimate:
* 3-D corrector: Tumble the USFSMAX (attached to the test object) in 3-Space trying to sample all possible orientations. Training should take ~60-90s. If done properly, R-square >= 0.98 is quite normal
* 2-D corrector: Rotate the USFSMAX (attached to the test object) in the X-Y (horizontal) plane. The 2D corrector algorithm has been updated correct for tilt if data points are within +/-10deg of level. For larger tilts, data are rejected. Training should complete in ~20 revolutions in the X-Y plane. If done properly, R-square >= 0.98 is quite normal
* ***Note that the host MCU must upload the local geomagnetic constants to the USFSMAX at startup for the 2D corrector to work properly***

#### Magnetic Constants
Your local geomagnetic constants are necessary to achieve the best heading accuracy. The constants in question are:
* Vertical field strength (M_V)
* Horizontal field strength (M_H)
* Magnetic declination (MAG_DECLINATION)

These constants are available from [online calculators](https://www.ngdc.noaa.gov/geomag/calculators/magcalc.shtml?#igrfwmm) for your latitude and longitude. Once your geomagnetic constants have been looked up:
* Define an entry for your location in the "Magnetic Constantants" section of the "config.h" tab

  Example:
  
      #define SUNNYVALE_CA_USA
  
* Define a new magnetic constants block in the "Magnetic Constants" section of the "def.h" tab

  Example:
  
      #ifdef SUNNYVALE_CA_USA
          #define M_V                                   41.8128f
          #define M_H                                   23.2519f
          #define MAG_DECLINIATION                      13.2197f
      #endif
  
* Comment out all location definitions in the "Magnetic Constants" section of the "config.h" tab except for your own

The DHI correction function is activated by defining "ENABLE_DHI_CORRECTOR" as "0x01". The 2-D corrector is selected by defining "USE_2D_DHI_CORRECTOR" as "0x01", otherwise the 3-D corrector is used instead. These definitions along with some simple instructions are located in the "BASIC SETUP" section in the "config.h" tab of the sketch.

### Gyroscope Bias Offset Calibration
The original version of the USFSMAX automatically performed gyroscope calibration at startup. The gyroscope biases need to be measured when the USFSMAX is truly at rest. Several users have pointed out that this isn't always the case at startup, depending on the test object/vehicle. Now the gyroscope calibration is done only in response to a command from the host MCU so that the calibration conditions can be guaranteed. All of the host MCU sketches in this repository show how to handle this in the host MCU startup sequence.

### Changing the USFSMAX I2C Slave Address
The default I2C slave address of the USFSMAX is 0x57. In some cases, it may be desirable to have more than one USFSMAX module on the same I2C bus. There can also be possible address conflicts with other I2C devices. Now the USFSMAX I2C address can be changed using the USFSMAX reset pin and an I2C command. The code snippet below shows how this is done:
```
#include "I2Cdev.h"
#include "USFSMAX.h"

#define NEW_I2C_SLAVE_ADDR                 0x6C                            // USFSMAX register for changing the I2C address is 0x6C
#define SENSOR_0_WIRE_INSTANCE             Wire
#define I2C_CLOCK                          1000000                         // Run state I2C clock frequency = 1MHz
#define MAX32660_0_SLV_ADDR                (0x57)                          // Default USFS MAX I2C slave address
#define MAX32660_1_SLV_ADDR                (0x57)                          // Default USFS MAX I2C slave address
#define ALT_MAX32660_SLV_ADDR              (0x6A)                          // Alternate USFS MAX I2C slave address
#define USFSMAX_1_RESET_PIN                11                              // GPIO pin connected to the USFSMAX_1 module

// Instantiate class objects
I2Cdev  i2c_0(&SENSOR_0_WIRE_INSTANCE);
USFSMAX USFSMAX_0(&i2c_0, 0);

// Declare global scope variables
uint8_t max32660_slv_addr     = MAX32660_SLV_ADDR;                         // Default I2C slave address
uint8_t max32660_new_slv_addr = ALT_MAX32660_SLV_ADDR;                     // New I2C slave address

// Declare global scope utility functions
void change_I2C_slaveADDR(uint8_t new_addr)

void setup()
{
  // Set up the reset pin and hold USFSMAX_1 in reset state
  pinMode(USFSMAX_0_RESET_PIN, OUTPUT);
  digitalWrite(USFSMAX_1_RESET_PIN, LOW);
 
  // Open serial port
  Serial.begin(115200);
  delay(1000);
 
  // Initialize the I2C bus
  SENSOR_0_WIRE_INSTANCE.begin();
  delay(100);
  SENSOR_0_WIRE_INSTANCE.setClock(100000);                                 // Set I2C clock speed to 100kHz for configuration
  delay(2000);
 
  Serial.println("USFXMAX_1 holding in the reset state...");
  delay(2000)
  Serial.println("");
  Serial.println("");
 
  // Initialize USFSMAX_0 while USFSMAX_1 holds in the reset state
  Serial.print("Initializing USFSMAX_0...");
  Serial.println("");
  USFSMAX_0.init_USFSMAX();                                                // Configure USFSMAX and sensors
  
  // Set the USFSMAX_0 I2C slave address to the alternative
  change_I2C_slaveADDR(max32660_new_slv_addr);                             // USFSMAX_0 slave I2C address changed
  SENSOR_0_WIRE_INSTANCE.setClock(I2C_CLOCK);                              // Set the I2C clock to high speed for run-mode
  delay(100);
  Serial.print("USFSMAX_0 I2C slave address updated...");
  Serial.println("");
  
  /*
    Release USFSMAX_1 by setting the reset pin high (digitalWrite(USFSMAX_1_RESET_PIN, HIGH)) and configure as usual.
	USFSMAX_0 will now respond at I2C address 0x6C and USFSMAX_1 will respond at the default I2C address, 0x57...
  */
}

void loop()
{
}

void change_I2C_slaveADDR(uint8_t new_addr)
{
  i2c_0.writeByte(max32660_slv_addr, NEW_I2C_SLAVE_ADDR, (new_addr<<1));
  delay(100);
}
```

## Example Host MCU Sketches
This repository contains example host MCU Arduino sketches to demonstrate basic use of the USFSMAX motion coprocessor.

### STM32L4
This version is written for the [Tlera Dragonfly STM32L476 development board](https://www.tindie.com/products/tleracorp/dragonfly-stm32l47696-development-board/) using the [STM32L4 core for the Arduino IDE](https://github.com/GrumpyOldPizza/arduino-STM32L4). The USFSMAX breakout board can be connected to the MCU development board on a prototyping "Breadboard" or it can be "Piggybacked" using pin headers.

### Teensy 3.x
This version was tested using the [Teensy 3.2 and 3.6 development boards](https://www.pjrc.com/teensy/pinout.html) in the piggyback configuration. The Teensy-specific ["i2c_t3"](https://github.com/stevenvo/arduino-libraries/tree/master/i2c_t3) I2C library is used here and will need to be installed in your sketchbook folder.

### ESP32
This verson was tested using the [Tlera ESP32 development board](https://www.tindie.com/products/onehorse/smallest-esp32-development-board/), which has been retired. However, the code can be used with the ESP32 development board of your choice. For your particular board, simply go to the "config.h" tab and re-define:
* "INT_PIN" USFSMAX data ready GPIO
* "LED_PIN" utility LED GPIO
* "USFS_GND_PIN "GND" GPIO (Piggyback configuration)
* "USFS_VCC" "3V3" GPIO (Piggyback configuration)
* "SDA_PIN" I2C data GPIO
* "SCL_PIN" I2C clock GPIO

The sketch configures the USFSMAX at startup and demonstrates the basic AHRS functions and sensor data acquisition according to the [USFSMAX register map](https://github.com/gregtomasch/USFSMAX/blob/master/USFSMAX_Reg_Map_0.0.pdf) included in this repository. The DHI corrector can be enabled/configured in the "config.h" tab of the sketch to evaluate its operation in 2-D and 3-D modes. The sketch's serial interface supports reset of the DHI corrector selected at startup.

### I2C Data Transactions
**The I2C slave address of the USFSMAX is currently set to 0x57.** There are plans to make the I2C slave address user-selectable and this feature should be available soon. An important aspect of the USFSMAX's I2C slave bus is that there is always a finite delay between when the host MCU requests to read data and when that data is available to be read. Consequently, the USFSMAX will work best with host MCU's that support [I2C clock stretching](https://www.i2c-bus.org/clock-stretching/).

It should be noted that the USFSMAX data registers are set up for efficient I2C data transfer by "Burst reading". That is, a starting data register is specified in the I2C transaction and additional data registers are read sequentially for the total number of bytes specified in the transaction. To improve overall I2C read request response time to the host MCU, not all data registers can be the starting register of an I2C read transaction. Data registers that are directly addressable at the beginning of a read transaction are highlighted in yellow in the register map. So, for example, if a user wanted to read just the two Y-axis gyroscope sensor data bytes from registers 0x07 and 0x08, that is not supported. Instead, the read transaction would begin at register 0x05 and be four bytes long to include the two Y-axis gyroscope data bytes desired.
