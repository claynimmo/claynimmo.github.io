---
layout: default
title: Portfolio |  Microcontroller Siren
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Siren
The code is written for a custom microcontroller, to run an interupt driven two tone siren.

```cpp
#include <avr/interrupt.h>
#include <avr/io.h>
#include <stdint.h>

#define f1 2190
#define f2 4670
#define t1 350
#define t2 630

#define TOP1 761
#define TOP2 356

void pwm_init(void);
void timer_init(void);

// this function is called once by the main function.
void init(void){
    pwm_init();
    timer_init();
}

void pwm_init(){

    PORTB.DIRSET = PIN0_bm;
    
    TCA0.SINGLE.CTRLA = TCA_SINGLE_CLKSEL_DIV2_gc;
    TCA0.SINGLE.CTRLB = TCA_SINGLE_WGMODE_SINGLESLOPE_gc | TCA_SINGLE_CMP0EN_bm;

    TCA0.SINGLE.PER = TOP1;
    TCA0.SINGLE.CMP0 = TOP1/2;
    TCA0.SINGLE.CTRLA |= TCA_SINGLE_ENABLE_bm;
}

void timer_init(){
    cli();

    TCB0.CTRLB = TCB_CNTMODE_INT_gc;
    TCB0.CCMP = 3333/2;
    TCB0.INTCTRL = TCB_CAPT_bm;
    TCB0.CTRLA = TCB_CLKSEL_DIV2_gc | TCB_ENABLE_bm;

    sei();
}

volatile uint16_t currentTick;
volatile uint16_t currentTime = t1;
volatile uint8_t part1;
ISR(TCB0_INT_vect){
    currentTick ++;
    if(currentTick >= currentTime){
        currentTick = 0;
        if(part1){
            TCA0.SINGLE.PERBUF = TOP2;
            TCA0.SINGLE.CMP0BUF = TOP2/2;
            currentTime = t2;
        }
        else{
            TCA0.SINGLE.PERBUF = TOP1;
            TCA0.SINGLE.CMP0BUF = TOP1/2;
            currentTime = t1;
        }
        part1 = !part1;
    }
    TCB0.INTFLAGS = TCB_CAPT_bm;
}
```