---
layout: default
title: Portfolio | Math Approximations
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Approximations
C functions approximating sin, cos, and logarithm using only ALU operations (add, subtract, divide, multiply) using taylor polynomials. The operations are tested against the standard math library for correctness.

## Logarithm

**include/log.h**
```c
#ifndef LOGARITHM
#define LOGARITHM

#include <math.h>
#include <stdint.h>
#include <stdio.h>
#include <string.h>

float c_ln(float x);

float c_log(float base, float x);

#endif
```

**src/log.c**
```c
#include "log.h"
#include "utils.h"

#define LOGPRECISION 10
#define LN2 0.69314718055994530941723212145818f

// uses the ln(x) = 2(y + y^3 / 3 + y^5 / 5 + ...) approximation, where y = (x-1) / (x+1)
float ln_approx(float x){
    float y = (x - 1) / (x + 1);
    float val = 0;
    for(int i = 1; i <= LOGPRECISION; i += 2){
        float term = c_pow(y, i) / i;

        val += term;
    }
    return val * 2;
}

/**
 * Calculates the natural logarithm using the approximation
 * \param x value inside the brackets of the ln function
 * \return returns ln(x)
 */
float c_ln(float x){
    if(x == 0){return -INFINITY;}
    if(x < 0){return NAN;}

    // dimension reduction is used from the ln(A*B) = ln(A) + ln(B) property
    // this can directly use floating point mantissa and exponent bits
    uint32_t bits;
    memcpy(&bits, &x, sizeof(bits));
    uint32_t exponent = (bits >> 23) & 0xFF;
    int e = exponent - 127;
    uint32_t mantissa = bits & 0x7FFFFF;
    float significand = 1.0f + ((float)mantissa / (1u << 23));

    return ln_approx(significand) + e * LN2;
}

/**
 * calculate log_a(b) using the log_a(b) = ln(b) / ln(a) relationship
 * \param base base of the logarith
 * \param x value inside the brackets of the log function
 * \return floating point evaluation from log_base(x)
 */
float c_log(float base, float x){
    float top = c_ln(x);
    float bottom = c_ln(base);
    if(isnan(top) || isnan(bottom)){return NAN;}
    return c_ln(x) / c_ln(base);
}
```

## Sin/Cosine

**include/trig.h**
```c
#ifndef TRIG
#define TRIG
float c_sin(float x);
float c_cos(float x);
#endif
```

**src/trig.c**
```c

#include "trig.h"
#include "utils.h"

/**
 * Uses the taylor series approximation to estimate sin(x)
 * \param x Input phase, in radians
 */
float c_sin(float x){
    // trig reduction, using symmetric properties to reduce the size of the input in the approximation
    // taylor polynomial is most accurate near 0, so it reduces to -pi to pi, then to -pi/2 to pi/2
    float twopi = 2.0f * PI;

    while(x > PI){
        x -= twopi;
    }
    while(x < -PI){
        x += twopi;
    }

    if(x > PI/2){
        x = PI - x;
    }
    else if(x < -PI/2){
        x = -PI - x;
    }

    return x - (c_pow(x,3U) / factorial(3)) + (c_pow(x,5U) / factorial(5)) 
             - (c_pow(x,7U) / factorial(7)) + (c_pow(x,9U) / factorial(9))
             - (c_pow(x,11U)/ factorial(11));
}

/**
 * Calculate cosine using the cos(x) = sin(x + pi/2) rule
 * \param x Input phase, in radians
 */
float c_cos(float x){
   return c_sin(x + (PI / 2));
}
```

## Other math functions

**include/utils.h**
```c
#ifndef UTILS
#define UTILS
#define PI 3.1415926535897932384626433832795
float c_pow(float x, unsigned int pow);
unsigned int factorial(unsigned int x);
float to_rad(float deg);
#endif
```

