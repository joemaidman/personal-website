---
title: "RF packet sniffing &#038; replay &#8211; You&#8217;re only coming through in waves"
date: 2017-05-08 12:04
---

This post will explain how I went about sniffing, analysing and replaying the RF signals from a pair of remote operated ceiling fans operating on the 434Mhz frequency, as well as some of the science I learned along the way. In a world where we expect our data to be transferred securely encrypted, it may come as a surprise that so much is hiding in plain sight! While this example may seem trivial, suppose this were your doorbell or maybe your garage door or perhaps your wireless alarm system...

<a href="https://github.com/joemaidman/blades-in-motion" target="_blank">Source code available on Github</a>

<strong>Summary</strong>

We will be using a cheap digital TV/Radio USB receiver dongle to receive signals sent by the remote control which will then be demodulated and recorded with <a href="http://airspy.com/download/" target="_blank">SDR#</a>, a software defined radio program. We will analyse these audio clips in <a href="http://www.audacityteam.org/" target="_blank">Audacity,</a> an open-source audio editor to understand the information being transmitted before reconstructing the signal in software and replaying it via an <a href="https://www.arduino.cc/" target="_blank">Arduino</a> microcontroller with an <a href="http://www.robotshop.com/uk/rf-link-transmitter-434-mhz.html" target="_blank">RF transmitter</a>. Finally, we will have the Arduino serve as a basic HTTP server (using an <a href="https://www.arduino.cc/en/Main/ArduinoEthernetShield" target="_blank">Ethernet shield</a>) which will allow us to send commands to the target device via a web page.

<strong>Contents</strong>

<a href="#part1"><strong>Part 1:</strong> Hardware &amp; software</a>

<a href="#part2"><strong>Part 2:</strong> Knowledge</a>

<a href="#part3"><strong>Part 3: </strong>Practical</a>

<a href="#part4"><strong>Part 4: </strong>The code</a>

<a href="#part5"><strong>Part 5: </strong>The final product</a>

<a name="part1"></a><strong>Part 1:</strong> Hardware &amp; software

<strong>Hardware</strong>

<ul class="no-indent">
<li><strong>An RF controlled device as the target device. </strong>Any remote device using ASK to transmit data will do.</li>
<li><strong>A <a href="http://www.ebay.co.uk/itm/like/322350334259?lpid=122&amp;chn=ps&amp;adgroupid=35352091421&amp;rlsatarget=pla-279351682698&amp;adtype=pla&amp;poi=&amp;googleloc=1007216&amp;device=c&amp;campaignid=738466455&amp;crdt=0" target="_blank">RTL-SDR TV dongle </a>to detect the RF signal(s) (c£4).</strong></li>
<li><strong>An <a href="https://www.amazon.co.uk/Arduino-A000066-UNO/dp/B008GRTSV6" target="_blank">Arduino Uno</a> to replay/transmit the RF signal (c£20).</strong></li>
<li><strong>An <a href="http://www.robotshop.com/en/rf-link-transmitter-434-mhz.html" target="_blank">RF transmitter module</a> (c£3).</strong></li>
</ul>
<strong>Software</strong>
<ul class="no-indent">
	<li><strong><a href="http://airspy.com/download/" target="_blank">SDR#</a> to listen for and record the RF signal (free).</strong></li>
	<li><strong><a class="" href="http://www.audacityteam.org/download/" target="_blank">Audacity</a> to analyse the RF packets (free). </strong>We are essentially using this as a basic logic analyser to inspect the waveforms sent by the remote.</li>
	<li><strong><a href="https://www.arduino.cc/en/main/software" target="_blank">Arduino IDE</a> to program the microcontroller to replay RF signals (free).</strong></li>
	<li><strong>An internet browser such as Google Chrome (free).</strong></li>
</ul>

<a name="part2"></a><strong>Part 2:</strong> Knowledge

<strong>Radio Frequencies (RF)</strong>

RFs are electromagnetic waves that lie in the range from 3 kHz to 300 GHz, comfortably at the right of figure 1 (low energy wavelengths).

