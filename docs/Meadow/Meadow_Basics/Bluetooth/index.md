---
layout: Meadow
title: Bluetooth
subtitle: Meadow b5.0 BLE Draft Implementation
---

![](Bluetooth_Logo.svg)

## Bluetooth Low-Energy (BLE) Beta Feature Set

The b5.0 release of Meadow contains a draft subset of BLE features and while covering a large number of basic bluetooth use cases, it's by no means a complete BLE implementation, but rather a starting point from which additional features will be added. The BLE specification contains a very large number of potential features, from common to rarely used. As you use and test the current implementation, please help us prioritize and curate what features get added next. If you have a use case that is not currently covered, please file an issue in the [Meadow_Issues GitHub repo](https://github.com/WildernessLabs/Meadow_Issues/issues) describing your use case and the features needed.

### Currently Available Features

The following features are available in this release:

- **User-Definable BLE Definition Tree** - You can create a BLE tree `Definition` of _services_ and _characteristics_ that contain primitive type values including; `int`, `double`, `string`, etc.
- **BLE Server** - Start a bluetooth server on the Meadow and initialize it with your BLE definition tree.
- **Accept Client Connections** - Connect to the server from a _client_ device application such as a mobile phone.
- **Edit Values at Runtime** - Write values to the graph from your managed application. Those values can be read by a BLE Client app.
- **Value Change Notifications** - Get notified in your Meadow application when a BLE client writes to a characteristic in you BLE tree.

#### Planned Features

The following features are not available today, but are on-deck for implementation:

 * **Paring & Bonding** - The ability to pair a client to the server.
 * **Encrypted Communications** - Once paired and bonded, server and client will use encrypted transport for communications.
 * **`NOTIFY` Properties** - Once connected, _NOTIFY_ properties allow characteristics to _push_ values to the client from server.

## BLE Clients

There are a variety of BLE client applications and libraries. When filing bluetooth issues, please let us know which client you're using.

### Mobile BLE Clients

