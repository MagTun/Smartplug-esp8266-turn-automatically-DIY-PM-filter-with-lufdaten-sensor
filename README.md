# Turn automatically a PM filter with a smartplug (esp8266) based on the Lufdaten sensor data

## What you need 
1) a smart plug with tasmota already installed (you can find a list of plug with plug preinstalled with tasmota [here](https://templates.blakadder.com/plug.html)) or any smart plug with an esp8266 or esp8285* (if you managed to "brick" the plug, go to the section "how to unbrick it?')
I bought mine on Aliexpress at 16€:  [ATHOM preflashed TASMOTA Smart Plug 16A](https://fr.aliexpress.com/item/4001230982267.html?spm=a2g0s.9042311.0.0.40696c375Omr8M)     

    *) EPS8285 is equivalent to an ESP8266 embedded with 1MB Flash memory, therefore, ESP8285 uses the same set of SDK as ESP8266's, namely ESP8266 SDK)


2) Build your luftadaten sensor 
(20$): https://sensor.community/en/

3) Build your air filter
It's just a hepa filter on a fan! Yes it works, even if there is air from the fan passing around the filter! https://smartairfilters.com/en/blog/how-to-make-diy-air-purifier/ 

![image](https://user-images.githubusercontent.com/15843700/132995672-2985cf44-918e-4db1-b9ed-494c5e20089f.png)

This design has been tested by Thomas Talhelm "a non-expert nerd who has dedicated years to understanding how to live under Beijing’s particulate cloud":  
https://smartairfilters.com/en/blog/what-is-the-best-air-purifier-on-the-market-today/  
https://smartairfilters.com/en/blog/diy-air-outlet-tests/  
https://smartairfilters.com/en/blog/comparison-diy-filter-effectiveness-iq-air/  

## Basic/general info about tasmota
With tasmota you can directly send a command and save/edit a script from a web interface (you don't need to build + upload the bin file to change your code). It takes ≈7seconds for the plug to save the script and restart.
To do that just use need to open the IP of you tasmota device in your browser, to save a script on the Consoles>Edit script, and to go to Consoles>Console and write in the text input :  `script 1`   (and then press Enter)

## Backup everything (just in case)
- If you have a tasmota plug already configured: connect your smartplug and go to Configuration> Backup Configuration (it will backup the most important pin configuration and probably the wifi 

## Building a custom tasmota firmware that supports scripting 
You can use the `tasmota_custom.bin.gz` (based on 9.5.0.8) that is provided in this repository or you can build the same custom tasmota by following these step:

- Install VSCode [Visual Studio Code](https://code.visualstudio.com/) and the extension [PlatformIO](https://platformio.org/) via the extension menu inside VScode.
- Go to https://github.com/arendst/Tasmota , click on `Code` then `Download ZIP` and extract the zip (mine was extracted in the folder in the "Tasmota-development")
- In VSCode, open the command palette and select `PlatformIO : Home`
- On the PlatformIO home page, click on `Open project`, select your folder (eg: Tasmota-development). If VSCode aks if you trust the authors, click `Yes`
- Then make a copy of the file `Tasmota-development\tasmota\user_config_override_sample.h` and rename the `copy user_config_override.h` 
- Open `user_config_override_sample.h`  inside VSCode and just before the very last `#endif` paste this:

```
    #ifndef USE_SCRIPT
    #define USE_SCRIPT
    #endif
    #ifdef USE_RULES
    #undef USE_RULES
    #endif

    #define USE_WEBSEND_RESPONSE
    #define HTTP_DEBUG

```

- Make sure that at the bottom of the VSCode window, the path is set to "env:tasmota (Tasmota-development)" (otherwise click on it and select the correct env:  https://imgur.com/ISQzFdp )
- Open the command palette and select PlatformIO Built 

## Upload the build to the tasmota device
In your browser open the IP of your smartplug and go to Firmware Upgrade then click on upload a file and select Tasmota-development\build_output\firmware\tasmota.bin.gz  (advice: always use the bin.gz from the build_output, not the .bin from the .pio) (Don't install the wrong bin/scrip, otherwise you will "brick the plug": you will have then to break it open and do some soldering to fix it!)

● If you get a "Upload buffer miscompare" error
- This means that the smartplug doesn't have enough free memory to install the custom firmware. So you need to first install a "tasmota minimal firmware" before uploading your custom firmware.
- go to https://github.com/arendst/Tasmota/releases and download the latest tasmota-minimal.bin.gz (you need to scroll down pass the migration info about older version and the Changelog to finally get to the list of all tasmota.bin)
- open your tasmota plug IP and go to Firmware Upgrade> and upload the tasmota-minimal.bin.gz 
- once it is done, upload your custom firmware (Tasmota-development\build_output\firmware\tasmota.bin.gz)

##  Creating the script that will retrieve the value from the lufdaten sensor
- in your browser open the IP of your smartplug
- click on  consoles> edit script 
- check the checkbox to enable script
- paste this: you need to modify the IP (cf http) with the one from your luftdaten sensor and if you want the threshold (cf pm10t and pm25t)
 
 
 ```
>D
; declare var. for pm10 and pm25: a string var (to extract the json text), a number var (to convert the string into a num) and a threshold var (to easily change it)
pm10=""
pm10n=0
; default 15
pm10t=15

pm25=""
pm25n=0
; default 5
pm25t=5

; declare var to make the http call 
res=0

; declare var to avoid many print
flg=0


>S
; make the call every x seconds (here it's 60)
if upsecs%60==0
then
flg=1
; get contents to http page, and when ready call section
res=http("192.168.1.39:888" "/data.json")
flg=0
endif

>E

if flg>0
then
; extract 4th/6th occurrence of text "value", eg.  ":"5.22"},{"
pm10=gwr("value" 4)
pm25=gwr("value" 6)

; parse the result (remove  " and keep only numbers)
pm10=st(pm10 " 2)
print PM10  %pm10%
pm25=st(pm25 " 2)
print PM2.5  %pm25%

; convert string to number
pm10n=pm10
pm25n=pm25


if pm25n>pm25t
then 
print Turn ON bcs PM25 >  %pm25t%
=>power1 1

else 

if pm10n>pm10t
then 
print Turn ON bcs PM10 >  %pm10t%
=>power1 1

else 
print PM25 and PM25 < threshold 
; will switch off!
=>power1 0
; endif pm10
endif
;endif pm25
endif
;endif flg
endif

``` 

- Now fo to consoles>console and write : `script 1`   (and then press Enter)
- Everything should be working now! 

Don't forget to plug your PM filter on the smartplug!

## Advices :
- if you have any problem (ex: during the build or a function that doesn't work properly), update tasmota : they regularly fix the bugs (two times in a week, I solved errors by  simply updating the tasmota to the latest)
- always upload the .bin.gz not the .bin





## How to fix a bricked smart plug (by erasing it and reinstalling tasmota) or installing tasmota on a esp8266 plug

■ How to open the plug:  
- best technique: https://www.youtube.com/watch?v=MjG07LQmJjQ   
- interesting comments: https://www.youtube.com/watch?v=mG4bAAHluMU   

■ How to erase /reinstall tasmota
- solder the wires (cf the picture below from https://codesandbolts.com/bsd29-smart-socket-esp8285)
- install esptool via python `python -m pip install esptool`
- download tasmota.bin on http://ota.tasmota.com/tasmota/release/
- put esp8266 in programming mode (connect  gpio0 to ground, plug the usb-serial connector- wait 10sec, disconnect gpio to ground)   
     - when you are in pogramming mode the red light turn ON   
     - when it uploads the orange light (in the middle) turns ON and the white light flash quickly (near the crystal clock of the board).  
- verify that esp8266 is in programming mode  (to get COM port : device manager > Ports (COM & LPT) . )   
`esptool.py -p COM3 read_mac`               (should return several line of info incl mac address)  
`esptool.py -p COM3 flash_id`                  (should return several line similar above)  
- erase flash an install tasmota   
`esptool.py --port COM3 erase_flash`  
`esptool.py --port COM3 write_flash -fs 1MB -fm dout 0x0 firmware.bin`  

![image](https://user-images.githubusercontent.com/15843700/133206729-8d54ea10-94e3-4dc6-a42e-dd4b5e6ca138.png)
others tuto:  
https://community.octoprint.org/t/make-your-own-sonoff-smart-plug/13230  
https://www.instructables.com/USB-to-Serial-Convertor-for-ESP8266/  

■ Configurate the plug
- connect the plug and enter wifi
- re-enter the config specs for the plug : cf configuration> Restore config







