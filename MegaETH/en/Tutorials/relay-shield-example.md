# Relay Shield Example

![Robotdyn Relay Shield](photo_angle01_0g-00004529_relayshielduno_assembled.jpg)

The RobotDyn Relay Shield provides four high quality relays with NO/NC interfaces. LED indicators show the on/off state of each relay.

In this tutorial we will show how to use a relay to control an LED, however, its real purpose is to control high current devices.

## Schematics

![Robotdyn Relay Shield with LED](robotdyn_relayshield_with_led.png)

## Code

```cpp
int RelayControl1 = 7;    
 
 
void setup()
{
    pinMode(RelayControl1, OUTPUT);
}
 
 
void loop()
{
    digitalWrite(RelayControl1,HIGH);// NO1 and COM1 Connected (LED on)
    delay(1000); // wait 1000 milliseconds (1 second)
    digitalWrite(RelayControl1,LOW);// NO1 and COM1 Disconnected (LED off)
    delay(1000); // wait 1000 milliseconds (1 second)
}
```