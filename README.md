# TFT35V1.0-Klipper-screen

First of all, you need to make a single hardware change on the screen.
MKS TFT35 V1.0 screen, the interrupt output from the touchscreen is not soldered. This is easy to fix with just one jumper.  Without this refinement, there will be no sensor poll, so the touch does not work in the system.I don't have the slightest idea how the Marlinde sensor activates. 
As far as I can see from the [schematics](https://github.com/zavarci/TFT35V1.0-Klipper-screen/blob/main/Improvement%20on%20the%20screen%20board/sch_mks_tft35_int_ts.GIF), this change has no harm in marlin.
On the board, the jumper looks like [this](https://github.com/zavarci/TFT35V1.0-Klipper-screen/blob/main/Improvement%20on%20the%20screen%20board/jumper%20cooper.jpg) it's not that hard, I had a harder time taking the photo.



In the process of writing/editing.



At this stage, there are 3 options:
1) You have nothing installed, then go to the "Installing from scratch" item.
2) You already have klipper running, no screen (no Klipperscreen). Then follow the instructions, just install the Klipperscreen.
3) You already have a Klipperscreen (for example, HDMI), and you want to connect your native 3.5 inch screen. - Let's move on to setting up the TFT35
Installation from scratch
So, using [Raspberry Pi Imagerupload](https://www.raspberrypi.com/software/) the above image to a microSD card. You can choose 32 bits - I did not notice the difference, I put it on both options.
after starting the system and connecting via SSH via [PUTTY](https://www.putty.org/) install GIT:
```shell
sudo apt install git
```
Next, install[KIAUH](https://github.com/th33xitus/kiauh), through which we will install klipper and all its components. We sequentially enter the commands:
```shell
cd ~
git clone https://github.com/th33xitus/kiauh.git
./kiauh/kiauh.sh
```
If everything went according to plan, after the third command we will see the "window" of the installation.
![main](https://github.com/zavarci/TFT35V1.0-Klipper-screen/blob/main/pictures/kiauh_menu.PNG)  
Install the following parts in sequence. If you get an error at some stage, you don't need to continue. you must first understand the cause of the error, eliminate it, and re-install the package that was not installed.
Klipper (1 pc) - required. If you have two or more printers working from one raspberry, then I just don’t understand what you are doing in such a noob instruction - scroll on!
Moonraker (1 pc) - required.
Fluidd or Mansail to choose from. My choice is Mansail.
Klipperscreen 

Sometimes during the installation of Klipperscreen a message pops up asking you to update PIP, just in case I will give the command here (you need to adjust the path if your username is not "pi"):

```shell
/home/pi/.KlipperScreen-env/bin/python -m pip install --upgrade pip
```
If all the previous installation steps were completed successfully and you see a working Mansail in the browser, then congratulations - the distribution kit is 100% suitable for us))).
If you already have Klipper installed and working, but without Klipperscreen, then it is enough to install only the latter.
If everything is already working for you, including Klipperscreen on HDMI, then you can continue the installation.


TFT35 setup
1) create Overlay
copy to home directory (/home/pi/) overlay files mkstft35_rpi.dts from[archive](https://github.com/zavarci/TFT35V1.0-Klipper-screen/raw/main/DTS.rar).
in the console we enter the following commands (we compile the overlays):
```shell
sudo dtc -@ -I dts -O dtb -o /boot/overlays/mkstft35_rpi.dtbo ~/mkstft35_rpi.dts
```
you should see something like this in the console:
![main](https://github.com/zavarci/TFT35V1.0-Klipper-screen/blob/main/pictures/overlay_compile.PNG) 


2) Activate SPI0 on RaspberryPi.
To do this, edit the file /boot/config.txt
```shell
sudo nano /boot/config.txt
```
look for the line " #dtparam=spi=on " and uncomment it - remove the "#" at the beginning of the line,

3) connect the screen overlay
If the screen is connected to SPI0, then add the following lines to the end of the /boot/config.txt file:
```shell
###### MKS TFT35
hdmi_force_hotplug=1
hdmi_cvt=hdmi_cvt=480 320 60 1 0 0 0
hdmi_group=2
hdmi_mode=1
hdmi_mode=87
display_rotate=0

dtoverlay=mkstft35_rpi,rotate=270,speed=24000000,touch,touchgpio=17,fps=20
###### MKS TFT35
```

