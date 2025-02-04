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
7. [Add a Shutdown Button](#-add-a-shutdown-button)
   - [Step 1: Wiring the Button](#step-1-wiring-the-button)
   - [Step 2: Button Setup Options](#step-2-option-a-basic-shutdown-button-no-led)
   - [Step 3: Run the Script on Startup](#step-3-run-the-script-on-startup)
   - [Step 4: Test the Shutdown Button](#step-4-test-the-shutdown-button)

> Replace `<username>` with your preferred user and hostname.

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

## 📴 Add a Shutdown Button

### Step 1: Wiring the Button
1. Connect one leg of the push button to GPIO 26
2. Connect the other leg to GND (Ground)
3. (Optional) For LED: Connect anode to GPIO 17 through 330Ω resistor, cathode to GND

### Step 2 (Option A): Basic Shutdown Button (No LED)

Install required libraries:
```sh
sudo apt-get update
sudo apt-get install python3-gpiozero
```

Create shutdown script:
```sh
nano shutdown_button.py
```

```python
from gpiozero import Button
import os
import time

shutdown_button = Button(26, hold_time=3)

def shutdown():
    print("Shutdown button pressed and held for 3 seconds")
    os.system("sudo shutdown now")

shutdown_button.when_held = shutdown

while True:
    time.sleep(1)
```

Make executable:
```sh
chmod +x shutdown_button.py
```

### Step 2 (Option B): Enhanced Shutdown Button (With LED)

Create enhanced script:
```sh
nano shutdown_button_with_led.py
```

```python
from gpiozero import Button, LED
import os
import time

shutdown_button = Button(26, hold_time=3)
status_led = LED(17)

def shutdown():
    print("Shutdown button pressed and held for 3 seconds")
    status_led.on()
    os.system("sudo shutdown now")

def blink_led():
    print("Button held, blinking LED...")
    for _ in range(6):
        status_led.toggle()
        time.sleep(0.5)

shutdown_button.when_held = blink_led
shutdown_button.when_released = shutdown
status_led.off()

while True:
    time.sleep(1)
```

Make executable:
```sh
chmod +x shutdown_button_with_led.py
```

### Step 3: Run the Script on Startup

#### Option 1: Using rc.local
```sh
sudo nano /etc/rc.local
```

Add before `exit 0`:
```sh
python3 /<script_path>/<script_name.py> &
```

#### Option 2: Using systemd
Create service file:
```sh
sudo nano /etc/systemd/system/shutdown_button.service
```

```
[Unit]
Description=Shutdown Button Service
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /<script_path>/<script_name.py>

[Install]
WantedBy=multi-user.target
```

Enable and start:
```sh
sudo systemctl enable shutdown_button.service
sudo systemctl start shutdown_button.service
```

### Step 4: Test the Shutdown Button
1. Press and hold button for 3 seconds
2. If using LED version:
   - LED will blink during hold
   - System shuts down on release
3. Check service status:
```sh
sudo systemctl status shutdown_button.service
```
