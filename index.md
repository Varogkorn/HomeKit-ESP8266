## HomeKit ESP8266 

# This Project Was Fork from Mixiaoxiao/Arduino-HomeKit-ESP8266 

This Arduino library is a native Apple HomeKit accessory implementation for the [ESP8266 Arduino core](https://github.com/esp8266/Arduino), and works without any additional bridges.

This project is mainly based on [esp-homekit](https://github.com/maximkulkin/esp-homekit) for [ESP-OPEN-RTOS](https://github.com/SuperHouse/esp-open-rtos).

I ported the RTOS-based implementation of [esp-homekit](https://github.com/maximkulkin/esp-homekit) to the pure Arduino environment, aimed at easy and fast building project using Arduino IDE (or Eclipse with sloeber, PlatformIO).

Enjoy the "one-key" build, "one-key" upload, and work to link various other Arduino libraries with Apple HomeKit!

Here is a [discussion](https://github.com/HomeACcessoryKid/Arduino-HomeKit/issues/1) about the RTOS is required for running Apple HomeKit, and this project is a proof of concept that Apple HomeKit can be implemented and work fine without the RTOS.

This library is built with ESP8266 Arduino Core 2.6.3. Lower versions may compile with errors.

For ESP32, see [Arduino-HomeKit-ESP32](https://github.com/Mixiaoxiao/Arduino-HomeKit-ESP32). The HomeKit running on ESP32 has a **GREAT PERFORMANCE** which is 10x faster than ESP8266.

## Preview

![Preview](https://raw.github.com/Mixiaoxiao/Arduino-HomeKit-ESP8266/master/extras/preview.jpg) 


## Setup code of the example sketch

``111-11-111``


## Usage

1. Add Src file to Arduino IDE Library
	- Download This project in ZIP
	- Arduino IDE program 
	- On toolbar click Sketch -----> Include Library -----> Add .ZIP Library
	- Select file from Download and You good to go.

2. Define your accessory in a .c file to enjoy the  convenient "Macro" style declaration. You can also define your accessory in a .ino file using C++ code.
	```C
		homekit_accessory_t *accessories[] = ...
		homekit_server_config_t config = {
			.accessories = accessories,
			.password = "111-11-111",
			//.on_event = on_homekit_event, //optional
			//.setupId = "ABCD" //optional
		};
	```
3. Config Your WIFI Settings at wifi_info.ino by
	```C
		const char *ssid = "Your WIFI NAME HERE";
		const char *password = "YOUR WIFI PASSWORD HERE";
	```
4. Change Your ESP32/8266 Setting by Using this.
	```C
		* Module: Generic ESP8266 Module (to enable full settings)
		* FlashSize: at least 470KB for sketch (see `WolfSSL` section if you want a smaller sketch) 
		* LwIP Variant: v2 Lower Memory (for lower memory use)
		* Debug Level: None (for lower memory use)
		* Espressif FW: nonos-sdk 2.2.1+119(191122) (which I used to build this project)
		* SSL Support: Basic SSL ciphers (lower ROM use)
		* VTables: Flash (does not matter maybe)
		* Erase Flash: select `All Flash Contents` when you first upload
		* CPU Frequency: 160MHz (must)
	```
	
Done.

## Performance

Notice: You should set the ESP8266 CPU to run at 160MHz (at least during the pairing process), to avoid the tcp-socket disconnection from iOS device caused by timeout.

* Preinit: ~9.1s (You can see the accessory on your iOS HOME app after Preinit)
* Pair Setup Step 1/3: ~0s (The heavy crypto computation is done in Preinit)
* Pair Setup Step 2/3: ~12.1s 
* Pair Setup Step 3/3: ~0.8s  (The pair-setup is only processed when first paired with iOS device)
* Pair Verify Step 1/2: ~0.3s
* Pair Verify Step 2/2: ~0.8s (The Verify Step is required every time iOS connects or reconnects to ESP8266 to establish secure session)

All pairing process takes ~14s after you input the setup-code on your iPhone. Notice that Preinit require ~9s before you can start to pair.


## Heap (memory)

The heap is critical for ESP8266 with full TCP/IP support. ESP8266 easily crashes when the memory is lower than ~5000.

I tried to make WolfSSL crypto work safely on ESP8266 with better performance and lower memory or a trade-off. See details in next section.

Here are the free heap values of running the example sketch:

* Boot: ~26000
* Preinit over: ~22000
* Pairing: ~17000 (or even low when crypto computing)
* Paired and connected with one iOS device: ~21700
* Paired and no iOS device connected: ~23400

After memory optimization in v1.1.0:

* Boot: ~46000
* Preinit over: ~41000
* Pairing: ~37000 (or even low when crypto computing)
* Paired and connected with one iOS device: ~41700
* Paired and no iOS device connected: ~43000


## WolfSSL

* Based on wolfssl-3.13.0-stable.
* Clean source code: the unused files are removed.
* `CURVE25519_SMALL` and `ED25519_SMALL`: ESP8266 can not directly run without `SMALL` defined since the memory is not sufficient. But the NO `SMALL` version is faster. I mark the big `ge_precomp base[32][8]` with PROGMEM to store it in Flash (around 70KB). Also the `ge_double_scalarmult_vartime` can not run caused by lack of heap. I define `ESP_GE_DOUBLE_SCALARMULT_VARTIME_LOWMEM` in `user_settings.h` to use LOWMEM version of `ge_double_scalarmult_vartime` in `ge_low_mem.c`. This is a trade-off of performance and memory. If you want more Flash space, you should define `CURVE25519_SMALL` and `ED25519_SMALL` and undefine `ESP_GE_DOUBLE_SCALARMULT_VARTIME_LOWMEM` in `user_settings.h` (this will lead the Pair Verify Steps to take 1.2s + 0.9s).
* `integer.c`(big integer operations): `MP_16BIT` and `ESP_FORCE_S_MP_EXPTMOD` are defined for better performance in ESP8266. `ESP_INTEGER_WINSIZE` (value is 3) is defined to avoid crash caused by memory exhaust and the values of {3, 4, 5} are of similar performance.

## Storage

* The pairing data is stored in the `EEPROM` address in ESP8266 Arduino core.
* This project does not use the `EEPROM` library with data-cache to reduce memory use (directly call flash_read and write). 
* The `EEPROM` is 4096B in ESP8266, this project uses max [0, 1408B).
* See the comments in `storge.c` and [ESP8266-EEPROM-doc](https://arduino-esp8266.readthedocs.io/en/2.6.3/libraries.html#eeprom).
* `EEPROM` of [1408, 4096) is safe for you to use. 
* This project do NOT use `FS(file system)`, so you can use `FS` freely.


## WatchDog

* There are software and hardware watchdogs in ESP8266 Arduino core. The heavy crypto computing will lead to watchdog reset.
* There are disable/enable api of software-watchdog in ESP8266 Arduino core.
* I found the [esp_hw_wdt](https://github.com/ComSuite/esp_hw_wdt) to disable/enable the hardware-watchdog.
* The two watchdogs are disabled while `Preinit` and `Pair Setup Step 2/3`.

## Recommended settings in IDE

* Module: Generic ESP8266 Module (to enable full settings)
* FlashSize: at least 470KB for sketch (see `WolfSSL` section if you want a smaller sketch) 
* LwIP Variant: v2 Lower Memory (for lower memory use)
* Debug Level: None (for lower memory use)
* Espressif FW: nonos-sdk 2.2.1+119(191122) (which I used to build this project)
* SSL Support: Basic SSL ciphers (lower ROM use)
* VTables: Flash (does not matter maybe)
* Erase Flash: select `All Flash Contents` when you first upload
* CPU Frequency: 160MHz (must)

## Arduino port

* `ESP8266WiFi` (WiFiServer and WiFiClient) is used for tcp connection.
* `ESP8266mDNS` is used for advertising (Bonjour) 

## Troubleshooting

* Check your serial output with [example_serial_output.txt](https://raw.github.com/Mixiaoxiao/Arduino-HomeKit-ESP8266/master/extras/example_serial_output_v1.1.0.txt)


You can use the [editor on GitHub](https://github.com/Varogkorn/HomeKit-ESP8266/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/Varogkorn/HomeKit-ESP8266/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.