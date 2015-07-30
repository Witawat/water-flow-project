# water-flow-project
project about water flow sensor information 

เป็นโปรเจ็ควัดระดับการไหลของน้ำโดยส่งข้อมูลผ่าน OLED, mobile wifi browser, micro sd card.

##อุปกรณ์
1. arduino mega 2560
2. Water Flow Sensor (YF-S201)
3. จอ OLED 
4. ESP8266
5. micro sd card adapter

##ขั้นตอนการประกอบ

###การประกอบเจ้า YF-S201 กับ Arduino(แบบฉบับผู้เขียน)
1. เจ้าตัว YF-S201 มีทั้งหมด 3 ขา
2. ขาสีดำเป็น ground
3. ขาสีเหลืองเป็น vcc(5v)
4. ขาสีแดงเป็นขาส่งข้อมูลต่อเข้ากับ mega ขาที่ 2

###การประกอบเจ้าจอ OLED กับ Arduino(แบบฉบับผู้เขียน)
1. มีด้วยกันทั้งหมด 6 ขาเราจะเริ่มพูดถึงขา GND ซึ่งอยู่ด้านซ้ายสุดก่อนและถัดมาทางขวาเรื่อยๆ
2. GND ให้ต่อเข้ากับ ground
3. VCC ให้ต่อกับไฟ 5V
4. SCL ใหต่อเข้ากับบอร์ด mega ขาที่ 32
5. SDA ให้ต่อเข้ากับบอร์ด mega ขาที่ 34
6. RST ให้ต่อเข้ากับบอร์ด mega ขาที่ 36
7. D/C ให้ต่อเข้ากับบอร์ด mega ขาที่ 38

###การประกอบเจ้าจอ ESP8266 กับ Arduino(แบบฉบับผู้เขียน)
ให้เข้าไปดูวิธีต่อในเว็บนี้
http://www.arduinoall.com/article/สอน-วิธี-ใช้งาน-arduino-wi-fi-module-esp8266
ปล.เว็บไซต์ ณ วันที่ 30 กรกฎาคม 2558

###การประกอบเจ้าจอ micro sd card adapter กับ Arduino(แบบฉบับผู้เขียน)
1.มีด้วยกัน 6 ขาโดยจะเริ่มกล่าวถึงขา GND ก่อนและไล่ไปทางขวาเรื่อยๆ
2. GND ต่อเข้ากับ ground
3. VCC ต่อเข้ากับแรงดันไฟ 5V
4. MISO ต่อเข้าบอร์ด mega ขาที่ 50
5. MOSI ต่อเข้าบอร์ด mega ขาที่ 51
6. SCK  ต่อเข้าบอร์ด mega ขาที่ 52
7. CS   ต่อเข้าบอร์ด mega ขาที่ 53

