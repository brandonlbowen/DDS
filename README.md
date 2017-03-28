# DDS
Drone Delivery Solution Arduino Code
// The SparkFun MG2639 Cellular Shield uses SoftwareSerial
// to communicate with the MG2639 module. Include that
// library first:
#include <SoftwareSerial.h>
// Include the MG2639 Cellular Shield library
#include <SFE_MG2639_CellShield.h>

// A character to confirm the end of a written message.
// Any message you write will not be able to include this
// character.
const char EOM_CHAR = '~';

char myPhone[15]; // Array to store local phone number

#include <TinyGPS++.h>
#include <SFE_MG2639_CellShield.h>
TinyGPSPlus gps;

//RX=8 TX=9
#include <AltSoftSerial.h>
AltSoftSerial ss;

int fsrReading;
int weightCheck;

void setup()
{

   // USB serial connection is used to print the information
  // we find and interact with the user.
  Serial.begin(9600);
  
  // serialTrigger() halts execution of the program until
  // any value is received over the serial link. Cell data
  // costs $, so we don't want to use it unless it's visible!
  serialTrigger();
  
  // Call cell.begin() to turn the module on and verify
  // communication.
  int beginStatus = cell.begin();
  if (beginStatus <= 0)
  {
    Serial.println(F("Unable to communicate with shield. Looping"));
    while(1)
      ;
  }
  // Delay a bit. If phone was off, it takes a couple seconds
  // to set up SIM.
  delay(2000);
  
  // Use cell.getPhoneNumber to get the local phone #.
  cell.getPhoneNumber(myPhone);
  Serial.println(F("Send me a text message!"));
  Serial.print(F("My phone number is: "));
  Serial.println(myPhone);
  Serial.println();

  // Set SMS mode to text mode. This is a more commonly used,
  // ASCII-based mode. The other possible mode is 
  // SMS_PDU_MODE - protocol data unit mode - an uglier,
  // HEX-valued data mode that may include address info or
  // other user data.
  sms.setMode(SMS_TEXT_MODE);
  
  Serial.print(F("To send a message begin by typing "));
  Serial.print(F("WnnnnnnnnnnX. Where n's are digits"));
  Serial.println(F(" in the destination phone number"));
  Serial.println();


  Serial.begin(9600);
  ss.begin(115200);
  Serial.println(F("Start:"));
}

int fsrSensor()
{
  //Read the voltage value from the FSR sensor
  fsrReading = analogRead(A5);

  //lower threshold (nothing on landing pad)
  if (fsrReading < 100)
  {
    weightCheck = 0;
  }
  //Weight is on landing pad
  else {
    weightCheck = 1;
  }
  return weightCheck;
}

  void loop()
  {

  weightCheck = fsrSensor();
  // Get first available unread message:
  byte messageIndex = sms.available(REC_UNREAD);
  // If an index was returned, proceed to read that message:
  if (messageIndex > 0)
  {
    printSMS(messageIndex);
    
    Serial.println(F("Would you like to delete the message? y/n"));
    while (!Serial.available())
      ;
    char c = Serial.read();
    if ((c == 'y') || (c == 'Y'))
    {
      sms.deleteMessage(messageIndex);
      Serial.println("Message deleted.");
      Serial.println();
    }
  }
  // If serial was received, maybe we'll send a message:
  if (Serial.available())
  {
    char c = Serial.read();
    // Begin an SMS send with the 'W' character.
    if (c == 'W')
    {
      writeSMS();
    }
  }
}

void writeSMS()
{
  char c = 0;
  char destinationPhone[16];
  uint8_t i = 0;
  // Loop until we see a terminating 'X', or receive too
  // many characters.
  while ((c != 'X') & (i < 16))
  {
    // Serial.read() should return -1 if there's nothing
    // there.
    c = Serial.read();
    if ((c >= '0') && (c <= '9'))
      destinationPhone[i++] = c;
  }
  
  Serial.print(F("Sending a message to "));
  Serial.println(destinationPhone);
  
  // To begin sending an SMS, use sms.start(phoneNumber):
  sms.start(destinationPhone);
  
  Serial.print(F("Type your message. End it by sending: "));
  Serial.println(EOM_CHAR);
  
  // Loop until we see the EOM character.
  while (c != EOM_CHAR)
  {
    if (Serial.available())
    {
      c = Serial.read();
      Serial.print(c);
      if ((c != EOM_CHAR) && (c >= 0))
        sms.print((char)c); // Add to a message with sms.print
    }
  }
  Serial.println();
  Serial.println("Sending the message.");
  Serial.println();
  
  // Send an sms message by calling sms.send()
  sms.send();
}

void printSMS(int index)
{
  // Call sms.read(index) to read the message:
  sms.read(index);
  
  Serial.print(F("Reading message ID: "));
  Serial.println(index);
  // Print the sending phone number:
  Serial.print(F("SMS From: "));
  Serial.println(sms.getSender());
  // Print the receive timestamp:
  Serial.print(F("SMS Date: "));
  Serial.println(sms.getDate());
  // Print the contents of the message:
  Serial.print(F("SMS Message: "));
  Serial.println(sms.getMessage());
  Serial.println();
  
}
void serialTrigger()
{
  Serial.println(F("Send some serial to start"));
  while (!Serial.available())
    ;
  Serial.read();
  // This sketch displays information every time a new sentence is correctly encoded.
  while (ss.available() > 0)
    if (gps.encode(ss.read()))
      displayInfo();

      

}

void displayInfo()
{
  Serial.print(F("Location: ")); 
  if (gps.location.isValid())
  {
    Serial.print(gps.location.lat(), 6);
    Serial.print(F(","));
    Serial.print(gps.location.lng(), 6);
  }
  else
  {
    Serial.print(F("INVALID"));
  }

  Serial.print(F("  Date/Time: "));
  if (gps.date.isValid())
  {
    Serial.print(gps.date.month());
    Serial.print(F("/"));
    Serial.print(gps.date.day());
    Serial.print(F("/"));
    Serial.print(gps.date.year());
  }
  else
  {
    Serial.print(F("INVALID"));
  }

  Serial.print(F(" "));
  if (gps.time.isValid())
  {
    if (gps.time.hour() < 10) Serial.print(F("0"));
    Serial.print(gps.time.hour());
    Serial.print(F(":"));
    if (gps.time.minute() < 10) Serial.print(F("0"));
    Serial.print(gps.time.minute());
    Serial.print(F(":"));
    if (gps.time.second() < 10) Serial.print(F("0"));
    Serial.print(gps.time.second());
    Serial.print(F("."));
    if (gps.time.centisecond() < 10) Serial.print(F("0"));
    Serial.print(gps.time.centisecond());
  }
  else
  {
    Serial.print(F("INVALID"));
  }

  Serial.println();
}