**src/utils.c**
```c
#include "utils.h"
/**
 * Calculate the floating point power function. Does not work for negative input, and so takes an unsigned integer
 * \param x Value to used as the base
 * \param pow Value used as the exponent
 */
float c_pow(float x, unsigned int pow){
    if(pow == 0){ return 1; }

    float ret = 1;
    for(unsigned int i = 1; i <= pow; i++){
        ret = ret * x;
    }
    return ret;
}

/**
 * Calculate the factorial of an integer
 * \param unsigned value, to calculate x!
 */
unsigned int factorial(unsigned int x){
    unsigned int current = x;
    unsigned int total = 1;
    while(current != 0){
        total = total * current;
        current--;
    }
    return total;
}

float to_rad(float deg){
    return deg * 2 * PI / 180;
}
```
## Test Cases

**include/test.h**
```c
#ifndef TEST
#define TEST

#include <stdio.h>
#include <stdbool.h>
#include <string.h>
#include <math.h>

#define EPSILON 1e-6f
int assert_fequal(float x, float y, char *fail_message);
int assert_iequal(int x, int y, char *fail_message);
bool fequal(float a, float b);
#endif
```

**tests/test.c**
```c

#include "test.h"

int assert_fequal(float x, float y, char *fail_message){
    if(fequal(x, y) == false ){
        printf("\nFAILED: %s, (%f != %f)\n", fail_message, x, y);
        return 0;
    }
    return 1;
}

int assert_iequal(int x, int y, char *fail_message){
     if(x != y){
        printf("\nFAILED: %s, (%d != %d)\n", fail_message, x, y);
        return 0;
    }
    return 1;
}

bool fequal(float a, float b){
    if( (isinf(a) && isinf(b)) || (isnan(a) && isnan(b)) ){
        return true;
    }
    return fabsf(a - b) < EPSILON;
}
```

**tests/main.c**
```c
#include "test.h"
#include "utils.h"
#include "trig.h"
#include "log.h"

void test_pow(){
    printf("\nPower tests: \n");
    int total = 0;
    int max = 4;
    total += assert_fequal(c_pow(0.3f, 4), 0.0081f, "");
    total += assert_fequal(c_pow(12.23010f, 0), 1, "");
    total += assert_fequal(c_pow(4, 2), 16, "");
    total += assert_fequal(c_pow(10, 1), 10, "");
    printf("Passed %d/%d tests\n", total, max);
}

void test_factorial(){
    printf("\nFactorial tests: \n");
    int total = 0;
    int max = 6;
    total += assert_iequal(factorial(1), 1, "");
    total += assert_iequal(factorial(2), 2, "");
    total += assert_iequal(factorial(3), 6, "");
    total += assert_iequal(factorial(4), 24, "");
    total += assert_iequal(factorial(5), 120, "");
    total += assert_iequal(factorial(11), 39916800, "");

    printf("Passed %d/%d tests\n", total, max);
}

void test_sin(){
    printf("\nSine tests:\n");
    int total = 0;
    int max = 13;
    total += assert_fequal(c_sin(0.0f), sin(0.0f), "sin(0) should be 0");
    total += assert_fequal(c_sin(0.1f), sin(0.1f), "");
    total += assert_fequal(c_sin(0.5f), sin(0.5f), "");
    total += assert_fequal(c_sin(1.0f), sin(1.0f), "");
    total += assert_fequal(c_sin(2.0f), sin(2.0f), "");
    total += assert_fequal(c_sin(20 * PI), sin(20 * PI), "Test large input (for wrapping)");
    total += assert_fequal(c_sin(PI/2), sin(PI/2), "sin(90 degrees) should be maximum (1)");
    total += assert_fequal(c_sin(PI),   sin(PI), "sin(180 degrees) should wrap back around to 0");
    total += assert_fequal(c_sin(3*PI / 2), sin(3*PI / 2), "sin(270 degrees) should wrap back to the maximum");
    total += assert_fequal(c_sin(2 * PI), sin(2*PI), "sin(360) should return to 0");
    total += assert_fequal(c_sin(-0.1f), sin(-0.1f), "Test negative input");
    total += assert_fequal(c_sin(-0.52908f), sin(-0.52908f), "Test negative input");
    total += assert_fequal(c_sin(-13*PI), sin(-13*PI), "Test negative input (large)");
    printf("Passed %d/%d tests\n", total, max);
}

void test_cos(){
    printf("\nCosine tests:\n");
    int total = 0;
    int max = 13;
    total += assert_fequal(c_cos(0.0f), cos(0.0f), "cos(0) should be 0");
    total += assert_fequal(c_cos(0.1f), cos(0.1f), "");
    total += assert_fequal(c_cos(0.5f), cos(0.5f), "");
    total += assert_fequal(c_cos(1.0f), cos(1.0f), "");
    total += assert_fequal(c_cos(2.0f), cos(2.0f), "");
    total += assert_fequal(c_cos(20 * PI), cos(20 * PI), "Test large input (for wrapping)");
    total += assert_fequal(c_cos(PI/2), cos(PI/2), "cos(90 degrees) should be maximum (1)");
    total += assert_fequal(c_cos(PI),   cos(PI), "cos(180 degrees) should wrap back around to 0");
    total += assert_fequal(c_cos(3*PI / 2), cos(3*PI / 2), "cos(270 degrees) should wrap back to the maximum");
    total += assert_fequal(c_cos(2 * PI), cos(2*PI), "cos(360) should return to 0");
    total += assert_fequal(c_cos(-0.1f), cos(-0.1f), "Test negative input");
    total += assert_fequal(c_cos(-0.52908f), cos(-0.52908f), "Test negative input");
    total += assert_fequal(c_cos(-13*PI), cos(-13*PI), "Test negative input (large)");
    printf("Passed %d/%d tests\n", total, max);
}

void test_log(){
    printf("\nLogarithm tests: \n");
    int total = 0;
    int max = 11;
    total += assert_fequal(c_ln(1.0f), log(1.0f), "ln 1 should be 0");
    total += assert_fequal(c_ln(0.5f), log(0.5f), "ln(0.5)");
    total += assert_fequal(c_ln(30000.0f), log(30000.0f), "ln(30000)");
    total += assert_fequal(c_ln(450.25f), log(450.25f), "ln(450.25)");
    total += assert_fequal(c_ln(0), log(0), "ln(0)");
    total += assert_fequal(c_ln(-10), log(-10), "ln(-10)");

    total += assert_fequal(c_log(10, 10), 1, "log10(10)");
    total += assert_fequal(c_log(0.25f, 0.25f), 1, "log0.25(0.25)");
    total += assert_fequal(c_log(2, 235.24f), log2f(235.24f), "log2(235.24)");
    total += assert_fequal(c_log(2,-5), log2f(-5), "log2(-5)");
    total += assert_fequal(c_log(-2,-5), NAN, "invalid base");
    printf("Passed %d/%d tests\n", total, max);
}

int main(){
    test_pow();
    test_factorial();
    test_sin();
    test_cos();
    test_log();
    return 0;
}
```