<strong>Figure 1</strong> Electromagnetic spectrum &amp; waves

<img class="plain " src="assets/images/2017-05-08/EMSpectrumcolor.jpg" alt="EMSpectrumcolor" width="428" height="321" />

<strong>Modulation</strong>

Modulation refers to the method used to encode digital data in a radio signal. The dark blue line in figure 2 shows a square wave representation of some digital information (1s and 0s) that we want to send or receive. To encode this information, a number of methods can be used, the most popular of which are <em>Amplitude Shift Keying</em> (ASK) and <em>Frequency Shirt Keying</em> (FSK) shown in light blue and green. These terms should be more familiar than they first appear; AM and FM radio use these encoding methods respectively;  Amplitude Modulated and Frequency Modulated radio respectively.

<strong>a. Amplitude Shift Keying (ASK). </strong> The signal changes <em>amplitude</em> (signal strength or wave height) while the frequency remains constant.

<strong>b. Frequency Shift Keying (FSK).</strong> The signal changes <em>frequency</em> (wavelength) while the amplitude remains constant.

<strong>Figure 2</strong> Modulation types
<a class="plain" href="assets/images/2017-05-08/modulations.png"><img src="assets/images/2017-05-08/modulations.png" alt="modulations" width="300" height="141" /></a>

The biggest difference to note is that the ASK signal doesn't change <em>frequency</em>, rather it changes <em>amplitude</em>; in figure 2 ASK's signal goes quiet (off) to send a zero and 'loud' (on) for a one. Our RF remote (and many other basic devices) uses a variation of ASK called On-Off Keying (OOK) that has no carrier signal for logic zero (more below). In contrast, FSK alters its <em>frequency </em>oscillating around its base frequency to transmit data but does so at a constant <em>amplitude</em>. FSK is actually switching frequencies slightly (within a few hundred kHz) to convey information. Consider tuning to your favourite FM radio station which operated at 98.5 FM; your radio is actually listening for signals in a range around 98.5Mhz.  Because of the variable frequency of FSK, it requires a larger bandwidth but this compromise is more than offset by its resistance to noise; ASK is more prone to interference which tends to manifest in changes in amplitude rather than frequency. FM demodulators can remove/ignore the amplitude spikes and still read the underlying signal clearly. AM does, however, have the benefit of a longer wavelength meaning it can move further and better cope with large obstacles.

<strong>Figure 3 </strong>Waves

&nbsp;

<a class="plain" href="assets/images/2017-05-08/wavelength.gif"><img src="assets/images/2017-05-08/wavelength.gif" alt="wavelength" width="300" height="213" /></a>

<strong>Figure 4</strong> Retro lesson in AM/FM radio basics c1964

<iframe width="420" height="315" src="https://www.youtube.com/embed/D65KXwfDs3s" frameborder="0" allowfullscreen></iframe>

Figure 5 shows the radio frequency allocations for the UK, from<em> Very Low Frequency</em> (VLF) right up to<em> Extremely High Frequency</em> (EHF) around 300Ghz. Our remote is operating at 434Mhz, putting it in the Ultra High Frequency (UHF) band (circled in red).

<strong>Figure 5</strong> UK RF allocations (click to enlarge)

<a class="plain" href="assets/images/2017-05-08/uk-spectrum-allocation-chart1.jpg"><img src="assets/images/2017-05-08/uk-spectrum-allocation-chart1.jpg" alt="uk-spectrum-allocation-chart1" width="550" height="383" /></a>

<strong>Reconnaissance</strong>

Any device that contained electronics capable of emitting RF energy by radiation, conduction, or other means above a certain threshold is required to be properly authorised and tested by the FCC in the USA and must display an FCC ID number. Looking at the back of our remote with the battery cover removed reveals the frequency (434Mhz), several dip switches as well as the FCC ID (figure 6). We can use this ID to search for documentation on <a href="https://fccid.io/">fccid.io</a> including manufacturer details, operating frequencies, photos, manuals and even the device submission covering letter,  allowing us to learn plenty about the device.

