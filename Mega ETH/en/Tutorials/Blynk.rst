.. highlight:: c++

#############
Blynk Example    
#############

Blynk is a platform with iOS and Android apps to control
Mega ETH and other popular microcontrollers over the Internet.
You can easily build graphic interfaces for all your
projects by simply dragging and dropping widgets.

   * Downloads, docs, tutorials: http://www.blynk.cc
   * Sketch generator:           http://examples.blynk.cc
   * Blynk community:            http://community.blynk.cc
   * Follow Blynk:                  http://www.fb.com/blynkapp
                                http://twitter.com/blynk_app

Blynk library is licensed under MIT license
This example code is in public domain.

You’ll need:

   * Blynk App (download from AppStore or Google Play)
   * RobotDyn Mega ETH board

::

    #include <SPI.h>
    #include <Ethernet3.h>
    #include <BlynkSimpleEthernet2.h>
    
    // You should get Auth Token in the Blynk App.
    // Go to the Project Settings (nut icon).
    char auth[] = "YourAuthToken";
    
    
    void setup()
    {
      // Debug console
      Serial.begin(9600);
    
      Blynk.begin(auth);
      // You can also specify server:
      //Blynk.begin(auth, "blynk-cloud.com", 80);
      //Blynk.begin(auth, IPAddress(192,168,1,100), 8080);
    }
    
    void loop()
    {
      Blynk.run();
      // You can inject your own code or combine it with other sketches.
      // Check other examples on how to communicate with Blynk. Remember
      // to avoid delay() function!
    }
    
