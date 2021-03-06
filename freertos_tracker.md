# Design notes for an APRS tracker

As part of IrishSat's high altitude balloon launches, we want a tracker that pretty much does what lightaprs does,
but with some more flexibillity for adding measurements from multiple sensors and possibly different
communications modes, all to be measured simultaneously. These are some notes about how using FreeRTOS can enable that.

This is a work in progess, assembling ideas before building anything.

## References

* [LightAPRS](https://github.com/lightaprs)
* [HamShield](https://github.com/EnhancedRadioDevices/HamShield)
* [MicroModem](https://github.com/markqvist/MicroModem)
* [libaprs from MicroModem](https://github.com/markqvist/LibAPRS)
* [Pico HAB Tracker](https://github.com/daveake/pico-tracker) <-- Raspberry Pi Pico
* [https://blog.adafruit.com/2017/09/21/lidsat-1-dra-818-vhf-uhf-feather-m0-fm-repeater-cubesat-in-3d-printed-enclosure/]
* [F4GOH dra818 tracker](https://hamprojects.wordpress.com/2015/07/01/afsk-dra818-aprs-tracker/)
* [NovaHammy, nice complete circuit for dra818](https://github.com/cogwheelcircuitworks/NovaHammy/blob/master/hardware/NovaHammyRevXDsch.pdf)


## What we want

* Multiple custom sensors (fabricated in on-campus clean rooms), simultaneous measurement, transmit+receive = FreeRTOS
* Enough RAM for an RTOS, low-power consumption = Raspberry Pi Pico
* Over-the-horizon comms = APRS, DRA818V radio module with minicircuits LFCN-160+ low pass filter. Custom breakout board from OSH Park
* Higher bitrate direct comms = LoRa [900 MHz for now](https://www.adafruit.com/product/3072)
* GPS for high altitude = u-blox [Generic M8030-DR based module](https://www.amazon.com/dp/B07PRDY6DS), but can use a ublox max module.
Notes on GPS: can save settings to flash. Maybe just do that, but keep code to flash new chips... After writing the code, I see that
you don't need to upload it every time if you have flash... otherwise you need to detect if GPS restarted and re-upload. On low power mode,
it restarts if it can't find a fix on start up.

## Typical non-RTOS code
1. Main loop: GPS is set to low power mode (1 Hz), so sleep for one second then read in (flush)
the GPS buffer. Every 1 minute, assemble an APRS packet and but that in to a TX buffer. All other operations (receiving packets,
collecting other data sources, processing other data) is done within the timing of this one loop, which becomes complicated
and makes low-power operation difficult (since it has to always be awake).
1. Receive: ADC set to 9600 Hz. On every ADC interrupt, read the input audio value, send it to a FIFO, then do HDLC decode on the FIFO.
1. Transmit: Use PWM for audio output (duty cycle proportional to output voltage).
On the same ADC interrupt, advance the phase of the sine wave output based on whether the current tone is 1200 or 2200 Hz.
Every 8 ADC interrupts, read in the next bit to send from the TX buffer.
If it's a 1, change the modulation frequency by changing how much phase gets advanced each ADC interupt.

### HDLC decoding
Nearly everyone does this: some convolution for mark and space detection, then compare which was larger - the mark or the space.
On every transition from 1 to 0 or 0 to 1, some kind of digital PLL is nudged.
Direwolf has a nice description implimentation. They have a 32 signed int as a counter that increments by (2^32)\*(1200) per second
per ADC time step. When it overflows (goes negative), takes a sample. When the PLL needs to be nudged, just divide the counter by a
factor (e.g., 2), moving it closer to zero. Eventually the counter will be at zero at each transition.

## Ideas for new RTOS code
Event-based tasks are run in parallel, and the MCU can sleep whenever it's not doing anything. Can essentially follow the lightAPRS
code, but use task based concepts.
* **APRS Tracker Task**: Every 1 minute (triggered off of some counter, or simply put it to sleep for 1 minute at the end of the loop)
grab the GPS position and any other telemtry for an APRS location packet. Make a struct with the binary data
and number of bytes. Send a pointer to the completed packet struct to the *transmit queue*. The transmit queue is a queue
of pointers to packet structs that will be sent. We can use the afsk struct from libaprs. (basically port libaprs).
* **Other APRS Packets**: Do the same above (wake up periodically, take data, build a packet) and add it to the transmit queue
* **Transmit Interrupt Service Routine**: every 1/1200th of a second, see if there's anything in the transmit queue.
If TX was off, do CSMA to see if we can get a channel, then key up transmitter and
queue up some number of prefix sync bytes. The ISR will get data from and modify the afsk struct like libaprs does to keep track
of what has been transmitted so far.
Send one bit (change the PWM between 1200 and 2000 Hz depending on the data). If 5 1s are in a row, bit stuff and send the 1 next time
(rather than get another one). When transmit queue is empty, send
then ending suffix, turn off the PTT, and disable the 1200 Hz timer. Unlike non-RTOS versions, maybe we just use PWM directly rather
than as DDS of a sine wave. But DDS sine wave is pretty cool, could just do that.
* **Receive ISR**: On ADC tick, read in a value and send it via notification to the *Receive Processing Task*.
* **Receive Processing Task**: On every bit, add it to a FIFO. Do the HDLC decoding. cosine + sine correlation detection.
Centered/shifted so that zero phase is the middle of the vector. maybe do what direwolf does and decode in parallel with differen gain.
Then low pass filter (the cos and sin correlator is terrible at filtering, and bell 202 has symbol rates at the same freq as the
modulation!!!) [Maybe try a transposed direct form II biquad?](https://ccrma.stanford.edu/~jos/filters/Biquad_Software_Implementations.html)
cool effect that TDFII and DFI have [internal overflow protection](https://ccrma.stanford.edu/~jos/filters/Direct_Form_I.html)
* **LoRa Task**: Every so many seconds, assemble LoRa packet and send.
