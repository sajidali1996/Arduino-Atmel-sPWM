# Arduino-Atmel-sPWM

#### Implementation of an sPWM signal on Ardunio and Atmel micros

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/pulses_1.JPG?raw=true "Figure")

## Introduction

This document covers key concepts in the generation of a sinusoidal signal using dynamic pulse width modulation (PWM) as well as it’s implementation on Atmel microcontrollers, including Arduino boards. The code in this document is mostly C with use of Arduino code/libraries in some cases. This document assumes the reader has a fundamental under- standing of C programming. It is the aim of this document to help the hobbyist or student make rapid progress in understanding and implementing an sPWM signal. If you find any mistakes in this document or can think of improvements, perhaps you are a university tutor or lecturer, make a suggestion in the associated forum. Any significant contributors will be listed as an author

## Key PWM Concepts
###Basic PWM

Pulse width modulation’s (PWM) main use is to control the power supplied to electric circuits, it does this by rapidly switching a load on and off. Another way of thinking of it is to consider it as a method for a digital system to output an analogue signal. The figure below shows an example of a PWM signal.

![Figure 1-1](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/basicPWM_3.png?raw=true "Figure 1.1")

There are two properties to a PWM signal, the frequency which is determined by the period of the signal and the duty cycle which is determined by the high-time of the signal. The signal in Figure 1.1 has a period of 250μS which means it switches at 4KHz. The duty- cycle is the percent high time in each period, in the last figure the duty-cycle is 60% because of the high-time and period of 150μS and 250μS respectively. It is the duty-cycle that determines average output voltage. In Figure 1.1 the duty-cycle of 60% with 5V switching voltage results in 3V average output as shown by the red line. After filtering the output a stable analogue output can be achieved. Figure 1.2 shows PWM signals with 80% and 10% duty-cycles. By dynamically changing the duty-cycle, signals other than a flat output voltage can be achieved.

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/basicPWM_4.png?raw=true "Figure")

### Typical micro-controller PWM implementation

This section describes how micro-controllers use timers/counters to implement a PWM signal, this description relies heavily on Figure 1.3. Here the blue line represents a counter that resets after 16000, this gives the period of the PWM and also the switching frequency (f s ), if this micro had a clock source of 16MHz, then f s would be 16×10^6/16×10^3 = 1KHz. There are two other values represented by the green and red lines, which determine the duty-cycle. The values shown are 11200 and 8000 and these give the high time of the PWM signal, and the duty-cycle is these values as a fraction of the PWM period. Therefore the red PWM has a duty-cycle of 11200/16000 = 70% and the red PWM has a duty-cycle of 8000/16000 = 50%

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/sawtooth_counter_1.png?raw=true "Figure")

### Sinusoidal PWM

A sinusoidal PWM (sPWM) signal can be constructed by dynamically changing the duty- cycle. The result is short pulses at the zero-crossings and long pulses at the wave peaks. This can be seen in Figure 1.1

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/PWMsin_2.png?raw=true "Figure")

Figure 1.4 shows negative pulses which is not possible on most micro-controllers. Instead normally this is implemented with two pins, one pulsing the positive half of the sin wave and the second pulsing the negative half, this is how it is implemented in this paper.

## Code & Implementation

In this chapter three examples of code are given, each adding more advanced functionality to the code. Even though this code was written with Arduino in mind the code is written in pure C, however in Chapter an example of modified code that is more Arduino friendly is given. If the signal is going to be viewed on an oscilloscope we suggest to modify the code to toggle a pin every interrupt service routine (ISR) to be used as a trigger.

### Manually entered look up table sPWM code

The following C code implements an sPWM on a Atmel micro-controller. The signal is generated on two pins, one responsible for the positive half of the sine wave and the other pin the negative half. The sPWM is generated by running an (ISR) every period of the PWM in order to dynamically change the duty-cycle. This is done by changing the values in the registers OCR1A and OCR1B from values in a look up table. There are two look up tables for each of the two pins, lookUpTable1 and lookUpTable2 and both have 200 values. The first half of lookUpTable1 has sin values from 0 to π and the second half is all zeroes. The first half of lookUpTable2 is all zeroes and the second half has sin values from 0 to π as shown in Figure 2.1.

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/lookup_2.png?raw=true "Figure")

