# Unbrick AVR Microcontrollers Using HVPP

Unbrick my bricked ATtiny1313A microcontroller using HVPP (High-Volatge Parallel Programming). I utilized an Arduino board and programmed it to function as an HVPP programmer.

![image](https://github.com/m3y54m/unbrick-avr-using-hvpp/assets/1549028/e8905640-c87e-4590-a0c2-e1f62ffed233)

HVPP pins in ATtiny2313A:

![image](https://github.com/m3y54m/unbrick-avr-using-hvpp/assets/1549028/8ca2dba6-3e4b-4c5e-9643-6452ff74aafe)

## Resources

### Tested. Working (with some small changes in circuit and code):

This is the code that finally worked for me.

**Note: I inverted logic level of `RST` output pin in the code according to my active-low RESET configuration in circuit**

![image](https://github.com/m3y54m/unbrick-avr-using-hvpp/assets/1549028/48a5a3d8-af4d-4baa-8c9a-fbe93c68e5a2)


```c
// User defined settings
#define  INTERACTIVE  0     // Set this to 0 to disable interactive (serial) mode
#define  BURN_EFUSE   1     // Set this to 1 to enable burning extended fuse byte
#define  BAUD         9600  // Serial port rate at which to talk to PC

// If interactive mode is off, these fuse settings are used instead of user prompted values
#define  LFUSE        0x64  // default for ATmega168 = 0x62
#define  HFUSE        0xDF  // default for ATmega168 = 0xDF
#define  EFUSE        0xFF  // default for ATmega168 = 0xF9
// Note: I modified the fuse byte values according to ATtiny2313 defaults
//       adopted from [AVR® Fuse Calculator](https://www.engbedded.com/fusecalc/)

// Pin assignments for the Arduino Uno board (Feel free to modify based on your specific board configurations)
// Note: Analog inputs 0-5 can be addressed as digital outputs 14-19
#define  DATA    PORTD // PORTD = Arduino Digital pins 0-7
#define  DATAIN  PIND  // Corresponding inputs
#define  DATAD   DDRD  // Data direction register for DATA port
#define  VCC     12
#define  RDY     13     // RDY/!BSY signal from target
#define  OE      11
#define  WR      10
#define  BS1     9
#define  XA0     8
#define  XA1     A5    
#define  RST     A4    // 12V RESET enable (active-low)
#define  XTAL1   A3
#define  BUTTON  A2    // Run button
// Target-specific Pin Assignments 
// For ATtiny2313A
#define  PAGEL   9   // same as BS1
#define  BS2     A5  // same as XA1


// Internal definitions
#define  LFUSE_SEL  0
#define  HFUSE_SEL  1
#define  EFUSE_SEL  2

void setup()  // run once, when the sketch starts
{
  // Set up control lines for HV parallel programming
  DATA = 0x00;  // Clear digital pins 0-7
  DATAD = 0xFF; // set digital pins 0-7 as outputs
  pinMode(VCC, OUTPUT);
  pinMode(RDY, INPUT);
  pinMode(OE, OUTPUT);
  pinMode(WR, OUTPUT);
  pinMode(BS1, OUTPUT);
  pinMode(XA0, OUTPUT);
  pinMode(XA1, OUTPUT);
  pinMode(PAGEL, OUTPUT);
  pinMode(RST, OUTPUT);  // signal to control +12V RESET
  pinMode(BS2, OUTPUT);
  pinMode(XTAL1, OUTPUT);
  
  pinMode(BUTTON, INPUT);
  digitalWrite(BUTTON, HIGH);  // turn on internal pullup resistor

  // Initialize output pins as needed
  digitalWrite(RST, HIGH); // Pull RESET pin to GND (according to NPN transistor circuit)
  digitalWrite(VCC, LOW); 
}

void loop()  // run over and over again
{
  byte hfuse, lfuse, efuse;  // desired fuse values from user
  byte read_hfuse, read_lfuse, read_efuse;  // fuses read from target for verify
  
#if (INTERACTIVE == 1)
  DATAD = 0x00;  // Set all DATA lines to input, including UART lines (so we can use them for serial comm)
  Serial.begin(BAUD);
  delay(10);
  //Serial.print("\n");
  Serial.println("Insert target AVR and press button.");
#endif

  // wait for button press, debounce
  while(1) {
    while (digitalRead(BUTTON) == HIGH);  // wait here until button is pressed
    delay(10);                            // simple debounce routine
    if (digitalRead(BUTTON) == LOW)       // if the button is still pressed, continue
      break;  // valid press was detected, continue on with rest of program
  }
  
  // Initialize pins to enter programming mode
  DATA = 0x00;
  DATAD = 0xFF;  // Set all DATA lines to output mode
  digitalWrite(PAGEL, LOW);
  digitalWrite(XA1, LOW);
  digitalWrite(XA0, LOW);
  digitalWrite(BS1, LOW);
  digitalWrite(BS2, LOW);
  digitalWrite(WR, LOW);  // ATtiny needs this to be low to enter programming mode, ATmega doesn't care
 
  // Enter programming mode
  digitalWrite(VCC, HIGH);  // Apply VCC to start programming process
  delay(1);
  digitalWrite(OE, HIGH);
  delay(1);
  digitalWrite(RST, LOW); // Apply 12V to RESET
  delay(1);
  digitalWrite(WR, HIGH);   // Now that we're in programming mode we can disable !WR

  // Now we're in programming mode until RST is set LOW again
  
#if (INTERACTIVE == 1)
  // Ask the user what fuses should be burned to the target
  // For a guide to AVR fuse values, go to http://www.engbedded.com/cgi-bin/fc.cgi
  // Serial.println("Ensure target AVR is removed before entering fuse values."); 
  Serial.print("Enter desired LFUSE hex value (ie. 0x62): ");
  lfuse = fuse_ask();
  Serial.print("Enter desired HFUSE hex value (ie. 0xDF): ");
  hfuse = fuse_ask(); 
  #if (BURN_EFUSE == 1)
    Serial.print("Enter desired EFUSE hex value (ie. 0xF9): ");
    efuse = fuse_ask();
  #endif 
#else
  hfuse = HFUSE;
  lfuse = LFUSE;
  efuse = EFUSE;
#endif

  // Now burn desired fuses
  // First, program HFUSE
  fuse_burn(hfuse, HFUSE_SEL);

  // Now, program LFUSE
  fuse_burn(lfuse, LFUSE_SEL);

#if (BURN_EFUSE == 1)
  // Lastly, program EFUSE
  fuse_burn(efuse, EFUSE_SEL);
#endif

  //delay(1000);  // wait a while to allow button to be released

  // Read back fuse contents to verify burn worked
  read_lfuse = fuse_read(LFUSE_SEL);
  read_hfuse = fuse_read(HFUSE_SEL);
  #if (BURN_EFUSE == 1)
    read_efuse = fuse_read(EFUSE_SEL);
  #endif


  // Done verifying
  //digitalWrite(OE, HIGH);

  DATAD = 0x00;
  Serial.begin(BAUD);  // open serial port
  Serial.print("\n");  // flush out any garbage data on the link left over from programming
  Serial.print("Read LFUSE: ");
  Serial.print(read_lfuse, HEX);
  Serial.print("\n");
  Serial.print("Read HFUSE: ");
  Serial.print(read_hfuse, HEX);
  Serial.print("\n");
  #if (BURN_EFUSE == 1)
    Serial.print("Read EFUSE: ");
    Serial.print(read_efuse, HEX);
    Serial.print("\n");
  #endif  
  Serial.println("Burn complete."); 
  Serial.print("\n");
  Serial.println("It is now safe to remove the target AVR.");
  Serial.print("\n");
  
  // All done, disable outputs
  DATAD = 0x00;
  DATA = 0x00;
  delay(1);
  digitalWrite(RST, HIGH);
  delay(1);
  digitalWrite(OE, LOW);
  digitalWrite(WR, LOW);
  digitalWrite(PAGEL, LOW);
  digitalWrite(XA1, LOW);
  digitalWrite(XA0, LOW);
  digitalWrite(BS1, LOW);
  digitalWrite(BS2, LOW);
  digitalWrite(VCC, LOW);

}

void send_cmd(byte command)  // Send command to target AVR
{
  DATAD = 0xFF;  // Set all DATA lines to outputs
  
  // Set controls for command mode
  digitalWrite(XA1, HIGH);
  digitalWrite(XA0, LOW);
  digitalWrite(BS1, LOW);

  DATA = command;
  digitalWrite(XTAL1, HIGH);  // pulse XTAL to send command to target
  delay(1);
  digitalWrite(XTAL1, LOW);
  delay(1);
}

void fuse_burn(byte fuse, int select)  // write high or low fuse to AVR
{
  // if highbyte = true, then we program HFUSE, otherwise LFUSE
  
  send_cmd(B01000000);  // Send command to enable fuse programming mode
  
  // Enable data loading
  digitalWrite(XA1, LOW);
  digitalWrite(XA0, HIGH);
  delay(1);
  
  // Load fuse value into target
  DATA = fuse;
  digitalWrite(XTAL1, HIGH);  // load data byte
  delay(1);
  digitalWrite(XTAL1, LOW);
  delay(1);
  // Decide which fuse location to burn
  switch (select) { 
  case HFUSE_SEL:
    digitalWrite(BS1, HIGH); // program HFUSE
    digitalWrite(BS2, LOW);
    break;
  case LFUSE_SEL:
    digitalWrite(BS1, LOW);  // program LFUSE
    digitalWrite(BS2, LOW);
    break;
  case EFUSE_SEL:
    digitalWrite(BS1, LOW);  // program EFUSE
    digitalWrite(BS2, HIGH);
    break;
  }
  delay(1);
   // Burn the fuse
  digitalWrite(WR, LOW); 
  delay(1);
  digitalWrite(WR, HIGH);
  delay(100);
  
  // Reset control lines to original state
  digitalWrite(BS1, LOW);
  digitalWrite(BS2, LOW);
  
}

byte fuse_read(int select) {
  byte fuse;
  
  send_cmd(B00000100);  // Send command to read fuse bits

  DATAD = 0x00;  // Set all DATA lines to inputs
  DATA = 0x00;
  digitalWrite(OE, LOW);  // turn on target output enable so we can read fuses
  
  // Set control lines and read fuses
  switch (select) {
  case LFUSE_SEL:  
    // Read LFUSE
    digitalWrite(BS2, LOW);
    digitalWrite(BS1, LOW);
    delay(1);
    fuse = DATAIN;
    break;
  case HFUSE_SEL:
    // Read HFUSE
    digitalWrite(BS2, HIGH);
    digitalWrite(BS1, HIGH);
    delay(1);
    fuse = DATAIN;
    break;
  case EFUSE_SEL:
    // Read EFUSE
    digitalWrite(BS2, HIGH);
    digitalWrite(BS1, LOW);
    delay(1);
    fuse = DATAIN;
    break;
  }
  
  digitalWrite(OE, HIGH);  // Done reading, disable output enable line
  
  return fuse;
}

byte fuse_ask(void) {  // get desired fuse value from the user (via the serial port)
  byte incomingByte = 0;
  byte fuse;
  char serbuffer[2];
  
  while (incomingByte != 'x') {  // crude way to wait for a hex string to come in
    while (Serial.available() == 0);   // wait for a character to come in
    incomingByte = Serial.read();
  }
  
  // Hopefully the next two characters form a hex byte.  If not, we're hosed.
  while (Serial.available() == 0);   // wait for character
  serbuffer[0] = Serial.read();      // get high byte of fuse value
  while (Serial.available() == 0);   // wait for character
  serbuffer[1] = Serial.read();      // get low byte
  
  fuse = hex2dec(serbuffer[1]) + hex2dec(serbuffer[0]) * 16;
  
  Serial.println(fuse, HEX);  // echo fuse value back to the user
  
  return fuse;
  
}

int hex2dec(byte c) { // converts one HEX character into a number
  if (c >= '0' && c <= '9') {
    return c - '0';
  }
  else if (c >= 'A' && c <= 'F') {
    return c - 'A' + 10;
  }
}

```

- [Bricked! ATTINY2313 Repair with HV Rescue Shield](https://community.robotshop.com/forum/t/bricked-attiny2313-repair-with-hv-rescue-shield/2898)
- [AVR HV Rescue Shield 2](https://mightyohm.com/blog/products/hv-rescue-shield-2-x/)

### Tested. Not working but helpful:

- [AVR HVPP Configurator](https://www.instructables.com/AVR-HVPP-Configurator/)
- [HVPP-Configurator](https://github.com/zsccat/HVPP-Configurator)

### Not tested but helpful:

- [How to make an AVR Programmer out of a Pro Mini – Part 2: HVSP/HVPP](https://prominimicros.com/how-to-make-an-avr-programmer-out-of-a-pro-mini-part-2-hvsp-hvpp/)

### References:

- [ATtiny2313A Datasheet >> 21.2 Parallel Programming](https://ww1.microchip.com/downloads/en/DeviceDoc/doc8246.pdf)
- [AVR® Programming Interfaces](https://microchipdeveloper.com/8avr:programminginterfaces)
- [Un-Brick an AVR® Xplained Board](https://microchipdeveloper.com/boards:debugbrick)
- [AVR® Fuse Calculator](https://www.engbedded.com/fusecalc/)
