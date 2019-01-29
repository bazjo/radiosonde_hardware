# RS41_ReverseEngineering
For general information about radiosondes [Wikipedia](https://de.wikipedia.org/wiki/Radiosonde).

For more information about the RS41 [my website](https://example.com).

The blockwise structure of the RS41 is described in the following. The schematic in Eagle format, Logic Analyzer recordings of the functional blocks and high-resolution scans of the printed circuit boards are also provided.

The examination of the RPM411 daughter board with barometric sensor and the OIF411 Ozone interface can be found, as soon as available, in the separate subfolders.

Pull requests with improvements, translations and bug fixes are welcome!

# ToDo
* identify unidentified components
* find unknown connections
* identify values of passive components
* find out which of the resistors entered in the schematic are in fact ESD-Surpressors etc..
* front end measurement in the climate chamber
* detailed description of the SPI bus
* detailed description of the UART between GPS and MCU
* functional investigation NFC interface
* sniffing of communication between RI41 Groundcheck Device and RS41
* receive and reverse flashdump of the controller

# Introduction
The sonde is to be divided into six functional blocks, which are highlighted in the following picture.

* [Power Supply](#Power Supply)
* [Microcontroller](#microcontroller)
* [Front end](#frontend)
* [GPS](#gps)
* [Radio](#radio)
* [Interface](#interface)

Reverse engineering is complicated by the fact that the circuit board has four layers.

# Power supply
![Power Supply](__used_asset__/supply_sch.png?raw=true "Power Supply")

The power supply can be divided into three parts

* a boost converter generates 3.8 V from the variable battery voltage
* three Low Dropout Regulators (LDOs) each generate a 3 V rail for different blocks
* a hard-wired logic determines the operating state of the boost converter and thus of the sonde

## Boost converter
A [TPS61200](http://www.ti.com/lit/ds/symlink/tps61200.pdf) `U502` from TI is used for the boost converter, whose circuitry corresponds to the typical application. The input circuit contains an SMD fuse `R502` and a clamping diode `D501`. Between battery and boost converter there is a P-channel MOSFET `Q501`, which is closed by the pull-up resistor `R501` during storage. `Q501` can be opened or closed by a hardwired logic (see below) to switch the sonde on or off.

## LDOs
TVS70030](http://www.ti.com/lit/ds/symlink/tlv700-q1.pdf) from TI are used. `U501` generates the voltage for the microcontroller (MCU), `U503` for the measurement frontend and `U504` for the GPS module. Pin 4 of the LDOs, which is NC according to the data sheet, is decoupled against ground, probably so that pin compatible versions like the [MAX8887](https://datasheets.maximintegrated.com/en/ds/MAX8887-MAX8888.pdf) can be used.

## Hard wired logic
The P-channel MOSFET Q501 discussed above is controlled by an N-channel MOSFET `Q502`. 
* The probe is on when this transistor is closed, so its gate is HIGH.
* The probe is off when this transistor is open, so its gate is LOW.

Via `R506` and `D502` a one-way rectified signal of the NFC coil reaches the gate. This allows the probe to be switched on via NFC. Furthermore this signal is also used for communication with the RI41 Groundcheck Device via the voltage divider `R510` and `R514`/`C525` to the MCU.

Once the probe is switched on, it is kept in this state by the switched battery voltage, which is fed to the gate via `R505`.

To switch it off again, the MCU can close the N-channel MOSFET `Q503`, which brings the gate from `Q502` to LOW.

The button at the bottom of the probe `S501` also switches the gate from `Q502` via `R507` to HIGH. So that the microcontroller can also query the status of the button, the gate voltage of `Q502` is fed via the voltage dividers `R506` and `R512`/`C524` to an ADC input of the MCU. The lower voltage drop via `R507`, which leads to a higher gate voltage when this is pressed, is evaluated here.

Finally, the battery voltage itself can also be evaluated via the voltage dividers `R508` and `R512`/`C524`.

# Microcontroller
![Microcontroller](__used_asset__/mcu_sch.png?raw=true "Microcontroller")

The microcontroller is a [STM32F100C8](https://www.st.com/resource/en/datasheet/stm32f100c8.pdf) `U101` from ST in LQFP48 package, which gets its clock from the 24 MHz crystal `X101`. Apart from the fact that all IO pins are occupied, it is only worth mentioning that RC low pass filters are present at many outputs.

# Measuring frontend
![Measuring frontend](__used_asset__/frontend_sch.png?raw=tr

Translated with www.DeepL.com/Translator
