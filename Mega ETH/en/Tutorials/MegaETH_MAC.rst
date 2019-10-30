.. highlight:: c++

###############################################
Setting up a Mega 2560 ETH Ethernet MAC address
###############################################

Network devices require two identifying addresses to operate on an Ethernet network: an IP address and a MAC (hardware) address. These two addresses operate at different layers of the network, IP is being assigned by a router or (rarely) manually, by an administrator of the network, whereas MAC is usually assigned at the factory, by the device manufacturer.
Since you are the manufacturer of the device that you build using a Mega ETH, you need to take care of configuring the MAC address yourself to connect it to a network.


Hard-coding the MAC address in your sketch
==========================================

The simplest way is to just hard-code the MAC address into the sketch. For example, start the Arduino IDE, go to File / Examples / Ethernet / WebServer and check out the code starting at line 20::

    // Enter a MAC address and IP address for your controller below.
    // The IP address will be dependent on your local network:
    byte mac[] = { 
      0xDE, 0xAD, 0xBC, 0xAF, 0xFE, 0xEB };
    IPAddress ip(192,168,0,101);

Later in the sketch those values are used when the Ethernet connection is initialized::

  Ethernet.begin( mac, ip );

You can choose to specify the IP address as shown (this is the case where IP address is being assigned manually by an administrator), or just specify only the MAC address and  obtain the IP address via DHCP (to make the network router choose and assign an unused IP address to the device).
As you can see in the givenexample, a MAC address is a sequence of 6 2-byte hexadecimal values. Some parts of the address carry a special meaning: for instance, the first three bytes are called the Organisationally Unique Identifier (OUI) while the last three bytes represent the specific Network Interface Controller (NIC). Specific bits within the OUI are also used to signify different modes of operqation of the network device, such as unicast or multicast. The safest way is to set the first byte to "0xDE" as in the example and modify the rest of the address.


Generating a unique MAC address

If you are going to deploy more than one Mega ETH to the same Ethernet network, hard-coding a MAC address into the sketch might not be the most convinient option, as you would have to change the sketch for every device and keep track of their values manually. The solution there is to generate a unique address from a random value, as the chances of you picking the same MAC address as some other device on your network is ridiculously small.

Storing MAC address in EEPROM

If you’re deploying a number of Mega ETH-based devices on a local network you’ll want them to retain their MAC addressess when powerded down and back up.

You don’t what the MAC address of each node to change each time it is restarted, this can cause certain issues with DHCP and also having a persistent MAC address is useful for sending messages over the network to uniquely identify which node it came from.

The example below generates a random MAC address and stores it in EEPROM. The random MAC address is generated on first power up, on each subsequent start after that it is read back out of EEPROM to ensure it stays the same::

    #include <SPI.h>
    #include <Ethernet.h>
    #include <EEPROM.h>
    
    byte mac[6] = { 0xDE, 0x00, 0x00, 0x00, 0x00, 0x00 };
    char macstr[18];
    
    void setup() {
      Serial.begin(9600);
      // Random MAC address stored in EEPROM
      if (EEPROM.read(0) == '#') {
        for (int i = 1; i <= 6; i++) {
          mac[i-1] = EEPROM.read(i);
        }
      } else {
        randomSeed(analogRead(0));
        mac[0] = 0xde;
        EEPROM.write(1, mac[0]);
        for (int i = 1; i < 6; i++) {
          mac[i] = random(0, 255);
          EEPROM.write(i+1, mac[i]);
        }
        EEPROM.write(0, '#');
      }
      snprintf(macstr, 18, "%02x:%02x:%02x:%02x:%02x:%02x", mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
    
      // Start up networking
      Ethernet.begin(mac);
    
      Serial.println(Ethernet.localIP());
    }
    
    void loop() {
    }

The only thing to watch out for is if you are using EEPROM elsewhere in your sketch that you don’t rewrite the values with some other stuff.


Storing MAC address on an SD card
=================================

The SD library is used to read from / write to an SD card using the built-in Mega ETH Card Reader. The library supports:
- FAT32 and FAT16 file systems;
- standard and high-capacity SD cards;
- 8.3 short filenames only.

The file names used with the SD library functions can include paths where directory names are separated by forward-slashes, "/", e.g. "dirname/filename.txt".

The microcontroller communicates to the SD card over SPI bus using pins 50, 51, and 52. Another pin must be used to select the SD card, this can be pin 53 or any other pin specified in the call to SD.begin(), it must be configured as output (even if not used), otherwise the SD library won't work.
Let’s change the last example to write the MAC address to an SD card file “mac.dat” instead of EEPROM::

    /*
      
      The circuit:
       SD card attached to SPI bus as follows:
    
      * MOSI - pin 11
      * MISO - pin 12
      * CLK - pin 13
      * CS - pin 4 
    
    */
    
    #include <SPI.h>
    #include <SD.h>
    #include <Ethernet.h>
    
    const int chipSelect = 4;
    byte mac[6] = { 0xDE, 0x00, 0x00, 0x00, 0x00, 0x00 };
    
    void setup() {
      // Open serial communications and wait for port to open:
      Serial.begin(9600);
      while (!Serial) {
        ; // wait for serial port to connect. Needed for native USB port only
      }
    
    
      Serial.print("Initializing SD card...");
    
      // see if the card is present and can be initialized:
      if (!SD.begin(chipSelect)) {
        Serial.println("Card failed, or not present");
        // don't do anything more:
        while (1);
      }
      Serial.println("card initialized.");
    
      // open the file.
    
      File dataFile = SD.open("mac.dat");
    
      // if the file is available, write to it:
    
      if (dataFile) {
        for (int i = 1; i <= 6; i++) {
          mac[i-1] = dataFile.read();
        }
      } else {
    
        // if not – create mac.dat, generate a random MAC
        // (first byte = 0xDE)
        // and store it to mac.dat
    
        dataFile = SD.open("mac.dat", FILE_WRITE);
        randomSeed(analogRead(0));
        mac[0] = 0xde;
        for (int i = 1; i < 6; i++) {
          mac[i] = random(0, 255);
        }
        dataFile.write(mac, 6);
        dataFile.close();
      }
      // Start up networking
      Ethernet.begin(mac);
    
      Serial.println(Ethernet.localIP());
    }
    
    void loop() {
    }