The code assumes implementation on a ATmega328 or Arduino Uno with a 16MHz clock source. This means the output pins will be PORTB1 and PORTB2 which are pins 9 and 10 on the Uno. The code is compatible with other Atmel micro-controllers/Arduinos though their data-sheets/schematics will need to be referenced to determine the output pins. Please see Chapter for a list of compatible devices.

```c
#include <avr/io.h>
#include <avr/interrupt.h>

// Look up tables with 200 entries each, normalised to have max value of 1600 which is the period of the PWM loaded into register ICR1.
int lookUp1[] = {50 ,100 ,151 ,201 ,250 ,300 ,349 ,398 ,446 ,494 ,542 ,589 ,635 ,681 ,726 ,771 ,814 ,857 ,899 ,940 ,981 ,1020 ,1058 ,1095 ,1131 ,1166 ,1200 ,1233 ,1264 ,1294 ,1323 ,1351 ,1377 ,1402 ,1426 ,1448 ,1468 ,1488 ,1505 ,1522 ,1536 ,1550 ,1561 ,1572 ,1580 ,1587 ,1593 ,1597 ,1599 ,1600 ,1599 ,1597 ,1593 ,1587 ,1580 ,1572 ,1561 ,1550 ,1536 ,1522 ,1505 ,1488 ,1468 ,1448 ,1426 ,1402 ,1377 ,1351 ,1323 ,1294 ,1264 ,1233 ,1200 ,1166 ,1131 ,1095 ,1058 ,1020 ,981 ,940 ,899 ,857 ,814 ,771 ,726 ,681 ,635 ,589 ,542 ,494 ,446 ,398 ,349 ,300 ,250 ,201 ,151 ,100 ,50 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0};
int lookUp2[] = {0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,0 ,50 ,100 ,151 ,201 ,250 ,300 ,349 ,398 ,446 ,494 ,542 ,589 ,635 ,681 ,726 ,771 ,814 ,857 ,899 ,940 ,981 ,1020 ,1058 ,1095 ,1131 ,1166 ,1200 ,1233 ,1264 ,1294 ,1323 ,1351 ,1377 ,1402 ,1426 ,1448 ,1468 ,1488 ,1505 ,1522 ,1536 ,1550 ,1561 ,1572 ,1580 ,1587 ,1593 ,1597 ,1599 ,1600 ,1599 ,1597 ,1593 ,1587 ,1580 ,1572 ,1561 ,1550 ,1536 ,1522 ,1505 ,1488 ,1468 ,1448 ,1426 ,1402 ,1377 ,1351 ,1323 ,1294 ,1264 ,1233 ,1200 ,1166 ,1131 ,1095 ,1058 ,1020 ,981 ,940 ,899 ,857 ,814 ,771 ,726 ,681 ,635 ,589 ,542 ,494 ,446 ,398 ,349 ,300 ,250 ,201 ,151 ,100 ,50 ,0};

int main(void){
    // Register initilisation, see datasheet for more detail.
    TCCR1A = 0b10100010;
       /*10 clear on match, set at BOTTOM for compA.
         10 clear on match, set at BOTTOM for compB.
         00
         10 WGM1 1:0 for waveform 15.
       */
    TCCR1B = 0b00011001;
       /*000
         11 WGM1 3:2 for waveform 15.
         001 no prescale on the counter.
       */
    TIMSK1 = 0b00000001;
       /*0000000
         1 TOV1 Flag interrupt enable. 
       */
    ICR1   = 1600;     // Period for 16MHz crystal, for a switching frequency of 100KHz for 200 subdevisions per 50Hz sin wave cycle.
    sei();             // Enable global interrupts.
    DDRB = 0b00000110; // Set PB1 and PB2 as outputs.
    //DDRB = 0xFF;
	
    while(1){; /* Do nothing. . . forever. */}
}

ISR(TIMER1_OVF_vect){
    static int num;
    // change duty-cycle every period.
    OCR1A = lookUp1[num];
    OCR1B = lookUp2[num];
    
    if(++num >= 200){ // Pre-increment num then check it's below 200.
       num = 0;       // Reset num.
       //PORTB  ^= 0b00100000; 
  }
}
```

