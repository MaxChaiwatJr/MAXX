#define led_on 2
#define led_off 1
#define PINK_MASK 0x3

int led = 0;
boolean pinChangeFlag;
uint8_t myPinK;
uint8_t myPinKTemp;
uint8_t pinChangeStatus;
float sensorvalue;
int sensor;


void timer4SetupFastPWMMode15() {
  TCCR4A = 0;
  TCCR4B = 0;
  OCR4A = 15625/5; //1s
  OCR4B = (15625/5)/2;
  OCR4C = (15625/5)/2;
  TCNT4 = 0x0;
  TIFR4 = 0x6;
  TIMSK4 = 0;
  TCCR4A = 0b01101011; //PWMmode15 COM1A mode1, COM1B mode2, COM1C mode2
  TCCR4B = 0b00011101; //1024
}

void adcSetup() {
  ADCSRA = 0x97;
  DIDR0 = 0x1;//ขา A0
  ADMUX = 0x40;
}

void setup() {

  DDRA = 0b11111111; //led8
  PORTA = 0b11111111;//led8

  EIMSK |= 0b00100000; //int5
  EICRB |= 0b00000100; //int5 Any edge

  timer4SetupFastPWMMode15();//led3
  DDRH = 0b00000000;//led3

  PORTK = 3;//switch
  DDRK = 0;//switch

  DDRD = 0b11111111;
  //PORTD = 0b11111111;
  
  pinChangeFlag = false;
  PCICR = 0x4; // enable PCINT2 group = PCINT23:16
  PCMSK2 = 0b11; // enable PCINT16:17 only
  myPinK = (PINK & PINK_MASK); // check for PK0:1

  DDRB = 0b00000001; //Relay
  PORTB = 0b00000000; //Relay
  adcSetup();

  Serial.begin(38400);
  Serial.println("Program begins");

}

ISR(INT5_vect) {
  DDRA ^= 0b11111111;//led8 toggle
  DDRH ^= 0b11111111;//led3 toggle
  PORTD ^= 0b11111111;
  
  Serial.println("int5");
}

ISR(PCINT2_vect) {
  pinChangeFlag = true;
  myPinKTemp = (PINK & PINK_MASK);
  pinChangeStatus = (myPinKTemp ^ myPinK) & PINK_MASK; // detect change
  myPinK = myPinKTemp; // update status
}


void loop() {
  //PORTA = 0b111111111;
  //Led
  if (pinChangeFlag) {
    pinChangeFlag = false;
    switch (pinChangeStatus) {
      case 1:
        if (myPinK == led_on) {
          DDRA = 0b11111111; //led8
          PORTA = 0b01010101;
          delay(1000);
          PORTA = 0b10101010;
          delay(1000);
          PORTA = 0b01111110;
          delay(1000);
          PORTA = 0b10111101;
          delay(1000);
          PORTA = 0b11011011;
          delay(1000);
          PORTA = 0b11100111;
          PORTA = 0b000000000;
          DDRH = 0b11111111;//led3
          PORTD = 0b11111111;//led8
        }
        break;

      case 2:
        if (myPinK == led_off) {
          DDRA = 0b00000000; //led8
          PORTA = 0b00000000;//led8
          DDRH = 0b00000000;//led3
          PORTD = 0b00000000;//led8
        }
        break;
    }
  }


  //water pump
  ADCSRA |= 0x40; //start ADC
  if (ADCSRA & 0x10) {
    sensor = ADC;
    ADCSRA |= 0x10;
    sensorvalue = map(sensor, 0, 1023, 100, 0);  //100=ชื้นมาก
    Serial.println(sensorvalue );
  }

  if (sensorvalue <= 40) {  //ตั้งค่า % ที่ต้องการจะรดน้ำต้นไม้
    PORTB = 0b11111111;//ส่ง high ไปยัง Relay
  }

  else {
    PORTB = 0b00000000;// ส่ง low ไปยัง Relay

  }

}
