# The open source centric smart home

Over the last two years or so, I adopted some smart home devices one by one. By now, I have connected devices for different aspects of my day to day life that serve me well.
However, when using multiple systems, you get to a point where you get annoyed by having to control every device with a different App and want scenes and configurations across different devices. 
After having some discussions on twitter [and being explicitly asked about it](https://twitter.com/oxygen0211/status/855782919725514754), I want to share the my opinion about the devices I own and the approach I took for combining them.

## Covered usecases
As a developer seeing products and frameworks come and go, I didn't want to use one system for everything that will lock me in and want to at least have the option to some hacking on the systems on my own, so I want the components and software in use to be open source or at least to have a relatively open API.
And since I am living in a rented apartment, pulling wires to all the rooms and installing components into the surge cabinet is not an option. Through this, the older systems based on a central Bus like KNX are not that interresting to me and I tend to go more towards the new, wireless and specialized products. 
Of course, this makes interconnection more difficult, but since wanted some automation across all devices, this was my main focus over the last couple weeks. For automation, I wanted things like:

* React to time and weather (turn on the lights on sunset)
* Centrally manage scenes (Light color and brightness, music, ...) for certain occasions (TV watching, Party, Going to bed,...)
* React to me being present or not (make sure the lights are off when I'm gone)
* Make my apartment react to events happening online (change the light color when I'm mentioned on Twitter, react to data in my fitness Apps)
* Adapt all smart devices based on operation of another one (set a dedicated light scene when turning on the TV)
* Start routines including multiple devices (wake me up using Music AND Light)

While the implementation of some of these automations is still work in progress, I managed to find a combination as a platform that allows me to do all that. Here's how it's built.

## Devices

### Phillips Hue lightbulbs
These are certainly my most used components. Two years ago, I got myself a Hue starter kit with three E27 Bulbs since I wanted something cool for my sleeping room ceiling light.
This is still in daily use (meaning that I am still using the version 1 bridge which lacks some functionality that the new one has, making some things seem more complicated than they are).
After having installed one in my sleeping room, I used the other two as a ceiling light in my home office and as an ambient light in my living room.
A few weeks ago, I added two E10-Spots to my doorway, a portable Hue Go lamp as a second ambient spot in my living room and a light strip for lighting the edge of my bed.
I am pretty pleased with the bulbs, the E27 are fairly bright, The E10 are not as powerful but four of them would still be sufficient to light my doorway (currently I am using two hues and two normal LED spots since I still had them spare).
The only thing I would consider somewhat negative are that all the components are fairly pricy and that I hat some initial trouble getting the Lightstrip to stick. The adhesive that was on there by default just wouldn't hold to the wood of my bed enough. However this was easily fixed with proper doublesided tape.

I think other bulbs of this kind like Lifx or the new Ikea ones are a equally good option, however, I don't have first hand experience with them and haven't seen the need to introduce a second system by now.

### Sonos connected speakers
Another pretty popular system. I am using one Play:3 speaker in my Sleeping room. Sonos offers speakers in different sizes that can be controlled via companion Apps for all common mobile and pc operation systems and offer an API and grouped by their location. 
The grouping is in a dynamic way so that you can transfer the playing music from one room to another or stream one playlist to multiple rooms simultaneously. I am very pleased with their sound and am planning to extend my system for a while now but haven't done so yet.
This is mainly caused since the new components are a significant investment (Play:1, the smalles one, costs around 250€, the bigger ones way more) and the combination of my apartment's room layout and the usage of other speakers reduces the urge to buy more components.
In my living room, I have a old but good Teufel concecpt E Magnum 5.1 surround sound system hooked to my TV that also easily covers my semi open kitchen and I also own a Bose mini soundlink Bluetooth speaker that delivers unbelievable sound for its size and prize that I use in my bathroom and home office when I want to.
My plans for extending this systems are to add a stationary Play:1 to my bathroom (I absolutely love blasting music while showering :-)) and a Sonos Connect to the surround sound system in my livingroom. If I had to replace my current surround sound system I would very likely opt for sonos or a similar system.

### Samsung Smart TV
When I moved into my apartment, I bought my first TV of my own. This was almost four years ago and a this time, the marked was flooded by different Smart TV systems. After getting advice from a friend of mine who was running a local TV shop for quite some decades, it was clear to me that a Samsung one would be a good choice, so I got myself a 55" Samsung smart TV.
It sports sufficient ports - most importantly a few HDMIs and USBs - and connectivity over Wifi and Ethernet. There are many Apps from games (haven't tried any of them) to ambient animation apps (like the cliche digital fireplace) to the biggest streaming services like Netflix, Spotify and YouTube. 
Additionally, there are rudimentary ways to connect devices wirelessly like Miracast (which I haven't got running immediately after switching to Mac and iPhone and haven't tried anymore yet), Samsung Screen share App (which is of no use for me since I have no Samsung mobile device) and rudimentary APIs (a bit more on that later, however I haven't looked into it in detail). 
My experience with this one is that the picture is really nice and very well balanced. Also I could hook up my old (PC-) surround sound system with a relatively cheap optical to tripple-cinch transmitter. However, on the connectivity side I have some recuring problems. While streaming, the TV will often drop out of my Wifi for no apparent reason. This might probably be a problem with my local Wifi setup since the TV is mounted on an outer wall and the signal needs to travel through some walls, I have also seen some reception problems on my laptop when having it placed in front of the TV while connect via HDMI.
In the recent time, especially when improving my home automation and looking into Apple HomeKit, I considered getting an Apple TV, especially since (at least subjectively) functions are decreasing and the platform seems to loose significance. However, I am currently not using streaming that much, it's basically the same as with extending my sonos system - the urge is currently not that big that I am willing to take the investment.

### Withings Smart Body Analyzer
I have been overweight for the bigger part of my life and for a long time it was a hard enough struggle to not gain even more weight. Thankfully, as my life got gradually more stressful towards the end of my studies, I found how much of a improvement in quality of life regular exercise is providing me.
I got much fitter in the last years and am in a process of loosing weight right now, however this is still a struggle and I need to gain proper data regularly for monitoring my progess and adapting my excercise and nutrition as needed.
For this, the Withings smart body analyzer is serving me very well. I have this one for a few years now, so it's still the first generation with a bit less functionality than the current ones but it reliably generates the data I need. It automatically reports measurements of weight, body fat and heart rate to Withings' backend from where it will be able on the companion app and other services. I have configured my iPhone companion app so that it will forward the data to apple health so it can be combined with data I am tracking in other apps.

## Software
Ok, I admit, until now, this post didn't have much to do with Open Source. But on the software side, we get our share of it, I promise. After all, the hardware systems alone are not much more than a fancier version of their non-smart versions without connectivity and software around them. Let's look into everything is interconnected in my system.

### Home Assistant
Let's dive right into our central (and Open Source!) component. As I said before, all the systems have their own apps that work nicely, but if you are using several components and want them to interact and synchronize, you will soon want to have a central system to control them from one place. This is exaclty what Home Assistant does. It packs A TON (639 at time of writing, according to their page) of integrations of all differrent Kinds from Smart devices including the ones mentioned above, Apps and Applications from which I wouldn't have thought they might be useful in a smart home system but totally make sense once you think about it. An example for this is an SNMP trap which they use for presence detection through the Wifi connection of your phone (if your Router supports SNMP, of course) or an influxDB adapter (and other monitoring systems) so you can monitor the state of your whole home with production server grade system monitoring. Home Assistant is written in Python and relies on YAML based configuration that you can make in one file or split up into several ones as you like to. You can define scenes that can include any mix of different devices, groups by different aspects like rooms, kind and so on (it's not one or the other but you can add a device to multiple groups), scripts and more I haven't even looked at yet. It is also relatively light on resources and even provides a special Raspberry Pi custom OS if you want to deploy it there easily. However, I have my instance running in a Docker container on an old laptop that is running elemantary OS.

One core element of Home Assistant (and probably all Smarthome systems) is automation. The triggers can be events and state changes that can be produced by any component connected to the system. Automations I have set up so fare are:
* Turn on my Lights on sunset if I am home (tracked via find my iPhone based presence detection)
* Turn off my Lights when I leave home 
* Activate "TV Evening" scene when I turn on my TV. This means it turns off the lights in my sleeping room and home office and the ones in my living room and doorway on

There are other similar projects, for example eclipse's OpenHAB, but frankly none of the projects I have tried before gave me the quick wins and easiness in operation I was searching for. For me, Home Assistant seems to be the right choice.

## Homebridge
One thing I found initially problematic with Home Assistant is operating it in everyday situations like walking into a room for picking up something. Espially in the first days I would walk into a room, hit the light switch turning the lights off since they were turned out over the automation before, notice it while walking into the room, turn around, hit the light switch again to turn it on and be annoyed that the light (which is now on default settings) doesn't fit to the scene anymore and that triggering the switch did only affect the light connected to it, not ambient lights plugged into the socket. The reason for this is in my opinion is that the home assistant app (or more precisely their react frontend that you pin to your phone/tablet home screen via a bookmark is not intuitiv enough for operating it while doing something other. 

Apple's HomeKit offers way better integration for this and also enables us to let Siri change the settings for us. Home Assistant does not offer HomeKit support natively, but for this, there's Homebridge. Homebridge is a Node.js based Open Source project that does bridge the gap between HomeKit and devices that don't offer support natively. Of course, there's also an adapter for Home Assistant. With this, you can use HomeKit to turn on devices and set scenes. For scenes, they had to trick a bit and implemented them as digital switches but it works like a charm. I had it set up as a docker container (caution: I had to set --net=host on the run command to cope with the random port association it does) and configured within minutes. The most tedious part was that Homebridge and HomeKit weren't able to copy my groups to rooms and I had to create rooms again in HomeKit and associate the devices coorectly, but I consider this a minor thing.

Using Siri with Homebridge has become my default way of making manual settings on my smart home components, followed by using the HomeKit settings in my iPhone's control center.

I have still to refine some configuration and fix a few problems like currently not seeing my sonos in HomeKit, but I am positive that this will be solvable. Adding Homebridge to my setup has shown me that voice assistants are a very effective and comfortable way of interaction with my smarthome. However, a stationary voice assistant system would still be better than talking into my phone since might have lying it around somewhere ob be carrying something that makes it hard to activate Siri on my phone. 
