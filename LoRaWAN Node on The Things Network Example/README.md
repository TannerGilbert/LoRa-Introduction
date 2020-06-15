# LoRaWAN Node on The Things Network

> The LoRaWAN® specification is a Low Power, Wide Area (LPWA) networking protocol designed to wirelessly connect battery operated 'things' to the internet in regional, national or global networks, and targets key Internet of Things (IoT) requirements such as bi-directional communication, end-to-end security, mobility and localization services. - [Lora Alliance](https://lora-alliance.org/about-lorawan)

[The Things Network](https://www.thethingsnetwork.org/) is a project dedicated to building an open, free, and decentralized internet of things network.

## Creating a The Things Network Account

![Create Account](doc/sign_up.png)

To create a The Things Network Account navigate to https://www.thethingsnetwork.org/ and click on the "Sign up" button. After that, you'll need to enter a username, email, and password. Be sure to enter a good username since you won't be able to change it after creating the account. 

## Creating an application

Next, you'll need to create an application. For this, navigate to the application page and click "add application".

![Create application](doc/create_application.png)

## Registering the device at The Things Network

Now that you've created an application you're ready to register a device.

![Adding device](doc/adding_device.PNG)

After adding the device go to the Settings tab and change the **Activation Method** to **ABP**.

![Change Activation Method](doc/device_settings.png)

Now you should see a device address, network session key, and app session key under the Overview tab. These three values will later be needed to get the script to work.

![Device Overview](doc/device_overview.png)

## Install arduino-lmic

LMiC (formerly 'LoRa MAC in C') is IBM's LoRa library. Arduino-LMIC contains the IBM LMIC (LoraMAC-in-C) library, slightly modified to run in the Arduino environment, allowing using the SX1272, SX1276 transceivers and compatible modules (such as some HopeRF RFM9x modules).

To install the library:
* go to **Sketch > Include Library > Manage Libraries** and search for lmic and install the **IBM LMIC Framework** library.

## Wiring RFM95 and ESP32

![RMF95 Pinout](doc/rmf95_pinout.jpg)

The RFM95 communicates with the microcontroller over [SPI](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface). To get the module to communicate with the ESP32 correctly connect the pins as follows:

* GND: GND
* 3.3V: 3.3V
* MISO: D19
* MOSI: D23
* SCK: D18
* RESET/RST: D14
* DIO0: 2
* DIO1: 15
* DIO2: 4
* NSS: 5

Note: Of the three GND pins, only one needs to be connected.

If you are using the RFM Adapter from school you'll have to use the following image:

![](doc/rfm_adapter.png)

**Important**: Don't forget to connect an antenna to the ANT pin.

## Creating a sender script

The LMIC library already includes a script for The Things Network, which can be accessed under **Examples > IBM LMIC Framework > ttn**.

To get the script to work, you have to enter the Network Session Key, App Session Key, and Device Address from the Overview tab of the Device. You'll also have to update the **lmic_pinmap lmic_pins** variables depending on how you connected it to your microcontroller.

In my case, I used the following pins:

```c
const lmic_pinmap lmic_pins = {
    .nss = 6,
    .rxtx = LMIC_UNUSED_PIN,
    .rst = 5,
    .dio = {2, 3, 4},
};
```

After running the file, you should see something like the following in the Serial Monitor

![Sending script output](doc/running_sending_script.PNG)

You should now be able to see the send data in the **Data tab** of the Application or Device page.

![Device data](doc/device_data.png)

## Encoding/Decoding data

LoraWAN and TTN transfer raw bytes, which can be hard to read. That's the reason why they provide libraries to encode/decode data. For Arduino they have the [Lora Serialization library](https://github.com/thesolarnomad/lora-serialization). The library allows you to encode your data on the Arduino side and decode it on the TTN side.

If you for example have temperature, humidity, pressure and sea_leavel readings you can encode/decode your data as follows:

Arduino side:
```c
LoraMessage message;

message
    .addTemperature(temperature)
    .addHumidity(humidity)
    .addUnixtime(pressure)
    .addUint16(sea_level);
```

TTN side (Payload Formats):
```javascript
// include src/decoder.js
function Decoder(bytes, port) {
  var json = decode(bytes, [temperature, humidity, unixtime, uint16], ['temperature', 'humidity', 'pressure', 'sea_level']);
  return json;
}
```

![Payload Formats](doc/payload_formats.PNG)

If you now look at the data tab you'll be able to see the data as plain text.

![](doc/decoded_data.PNG)

## Gateway data retrieval over MQTT

Now we are receiving the data in TTN, but how can we now get the data from TTN? TTN allows you to get the data over MQTT, an extremely lightweight machine-to-machine(M2M) connectivity protocol using a publish/subscribe model.

In the following examples, I'll show you how to receive data by using [Mosquitto's CLI](https://mosquitto.org/download/), but TTN also provides libraries for multiple programming languages, including Java, Node.js, and Python. For more information, check out the [SDKs & Libraries section in the documentation](https://www.thethingsnetwork.org/docs/applications/sdks.html).

### Receiving Messages (up)

To receive data from a TTN application, execute the following command:

```bash
mosquitto_sub -h <Region>.thethings.network -t '+/devices/+/up' -u '<AppID>' -P '<AppKey>' -v
```

> Don't forget to replace \<Region>, \<AppID>, \<AppKey> with the right values for your application. You can find them in the Overview tab of your application. The region is can be found under **Handler**. You will only need the part that follows **ttn-handler-**, e.g. **eu**.

> Note: ```-t``` stands for the topic do subscribe to. The topic has the following structure: ```AppID/devices/deviceID/up```. Attributes between // can also be replaced with a ```+```, which stands for everything. So if instead of a deviceID you put a ```+``` you'll listen to every device registered in the application.

If you only want to get a specific field you can write the name of the field after **up/**.

```bash
mosquitto_sub -h <Region>.thethings.network -t '+/devices/+/up/led' -u '<AppID>' -P '<AppKey>' -v
```

### Sending Messages (down)

MQTT can also be used to send messages to TTN. For this you will have to address a specific device by its **Device ID**.

```
mosquitto_pub -h <Region>.thethings.network -t "<AppID>/devices/<DevID>/down" -u "<AppID>" -P "<AppKey>" -m "{""payload_fields"":{""led"":true}}"
```