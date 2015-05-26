---
layout: post
title:  "Windows 10 IoT with Raspberry Pi 2"
date:   2015-05-26T21:58:00+12:00
categories: Blog IoT Pi2
disqus: true
---

For some time I have had a Si4703 FM Tuner on my desk. I've had sample code running on Arduino with rudimentary operations (seek, volume) and incomplete RDS decode implementations. I've had some good excuses for not getting stuck in; I have been holding out for Windows 10 on Raspberry Pi, but mostly finding time is difficult!

There are many code samples out there for Arduino, but I really wanted to implement the Radio Tuner in C#.
I've had a few nights this past week to get stuck in and really figure out how the Si4703 works. I have made some meaningful progress toward making it go. 

All the Arduino samples I have seen rely on `delay()` or `while(true)` loops. I wanted to design the interface take advantage of `async`/`await` and decided to use the optional interrupt GPIO pin.

Using the interrupt pin works really well with RDS which is continually updating. Decode is much more reliable with far fewer errors than constantly polling for updates.

###Sample code
Last night I posted the [c# S14703 implementation][link-code] to GitHub.
It has a *very* basic WPF interface to drive it, which at the moment requires a mouse to click the buttons.

###Si4703 Features Implemented:

* Seek (up and down)
* Tune (to a specific frequency)
* Volume
* RDS 
	* `PS` Program Service (Typically The radio station name)
	* `RT` Radio Text (typically the song name)
	* `PI` Program Identifier (constant across geographic areas and alternative frequencies)
	* `PTY` Program Type (Pop, Rock, Phone-In, Blues etc)
	* `CT` Clock (only one station in my area broadcasts this)
* RSSI (signal strength)
* Stereo indicator (radio switches to mono when it has poor signal strength)
* Power On (obviously)
* Power Off

###Issues
I would like to update the RDS decoder to handle more types of data. `AF` Alternate Frequencies appears to be broadcast in my area so this may be next.

RDS `CT` (clock) decode works, but is not fully plumbed to the UI yet. 

The program type code (`PTY`) looks dubious. The [PTY category][link-pty] indicates what the broadcast is, Rock, Pop etc.

I have several years of c# development experience. However I have never laid a finger on WPF before. I know the Asp.Net MVC and WebApi stack inside out. I can write a Windows Service no problem, but WPF is completely new to me.

I clearly have a lot to learn here but have managed the bare basics to get a basic UI working. With one exception.

`ICommand`

This is the first time I have run up against Win RT vs the full .Net runtime. 

```c#
public event EventHandler CanExecuteChanged
{
	add { System.Windows.Input.CommandManager.RequerySuggested += value; }
	remove { System.Windows.Input.CommandManager.RequerySuggested -= value; }
}
```
![CS0234 The type or namespace name 'Command Manager' does not exist in the namespace 'System.Windows.Input'](/assets/images/2015-05/cs0234_error.png)


`CommandManager` is not available to UWA apps so I need to discover how to do this properly. A [hack][link-cmdmgrhack] is to set a timer to fire the event every 100ms to refresh the `CanExecute` state. I already have a time running so this simple hack may be the easiest way forward.

In the meantime, this version of code always returns `true` for `CanExecute` in the `RelayCommand` implementation rather than using the supplied predicate. 
This means you can seek without powering on the radio first. This will undoubtedly result in a crash.



###Future
I have various other sensors and breakout boards waiting for c# code to drive them
* DHT11 temperature/humidity
* Gyroscope/Compass
* Pir Sensor
* Stepper Motors
* Serial Bluetooth (probably more useful connected to an Arduino)
* Mains Power Relays
* Solenoids to control mains pressure water
* Arduino (to connect over I2C perhaps to drive other components)

All of these things (and more) will eventually come. Perhaps they will miraculously assemble themselves into a carputer!





[link-code]:https://github.com/williamdenton/IoT
[link-pty]:http://en.wikipedia.org/wiki/Radio_Data_System#Program_types
[link-cmdmgr]:https://msdn.microsoft.com/en-us/library/system.windows.input.commandmanager
[link-cmdmgrhack]:https://social.msdn.microsoft.com/Forums/vstudio/en-US/477cdd19-ee88-4746-97fe-59b8dbd44e0a/implement-icommandcanexecutechanged-in-portable-class-library-pcl?forum=netfxbcl