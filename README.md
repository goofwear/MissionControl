<p align="left">
<img alt="GitHub" src="https://img.shields.io/github/license/ndeadly/MissionControl">
<img alt="GitHub release (latest by date)" src="https://img.shields.io/github/v/release/ndeadly/MissionControl">
<img alt="GitHub All Releases" src="https://img.shields.io/github/downloads/ndeadly/MissionControl/total">
<img alt="GitHub Releases" src="https://img.shields.io/github/downloads/ndeadly/MissionControl/latest/total">
<a href="https://ko-fi.com/J3J01BZZ6">
    <img border="0" alt="ko-fi" src="https://www.ko-fi.com/img/githubbutton_sm.svg" align="right">
</a>
</p>

# MissionControl

Use controllers from other consoles natively on your Nintendo Switch via Bluetooth. No dongles or other external hardware neccessary.

### Features
* Supports all firmware versions
* Pair Bluetooth controllers natively via `Change Grip/Order` screen
* Native button remapping (firmware 10.0.0+)
* `btdrv-mitm` module adds extension IPC commands that can be used to interact with the `bluetooth` process without interfering with the state of the system. 

### Supported Controllers
 Currently, the following controllers are supported. If you have a third-party variant of one of the below, or a Bluetooth controller that isn't listed, consider submitting an issue with the controller details, including vid/pid, to request support.

* __Nintendo Wii Remote + extensions (Nunchuck, Classic Controller, Classic Controller Pro, SNES Mini)__
* __Nintendo WiiU Pro Controller__
* __Sony DualShock4 Controller__
* __Microsoft Xbox One S Controller__*
* __Ouya Controller__
* __Gamestick Controller__
* __Gembox Controller__
* __Ipega Controllers (9062S, 9023 confirmed working. Others may also work)__
* __Xiaomi Mi Controller__

