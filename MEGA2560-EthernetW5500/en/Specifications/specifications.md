## MEGA 2560 ETH R3 with ATmega2560 & W5500

<style>
    th {
        display: none;
    }
</style>

|<!-- -->|<!-- -->|
|---|---|
|Input Voltage (VIN/DCВ jack)|7~12V|
|PowerВ IN (USB)    |5V-limit 500mAВ |
|PoE Type    |No PoE/Active PoE/Passive PoE|
|PowerВ IN (PoE) |Optional module, 48V(Input), 9V(Output)|
|Digital I/O |54|
|PWM Output  |12|
|Analog I/O  |16|
|Reserved Pins   | &bull; D4В is used forВ SD card select;<br/> &bull; D10В is used for W5500 CS;<br/> &bull; *Optional: D8В is used for W5500В interrupting, D7В is used for W5500 initialization, D9В is used forВ SD card detect|
|USB socket  |Micro-USB|
|Ethernet socket |RJ45|
|PCB Size    |53.35Г—101.61mm|
|Card Reader |Micro SDВ card, with logic level convertor|
|Weight  |63g|

#### Mega MCU

|<!-- -->|<!-- -->|
|---|---|
|Microcontroller |ATmega2560(AVR 8-bit)|
|Operating Voltage   |5VВ |
|Memory Size |256KB|
|SRAM    |8KB|
|EEPROM  |4KB|
|Clock Speed |16MHz|

#### Ethernet MCU

|<!-- -->|<!-- -->|
|---|---|
|Microcontroller |Wiznet W5500|
|PHY compliance  |10BaseT/100BaseTX Ethernet. Full and half duplexВ|
|Operating Voltage   |3.3V|
|Memory  |Internal 32Kbytes Memory for Tx/Rx Buffers|
|TCP/IP Protocols    |TCP, UDP, ICMP, IPv4, ARP, IGMP, PPPoE|
|PHY Clock Speed |25MHz|

#### Passive PoE(optional)

|<!-- -->|<!-- -->|
|---|---|
|Ethernet Power (IN) |12~48V DC|
|Output  |DCВ 9V В|

#### Active PoE(optional)

|<!-- -->|<!-- -->|
|---|---|
|Ethernet Power (IN) |12~48V DC|
|Output  |DCВ 12V|

#### WiFi(optional)

|<!-- -->|<!-- -->|
|---|---|
|WiFI module |ESP-01(ESP8266)|
|Reserved Pins   | &bull; D14/TX3<br/> &bull; D15/RX3|


