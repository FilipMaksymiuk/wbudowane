// CONFIG2
#pragma config POSCMOD = HS
#pragma config OSCIOFNC = OFF
#pragma config FCKSM = CSDCMD
#pragma config FNOSC = PRI
#pragma config IESO = ON

// CONFIG1
#pragma config WDTPS = PS32768
#pragma config FWPSA = PR128
#pragma config WINDIS = ON
#pragma config FWDTEN = OFF
#pragma config ICS = PGx2
#pragma config GWRP = OFF
#pragma config GCP = OFF
#pragma config JTAGEN = OFF

#include <xc.h>
#include <libpic30.h>

#define FOSC 8000000
#define PRESCALAR 256
#define TPERSEC ((unsigned long)(FOSC / 2 / PRESCALAR))

void setTimer() {
    T1CON = 0x8030;
}

void delay() {
    TMR1 = 0;
    PR1 = TPERSEC * 0.5;
    while (TMR1 < PR1);
}

volatile int current_mode = 3;
volatile int btn_flag = 0;

// ISR — przerwanie przy RD6/RD13
void __attribute__((__interrupt__, no_auto_psv)) _CNInterrupt(void) {
    static int prev6 = 1;
    static int prev13 = 1;

    int btn6 = PORTDbits.RD6;
    int btn13 = PORTDbits.RD13;

    __delay32(30000); // debounce

    btn6 = PORTDbits.RD6;
    btn13 = PORTDbits.RD13;

    if (prev6 == 1 && btn6 == 0) {
        current_mode++;
        if (current_mode > 6) current_mode = 3;
        btn_flag = 1;
    }

    if (prev13 == 1 && btn13 == 0) {
        current_mode--;
        if (current_mode < 3) current_mode = 6;
        btn_flag = 1;
    }

    prev6 = btn6;
    prev13 = btn13;
    IFS1bits.CNIF = 0;
}

int main(void) {
    int portValue = 0x00;
    int bcd = 0;

    setTimer();

    TRISA = 0x0000;
    LATA = 0x00;
    TRISD = 0xFFFF;
    AD1PCFG = 0xFFFF;

    CNPU1bits.CN15PUE = 1;
    CNPU2bits.CN19PUE = 1;

    CNEN1bits.CN15IE = 1;
    CNEN2bits.CN19IE = 1;
    IEC1bits.CNIE = 1;
    IFS1bits.CNIF = 0;

    while (1) {
        if (btn_flag) {
            portValue = (current_mode == 4) ? 0xFF : 0x00;
            bcd = (current_mode == 6) ? 99 : 0;
            btn_flag = 0;
        }

        switch (current_mode) {
            case 3: // Gray UP
                LATA = portValue ^ (portValue >> 1);
                portValue++;
                if (portValue > 0xFF) portValue = 0;
                break;

            case 4: // Gray DOWN
                LATA = portValue ^ (portValue >> 1);
                portValue--;
                if (portValue < 0) portValue = 0xFF;
                break;

            case 5: // BCD UP
                LATA = ((bcd / 10) << 4) | (bcd % 10);
                bcd++;
                if (bcd > 99) bcd = 0;
                break;

            case 6: // BCD DOWN
                LATA = ((bcd / 10) << 4) | (bcd % 10);
                bcd--;
                if (bcd < 0) bcd = 99;
                break;
        }

        delay();
    }

    return 0;
}
