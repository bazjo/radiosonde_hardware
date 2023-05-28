# Meteomodem M20

## Description

![Assembled Radiosonde](assembled.jpg)
**Assembled Radiosonde**

![Exploded View](exploded_view.jpg)
**Exploded View**

### MCU and GPS
![MCU/GPS Block](__used_asset__/m20.svg)

### Radio
![Radio Block](__used_asset__/m20-radio.svg)

The Analog Devices ADF7012 FSK transmitter gets its reference clock from the 8 MHz MCO output of the MCU. The MCU handles the control interface of the radio via bit-banging over standard port pins. TX data is sent asynchronously to the ADF7012 TXDATA pin. The MCU generates transmit data with the help of TIMER21 and its TIM21_CH1 output. A PA (unknown type) boosts the output signal.

Modulation is 2FSK with 5.5 kHz deviation. Data bits are sent Biphase-M encoded in short bursts with 4800 bit/s (9600 smbols/s).

### Analog
![Analog Block](__used_asset__/m20-analog.svg)

### Powersupply
![Power Supply](__used_asset__/m20-power.svg)

## Theory of Operations

![Main PCB Scan](pcb.jpg)
**Main PCB Scan**

![Sensor Boom Scan](sensorboom.jpg)
**Sensor Boom Scan**

Still some stuff missing here...

## Parts List

Still some stuff missing here...

## Detail Photos

![Detail](detail/detail01.jpg)

![Detail](detail/detail02.jpg)

![Detail](detail/detail03.jpg)

![Detail](detail/detail04.jpg)

![Detail](detail/detail05.jpg)