**Not all Xbox One wireless controllers support Bluetooth. Older variants use a proprietary 2.4Ghz protocol and cannot be used with the Switch. See [here](https://support.xbox.com/help/hardware-network/accessories/connect-and-troubleshoot-xbox-one-bluetooth-issues) for information on identifying the Bluetooth variant.*

### Requirements
* Hackable Nintendo Switch running ***at least*** version 0.14.2 of [Atmosphère](https://github.com/Atmosphere-NX/Atmosphere/releases) custom firmware. Other custom firmwares are not supported.
* Compatible Bluetooth controller

### Installation
Download the [latest release](https://github.com/ndeadly/MissionControl/releases) .zip and extract to the root of your SD card, allowing the folders to merge and overwriting any existing files. A console reboot is required in order for MissionControl to become active.

It is recommended after rebooting to also disconnect any physically connected controllers from the console, delete the pairing database, and re-pair via bluetooth to make sure your existing Switch controllers have the correct hardware ID stored and can be recognised properly. To delete the database, navigate to `System Settings->Controllers and Sensors->Disconnect Controllers`

***IMPORTANT: Atmosphère >= 0.14.2 is required to run this version. Older versions will hang at the Nintendo logo during boot. 
Users upgrading from the previous version of MissionControl are also advised to wipe their pairing database and re-pair controllers running v0.2.0. Failure to wipe the old entries may result in non-switch controllers being detected incorrectly.***

### Usage
Install MissionControl to your SD card, reboot the console and then pair controllers as you normally would via the `Controllers->Change Grip/Order` screen. Once paired, controllers will reconnect automatically when woken up.

Most native features *should* just work (with the exception of things like firmware update). If you find something that's broken please create an issue.

### Pairing controllers
The supported controllers each have their own methods to enter pairing/sync mode. Below are instructions on entering this mode for each supported type.

***Wii(U) Controllers***
Press the red sync button on the back of the controller. The controller LEDs will blink rapidly.

***Dualshock4 Controllers***
Press and hold the `PS` + `share` buttons simultaneously until the lightbar starts blinking. When done correctly the blink pattern will resemble a heartbeat, otherwise it will blink periodically.

***Xbox One Controllers***
Press and hold the `guide`(`X`) button until the LED starts blinking. Then press and hold the small sync button on the back near the charging port until the LED starts blinking more rapidly.
You should also ensure your controller firmware is up to date, as old firmwares were known to have flakey bluetooth.

***Other Controllers***
Please refer to your controller's user manual for information on how to put it in sync mode.

### How it works
MissionControl works by Man-In-The-Middling the `bluetooth` system module and intercepting its initialisation IPC commands and system events, and translating incoming/outgoing data to convince the Switch that it's communicating with an official Pro Controller.

To achieve this, the `btdrv-mitm` module obtains the handles to `bluetooth` system events and shared memory when the system attempts to initialise them over IPC via the `btm` and `hid` modules. It then creates its own secondary versions of these and passes their handles on instead of the original. This allows modifications to be made to any data buffers before notifying (or not) the system. Additionally, the `WriteHidData` IPC command is intercepted to translate or drop outgoing requests to the controller. In the case of the latter, fake responses can be written directly to the buffer in shared memory.

Intercepting initialisation IPC commands also allows homebrew to properly make use of the `bluetooth` service. Normally, calling any of the IPC commands that would initialise or finalise system events would either crash the console, or invalidate the event handles held by system processes. With `btdrv-mitm` we are able to hand out alternative event handles when homebrew attempts to initialise an interface, and redirect the real system events to those instead of the events held by system processes.

IPS patches to the `bluetooth` module are provided to (re)enable the passing of abitrary pincodes when Bluetooth legacy pairing is used (Nintendo hardcodes a value of `'0000'`, ignoring IPC arguments). This enables Wii(U) devices to be paired with the console.

The `btm` service is now also MITM'd, allowing for faking controller names on the fly while retaining the original names in the pairing database.

### Building from source
First, clone the repository to your local machine and switch to the newly cloned directory
```
git clone --recurse-submodules https://github.com/ndeadly/MissionControl.git
cd MissionControl
```

MissionControl currently uses a custom fork of `libnx` that adds Bluetooth service wrappers and type definitions. This needs to be built and installed first

```
cd libnx
make && make install
```

Next build `Atmosphere-libs`
```
cd ../Atmosphere-libs
make
```

Finally, build and package the distribution .zip. This will build `bluetooth-mitm` and package it up with bluetooth exefs patches. 
```
cd ..
make dist
```

The resulting package can be installed as described above.

### Planned Features
* Controller management application
* Rumble support
* Motion support
* Keyboard and mouse support

### Known Issues and Limitations
* Non-Switch controllers cannot be used to wake the system from sleep.
* Controllers that haven't had their hardware ID whitelisted for identification will not be recognised as connected. They will however still store their details to the database if they are able to successfully sync. If you encounter such a controller, please create an issue requesting support. 
* Wii(U) controllers can take a while to pair with the console. For some reason they are only detected at the end of a device discovery cycle. Be patient and re-press the sync button on the controller if neccessary.
* Some Dualshock4 v1 controllers may appear to not be working with the console. Users have reported that repeatedly attempting to pair may eventually cause them to sync up. Once synced they should work as usual.
* Xbox One button layout was changed at some point in a firmware update. Please ensure your controller firmware is up to date if you have issues with incorrect button mappings.

### Frequently Asked Questions
***Does this run on \<insert CFW here\>?***
No. MissionControl will only run under Atmosphère. This is not some petty act of malice toward other CFWs and their users. Others simply don't (currently) provide the framework neccessary for this sysmodule to work.

***Does this support USB controllers?***
No. MissionControl is Bluetooth-only for now.

***Does this work with sys-con installed?***
Yes, I have had several reports from users that the two can be used together without issue.

***Installing this bricked my console!!!!!11!!1!11***
No. If you are seeing errors about title `0100000000000008` upon rebooting your console you are almost certainly using an outdated Atmosphere version. Install the latest version from the [official github release](https://github.com/Atmosphere-NX/Atmosphere/releases) and follow the MissionControl [installation instructions](https://github.com/ndeadly/MissionControl#installation) again.

***How can I use this with multiple sysNAND/emuMMC launch configs?***
Pairing controllers across multiple unique HOS installations requires multiple pairing databases and is essentially the same as pairing with two different consoles. The only exception being the case where you paired the controller(s) prior to making sysNAND copies. For now you will have to re-pair your controllers when switching back and forth. In the future I hope to include an option to load/store the database on the SD card to avoid this issue.

***Why have my official Joycon/Switch Pro Controllers stopped working over Bluetooth after installing MissionControl?***
~~It is possible to pair these controllers via the Joycon rails or a USB connection. In these cases a garbage hardware ID gets stored to the pairing database and the controller fails to be identified. Failure to identify a controller means I don't know how its input data should be handled, so I ignore it. This issue can be resolved by physicall disconnecting the controller from the console, deleting the pairing database with the `System Settings->Controllers and Sensors->Disconnect Controllers` option, and then re-pairing via bluetooth in the `Change Grip/Order` screen.~~ This should no longer be an issue with the latest version of MissionControl.

***Why doesn't my 3rd-party knockoff controller work?***
~~Many 3rd-party controllers also use garbage hardware IDs, making them difficult to identify reliably. If a controller can't be identified, I ignore it, since I have no idea how to process the incoming data. I am currently trying to find an alternate way to identifying them without creating problems elsewhere.~~ Same as above.

***My Xbox One Controller buttons are mapped incorrectly. Can you fix it?***
You didn't read the [Known Issues and Limitations](https://github.com/ndeadly/MissionControl#known-issues-and-limitations) section, did you? You need to update your controller firmware to the latest version. This can be done on Windows using the [Xbox Accessories](https://www.microsoft.com/en-us/p/xbox-accessories/9nblggh30xj3?activetab=pivot:overviewtab) app. You can also do this on the Xbox itself.

***Can you add support for PS3 controllers?***
It's on my list of things to look into. The pairing process is non-standard and may require modifications to the bluetooth driver. If it can be done non-destructively I will add support eventually.

***Can you add support for Xbox 360 controllers?***
No. These don't use Bluetooth. Try sys-con with a wireless USB adapter.

***Can you add bluetooth audio support?***
No. The bluetooth module on the switch only implements a small set of services required to make hid controllers work. Of this small set of services, only a handful of high-level functions are exposed to the rest of the system. Adding audio support would require implementing the services neccessary for audio transport, for which any sane person would require an open-source re-implementation of the bluetooth module in order to have access the low-level functions required to pull it off.


### Credits
* [__switchbrew__](https://switchbrew.org/wiki/Main_Page) for the extensive documention of the Switch OS.
* [__devkitPro__](https://devkitpro.org/) for the homebrew compiler toolchain.
* __SciresM__ for his dedicated work on the [Atmosphère](https://github.com/Atmosphere-NX) project, libstratosphere and general helpfulness with all things Switch related.
* __misson20000__ for his handy debug monitor [Twili](https://github.com/misson20000/twili) and IPC logger [Ilia](https://github.com/misson20000/ilia)
* __dekuNukem__, __CTCaer__, __shinyquagsire23__ and others for their work in reversing and documenting the switch controller communication protocol.
* __friedkeenan__ for helping to test Wii extension controller support.
* Everyone else over at the __ReSwitched__ discord server who helped answering technical questions.