<strong>Figure 6</strong> Target device (rear view)

<a class="plain" href="assets/images/2017-05-08/20170414_175006.jpg"><img src="assets/images/2017-05-08/20170414_175006.jpg" alt="20170414_175006" height="424" /></a>

&nbsp;

<a name="part3"></a><strong>Part 3:</strong> Practical

Follow <a href="http://www.rtl-sdr.com/rtl-sdr-quick-start-guide/" target="_blank">this guide</a> to set-up SDR# and once you have it running and tuned to your devices frequency, you should see a spike in the spectrum like this when a button is pressed:

<strong>Figure 7</strong> SDR#

<a class="plain" href="assets/images/2017-05-08/Arduino_RF_Tx_433_ModuleTesting_SDRSharp_03.png"><img src="assets/images/2017-05-08/Arduino_RF_Tx_433_ModuleTesting_SDRSharp_03.png" alt="Arduino_RF_Tx_433_ModuleTesting_SDRSharp_03" width="569" height="474" /></a>

Use SDR#'s built-in recording function to record the signal as an audio file.

<strong>Packet analysis</strong>

Opening an audio recording of one of the signals in Audacity presents us with the following.

<strong>Figure 8</strong> Audacity high level

<a href="assets/images/2017-05-08/11.png"><img src="assets/images/2017-05-08/11.png" alt="1" width="550" height="169" /></a>

Once we have trimmed the white noise and zoomed in on the transmission, we see a series of twelve individual repeating signals with an interval of c12,000 microseconds.

<strong>Figure 9</strong> Audacity repeating signal

<a class="plain" href="assets/images/2017-05-08/2.png"><img src="assets/images/2017-05-08/2.png" alt="2" width="550" height="155" /></a>

Zooming in further to one of these signals shows us the individual bits being transmitted along with their transmission periods using the time index as a measurement.

<strong>Figure 10</strong> Audacity individual transmission

<a class="plain" href="assets/images/2017-05-08/3.png"><img src="assets/images/2017-05-08/3.png" alt="3" width="550" height="206" /></a>

This screenshot reveals a signal of:

<strong>[1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0] </strong>

Each bit is being transmitted for approximately 350 microseconds. We can use this information to program the microcontroller to replay the signal. Our RF remote control is sending data packets that appear to be 38 bits long and (after looking at several different signals) contain a fairly obvious preamble code comprising roughly the first 18 bits.

<a name="part4"></a><strong>Part 4: </strong>The code

Next, we will create a simple web page served over LAN via an Arduino Ethernet shield module where the user will be able to click buttons to send HTTP requests to the board to trigger each transmission.

We include the ethernet library file and define a constant of 38 for the array length representing the signals to transmit as well as two floating point numbers representing periods of time in microseconds. The first is the period to transmit each bit of data for. The second represents the interval between sending packets. We also define two integers,  a digital pin number (9) and the number of times to repeat each transmission (12). Four arrays are defined containing the 38-bit data packets to transmit. While the remote does have additional features, I have chosen just four signals due to storage constraints on the board and to avoid adding complexity with the additional of an SD card. Finally, we specify an IP address and port for the ethernet shield (192.168.1.177:80) to host our web server.

```c++
//Libraries
#include <SPI.h>
#include <Ethernet.h>
//RF data
#define BITS 38
int rfPin = 9;
int repeat = 12;
float period = 350;
float pausePeriod = 12000;
int lightCode [BITS] =  {1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 1, 1, 0};
int fanMinCode [BITS] = {1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0};
int fanMaxCode [BITS] = {1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0};
int fanOffCode [BITS] = {1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 1, 1, 0, 0, 1, 0};
//WebServer data
byte mac[] = {0x90, 0xA2, 0xDA, 0x0D, 0x9C, 0x79};
IPAddress ip(192, 168, 1, 177);
EthernetServer server(80);
boolean reading = false;
String readString = "";
```

Our main setup function sets pin 9 on the board to an output pin and starts the HTTP server.

