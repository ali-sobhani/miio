# miIO Device Library

This is a small library for controlling Mi Home devices that implement the miIO
protocol, such as the Mi Air Purifier, Mi Robot Vacuum and Mi Smart Socket.

This library requires at least Node 6.0.0.

## Devices types

The intent of this library is to support all miIO-compatible devices and to
provide an easy to use API for them. The library maps specific device models to
generic device types to simplify interacting with them.

Currently supported devices are:

* Air Purifiers (1, 2 and Pro)
* Mi Smart Socket Plug
* Mi Robot Vacuum
* Mi Smart Home Gateway (Aqara) and accessories

See [documentation for devices](docs/devices/README.md) for information about
the types, their API and supported device models.

## Installation

```
npm install --save miio
```

## Usage

```javascript
const miio = require('miio');
```

Resolve a handle to the device:

```javascript
// Resolve a device, resolving the token automatically if possible
miio.device({ address: '192.168.100.8' })
	.then(console.log)
	.catch(console.error);

// Resolve a device, specifying the token (see below for how to get the token)
miio.device({ address: '192.168.100.8', token: 'token-as-hex' })
	.then(console.log)
	.catch(console.error);
```

Call methods to interact with the device:

```javascript
// Call any method via call
device.call('set_mode', [ 'silent' ])
	.then(console.log)
	.catch(console.error);

// Fetch some properties
device.getProperties([ 'power', 'mode', 'aqi', 'temp_dec', 'humidity' ])
	.then(console.log)
	.catch(console.error);

// Or if a model has been specified use some of its methods
device.setPower(false)
	.then(on => console.log('Power is now', on));
```

Monitor and transform properties:
```javascript
// Define a property that should be monitored
device.defineProperty('mode');

// Define that a certain property should be run through a custom conversion
device.defineProperty('temp_dec', v => v / 10.0);

// Listen for changes to properties
device.on('propertyChanged', e => console.log(e.property, e.oldValue, e.value));

// Activate automatic property monitoring (activated by default for most devices)
device.monitor();

// Stop automatic property monitoring
device.stopMonitoring();

// Fetch the last value of a monitored property
const value = device.property('temp_dec');
```

If you are done with the device call `destroy` to stop all network traffic:

```javascript
device.destroy();
```

## Tokens

A few miIO devices send back their token during a handshake and can be used
without figuring out the token. Most devices hide their token, such as
Yeelights and the Mi Robot Vacuum.

[Finding tokens](docs/tokens.md) contains more information about how to get the
token of those devices.

## Discovering devices

Use `miio.browser()` to look for devices on the local network. This method of
discovery will tell you directly if a device reveals its token and can be
auto-connected to. It will not tell you the model of devices until they are
connected to via `miio.device()`.

Example:

```javascript
const browser = miio.browser({
	cacheTime: 300 // 5 minutes. Default is 1800 seconds (30 minutes)
});

const devices = {};
browser.on('available', reg => {
	if(! reg.token) {
		console.log(reg.id, 'hides its token');
		return;
	}

	miio.device(reg)
		.then(device => {
			devices[reg.id] = device;

			// Do something useful with the device
		})
		.catch(handleErrorProperlyHere);
});

browser.on('unavailable', reg => {
	const device = devices[reg.id];
	if(! device) return;

	device.destroy();
	delete devices[reg.id];
})
```

You can also use mDNS for discovery, but this library does not contain a mDNS
implementation. You can choose a mDNS-implementation suitable for your
needs. Devices announce themselves via `_miio._udp` and should work for most
devices, in certain cases you might need to restart your device to make it
announce itself.

## Advanced: Skip model and token checks

The `miio.device` function will return a promise that checks that we can
communicate with the device and what model it is. If you wish to skip this
step and just create a reference to a device use `miio.createDevice`:

```javascript
const device = miio.createDevice({
	address: '192.168.100.8',
	token: 'token-as-hex',
	model: 'zhimi.airpurifier.m1'
});
```

## Advanced: Device management

Get information and update the wireless settings of devices via the management
API.

Discover the token of a device:
```javascript
device.discover()
	.then(info => console.log(info.token));
```

Get internal information about the device:
```javascript
device.management.info()
	.then(console.log);
```

Update the wireless settings:
```javascript
device.management.updateWireless({
	ssid: 'SSID of network',
	passwd: 'Password of network'
}).then(console.log);
```

Warning: The device will either connect to the new network or it will end up
creating its own network if the settings are invalid.

## Protocol documentation

This library is based on the documentation provided by OpenMiHome. See https://github.com/OpenMiHome/mihome-binary-protocol for details. For details
about how to figure out the commands for new devices look at the
[documentation for protocol and commands](docs/protocol.md).
