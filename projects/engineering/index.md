[Back to Home](../../index.md)

**Analogue to Digital Voice Recorder:** https://github.com/claynimmo/Analogue-to-Digital-Voice-Recorder Uses an arduino device to sample the voltage output of a custom designed input conditioning circuit, including DC offset removal, anti-alising filters, and a gain component. The arduino transmits the voltage encoded in a 10-bit integer across the serial port, where it is then read through a python script for centring the signal and converting it into a wav file.
