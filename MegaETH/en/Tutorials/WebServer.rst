.. highlight:: c++

##########
Web Server
##########

In the following example, we will create a sketch that turns your Mega ETH board to a Web Server that serves just one static HTML page that it reads from a file on an SD card.


Creating the Web Page
=====================

Let’s create a simple HTML page and store it on an SD card using your computer and a card reader. While you can use almost any text editor to write HTML code, it will be much easier if it can highlight the HTML syntaxis and automatically close HTML tags. Geany (geany.org) is an example of such an editor available on almost all operating systems.
Create the following web page in the editor:

.. code-block:: HTML

	<!DOCTYPE html>
	<html>
	    <head>
	        <title>Test SD Card Web Page</title>
	    </head>
	    <body>
	        <h1>Hello from the Mega ETH Web Server!</h1>
	        <p>An example web page stored on the SD card.</p>
	    </body>
	</html>

Store it on the SD card in the root directory as *index.html*.


Preparing the sketch
====================

We will need to configure the pin 4 for the SD card reader to work, then we initialize the network library, generate a presumably unique MAC address following our previous recommendations of setting the first byte of MAC address to 0xDE for better network equipment compatibility and store that MAC address in the microcontroller’s EEPROM.

We will use DHCP to query an available IP address from the network router and display the IP address on a 16x2 LCD monitor because we need to type that IP address into a browser, that must be running on a computer on the same local network.

We assume that the HTML file is already stored on the SD card and the card is inserted into the onboard card reader.

Let’s get down to the code:

::

	#include <SPI.h>
	#include <Ethernet3.h>
	#include <EEPROM.h>
	#include <LiquidCrystal.h>
	#include <SD.h>

	byte mac[6] = { 0xDE, 0x00, 0x00, 0x00, 0x00, 0x00 };
	char macstr[18];
	
	// initialize the LCD library with the numbers of the interface pins
	LiquidCrystal lcd(12, 11, 5, 4, 3, 2);
	
	EthernetServer server(80);  // create a server at port 80
	
	void setup() {
	
	  lcd.begin(16, 2); //Initialize the 16x2 LCD screen
	
	  // Check EEPROM byte with offset==0 for “#” char. The presence of “#” in the first byte of EEPROM will be a sign that the MAC address was already generated and stored in bytes 1-7 of EEPROM. Otherwise, we need to generate a random MAC, put “#” at offset 0 and the MAC in the following 6 bytes
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
	
	  // Startup networking, get IP, DNS, and subnet from the DHCP server
	
	  if (Ethernet.begin(mac) == 0) {
	    if (Ethernet.linkStatus() == LinkOFF) {
	      lcd.print("LINK FAILURE");
	      exit()
	    }
	    else {
	            lcd.print("DHCP ERROR!");
	            exit(0)
	    }
	  } else // success, display your local IP address:
	         lcd.print(Ethernet.localIP()); //You can enter this IP in your web browser
	}
	
	void loop() {
	  // listen for incoming clients
	  EthernetClient client = server.available();
	  if (client) {  // got client?
	        boolean currentLineIsBlank = true;
	        while (client.connected()) {
	            if (client.available()) {   // client data available to read
	                char c = client.read(); // read 1 byte (character) from client
	                // last line of client request is blank and ends with \n
	                // respond to client only after last line received
	                if (c == '\n' && currentLineIsBlank) {
	                    // send a standard http response header
	                    client.println("HTTP/1.1 200 OK");
	                    client.println("Content-Type: text/html");
	                    client.println("Connection: close");
	                    client.println();
	                    // send web page
	                    webFile = SD.open("index.htm");        // open web page file
	                    if (webFile) {
	                        while(webFile.available()) {
	                            client.write(webFile.read()); // send web page to client
	                        }
	                        webFile.close();
	                    }
	                    break;
	                }
	                // every line of text received from the client ends with \r\n
	                if (c == '\n') {
	                    // last character on line of received text
	                    // starting new line with next character read
	                    currentLineIsBlank = true;
	                } 
	                else if (c != '\r') {
	                    // a text character was received from client
	                    currentLineIsBlank = false;
	                }
	            } // end if (client.available())
	        } // end while (client.connected())
	        delay(1);      // give the web browser time to receive the data
	        client.stop(); // close the connection
	    } // end of session with client
	}