## Makefile

```makefile
SRC_DIR := src
TST_DIR := tests
OBJ_DIR := obj
BIN_DIR := bin

APP_EXE := $(BIN_DIR)/physics
TST_EXE := $(BIN_DIR)/tests

APP_SRC := $(wildcard $(SRC_DIR)/*.c)
TST_SRC := $(wildcard $(TST_DIR)/*.c)

APP_OBJ := $(APP_SRC:$(SRC_DIR)/%.c=$(OBJ_DIR)/src_%.o)
TST_OBJ := $(TST_SRC:$(TST_DIR)/%.c=$(OBJ_DIR)/test_%.o)

CC = gcc
CPPFLAGS := -I include
CFLAGS   := -Wall

.PHONY: all tests clean

all: $(APP_EXE)

tests: $(TST_EXE)

$(APP_EXE): $(OBJ) | $(BIN_DIR)
	$(CC) $(LDFLAGS) -o $@

$(TST_EXE): $(APP_OBJ) $(TST_OBJ) | $(BIN_DIR)
	$(CC) $^ $(LDFLAGS) -o $@

$(OBJ_DIR)/src_%.o: $(SRC_DIR)/%.c | $(OBJ_DIR)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

$(OBJ_DIR)/test_%.o: $(TST_DIR)/%.c | $(OBJ_DIR)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

$(BIN_DIR) $(OBJ_DIR):
	mkdir $@

clean:
	@$(RM) -rv $(BIN_DIR) $(OBJ_DIR)
	
-include $(OBJ:.o=.d)
```