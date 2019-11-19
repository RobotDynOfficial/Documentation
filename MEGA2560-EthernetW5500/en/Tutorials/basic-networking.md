# Basic Networking Set-up with Mega 2560 ETH

## Table of Contents

[**Minimal code structure**](#minimal-code-structure)

[**Setting up MAC address**](#setting-up-ethernet-mac-address)

  [Hard-coding the MAC address in your sketch](#hard-coding-the-mac-address-in-your-sketch)

  [Generating a unique MAC address](#generating-a-unique-mac-address)

  [Storing MAC address in EEPROM](#storing-mac-address-in-eeprom)

  [Storing MAC address on an SD card](#storing-mac-address-on-an-sd-card)

[**IP address**](#ip-address)

  [IPv4 vs IPv6]|(#ipv4-vs-ipv6)

  [Hard-coding IP address into the sketch](#hard-coding-ip-address-into-the-sketch)

  [Dynamic IP address assignment](#dynamic-ip-address-assignment)



## Minimal code Structure

The minimal valid code to run on Mega 2560 ETH has the following structure:

```cpp

void setup() {
  // code that you put here will run exactly once, at startup

}

void loop() {
  // code that you put here will run repeatedly until power off or reset to run repeatedly:

}
```

Every line that starts with "//" is ignored by the compiler, so you can put any descriptions and comments in the code where you need them.


## Setting up a Mega 2560 ETH Ethernet MAC address

Network devices require two identifying addresses to operate on an Ethernet network: an IP address and a MAC (hardware) address. These two addresses operate at different layers of the network, IP is being assigned by a router or (rarely) manually, by an administrator of the network, whereas MAC is usually assigned at the factory, by the device manufacturer.
Since you are the manufacturer of the device that you build using a Mega ETH, you need to take care of configuring the MAC address yourself to connect it to a network.


### Hard-coding the MAC address in your sketch

The simplest way is to just hard-code the MAC address into the sketch. For example, start the Arduino IDE, go to File / Examples / Ethernet / WebServer and check out the code starting at line 20:
```cpp

    // Enter a MAC address and IP address for your controller below.
    // The IP address will be dependent on your local network:
    byte mac[] = { 
      0xDE, 0xAD, 0xBC, 0xAF, 0xFE, 0xEB };
    IPAddress ip(192,168,0,101);

```
Later in the sketch those values are used when the Ethernet connection is initialized:
```cpp
  Ethernet.begin( mac, ip );
```
You can choose to specify the IP address as shown (this is the case where IP address is being assigned manually by an administrator), or just specify only the MAC address and  obtain the IP address via DHCP (to make the network router choose and assign an unused IP address to the device).
As you can see in the given example, a MAC address is a sequence of 6 2-byte hexadecimal values. Some parts of the address carry a special meaning: for instance, the first three bytes are called the Organizationally Unique Identifier (OUI) while the last three bytes represent the specific Network Interface Controller (NIC). Specific bits within the OUI are also used to signify different modes of operation of the network device, such as unicast or multicast. The safest way is to set the first byte to "0xDE" as in the example and modify the rest of the address.


### Generating a unique MAC address

If you are going to deploy more than one Mega ETH to the same Ethernet network, hard-coding a MAC address into the sketch might not be the most convenient option, as you would have to change the sketch for every device and keep track of their values manually. The solution there is to generate a unique address from a random value, as the chances of you picking the same MAC address as some other device on your network is ridiculously small.

### Storing MAC address in EEPROM

If you’re deploying a number of Mega ETH-based devices on a local network you’ll want them to retain their MAC addresses when powered down and back up.

You don’t what the MAC address of each node to change each time it is restarted, this can cause certain issues with DHCP and also having a persistent MAC address is useful for sending messages over the network to uniquely identify which node it came from.

The example below generates a random MAC address and stores it in EEPROM. The random MAC address is generated on first power up, on each subsequent start after that it is read back out of EEPROM to ensure it stays the same::
```cpp
    
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

```
The only thing to watch out for is if you are using EEPROM elsewhere in your sketch that you don’t rewrite the values with some other stuff.


### Storing MAC address on an SD card

The SD library is used to read from / write to an SD card using the built-in Mega ETH Card Reader. The library supports:
- FAT32 and FAT16 file systems;
- standard and high-capacity SD cards;
- 8.3 short filenames only.

The file names used with the SD library functions can include paths where directory names are separated by forward-slashes, "/", e.g. "dirname/filename.txt".

The micro controller communicates to the SD card over SPI bus using pins 50, 51, and 52. Another pin must be used to select the SD card, this can be pin 53 or any other pin specified in the call to SD.begin(), it must be configured as output (even if not used), otherwise the SD library won't work.
Let’s change the last example to write the MAC address to an SD card file “mac.dat” instead of EEPROM::
```cpp
    
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

```


## IP address

IP address is a numerical identifier assigned to each device that is connected to a network that uses Internet Protocol for communication. An IP address can be assigned either manually, by a network administrator, or automatically by a network router or a similar device. The former is called "static IP address setup", the latter is called "dynamic" or DHCP setup, since it uses the DHCP protocol for the IP address queries and assignments.

### IPv4 vs IPv6

There are 2 version of the IP protocol and addresses: IPv4 and, newer, IPv6. IPv4 defines an IP address as a 32-bit integer number denoted as 4 0-255 values separated with ".", e.g.: 192.168.0.1
Because of the depletion of the IPv4 IP address space the newer IPv6 protocol uses 128-bit addresses, denoted as 6 16-bit hexadecimal numbers (0x0 - 0xFFFF) separated by ":", e.g.: 2001:db8:0:1234:0:567:8:1
The standard Arduino networking libraries (Ethernet3, Ethernet4) support IPv4 only. The W5500 chip itself supports only IPv4 natively, however there is a low-level MACRAW mode for sending Ethernet frames directly and there are some alternative networking libraries (e.g. [Ethersia](https://github.com/njh/ethersia))with IPv6 support built-in.

### Hard-coding IP address into the sketch

The simplest way to assign an IP address to your Mega ETH-based device is to hard-code the IP address into the sketch:

```c++
   
    #include <SPI.h>
    #include <Ethernet3.h>

    // network configuration. dns server, gateway and subnet are optional.

     // the media access control (ethernet hardware) address:
    byte mac[] = { 0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED };  

    // the dns server ip
    IPAddress dnServer(192, 168, 0, 1);
    // the router's gateway address:
    IPAddress gateway(192, 168, 0, 1);
    // the subnet:
    IPAddress subnet(255, 255, 255, 0);

    //the IP address is dependent on your network
    IPAddress ip(192, 168, 0, 2);

    void setup() {
      Serial.begin(9600);

      // initialize the ethernet device
      Ethernet.begin(mac, ip, dnServer, gateway, subnet);
      //print out the IP address
      Serial.print("IP = ");
      Serial.println(Ethernet.localIP());
    }

    void loop() {
    }

```

### Dynamic IP address assignment

If you are going to connect several Mega ETH devices to the same network segment, hard-coding an IP address might not be very convenient, since you don't want to change the sketch for every device you deploy. In this case and in all cases where DHCP service is available on the network, it would be much better to automatically obtain an IP address from the DHCP server and store it in non-volatile memory, EEPROM or SD card, because, in most cases, it is desirable for a network device to retain it's assigned IP address every time it is powered up.
In the following example Mega ETH will obtain an IP address via DHCP protocol using the [Ethernet3 library](https://github.com/sstaub/Ethernet3) and display it on a 16x24 LCD monitor:

```c
  
    /*

      This sketch uses the DHCP to get an IP address
      and display the address on an LCD.

     */

    #include <SPI.h>
    #include <Ethernet3.h>
    #include <LiquidCrystal.h>

    // Enter a MAC address for your controller below.

    byte mac[] = {
      0xDE, 0xAB, 0xCD, 0xEF, 0xDE, 0x02
    };
    // initialize the LCD library with the numbers of the interface pins
    LiquidCrystal lcd(12, 11, 5, 4, 3, 2);

    void setup() {
     
      
      lcd.begin(16, 2);

      if (Ethernet.begin(mac) == 0) {
        if (Ethernet.linkStatus() == LinkOFF) {
          lcd.print("LINK FAILURE");
        }
        else lcd.print("DHCP ERROR!");
      } else // success, display your local IP address:
             lcd.print(Ethernet.localIP());

        // do nothing more:
      while (true) {
          delay(1);
      }
    }

    void loop() {
    }

```


