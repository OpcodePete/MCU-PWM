# PIC microcontroller demo of PWM

This is an early prototype (in 2009) using a PIC microcontroller and assembly language to demonstarte movement using pulse-width modulation.

<br />

**Central Processing Unit**  
PIC16F84 is an 8-bit microcontroller from Microchip (http://www.microchip.com/).

**Motor Control**  
SN754410 is a quadruple half H-Bridge from Texas Instruments (http://www.ti.com/). This interfaces the microcontroller to the DC motors as TTL voltages are not sufficient to drive motors.

**Power Sources**  
Primary Power Unit: Sealed Lead Acid (SLA) battery.

**Framework**  
Chassis has been built with parts from Tamiya (http://www.tamiya.com/).

**Software**  
An interrupt design program developed in assembly language. Pulse width modulation (PWM) is used for speed control and also maintaining torque

```bash

                LIST    P=16F84A                ; List directive to define processor
                INCLUDE    <p16F84.inc>         ; Processor specific variable definitions

                __CONFIG _CP_OFF & _WDT_ON & _PWRTE_ON & _RC_OSC

                ;PWM        EQU    0x0C
                W_TEMP    EQU 0x0D
                S_TEMP    EQU 0x0E
                SPEED    EQU 0x0F
                PWM_1    EQU    0x10
                PWM_2    EQU    0x11
                PWM_3    EQU    0x12
                PWM_4    EQU    0x13


RESET_VECTOR    ORG        0x000                ; Processor reset vector
                GOTO    MAIN                    ; Go to beginning of program


INT_VECTOR      ORG        0x004                ; Interrupt vector location           
                MOVWF    W_TEMP                 ; Copy W register to W_TEMP storage location
                SWAPF    STATUS, W              ; Swap STATUS to be saved into W register
                MOVWF    S_TEMP                 ; Copy STATUS register to S_TEMP storage location

                ; Generate begin or (start of) end of pulse
                BTFSC    PWM_1, 0               ; Check current pulse state, high or low
                GOTO    PWM_ON_1                ; Pulse is high, go to low pulse processing
                NOP                             ; High processing, NOP to match 3 cycles with low processing
                BSF        PWM_1, 0             ; High processing, beginning of pulse               
                MOVLW    0xFF                   ; High processing, set timer for duration of high pulse, TMR0 = 255 cycles - SPEED - ISR cycles - 2 cycles for TMR0 inhibit
                NOP                             ; Match 1 cycle of low processing (period calculation)
                SUBLW    SPEED                  ;         SPEED = 0 - 50 cycles
                SUBLW    21                     ;         ISR no. of cycles
                SUBLW    2                      ;        TMR0 inhibit cycles
                GOTO    PWM_DONE_1              ; High processing end
PWM_ON_1        BCF        PWM_1, 0             ; Low processing, output low pulse, i.e. end of the high period, low from here
                MOVLW    0xFF                   ; Low processing, set timer for duration of low pulse
                                                ;     TMR0 = 255 cycles - (Period - SPEED) - ISR cycles - 2 cycles for TMR0 inhibit
                SUBLW    50                     ; Period = 50 cycles
                ADDLW    SPEED                  ; SPEED = 0 - 50 cycles
                SUBLW    21                     ; ISR no. of cycles
                SUBLW    2                      ; TMR0 inhibit cycles
                GOTO    PWM_DONE_1              ; Low processing end (also match 2 cycle of high processing)

PWM_DONE_1        MOVWF    TMR0                 ; Set timer for next (high or low pulse) interrupt
                BCF        INTCON, T0IF         ; Clear interrupt overflow flag

                SWAPF    S_TEMP, W              ; Swap S_TEMP storage location to be saved into W register
                MOVWF    STATUS                 ; Copy W register to STATUS
                SWAPF    W_TEMP, F              ; Swap W_TEMP storage location to be saved into W_TEMP storage location
                SWAPF    W_TEMP, W              ; Swap W_TEMP storage location to be saved into W register
                RETFIE


MAIN            ; Initialise Ports
                BCF        STATUS, RP0          ; Select Bank 0
                CLRF    PORTA                   ; Initialise PORT A by clearing output data latches
                CLRF    PORTB                   ; Initialise PORT B by clearing output data latches
           
                ; Configure Interrupt
                BSF        STATUS, RP0          ; Select Bank 1
                BSF        INTCON, GIE          ; Enable all interrupts <8Bh:7>
                BSF        INTCON, T0IE         ; Enable TMR0 interrupt <8Bh:5>
                BCF        INTCON, T0IF         ; Reset interrupt flag <8Bh:2>
           
                ; Configure Timer
                BCF        STATUS, RP0          ; Select Bank 0
                CLRF    TMR0                    ; Initialise Timer0
                BSF        STATUS, RP0          ; Select Bank 1
                BCF        OPTION_REG, T0CS     ; Select internal clock as source, i.e. Timer mode
                BSF        OPTION_REG, PSA      ; Prescaler is assigned to WDT <81h:3>. TMR0 has a 1:1 prescale assignment (0-255)

                ; Configure Ports
                MOVLW    B'00010000'            ; Set data direction for PORT A
                MOVWF    TRISA                  ; Set RA<7:5> are always `0`, RA<4> as input, RA<3:0> as output
                MOVLW    B'00110000'            ; Set data direction for PORT B
                MOVWF    TRISB                  ; Set RB<7:6> as output, RB<5:4> as input, RB<3:0> as output

                ; Setup H-Bridge
                BCF        STATUS, RP0          ; Select Bank 0
                MOVLW    B'00000110'            ; Set RB<2:1> to enable H-Bridge, set 1,2 EN & 3,4 EN
                MOVWF    PORTB                  ; Enable both full bridges

                ; Main program here
                MOVLW    0x19                   ; Set to half-speed 25 (decimal)
                MOVWF    SPEED                  ; Set speed
HERE            GOTO    HERE                    ; Wait here (interrupt driven program)
                END


                ; Set movement with direction 1A,2A or 3A,4A
                BTFSC    PORTA, 04h             ; Check input on RA<4>
                GOTO    MOVEBKWD
                BTFSS    PORTA, 04h             ; Check input on RA<4>
                GOTO    MOVEFWD
                GOTO    START

MOVEBKWD        MOVLW    B'00000101'            ; Set values (logic 0s & 1s) to send to PORT A
                MOVWF    PORTA                  ; Send values to port
                GOTO    START

MOVEFWD         MOVLW    B'00001010'            ; Set value of 'forward'
                MOVWF    PORTA                  ; Send value to Port A
```