###CODE OF PROJECT
        #include <SPI.h>
        #include <Wire.h>
        #include <Adafruit_GFX.h>
        #include <Adafruit_SSD1306.h>
        #include <SoftwareSerial.h>
        #include <SD.h>

        #define DEBUG true

        #define OLED_MOSI  34  //SDA
        #define OLED_CLK   32  //SCL
        #define OLED_DC    38
        #define OLED_CS    13
        #define OLED_RESET 36
        Adafruit_SSD1306 display(OLED_MOSI, OLED_CLK, OLED_DC, OLED_RESET, OLED_CS);
        SoftwareSerial dbgSerial(10,11 ); // TX, RX

        byte sensorInterrupt = 0;  // 0 = digital pin 2
        byte sensorPin       = 2;
        float calibrationFactor = 4.5;

        volatile byte pulseCount;  

        float flowRate;
        unsigned int flowMilliLitres;
        unsigned long totalMilliLitres;

        unsigned long oldTime;

        //sd part value 
        File myFile; // สร้างออฟเจก File สำหรับจัดการข้อมูล
        const int chipSelect = 4;
        int countOfSDData = 0;
        String data1 = "";
        String data2 = "";
        String data3 = "";

        void setup() 
        { 
           display.begin();
           Serial.begin(9600); 
            dbgSerial.begin(9600);
  
          // sd part
          while (!Serial) {
            ; // รอจนกระทั่งเชื่อมต่อกับ Serial port แล้ว สำหรับ Arduino Leonardo เท่านั้น
          }
            Serial.print("Initializing SD card...");
            pinMode(SS, OUTPUT);
            if (!SD.begin(chipSelect)) {
            Serial.println("initialization failed!");
            return;
          }
          Serial.println("initialization done."); 
  
          if (SD.exists("wfdata.txt")) {
            SD.remove("wfdata.txt");
          }
          myFile = SD.open("wfdata.txt", FILE_WRITE);
            // ถ้าเปิดไฟล์สำเร็จ ให้เขียนข้อมูลเพิ่มลงไป
            if (myFile) {
              Serial.print("Writing...");
              myFile.println("START NEW DATA WATER FLOW"); // สั่งให้เขียนข้อมูล
              myFile.close(); // ปิดไฟล์
              Serial.println("done.");
            } else {
              // ถ้าเปิดไฟลืไม่สำเร็จ ให้แสดง error 
              Serial.println("error opening");
            }
  
  
  
          // water flow sensor part
          pinMode(sensorPin, INPUT);
          digitalWrite(sensorPin, HIGH);
          pulseCount        = 0;
          flowRate          = 0.0;
          flowMilliLitres   = 0;
          totalMilliLitres  = 0;
          oldTime           = 0;
          attachInterrupt(sensorInterrupt, pulseCounter, FALLING);
  
          // esp 8266 part
          sendData("AT+RST\r\n",2000,DEBUG); 
          sendData("AT+CWMODE=2\r\n",1000,DEBUG); 
          sendData("AT+CIPMUX=1\r\n",1000,DEBUG); 
          sendData("AT+CIPSERVER=1,80\r\n",1000,DEBUG);
          sendData("AT+CIFSR\r\n",1000,DEBUG);
 
  
        } 
        void loop ()    
        {
  
          //
          // water flow sensor part
          //
          if((millis() - oldTime) > 1000)    // Only process counters once per second
          { 
            detachInterrupt(sensorInterrupt);
    
            flowRate = ((1000.0 / (millis() - oldTime)) * pulseCount) / calibrationFactor;
            oldTime = millis();
    
            flowMilliLitres = (flowRate / 60) * 1000;
            totalMilliLitres += flowMilliLitres;
            data1 = "";
            data2 = "";
            data3 = "";
            unsigned int frac;
            Serial.print("Flow rate: ");
            data1 += "Flow rate: ";
            Serial.print(int(flowRate));  // Print the integer part of the variable
            data1 += int(flowRate);
            Serial.print(".");  
            data1 += ".";  
            Serial.print(frac, DEC) ;      // Print the fractional part of the variable
            int fracUSE = (frac, DEC);
            data1 += fracUSE;
            Serial.print("L/min");
            data1 += "L/min";
            Serial.print("  Current Liquid Flowing: ");             // Output separator
            data2 += "Current Liquid Flowing: ";
            Serial.print(flowMilliLitres);
            data2 += flowMilliLitres;
            Serial.print("mL/Sec");
            data2 += "mL/Sec";
            Serial.print("  Output Liquid Quantity: ");             // Output separator
            data3 += "Output Liquid Quantity: ";
            Serial.print(totalMilliLitres);
            data3 += totalMilliLitres;
            Serial.println("mL");
            data3 += "mL";
            // SD PART
            //
            //
            //
            myFile = SD.open("wfdata.txt", FILE_WRITE); // เปิดไฟล์ที่ชื่อ test.txt เพื่อเขียนข้อมูล โหมด FILE_WRITE
            
            countOfSDData++;
            // ถ้าเปิดไฟล์สำเร็จ ให้เขียนข้อมูลเพิ่มลงไป
            if (myFile) {
              Serial.print("Writing...");
              myFile.print("                ========  ");
              myFile.print(countOfSDData);
              myFile.println("  ========");
              myFile.println(data1); // สั่งให้เขียนข้อมูล
              myFile.println(data2);
              myFile.println(data3);
              myFile.close(); // ปิดไฟล์
              Serial.println("done.");
            } else {
              // ถ้าเปิดไฟลืไม่สำเร็จ ให้แสดง error 
              Serial.println("error opening");
            }
            //
            //
            //
            //
  
            pulseCount = 0;
            showSensorValue();
            attachInterrupt(sensorInterrupt, pulseCounter, FALLING);
          }
  
          //
          // esp 8266 part
          //
          if(dbgSerial.available()) // check if the esp is sending a message 
          {
            //Serial.println("available");
            if(dbgSerial.find("+IPD,"))
            {
     
            showSensorValue();
     
     
            //Serial.println("Web Start\r\n");
            delay(1000);
 
            int connectionId = dbgSerial.read()-48; 
     
            String webpage = "<h1><center><font size=7>Current Liquid Flowing: " + (String)flowMilliLitres + " mL/Sec</font></center></h1>";
 
            String cipSend = "AT+CIPSEND=";
            cipSend += connectionId;
            cipSend += ",";
            cipSend +=webpage.length();
            cipSend +="\r\n";
     
            sendData(cipSend,1000,DEBUG);
            sendData(webpage,1000,DEBUG);
          }
        }
 
        }

        void showSensorValue()
        {
          display.setTextSize(2);
          display.setTextColor(WHITE);
          display.setCursor(0,50);
          display.clearDisplay();
          display.print(flowMilliLitres);
          display.print(" mL/Sec\r\n");
          display.display(); 
        }



        // send a AT command for esp 8266 and debuging
        String sendData(String command, const int timeout, boolean debug)
        {
            String response = "";
    
            dbgSerial.print(command); // send the read character to the esp8266
            //Serial.println("dataSEND");
            long int time = millis();
    
            while( (time+timeout) > millis())
            {
              while(dbgSerial.available())
              {
        
                // The esp has data so display its output to the serial window 
                char c = dbgSerial.read(); // read the next character.
                response+=c;
              }  
            }
    
            if(debug)
            {
              Serial.print(response);
            }
    
            return response;
        }

        // interrupt sensor pin 2
        void pulseCounter()
        {
          pulseCount++;
        }
        
#GOOD LUCK
