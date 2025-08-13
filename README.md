# Major-project
Source code of final year project
#include <LiquidCrystal_I2C.h>
LiquidCrystal_I2C lcd(0x27, 16, 2);  // set the LCD address to 0x27 for a 16 chars and 2 line display
#include <EEPROM.h>

#include <SimpleTimer.h>
SimpleTimer scantimer;

#include <WiFi.h>
#include <HTTPClient.h>

char ssid[] = "project4G";    // your SSID
char pass[] = "project1234";  // your SSID Password

const char* host = "http://microembeddedtech.com/appinventor";
String get_host = "http://microembeddedtech.com/appinventor";

WiFiServer server(80);  // open port 80 for server connection
WiFiClient client;
String tablename = "banvoting";

///////////////////////////////////////////////////////////////////////////
#define RXD2 16
#define TXD2 17

#include <Adafruit_Fingerprint.h>
uint8_t id;
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&Serial2);

////////////////////////////////////////////////////////

#define enroll 15
#define buzzer 13
#define sw1 5
#define sw2 18
#define sw3 19
#define sw4 23

#define votingmode 4 
#define emgbutton 27

int votemode = 1;
int votingstart = 0;
int displayvotemsg = 0;
int startdata = 0;
int startvotepressed = 0;
int vote1, vote2, vote3,vote4;

int flag;
int uid;
int resultid;
int totalvotes;


int count = 0;
String winnersend;
int emgbtnval;
int emgupdate=0;

#include <vector>

std::vector<int> votedIDs; // List to store IDs of users who have voted

void setup() {
  Serial.begin(9600);
  Serial2.begin(57600, SERIAL_8N1, RXD2, TXD2);
  pinMode(enroll, INPUT_PULLUP);
  pinMode(votingmode, INPUT_PULLUP);
  pinMode(emgbutton, INPUT_PULLUP);
  pinMode(buzzer, OUTPUT);
  pinMode(sw1, INPUT_PULLUP);
  pinMode(sw2, INPUT_PULLUP);
  pinMode(sw3, INPUT_PULLUP);
  pinMode(sw4, INPUT_PULLUP);


  lcd.begin();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("FINGER PRINT BASED");
  lcd.setCursor(0, 1);
  lcd.print("VOTING MACHINE");
  delay(2000);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Connecting WiFi...");
  delay(1000);
  Serial.printf("\nConnecting to %s", ssid);
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  // print out info about the connection:
  Serial.println("\nConnected to network");
  Serial.print("My IP address is: ");
  Serial.println(WiFi.localIP());
  //starting the server
  server.begin();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Wifi Connected");
  //Enroll();
   lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("STAND BY MODE");

  delay(1000);
}

void loop() {
 checkKeys();
 delay(100);
 if (votemode == 0) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("- VOTING MODE -");
    votingstart = 1;
    resultid = getFingerprintIDez();
    if(resultid!=-1){
      if (resultid!=402){
          
          //Serial.println(resultid);
         // Vote();

          if (std::find(votedIDs.begin(), votedIDs.end(), resultid) == votedIDs.end()) {
                    // User has not voted, allow voting
                    Serial.println(resultid);
                    votedIDs.push_back(resultid); // Store the ID after voting
                    Vote();
                } else {
                    // User has already voted
                        Serial.println("-- ALERT: ALREADY VOTED --");
                        lcd.clear();
                        lcd.setCursor(0, 0);
                        lcd.print("ALREADY VOTED");
                        delay(2000);
                        buzzering();
               }
      }
      else{
        Serial.println("-- ALERT FINGER PRINT NOT AVAILABLE ---");
        userupdate_status(tablename,"0","1"); 
        buzzering();
      }
        
    }
    
  }
  else {
     if (votingstart == 1) {
      Serial.println("---------  VOTING COMPLETED RESULTS WILL BE ANNOUNCED SOON  --------- ");  lcd.clear();
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("VOTING CLOSED");
      lcd.setCursor(0, 1);
      lcd.print(" --RESULTS --");
      delay(2000);
      displayresult();
      votingstart = 0;
    }
    else
    {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("STAND BY MODE");
    }

  }
}


void checkKeys()
{
  if (digitalRead(enroll) == 0)
  {
    Serial.println("------------ EROLL BUTTON PRESSED ----------");
    lcd.clear();
    lcd.print("Please Wait");
    delay(1000);
    while (digitalRead(enroll) == 0);
    Enroll();
  }

    if (digitalRead(votingmode) == 0) {
    votemode = 0;
  }
  else {
    votemode = 1;
  }

 if (digitalRead(emgbutton) == 0) {
  if(emgupdate==0){
    Serial.println("EMERGENCY BUTTON PRESS UPDATE");
    lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("EMERGENCY PRESSED");
      userupdate_status(tablename,"1","0");
      buzzering();
      emgupdate=1;
  }

 }
 else{
  if(emgupdate==1){
    emgupdate=0;
  }

 }

}

void userupdate_status(String table_name, String emgv,String wfpv) {
//http://microembeddedtech.com/appinventor/cloningattackupdate.php?table_name=arduinoencryption&updatebit=t2bit&bitval=1
  WiFiClient client = server.available();
//String constructstring = '"'+sfanstatus+'"';
//#55/66/DETECTED/DETECTED/DETECTED/NORMAL
String onebit="1";


  HTTPClient http;
  String url = get_host+"/banvoteemgupdate.php?table_name="+table_name+"&emg="+emgv+"&wfp="+wfpv;

  Serial.println(url);

  http.begin(url);

  //GET method
  int httpCode = http.GET();
  String payload = http.getString();
  Serial.println(payload);
  http.end();
  delay(1000);
}

void buzzering(){
  digitalWrite(buzzer, HIGH);
  delay(1500);
  digitalWrite(buzzer, LOW);
}
