## Configuration

The configuration functions can be used to control aspects of pigpio such as
C library initialization, C library termination and the sample rate for
alerts. Many applications will not need to use the configuration functions as
the default behavior will suffice.

#### Functions
  - [hardwareRevision()](#hardwarerevision)
  - [initialize()](#initialize)
  - [terminate()](#terminate)
  - [configureClock(microseconds, peripheral)](#configureclockmicroseconds-peripheral)
  - [configureSocketPort(port)](#configuresocketportport)

#### Constants
  - [CLOCK_PWM](#clock_pwm)
  - [CLOCK_PCM](#clock_pcm)

### Functions

#### hardwareRevision()
Returns the Raspberry Pi hardware revision as an unsigned integer. Returns 0
if the hardware revision can not be determined.

The following program can be used to print the hardware revision of a
Raspberry Pi in HEX.

```js
var pigpio = require('pigpio');

console.log('Hardware Revision: ' + pigpio.hardwareRevision().toString(16));
```

For further details related to Raspberry Pi hardware revisions see
[here](http://elinux.org/RPi_HardwareHistory) and
[here](https://github.com/joan2937/pigpio#gpio).

#### initialize()
Initialize the pigpio C library.

In most applications it will not be necessary to call the `initialize` and
`terminate` functions directly as they are indirectly called automatically.

There is however an exception to this rule. If a Node.js application uses
[signal events](https://nodejs.org/dist/latest/docs/api/process.html#process_signal_events),
initialization and termination of the pigpio C library can no longer be
handled automatically in all cases. Linux only allows one signal event handler
to be registered for a signal. When the pigpio C library is initialized, it
registers signal event handlers for all signal events. These signal event
handlers are used by the pigpio C library to terminate the pigpio C library.
If a Node.js application registers it own signal event handlers before the
pigpio C library is initialized, the event handlers registered by the Node.js
code will be deregistered and replaced with those registered by the pigpio C
library initialization code.

To resolve this issue the `initialize` and `terminate` functions can be used.
The `initialize` function should be called before the Node.js application
registers any signal event handlers. The signal event handlers should call the
`terminate` function to terminate the pigpio package.

As an example, let's say we want to implement a simple program that blinks an
LED once per second. If the user terminates the program by hitting ctrl+c we
would like to turn the LED off before the program terminates.

The following program might look like it should do what we would like it to do
but unfortunately it doesn't. The `SIGINT` handler is not executed when the
user hits ctrl+c.

```js
var Gpio = require('pigpio').Gpio,
  led,
  iv;

process.on('SIGINT', function () {
  led.digitalWrite(0);
  clearInterval(iv);
  console.log('Terminating...');
});

led = new Gpio(17, {mode: Gpio.OUTPUT}); // pigpio C library automatically initialized here

iv = setInterval(function () {
  led.digitalWrite(led.digitalRead() ^ 1);
}, 1000);
```

The program registers a `SIGINT` handler and then creates a `Gpio` object. The
pigpio C library is automatically initialized when the `Gpio` object is
created. Unfortunately this automatically initialization occurs after the
`SIGINT` handler is registered and it registers a new `SIGINT` handler which
in turn deregisters the `SIGINT` handler registered by the JavaScript code.

To resolve this issue the pigpio `initialize` and `terminate` functions can be
used.

```js
var pigpio = require('pigpio'),
  Gpio = pigpio.Gpio,
  led,
  iv;

pigpio.initialize(); // pigpio C library initialized here

process.on('SIGINT', function () {
  led.digitalWrite(0);
  pigpio.terminate(); // pigpio C library terminated here
  clearInterval(iv);
  console.log('Terminating...');
});

led = new Gpio(17, {mode: Gpio.OUTPUT});

iv = setInterval(function () {
  led.digitalWrite(led.digitalRead() ^ 1);
}, 1000);
```

#### terminate()
Terminate the pigpio C library. See
[initialize()](#initialize).

After `terminate` has been called any pigpio objects created up to that point
can no longer be used.

#### configureClock(microseconds, peripheral)
- microseconds - an unsigned integer specifying the sample rate in microseconds (1, 2, 4, 5, 8, or 10)
- peripheral - an unsigned integer specifying the peripheral for timing (CLOCK_PWM or CLOCK_PCM)

Under the covers, pigpio uses the DMA and PWM or PCM peripherals to control
and schedule PWM and servo pulse lengths. pigpio can also detect GPIO state
changes. A fundamental parameter when performing these activities is the
sample rate. The sample rate is a global setting and the same sample rate is
used for all GPIOs.

The sample rate can be set to 1, 2, 4, 5, 8, or 10 microseconds.

The number of samples per second and approximate CPU % for each sample rate
is given by the following table:

Sample rate (us) | Samples per second | CPU % |
---: | ---: | ---: |
1 | 1000000 | 25 |
2 | 500000 | 16 |
4 | 250000 | 11 |
5 | 200000 | 10 |
8 | 125000 | 15 |
10 | 100000 | 14 |

The `configureClock` function can be used to configure the sample rate and
timing peripheral.

If `configureClock` is never called, the sample rate will default to 5
microseconds timed by the PCM peripheral.

If `configureClock` is called, it must be called before creating `Gpio` objects.
For example:

```js
var pigpio = require('pigpio'),
  Gpio = pigpio.Gpio,
  led;

// Call configureClock before creating Gpio objects
pigpio.configureClock(1, pigpio.CLOCK_PCM);

led = new Gpio(17, {mode: Gpio.OUTPUT});
```

#### configureSocketPort(port)
- port - an unsigned integer specifying the pigpio socket port number.

Configures pigpio to use the specified socket port.

The default setting is to use port 8888.

If `configureSocketPort` is called, it must be called before creating `Gpio`
objects. For example:

```js
var pigpio = require('pigpio'),
  Gpio = pigpio.Gpio,
  led;

// Call configureSocketPort before creating Gpio objects
pigpio.configureSocketPort(8889);

led = new Gpio(17, {mode: Gpio.OUTPUT});
```

### Constants

#### CLOCK_PWM
PWM clock.

#### CLOCK_PCM
PCM clock.