```c++
void setup() {
 pinMode(rfPin, OUTPUT);
 Serial.begin(9600);
 Ethernet.begin(mac, ip);
 server.begin();
 Serial.print("Server is at IP: ");
 Serial.println(Ethernet.localIP());
}
```

The main loop of the sketch checks if a client to connected to the server and reads the client request. If the client requested the home page directly, we render an HTML page containing control buttons. If the request contains a '?' character, it means a button has been pressed and is interpreted as a trigger action. The request header is checked for a number between 1 and 4 to determine which action to take and the appropriate packet array is passed to the sendRF function for transmission.

```c++
void loop() {
  EthernetClient client = server.available();
  if (client) {
    while (client.connected()) {
      if (client.available()) {
        char c = client.read();

        if (readString.length() < 100) {
          readString += c;
        }

        if (c == '\n') {
          Serial.println(readString); //print to serial monitor for debuging

          if (readString.indexOf('?') >= 0) { //don't send new page
            client.println("HTTP/1.1 204 Awesome Fan Controller\r\n\r\n");
          }
          else {
            client.println("HTTP/1.1 200 OK"); //send new page
            client.println("Content-Type: text/html");
            client.println();

            client.println("<HTML>");
            client.println("<HEAD>");
            client.println("<TITLE>Awesome Fan Controller</TITLE>");
            client.println("</HEAD>");
            client.println("<BODY style='font-family:verdana;'>");

            client.print("<center><h3>Awesome Fan Controller</h3>");
            client.print("<a href='/?1'>Lights</a>");
            client.print("<a href='/?2'>Fans Min</a>");
            client.print("<a href='/?3'>Fans Max</a>");
            client.print("<a href='/?4'>Fans Off</a>");
            client.print("</center>");

            client.println("</BODY>");
            client.println("</HTML>");
          }

          delay(1);
          client.stop();

          if (readString.indexOf("/?1") > 0) //checks for lights
          {
            sendRF(lightCode);
            Serial.println("Lights");
          }
          if (readString.indexOf("/?2") > 0) //checks for min
          {
            sendRF(fanMinCode);
            Serial.println("Fans min");
          }
          if (readString.indexOf("/?3") > 0) //checks for max
          {
            sendRF(fanMaxCode);
            Serial.println("Fans max");
          }
          if (readString.indexOf("/?4") > 0) //checks for off
          {
            sendRF(fanOffCode);
            Serial.println("Fans off");
          }

          readString = "";

        }
      }
    }
  }
}
```

Last but not least, we define a function to transmit the data packet which is passed as an argument. The function loops through each bit in the array and sets the digital output pin 9 high/low for the signal time and repeats the entire signal twelve times.

```c++
void sendRF(int code []) {
  for (int i = 0; i < repeat; i++) {
    for (int j = 0; j < BITS; j++) {
      if (code[j] == 0) {
        digitalWrite(rfPin, LOW);
      }
      else {
        digitalWrite(rfPin, HIGH);
      }
      delayMicroseconds(period);
    }
    delayMicroseconds(pausePeriod);
  }
}
```

<strong>Figure 11 </strong>Fritzing schematic
<a class="plain" href="assets/images/2017-05-08/Fan-Controller_bb.png"><img src="assets/images/2017-05-08/Fan-Controller_bb.png" alt="Fan Controller_bb" width="550" height="428" /></a>

<strong>Figure 12</strong> Reality

<a class="plain" href="assets/images/2017-05-08/20170415_161523.jpg"><img src="assets/images/2017-05-08/20170415_161523.jpg" alt="20170415_161523" width="550" height="413" /></a>

<a name="part5"></a><strong>Part 5: </strong>The final product

<strong>Figure 13</strong> Control web page

<a class="plain" href="assets/images/2017-05-08/webpage1.png"><img src="assets/images/2017-05-08/webpage1.png" alt="webpage" width="550" height="343" /></a>

<strong>Figure 14</strong> Demo

<iframe width="420" height="315" src="https://www.youtube.com/embed/-oWBbTvQUzc" frameborder="0" allowfullscreen></iframe>

It works!
