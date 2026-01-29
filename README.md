//# musicadelasplantas
//Dispositivo de BioData

#include <EEPROMex.h> //store and read variables to nonvolatile memory
#include <Bounce2.h> //https://github.com/thomasfredericks/Bounce-Arduino-Wiring
#include <LEDFader.h> //manage LEDs without delay() jgillick/arduino-LEDFader.git

int maxBrightness = 190;
///// este codigo ha sido aportado por ARIDEEJAY Adriana Maria Gutierrez Grisales
//// Disfruta y si tienes alguna pregunta, por favor contactame. Gracias a Sam Cusumano
//******************************
// Definimos la escala Pentatónica Menor de C en 6 octavas
const int numNotasEscala = 30;  // Total de notas en la escala (5 notas por octava * 6 octavas)
int escalaPentatonicaC[numNotasEscala] = {
  36, 39, 41, 43, 46,  // C2, Eb2, F2, G2, Bb2
  48, 51, 53, 55, 58,  // C3, Eb3, F3, G3, Bb3
  60, 63, 65, 67, 70,  // C4, Eb4, F4, G4, Bb4
  72, 75, 77, 79, 82,  // C5, Eb5, F5, G5, Bb5
  84, 87, 89, 91, 94,  // C6, Eb6, F6, G6, Bb6
  96, 99, 101, 103, 106 // C7, Eb7, F7, G7, Bb7
};

//int acordesSeptimasPentatonicaC[][4] = {
 // {36, 39, 43, 46},  // Cm7 (C2, Eb2, G2, Bb2)
//  {39, 43, 46, 53},  // Ebmaj7 (Eb2, G2, Bb2, F3)
 // {41, 48, 53, 55},  // Fm7 (F2, C3, F3, G3)
 // {43, 46, 55, 58},  // Gm7 (G2, Bb2, G3, Bb3)
 // {46, 53, 58, 65}   // Bbmaj7 (Bb2, F3, Bb3, Eb4)
//};
//*******************************

const byte interruptPin = INT0; // Galvanometer input
const byte knobPin = A0; // Knob analog input
Bounce button = Bounce(); // Debounce button using Bounce2
const byte buttonPin = A1; // Tact button input
int menus = 5; // Number of main menus
int mode = 0; //0 = Threshold, 1 = Scale, 2 = Brightness
int currMenu = 0;
int pulseRate = 350; // Base pulse rate

const byte samplesize = 10; // Set sample array size
const byte analysize = samplesize - 1;  // Trim for analysis array

const byte polyphony = 5; // Above 8 notes may run out of RAM
int channel = 1;  // Setting channel to 1
int noteMin = 36; // C2  - keyboard note minimum
int noteMax = 96; // C7  - keyboard note maximum
byte controlNumber = 80; // Set to mappable control, low values may interfere with other soft synth controls
byte controlVoltage = 1; // Output PWM CV on controlLED
long batteryLimit = 3000; // Voltage check minimum, 3.0~2.7V under load; causes lightshow to turn off (save power)
byte checkBat = 1;

volatile unsigned long microseconds; // Sampling timer
volatile byte index = 0;
volatile unsigned long samples[samplesize];

float threshold = 1.7;   // Change threshold multiplier
float threshMin = 1.61;  // Scaling threshold min
float threshMax = 3.71;  // Scaling threshold max
float knobMin = 1;
float knobMax = 1024;

unsigned long previousMillis = 0;
unsigned long currentMillis = 1;

#define LED_NUM 6
LEDFader leds[LED_NUM] = { // 6 LEDs
  LEDFader(3),
  LEDFader(5),
  LEDFader(6),
  LEDFader(9),
  LEDFader(10),
  LEDFader(11)  // Control Voltage output or controlLED
};
int ledNums[LED_NUM] = {3,5,6,9,10,11};
byte controlLED = 5; // Array index of control LED (CV out)
byte noteLEDs = 1;  // Performs lightshow set at noteOn event

typedef struct _MIDImessage { // Build structure for Note and Control MIDImessages
  unsigned int type;
  int value;
  int velocity;
  long duration;
  long period;
  int channel;
} 
MIDImessage;
MIDImessage noteArray[polyphony]; // Manage MIDImessage data as an array with size polyphony
MIDImessage controlMessage; // Manage MIDImessage data for Control Message (CV out)

void setup()
{
  pinMode(knobPin, INPUT);
  pinMode(buttonPin, INPUT_PULLUP);
  button.attach(buttonPin);
  button.interval(5);
  
  randomSeed(analogRead(0)); // Seed for QY8 4 channel mode
  Serial.begin(31250);  // Initialize at MIDI rate
  
  controlMessage.value = 0;  // Begin CV at 0
  attachInterrupt(interruptPin, sample, RISING);  // Begin sampling from interrupt
}

