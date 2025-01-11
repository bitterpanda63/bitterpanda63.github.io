---
layout: page
title: Reverse-Engineering the Larq PureVis 2
order: 2
---

Hiya! I recently bought the LARQ PureVis 2, which is a smart waterbottle, to track my water intake.
I've noticed the app is not super reliant so I've started a new quest, figuring out how it works (and fixing it).

## My first guess
So I've had the bottle for a while now, here is my first guess at how it works :

To track water intake the bottle has a sonar sensor, which only triggers after a gyroscope confirms that the bottle is upright and standing still.
It then checks the difference between two sonar readings so it knows the length `l` which somebody drank. To get the ml, you just have to determine the volume.
So that's as easy as $V = 2* \pi * r^2 * l$, where `r` is the circumference of the bottle. 

The filter is just a passive filter which does not really have a lot of technically interesting details afaik. 
The cleaning is done using a UV-light : This goes off every **2h** to clean the bottle but you can also trigger it yourself if your water source is not healthy.

When using the app I discovered that it warned me to turn on Bluetooth so I'm guessing the bottle uses either standard Bluetooth or BLE.

## What's next
A good next step in my effort to reverse-engineer the bottle is to find the **patent**, that way I can confirm a lot of my findings, figure out the actual circumference, the technology behind it.
And I can maybe even figure out which wireless technology they are using.

Using JUSTIA and Google Patents I found the following patents :
- [Device for UV-LED liquid monitoring and treatment](https://patents.google.com/patent/US10959443B2/) : Talks about UV-B and UV-C light for cleaning, a way of transmitting data and using pressure sensors to measure the volume of the water.
- [Filtering container with time-based capacitive flow monitoring](https://patents.google.com/patent/US10969262B1) : This references a capacitive strip with discrete points along the bottle to measure where the water level is and it also references Bluetooth as a communication technology. Here in the sketches we also begin to see the bottle?
- [Liquid sanitation device and method](https://patents.google.com/patent/US10906819B2)

Looking at our first patent we can find an image containing what I think was a first prototype :

<img src="https://github.com/user-attachments/assets/384116f3-b179-4cbb-b805-b8dc79a1383b" alt="image containing a design of a bottle where the circuits are on the bottom" width="200"/>

This makes me believe it's an outdated patent and they have since moved away from a pressure sensor on the bottom of the bottle.

Now looking at the third patent we start seeing something that closely resembles the current bottle, just take a look at the sketch for the cap :

<img src="https://github.com/user-attachments/assets/7f2891cb-4978-4fc1-8a1c-522b7334d28f" alt="sketch of smart bottle cap" width="250"/>

According to the patent **210** is a photosensor, which is used to detect ambient light (i.e. bottle being open). But a photosensor could also be used to measure distances. So our original guess might be right? The patent makes no reference to capacitive strips or another way of measuring the contents, it does make a claim that the bottle can monitor water intake. Is this how they do it?

## Bluetooth and beyond!
### Using ADB
Alright, the patent was interesting, but how does the communication work between your phone and the water bottle. It's **Bluetooth**, yes, but let us try and take a peek behind the curtain. To do so I'll be using my Android phone with **USB Debugging** and **Bluetooth HCI snoop logs**, enabled.

To extract the Bluetooth HCI snoop logs just use the following **adb** commands :
```bash
adb devices # Verify that your phone shows up here
adb bugreport my_bugreport
unzip my_bugreport.zip -d my_bugreport && cd my_bugreport
```
Now in the `my_bugreport` folder you'll have to locate he HCI snoop logs, I found mine in the following subdirectory : 
```
my_bugreport/FS/data/misc/bluetooth/logs/btsnoop_hci.log
```
Now we want to inspect the logs, you can open them using a tool like [**Wireshark**](https://www.wireshark.org/).  
#### Using nRF Connect
Using a tool on my phone called *nRF Connect* I was also able to identify the LARQ Connection and play around with it, I was able to read out the serial number, hardware revision, manufacturer, firmware & software version, battery level and some other data.
I also was able to debug the messages sent between my phone and the bottle, which I will be analysing.
### Bluetooth findings
#### Status updates : ResponseGetCapUiState
One message which just kept coming back was the one below, which I've split into three parts, one part was an identifier `type.googleapis.com/ResponseGetCapUiState`. The un-decoded data could mean a couple of things, my guess right now is that it represents the filtering status (Adventure, Purifying, off, ...) and the current volume.
```
?
0D-03-00-00-00-10-01-1A-31-0A-29

type.googleapis.com/ResponseGetCapUiState
74-79-70-65-2E-67-6F-6F-67-6C-65-61-70-69-73-2E-63-6F-6D-2F-52-65-73-70-6F-6E-73-65-47-65-74-43-61-70-55-69-53-74-61-74-65

?
12-04-08-0F-10-01
```
Another message which was only sent when the  purifying was turned on is exactly the same, except for the last part :
```
[?]  [?]  [?]  [Purifying]   [?]  [?]
12 - 04 - 08 - 03          - 10 - 01
```
By switching between the different modes and analysing the send commands I've created the following table : 

| Hex | Meaning         |
| --- | --------------- |
| 03  | Purifying       |
| 0F  | Nothing         |
| 04  | Adventure Mode  |


#### The tale of the mysterious counter
Another common message is this one :
```
(0x) 0D-{?}-00-00-00-10-01
```
Where the question mark is incrementing constantly, the messages get sent some ms apart from each other and keep coming. I think it's a sort of heartbeat?

Continuing later...