Save (Ctrl+S) and exit the nano editor (Ctrl+X).

Important note!
At the time of the experiments, when working through SPI0, the refresh rate by eye corresponds to the set one (about 20 frames per second). 

you cannot connect an ADXL345 accelerometer for use in a Klipper. Therefore, when the screen is connected to SPI0, to test the resonances, you will need to turn off (comment out) the display overlay .This requires removing a jumper, but was probably somewhat unnecessary. And this is not always convenient.. Or You have several options.
You can use [MPU6050]( https://www.klipper3d.org/Measuring_Resonances.html) sensor from I2C port. You can connect ADLX345 sensor to another MCU. For example [Robin Nano](https://www.reddit.com/r/klippers/comments/ul5h6p/accelerometer_adxl345_wired_to_robin_nano_v1x/), [Nano](https://nate15329.com/klipper-input-shaper-w-arduino-nano/), [Pico](https://klipper.discourse.group/t/raspberry-pi-pico-adxl345-portable-resonance-measurement/1757) ...
I think MPU6050 is cheaper and more common.

4) installation[FBCP](https://github.com/tasanakorn/rpi-fbcp)
necessary to copy the output of the primary framebuffer to the secondary one (for example, as we have - FBTFT).
run the commands in sequence:
```shell
sudo apt-get install cmake
cd ~
sudo git clone https://github.com/tasanakorn/rpi-fbcp
cd rpi-fbcp/
sudo mkdir build
cd build
sudo cmake ..
sudo make
sudo install fbcp /usr/local/bin/fbcp
```
Create a service file:
```shell
sudo nano /etc/systemd/system/fbcp.service
```
Add the following lines to it:
```shell
[Unit]
Description=fbcp
After=KlipperScreen.service
StartLimitIntervalSec=0
[Service]
Type=simple
Restart=always
RestartSec=1
User=root
ExecStart=/usr/local/bin/fbcp

[Install]
WantedBy=multi-user.target
```
Save (Ctrl+S) and exit the nano editor (Ctrl+X).

We allow the service to work:
```shell
sudo systemctl enable fbcp.service
```
After loading, you should see console lines at the beginning, after which Klipperscreen will start. If it does not start, then you need to run through it[trouble shooter](https://github.com/jordanruthe/KlipperScreen/blob/master/docs/Troubleshooting.md).
From experience, the most common error is: "xf86OpenConsole: Cannot open virtual console 2 (Permission denied)"
You just need to add one line "needs_root_rights=yes" to the file "/etc/X11/Xwrapper.config".
To do this, enter in the console:
```shell
sudo nano /etc/X11/Xwrapper.config
```
add the line "needs_root_rights=yes", if it is missing, save (Ctrl+S) and exit the nano editor (Ctrl+X).
Restart KlipperScreen:
```shell
sudo service KlipperScreen restart
```

5) installation[xinput-calibrator](https://github.com/kreijack/xlibinput_calibrator)
A calibrator must be installed to calibrate the sensor. First, install the required libraries:
```shell
DISPLAY=:0 ./xlibinput_calibrator --output-file-x11-config=x11_config.txt
```
At this time, a proposal will appear on the screen to poke crosses - we execute. The result of the work will be several lines in the console, such as this:
![main](https://github.com/zavarci/TFT35V1.0-Klipper-screen/blob/main/pictures/xlibinput_calibrator_result.PNG)  

Add this result to the file:
```shell
sudo nano /usr/share/X11/xorg.conf.d/99-calibration.conf
```
I will leave here my parameters for an example:
```shell
Section "InputClass"
        Identifier      "calibration"
        MatchProduct    "ADS7846 Touchscreen"
        Option          "CalibrationMatrix"     "0.003395 -1.121566 1.056742 1.102245 -0.008974 -0.048807 0.000000 0.000000 1.000000 "
EndSection
```
Reboot
```shell
sudo reboot
```
The touchscreen should work correctly.

I got into this kind of complex business because of the dictates of the participants and developers.