There are several good BLE mobile client applications available from iOS and Android app stores that can expedite testing and validation of BLE endpoints including:

 * Android: [BLE Scanner](https://play.google.com/store/apps/details?id=com.macdom.ble.blescanner)
 * iOS: [BLE Scanner 4.0](https://apps.apple.com/us/app/ble-scanner-4-0/id1221763603#?platform=iphone)

### C# and Xamarin BLE Clients

* **Bluetooth LE plugin for Xamarin** - A dedicated mobile BLE client that makes cross-platform BLE easy. [Nuget](https://www.nuget.org/packages/Plugin.BLE/), [GitHub Site](https://github.com/xabre/xamarin-bluetooth-le)
* **Shiny** - Cross-platform device services (including BLE) project. [Nuget](https://www.nuget.org/packages/Shiny.BluetoothLE/), [GitHub Site](https://github.com/shinyorg/shiny).

# Using Meadow's BLE Server 

## Pre-requisites

BLE requires your meadow be updated to the latest b5.0 binaries.  This includes both a Meadow OS and new firmware for the ESP32.  See the [Deploying Meadow.OS Guide](/Meadow/Getting_Started/Deploying_Meadow/) for more information.

## Defining a BLE Service Definition

The Meadow BLE server must be initialized with a `Definition` tree which includes a graph of the following three things:

 * **`Device`** - This is the Meadow device that hosts the BLE server. A BLE definition should include a `deviceName` property that provides a friendly name to identify the device.
 * **`Services`** - A service is a high-level group of accessible endpoints points, identified by a _UUID_ that define a particular "feature" that can be interacted with in BLE. There are a number of pre-defined services such as _Battery_, _Blood Pressure_, and _Device Information_ that have known UUIDs, but you can also define your own, custom services.
 * **`Characteristics`** - These are properties within a given service (also identified by a UUID) exposed as data endpoints that can be read from, and optionally, written to, by clients. As with Services, there are known characteristics such as _Apparent Wind Speed_, or _Humidity_, but again, you can also define your own custom characteristics.

### Important Note about Known Services and Characteristics

When creating known services and characteristics with established UUIDs, some of the features of them (such as whether they can be written to) might be locked down. Meaning that if a known characteristic is read-only, the underlying library will not allow it to be written to.

For a full list of known IDs for services and characteristics, see the [Bluetooth Assigned Numbers documents](https://www.bluetooth.com/specifications/assigned-numbers/)

### Creating a BLE `Definition` Tree

To define your server's characteristic graph, you must create a BLE `Definition` object tree. 

For example, the following code specifies a Meadow BLE server instance that advertises to BLE clients as a device named `MY MEADOW` and contains a single service with three simple properties:

```csharp
var definition = new Definition(
    "MY MEADOW",
    new Service(
        "ServiceA",
        253,
        new CharacteristicBool(
            "My Bool",
            uuid: "017e99d6-8a61-11eb-8dcd-0242ac1300aa",
            permissions: CharacteristicPermission.Read,
            properties: CharacteristicProperty.Read
            ),
        new CharacteristicInt32(
            "My Number",
            uuid: "017e99d6-8a61-11eb-8dcd-0242ac1300bb",
            permissions: CharacteristicPermission.Write | CharacteristicPermission.Read,
            properties: CharacteristicProperty.Write | CharacteristicProperty.Read
            ),
        new CharacteristicString(
            "My Text",
            uuid: "017e99d6-8a61-11eb-8dcd-0242ac1300cc",
            maxLength: 20,
            permissions: CharacteristicPermission.Write | CharacteristicPermission.Read,
            properties: CharacteristicProperty.Write | CharacteristicProperty.Read
            )
        )
    );

```

Examining the previous code, there are some important details:

 * The first property in the `Definition` tree is `deviceName`, which defines the name of the device/server.
 * The tree above creates a single service with a randomly chosen ID of `253`
 * Service Names are for convenience only, they are not viewable by the client.
 * Characteristics are typed to try to make programming easier without passing `byte[]` around.
 * The Uuid in the Bluetooth spec can be either a `Guid` or a `ushort` but the `ushort` gets translated to a `Guid` anyway, so we've opted for just `Guid` support in this release.
 *	Permissions versus properties are nuanced.  See the Bluetooth spec for details, but for general purposes just make them the same
 *  *Meadow currently only supports `Read` or `Write` even though the `enum`s have all of the BLE supported values*
 * Strings require a maxLength. Try not to exceed it.  Client writes of larger than this length may be problematic (we need to do more testing)

## Initializing the Bluetooth Server

Once you have a BLE tree definition you can start initialize the BLE server with the following code:

```csharp
Device.InitCoprocessor();
Device.BluetoothAdapter.StartBluetoothServer(definition);
```

## Setting Data for a Client to Read

Interacting with the Bluetooth Characteristics will be done through your `Definition`.  When you want to set a value for a Client to read, use the `SetValue()` method on a `Characteristic`.

For example, with our example definition, we could set the boolean and integer Characteristics every two seconds with a loop like this:

```csharp
bool state = false;
int value = 0;

while (true)
{
    // we can access the characteristics by index by name if we want:
    definition.Services[0].Characteristics["My Number"].SetValue(value);

    // or even by the UUID:
    definition.Services[0].Characteristics["017e99d6-8a61-11eb-8dcd-0242ac1300aa"].SetValue(state);

    value++;
    state = !state;

    Thread.Sleep(2000);
}
```

Note that you can access a Characteristic by index, name or UUID (the latter two are case-insensitive as well);

## Knowing when a Client Writes a Value

Your application can be notified when a Client sets a `Characteristic` value through the `ValueSet` event.  You can wire up any of the `Characteristics` you're interested in.

To continue our example, if we wanted to wire up all of the Characteristics (yes, in this example even the read-only ones that will never actually get written to) we could use the following:

```csharp
foreach (var characteristic in bleTreeDefinition.Services[0].Characteristics) {
    characteristic.ValueSet += (c, d) => {
        Console.WriteLine($"HEY, I JUST GOT THIS BLE DATA for Characteristic '{c.Name}' of type {d.GetType().Name}: {d}");
    };
}
```

The incoming parameters are the `Characteristic` definition being set and a type-safe value.

