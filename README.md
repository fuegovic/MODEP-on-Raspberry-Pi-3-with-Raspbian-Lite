# **Complete Guide: Setting Up MODEP on Raspberry Pi 3 (Raspbian Lite & USB audio)**

> Replace `<username>` with your preferred user and hostname.

## **0. Before You Begin**

### **Step 1: Prepare the SD Card**

- Download and install [Pi-imager](https://github.com/guysoft/pi-imager/releases).
- Select **Raspbian Lite** as the OS.
- Configure your **username/hostname** in the settings.
- Write the image to the SD card.

### **Step 2: Initial Setup**

- Insert the SD card into the Raspberry Pi and power it on.
- Connect to the internet (Ethernet or Wi-Fi).
- Update the system:
  ```sh
  sudo apt update && sudo apt upgrade -y
  ```

## **1. Install MODEP**

MODEP (MOD Emulation Pedalboard) turns your Raspberry Pi into a powerful multi-effects unit. Follow these steps to install it:

### **Step 1: Install MODEP**

```bash
curl https://blokas.io/apt-setup.sh | sh
sudo apt update
sudo apt install modep -y
```

### **Step 2: Enable Auto-Login (Console Mode)**

```bash
sudo raspi-config
```

- Navigate to **System Options > Boot/Auto Login > Console Autologin**
- Save and reboot:

```bash
sudo reboot
```

## **2. Check and Start MODEP Services**

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

If `jack` fails, it's likely a configuration issue.

## **3. Configure JACK for USB Audio Devices**

### **Step 1: Identify Audio Devices**

Run:

```bash
aplay -l
```

Look for your USB audio device in the output. It may be listed as **Headset**, **CODEC**, or another name (e.g., `hw:DeviceName,0`).

### **Step 2: Set Up JACK**

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

## **4. Optimize Performance (Reduce Audio Crackles)**

### **Step 1: Adjust Realtime Priority**

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
> **Note:** Replace `<username>` with your username

Save and exit.

### **Step 2: Increase USB Audio Stability**

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

### **Step 3: Optional Overclocking**

> ⚠️ **Caution:** Overclocking can improve performance but may increase heat and power consumption. It is only recommended if you experience audio glitches (xruns). Ensure proper cooling and power stability.

If you decide to overclock, edit the config file:

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

## **5. Set Up a Wi-Fi Hotspot (For MODEP Web UI Access)**

### **Step 1: Identify Your Wi-Fi Adapter**

Run the following command to check available network devices:

```sh
nmcli device
```

You should see an output similar to this:

```sh
DEVICE         TYPE      STATE        CONNECTION
wlan0          wifi      disconnected --
eth0           ethernet  connected    Wired connection 1
lo             loopback  unmanaged    --
```

> The built-in Raspberry Pi Wi-Fi module is typically listed as **wlan0**.

### **Step 2: Create a Wi-Fi Hotspot**

Run the following command, replacing `<hotspot-name>` and `<password>` with your desired network name and password:

```sh
sudo nmcli device wifi hotspot ssid <hotspot-name> password <password> ifname wlan0
```

### **Step 3: Configure Auto-Connect on Boot**

Run the following command to list network connections:

```sh
nmcli connection
```

Copy the **UUID** of the hotspot from the output and use it in the next command:

```sh
nmcli connection show <hotspot-UUID>
```

Check these properties:

```sh
connection.autoconnect:                 no
connection.autoconnect-priority:        0
```

Enable auto-connect and set priority:

```sh
sudo nmcli connection modify <hotspot-UUID> connection.autoconnect yes connection.autoconnect-priority 100
```

Verify the changes:

```sh
nmcli connection show <hotspot-UUID>
```

Now, your Wi-Fi hotspot will automatically activate whenever the Raspberry Pi boots.

Now you should have **MODEP running smoothly on your Raspberry Pi 3**! 🚀

