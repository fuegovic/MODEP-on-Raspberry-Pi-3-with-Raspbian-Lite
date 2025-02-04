# Complete Guide: Setting Up MODEP on Raspberry Pi 3 (Raspbian Lite & USB audio)

## Table of Contents
1. [Before You Begin](#-before-you-begin)
   - [Step 1: Prepare the SD Card](#step-1-prepare-the-sd-card)
   - [Step 2: Initial Setup](#step-2-initial-setup)
2. [Install MODEP](#️-install-modep)
   - [Step 1: Install MODEP](#step-1-install-modep)
   - [Step 2: Enable Auto-Login](#step-2-enable-auto-login-console-mode)
3. [Check and Start MODEP Services](#-check-and-start-modep-services)
4. [Configure JACK for USB Audio Devices](#-configure-jack-for-usb-audio-devices)
   - [Step 1: Identify Audio Devices](#step-1-identify-audio-devices)
   - [Step 2: Set Up JACK](#step-2-set-up-jack)
5. [Optimize Performance](#-optimize-performance-reduce-audio-crackles)
   - [Step 1: Adjust Realtime Priority](#step-1-adjust-realtime-priority)
   - [Step 2: Increase USB Audio Stability](#step-2-increase-usb-audio-stability)
   - [Step 3: Optional Overclocking](#step-3-optional-overclocking-)
6. [Set Up a Wi-Fi Hotspot](#-set-up-a-wi-fi-hotspot-for-modep-web-ui-access)
   - [Step 1: Identify Your Wi-Fi Adapter](#step-1-identify-your-wi-fi-adapter)
   - [Step 2: Create a Wi-Fi Hotspot](#step-2-create-a-wi-fi-hotspot)
   - [Step 3: Configure Auto-Connect on Boot](#step-3-configure-auto-connect-on-boot)
7. [Add a Shutdown and Power Up Button](#-add-a-shutdown-and-power-up-button)
   - [Step 1: Wiring the Button](#step-1-wiring-the-button)
   - [Step 2: Enable the Button](#step-2-enable-the-shutdown-button-in-configtxt)
   - [Step 3: Test the Button](#step-3-test-the-shutdown-button)
   - [Why This Method?](#why-this-method)
8. [Add an Activity LED (Optional)](##-add-an-activity-led-optional)
   - [Step 1: Wiring the LED](#step-1-wiring-the-led)
   - [Step 2: Enable the LED](#step-2-enable-the-serial-port-for-led-activity)
   - [Step 3: Test the LED](#step-3-test-the-activity-led)


## 💻 Before You Begin

### Step 1: Prepare the SD Card
- Download and install [Pi-imager](https://github.com/guysoft/pi-imager/releases)
- Select **Raspbian Lite** as the OS
- Configure your **username/hostname** in the settings
- Write the image to the SD card

### Step 2: Initial Setup
- Insert the SD card into the Raspberry Pi and power it on
- Connect to the internet (Ethernet or Wi-Fi)
- Update the system:
  ```sh
  sudo apt update && sudo apt upgrade -y
  ```

## 🎚️ Install MODEP

MODEP (MOD Emulation Pedalboard) turns your Raspberry Pi into a powerful multi-effects unit. Follow these steps to install it:

### Step 1: Install MODEP
```bash
curl https://blokas.io/apt-setup.sh | sh
sudo apt update
sudo apt install modep -y
```

### Step 2: Enable Auto-Login (Console Mode)
```bash
sudo raspi-config
```
- Navigate to **System Options > Boot/Auto Login > Console Autologin**
- Save and reboot:
```bash
sudo reboot
```

## 🚀 Check and Start MODEP Services

Verify that MODEP services are running:
```bash
systemctl list-units --type=service | grep modep
```

Critical services:
- `modep-mod-ui`
- `modep-mod-host`
- `jack`

If the UI is not accessible, restart the services:
```bash
sudo systemctl stop modep-mod-ui modep-mod-host jack
sudo systemctl start jack modep-mod-host modep-mod-ui
```

Check statuses:
```bash
sudo systemctl status jack
sudo systemctl status modep-mod-host
sudo systemctl status modep-mod-ui
```

## 🔊 Configure JACK for USB Audio Devices

### Step 1: Identify Audio Devices
Run:
```bash
aplay -l
```

Look for your USB audio device in the output. It may be listed as **Headset**, **CODEC**, or another name (e.g., `hw:DeviceName,0`).

### Step 2: Set Up JACK
Edit `/etc/jackdrc`:
```bash
sudo nano /etc/jackdrc
```

Replace with:
```
exec /usr/bin/jackd -t 2000 -R -P 70 -d alsa -d hw:<DeviceName>,0 -r 48000 -p 512 -n 4 -X seq -s -S
```

> Replace `<DeviceName>` with the name identified in the previous step.

Save and exit (`CTRL+X`, then `Y`, then `Enter`).

Restart JACK:
```bash
sudo systemctl restart jack
```

Check logs:
```bash
sudo journalctl --unit=jack --no-pager | grep XRUN
```

## ✨ Optimize Performance (Reduce Audio Crackles)

### Step 1: Adjust Realtime Priority
Edit `/etc/security/limits.conf`:
```bash
sudo nano /etc/security/limits.conf
```

Add at the end:
```
@audio   -  rtprio     99
@audio   -  memlock    unlimited
@audio   -  nice       -20
<username>  -  rtprio     99
<username>  -  memlock    unlimited
<username>  -  nice       -20
```
> Replace `<username>` with your username

### Step 2: Increase USB Audio Stability
```bash
sudo nano /boot/cmdline.txt
```

Add this at the end of the existing line (don't start a new line):
```
usbcore.autosuspend=-1
```

Save and reboot:
```bash
sudo reboot
```

### Step 3: Optional Overclocking 💪

> ⚠️ **Caution:** Overclocking can improve performance but may increase heat and power consumption. It is only recommended if you experience audio glitches (xruns). Ensure proper cooling and power stability.

Edit the config file:
```sh
sudo nano /boot/firmware/config.txt
```

Add the following:
```sh
# Run as fast as firmware / board allows
arm_boost=1
kernel=zImage
arm_freq=1375
core_freq=525
gpu_freq=525
over_voltage=3
force_turbo=1
sdram_freq=625
sdram_schmoo=0x02000020
over_voltage_sdram_p=4
over_voltage_sdram_i=3
over_voltage_sdram_c=3
dtparam=sd_overclock=100
```

> **Note:** 
> `force_turbo=1` can be removed to avoid forcing max clock speed, which can reduce lifespan.
> `sdram_freq` could be set lower, to a safer 500MHz.

Save and reboot:
```bash
sudo reboot
```

## 🛜 Set Up a Wi-Fi Hotspot (For MODEP Web UI Access)

### Step 1: Identify Your Wi-Fi Adapter
```sh
nmcli device
```

You should see output similar to:
```sh
DEVICE         TYPE      STATE        CONNECTION
wlan0          wifi      disconnected --
eth0           ethernet  connected    Wired connection 1
lo             loopback  unmanaged    --
```

### Step 2: Create a Wi-Fi Hotspot
```sh
sudo nmcli device wifi hotspot ssid <hotspot-name> password <password> ifname wlan0
```

### Step 3: Configure Auto-Connect on Boot
List network connections:
```sh
nmcli connection
```

View connection details:
```sh
nmcli connection show <hotspot-UUID>
```

Enable auto-connect:
```sh
sudo nmcli connection modify <hotspot-UUID> connection.autoconnect yes connection.autoconnect-priority 100
```

Verify changes:
```sh
nmcli connection show <hotspot-UUID>
```

## 📴 Add a Shutdown and Power-Up Button

### Step 1: Wiring the Button
1. Connect one leg of the push button to GPIO 3 (pin 5)  
2. Connect the other leg to GND (pin 6)

*Note:* GPIO 3 is part of the Raspberry Pi's hardware and supports both shutdown (with software configuration) and power-up functionality natively.

### Step 2: Enable the Shutdown Button in `config.txt`

Edit the Raspberry Pi's configuration file:
```sh
sudo nano /boot/config.txt
```

Add the following line at the end:
```sh
dtoverlay=gpio-shutdown,debounce=3000
```
>**Note:** The `debounce=3000` parameter ensures the button must be held for at least 3 seconds to trigger a shutdown. Short presses will be ignored.

Save and exit:
- Press `CTRL+X`, then `Y`, then `Enter`.

Reboot your Pi to apply the changes:
```sh
sudo reboot
```

### Step 3: Test the Shutdown Button
1. Press the button briefly while the Raspberry Pi is running.
   - The system will safely shut down.
2. Press the button again (post-shutdown) to power the Pi back on.

---

### Why This Method?
- **No dependencies or Python scripts required**: Configuration happens directly in `config.txt`.
- **Built-in power-up support**: GPIO 3 (pin 5) works for both shutdown and power-up.
- **Efficient and lightweight**: Uses system-level hardware features without running extra processes or services.

---

### 🟢 Add an Activity LED (Optional)
**Show activity feedback for disk writes.** You can connect an LED to monitor the system's disk activity.

### Step 1: Wiring the LED
1. Connect the **anode** (long leg) of the LED to GPIO 14 (TX) *(pin 8)* through a 330Ω resistor.
2. Connect the **cathode** (short leg) to GND *(pin 6)*.

### Step 2: Enable the Serial Port for LED Activity
Edit the `config.txt` file:
```sh
sudo nano /boot/config.txt
```

Add the following lines:
```sh
enable_uart=1
dtoverlay=disable-bt #Optional
```

Save and exit:
- Press `CTRL+X`, then `Y`, then `Enter`.

Reboot your Pi:
```sh
sudo reboot
```

### Step 3: Test The Activity LED
Once configured:
1. The LED will blink on GPIO14 (TX pin) whenever the system sends serial data.
2. This activity reflects disk I/O or other data being output from the Raspberry Pi.