void loop()
{
  currentMillis = millis();   // Manage time
  checkButton();  // Check button interaction

  if(index >= samplesize)  { analyzeSample(); }  // If samples array full, call sample analysis   
  checkNote();  // Turn off expired notes 
  checkControl();  // Update control value
  checkLED();  // LED management without delay()

  if(currMenu > 0) checkMenu(); // Allow main loop by checking current menu mode, and updating millis
}

void setNote(int value, int velocity, long duration, int notechannel)
{
  // Find available note in array (velocity = 0)
  for(int i=0;i<polyphony;i++){
    if(!noteArray[i].velocity){
      // If velocity is 0, replace note in array
      noteArray[i].type = 0;
      noteArray[i].value = value;
      noteArray[i].velocity = velocity;
      noteArray[i].duration = currentMillis + duration;
      noteArray[i].channel = notechannel;

      midiSerial(144, channel, value, velocity); // Send MIDI note-on

      // Handle LEDs when note is played
      if(noteLEDs == 1){ 
          for(byte j=0; j<(LED_NUM); j++) {   // Find available LED and set
            if(!leds[j].is_fading()) { rampUp(j, maxBrightness, duration);  break; }
          }
      } 
      break;
    }
  }
}

void checkLED(){
  // Iterate through LED array and call update  
  for (byte i = 0; i < LED_NUM; i++) {
    LEDFader *led = &leds[i];
    led->update();    
  }
}

void setControl(int type, int value, int velocity, long duration)
{
  controlMessage.type = type;
  controlMessage.value = value;
  controlMessage.velocity = velocity;
  controlMessage.period = duration;
  controlMessage.duration = currentMillis + duration; // Schedule for update cycle
}

void checkControl()
{
  signed int distance =  controlMessage.velocity - controlMessage.value; 
  if(distance != 0) {
    if(currentMillis > controlMessage.duration) { // If duration expired
      controlMessage.duration = currentMillis + controlMessage.period; // Extend duration
      if(distance > 0) { controlMessage.value += 1; } else { controlMessage.value -=1; }
      midiSerial(176, channel, controlMessage.type, controlMessage.value); 
    }
  }
}

void checkNote()
{
  for (int i = 0; i < polyphony; i++) {
    if(noteArray[i].velocity) {
      if (noteArray[i].duration <= currentMillis) {
        midiSerial(144, channel, noteArray[i].value, 0); // Send noteOff
        noteArray[i].velocity = 0;
        rampDown(i, 0, 225); // Turn off LED
      }
    }
  }
}

void midiSerial(int type, int channel, int data1, int data2) {
  cli(); // Disable interrupts
  data1 &= 0x7F;  // Number
  data2 &= 0x7F;  // Velocity
  byte statusbyte = (type | ((channel-1) & 0x0F));
  Serial.write(statusbyte);
  Serial.write(data1);
  Serial.write(data2);
  sei(); // Enable interrupts
}

void rampUp(int ledPin, int value, int time) {
  LEDFader *led = &leds[ledPin];
  led->fade(map(value, 0, 255, 0, maxBrightness), time); 
}

void rampDown(int ledPin, int value, int time) {     
  LEDFader *led = &leds[ledPin];
  led->fade(value, time); // Fade out
}

void analyzeSample()
{
  unsigned long averg = 0;
  unsigned long maxim = 0;
  unsigned long minim = 100000;
  float stdevi = 0;
  unsigned long delta = 0;
  byte change = 0;

  if (index == samplesize) { // Array is full
    unsigned long sampanalysis[analysize];
    for (byte i=0; i<analysize; i++){ 
      sampanalysis[i] = samples[i+1];  // Load analysis table (due to volatile)
      if(sampanalysis[i] > maxim) { maxim = sampanalysis[i]; }
      if(sampanalysis[i] < minim) { minim = sampanalysis[i]; }
      averg += sampanalysis[i];
      stdevi += sampanalysis[i] * sampanalysis[i];  // Prep stdev
    }

    averg = averg / analysize;
    stdevi = sqrt(stdevi / analysize - averg * averg);
    if (stdevi < 1) { stdevi = 1.0; } // Min stdev of 1
    delta = maxim - minim;
    
    if (delta > (stdevi * threshold)){
      change = 1;
    }

    if (change) {
      int dur = 150 + (map(delta % 127, 1, 127, 100, 2500)); // Length of note
      int setnote = escalaPentatonicaC[random(0, numNotasEscala)];  // Elige una nota aleatoria de la escala
      setNote(setnote, 100, dur, channel); // Envía la nota seleccionada
      
      // Set control parameters and update LEDs
      setControl(controlNumber, controlMessage.value, delta % 127, 3 + (dur % 100));
    }

    index = 0;  // Reset for next sample
  }
}

void checkButton() {
  button.update();
  if(button.fell()) {
    // Implementación del menú o cambio de modo si es necesario
  }
}

void checkMenu() {
  // Implementa el manejo del menú aquí
}

void sample() {
  if(index < samplesize) {
    samples[index] = micros() - microseconds;
    microseconds = samples[index] + microseconds; //rebuild micros() value w/o recalling
    index += 1;
  }
}
