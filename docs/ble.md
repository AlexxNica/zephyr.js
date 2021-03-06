ZJS API for Bluetooth Low Energy (BLE)
======================================

* [Introduction](#introduction)
* [Web IDL](#web-idl)
* [API Documentation](#api-documentation)
* [Client Requirements](#client-requirements)
* [Sample Apps](#sample-apps)

Introduction
------------
The BLE API is based off the [bleno API](https://github.com/sandeepmistry/bleno). Bluetooth Low Energy (aka
[Bluetooth Smart](https://www.bluetooth.com/what-is-bluetooth-technology/bluetooth-technology-basics/low-energy)) is a power-friendly version of Bluetooth
intended for IoT devices. It provides the ability to support low-bandwidth data
services to nearby devices.

A note about UUIDs. The BLE standard lets you have full 128-bit UUIDs or short
16-bit UUIDs (4 hex chars). We only support the short ones so far.

Based on the bleno API, these UUIDs are specified as a hexadecimal string, so
"2901" really means 0x2901. *NOTE: This seems like a bad practice and we should
perhaps require these numbers to be specified as "0x2901" instead, or else
treat them like decimals.*

Web IDL
-------
This IDL provides an overview of the interface; see below for documentation of
specific API functions.

```javascript
// require returns a BLE object
// var ble = require('ble');

[NoInterfaceObject]
interface BLE {
    void disconnect();
    void on(string eventType, EventCallback callback);
    void startAdvertising(string name, string[] uuids, string url);
    void stopAdvertising();
    void setServices();
    void updateRssi();
    PrimaryService PrimaryService(PrimaryServiceInit init);
    Characteristic Characteristic(CharacteristicInit init);
    Descriptor Descriptor(DescriptorInit init);
};

callback EventCallback = void (various);  // callback arg depends on event

dictionary PrimaryServiceInit {
    string uuid;
    Characteristic[] characteristics;
};

dictionary CharacteristicInit {
    string uuid;
    string[] properties;                // 'read', 'write', 'notify'
    Descriptor[] descriptors;
    ReadCallback onReadRequest;         // optional
    WriteCallback onWriteRequest;       // optional
    SubscribeCallback onSubscribe;      // optional
    UnsubscribeCallback onUnsubscribe;  // optional
    NotifyCallback onNotify;            // optional
};

callback ReadCallback = void (unsigned long offset, FulfillReadCallback);
callback WriteCallback = void (Buffer data, unsigned long offset,
                               boolean withoutResponse, FulfillWriteCallback);
callback SubscribeCallback = void (unsigned long maxValueSize,
                                   FulfillSubscribeCallback);
callback FulfillReadCallback = void (CharacteristicResult result, Buffer data);
callback FulfillWriteCallback = void (CharacteristicResult result);
callback FulfillSubscribeCallback = void (Buffer data);

dictionary DescriptorInit {
    string uuid;
    string value;
};

[NoInterfaceObject]
interface Characteristic {
    attribute ReadCallback onReadRequest;
    attribute WriteCallback onWriteRequest;
    attribute SubscribeCallback onSubscribe;
    attribute UnsubscribeCallback onUnsubscribe;
    attribute NotifyCallback onNotify;
    unsigned long RESULT_SUCCESS;
    unsigned long RESULT_INVALID_OFFSET;
    unsigned long RESULT_INVALID_ATTRIBUTE_LENGTH;
    unsigned long RESULT_UNLIKELY_ERROR;
};
```

API Documentation
-----------------
### BLE.disconnect

`void disconnect();`

Disconnect the remote client.

### BLE.on

`void on(string eventType, EventCallback callback);`

The `eventType` can be one of the following:
* 'accept' - a BLE client has connected
  * callback should expect an "client address"
  * *NOTE: currently the address is dummy string "AB:CD:DF:AB:CD:EF"*
* 'advertisingStart' - BLE services have begun to be advertised
  * callback should expect an error integer (0 for success)
  * *NOTE: this may change*
* 'disconnect' - a BLE client has disconnected
  * callback should expect an "client address"
  * *NOTE: currently the address is dummy string "AB:CD:DF:AB:CD:EF"*
* 'rssiUpdate' - most recent RSSI (Received signal strength indication) value
  * callback should expect an integer value measured in dBm
  * *NOTE: currently the value is dummy value of "-50"*
* 'stateChange' - state of BLE stack has changed
  * callback should expect a string state:
    * 'poweredOn' - the BLE stack is now ready to be used
    * no others supported at this time

### BLE.startAdvertising

`void startAdvertising(string name, string[] uuids, string url);`

The `name` is limited to 26 characters and will be advertised as the device
name to nearby BLE devices.

The `uuids` array may contain at most 7 16-bit UUIDs (four hex digits each).
These UUIDs identify available services to nearby BLE devices.

The `url` is optional and limited to around 24 characters (slightly more
if part of the URL is able to be [encoded](https://github.com/google/eddystone/tree/master/eddystone-url). If provided,
this will be used to create a physical web advertisement that will direct users
to the given URL. At that URL they might be able to interact with the
advertising device somehow.

### BLE.stopAdvertising

`void stopAdvertising();`

Currently does nothing.

### BLE.setServices

`void setServices(PrimaryService[]);`

Pass an array of PrimaryService objects to set up the services that are
implemented by your app.

### BLE.updateRssi()

`void updateRssi();`

Query the RSSI (received signal strength indication) value, it will trigger
a 'rssiUpdate' event that will contain the relative signal value.

### BLE.PrimaryService constructor

`PrimaryService(PrimaryServiceInit init);`

The `init` object should contain a `uuid` field with a 16-bit service UUID (4
hex chars) and a `characteristics` field with an array of Characteristic
objects.

### BLE.Characteristic constructor

`Characteristic(CharacteristicInit init);`

The `init` object should contain:
* `uuid` field with a 16-bit characteristic UUID (4 hex chars)
* `properties` field with an array of strings that may include 'read', 'write',
  and 'notify', depending on what is supported
* `descriptors` field with an array of Descriptor objects

It may also contain these optional callback fields:
* `onReadRequest` function(offset, callback(result, data))
  * Called when the client is requesting to read data from the characteristic.
  * See below for common argument definitions
* `onWriteRequest` function(data, offset, withoutResponse, callback(result))
  * Called when the client is requesting to write data to the characteristic.
  * `withoutResponse` is true if the client doesn't want a response
    * *TODO: verify this*
* `onSubscribe` function(maxValueSize, callback(data))
  * Called when a client signs up to receive notify events when the
      characteristic changes.
  * `maxValueSize` is the maximum data size the client wants to receive.
* `onUnsubscribe` function()
  * *NOTE: Never actually called currently.*
* `onNotify` function()
  * *NOTE: Never actually called currently.*

Explanation of common arguments to the above functions:
* `offset` is a 0-based integer index into the data the characteristic
    represents.
* `result` is one of these values defined in the Characteristic object.
  * RESULT_SUCCESS
  * RESULT_INVALID_OFFSET
  * RESULT_INVALID_ATTRIBUTE_LENGTH
  * RESULT_UNLIKELY_ERROR
* `data` is a [Buffer](./buffer.md) object.

### BLE.Descriptor constructor

`Descriptor(DescriptorInit init);`

The `init` object should contain:
* `uuid` field with a 16-bit descriptor UUID (4 hex chars)
  * Defined descriptors are listed here in [Bluetooth Specifications](https://www.bluetooth.com/specifications/gatt/descriptors)
* `value` field with a string supplying the defined information
  * *NOTE: Values can also be Buffer objects, but that's not currently supported.*

Client Requirements
-------------------
You can use any device that has BLE support to connect to the Arduino 101 when
running any of the BLE apps or demos. We've successfully tested the following
setup:

* Update the Bluetooth firmware, follow instructions [here](https://wiki.zephyrproject.org/view/Arduino_101#Bluetooth_firmware_for_the_Arduino_101)
* Any Android device running Android 6.0 Marshmallow or higher (We use Nexus 5/5X, iOS devices not tested)
* Let the x86 core use 256K of flash space with ROM=256 as described [here](https://github.com/01org/zephyr.js#getting-more-space-on-your-arduino-101)

For the WebBluetooth Demo which supports the Physical Web, you'll need:
* Chromium version 50.0 or higher
* Make sure Bluetooth is on and Location Services is enabled
* Go to chrome://flags and enable the #enable-web-bluetooth flag

Sample Apps
-----------
* [BLE with multiple services](../samples/BLE.js)
* [WebBluetooth Demo](../samples/WebBluetoothDemo.js)
* [WebBluetooth Demo with Grove LCD](../samples/WebBluetoothGroveLcdDemo.js)
* [Heartrate Demo with Grove LCD](../samples/HeartRateDemo.js)