## Testing the Signal

This chapter discusses ways to test and monitor the sPWM signal, the first section discusses using as oscilloscope which is the best method to verify the signal however an alternate and cheaper method is to use a small speaker. The 50hz signal and the underlying switching frequency is easy to hear.

### Viewing Pulse Widths with an osilloscope

Here we aim to view the individual pulses that make up the sPWM to see if they look how we would expect, that is thin pulses becoming thick and then thin again. Assuming the reader is using the code of Listing ??, there are a few modifications recommended. We recommend lowering the switching frequency of the sPWM by changing ’ #define SinDivisions ... (200) ’ to ’ #define SinDivisions (50) ’. This will make the pulses easier to see as there will be fewer of them. We also recommend toggling a pin every time the look up table resets. If you are using the Uno, this can be done by changing ’ DDRB = 0b00000110; ’ to ’ DDRB = 0b00100110; ’ and to place ’ PORTB ˆ= 0b00100000; ’ inside the ’ else ... if(num >= SinDivisions/2) ’ statement. This will toggle PortB pin2 which is the led pin on the Uno (pin13). Now this pin can be used as a trigger with the oscilloscope. Figure 4.1 shows the wiring to the Ardurino Uno.

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/PulseOscill_1.png?raw=true "Figure")

The wave form produced should look like Figure 4.2a which is similar to have the wave form from Figure 1.4. Figures 4.2b and 4.2c show the pulse widths at their narrowest and thickest.

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/pulses_2.png?raw=true "Figure")

### Viewing the filtered signal with an oscilloscope

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/filterOscill_2.png?raw=true "Figure")

Here we aim to view a smooth sinusoidal wave by smoothing the output of the micro with a simple low-pass RC filter as shown in Figure 4.3. 
The purpose of the filter is to remove the much higher switching frequency (f s ) and leave only the 50Hz sin wave. The cut-off frequency (f c ) of the filter determined by f c = 1 2πRC should lie somewhere between f s and 50Hz, closer to 50Hz is more optimal. f s is determined by the line of code ’ #define SinDivisions (a number) ’ in Listings 2.2, 2.3 and 3. The switching frequency is given by f s = a number ×50Hz, replacing a number with any arbitrary number may produce unexpected results, we recommend using 50,200 and 400 to produced f s ’s of 2.5,10 and 20 KHz respectively.

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/smoothed_2.png?raw=true "Figure")

Alternatively a potentiometer can be used as a variable resistor as shown in Fig-
ure 4.5a. This allows the f c to be varied. Figure 4.5b to 4.5f shows the same signal with
progressively lower f c

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/trimpot_RCfilter_2.png?raw=true "Figure")

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/smothing_1.png?raw=true "Figure")

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/smothing_2.png?raw=true "Figure")

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/smothing_3.png?raw=true "Figure")

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/smothing_4.png?raw=true "Figure")

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/smothing_5.png?raw=true "Figure")

### Listening to the Signal

If you don’t have an oscilloscope, listening to the signal is a useful way to determine if the sPWM is working since it is easy to hear both the 50Hz hum and the switching frequency. We can use the micro to drive a small pair of head-phones directly, putting them in series with a 1KΩ resister should protect most head-phones, as seen in Figure 4.6.

![Figure what](https://github.com/Terbytes/Arduino-Atmel-sPWM/blob/master/im/speaker_2.png?raw=true "Figure")

It is recommended to change ’ #define SinDivisions (200) ’ down to 50 and up to 400 in order to hear the difference in switching frequencies.