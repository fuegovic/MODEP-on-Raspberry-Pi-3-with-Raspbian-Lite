# **Complete Guide: Setting Up MODEP on Raspberry Pi 3 (Raspbian Lite & USB audio)**

> Replace `<username>` with your preferred user and hostname.

## **💻 Before You Begin**

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

## **🎚️ Install MODEP**

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

## **🚀 Check and Start MODEP Services**

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

## **🔊 Configure JACK for USB Audio Devices**

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

## **✨ Optimize Performance (Reduce Audio Crackles)**

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

### **Step 3: Optional Overclocking 💪**

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

## **🛜 Set Up a Wi-Fi Hotspot (For MODEP Web UI Access)**

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

Now you should have **MODEP running smoothly on your Raspberry Pi 3**! 🎚️

## **📴 Add a Shutdown Button**

This section explains how to add a physical shutdown button for your Raspberry Pi using a GPIO pin and a Python script. Additionally, you'll have the option to add an LED for visual feedback.

---

### **Step 1: Wiring the Button**

1. Connect one leg of the push button to a GPIO pin (e.g., `GPIO 26`).
2. Connect the other leg of the push button to a **GND (Ground)** pin.

> If you're also adding an LED, connect the **long leg (anode)** of the LED to a GPIO pin (e.g., `GPIO 17`) through a resistor (330Ω is recommended). Connect the **short leg (cathode)** to **GND**.

### **Step 2 (Option A): Basic Shutdown Button (No LED)**

Using just the button, we'll create a script that requires the button to be held for **3 seconds** before shutting down the Raspberry Pi.

#### **Install required libraries**

Update the package repository and install `gpiozero`:

```sh
sudo apt-get update
sudo apt-get install python3-gpiozero
```

#### **Create the basic shutdown script**

1. Create a new Python file:
   ```sh
   nano shutdown_button.py
   ```

2. Paste the following code:

   ```python
   from gpiozero import Button
   import os
   import time

   # Define the GPIO pin the button is connected to
   shutdown_button = Button(26, hold_time=3)

   # Function to shutdown the Raspberry Pi
   def shutdown():
       print("Shutdown button pressed and held for 3 seconds")
       os.system("sudo shutdown now")

   # When the button is held for 3 seconds, call the shutdown function
   shutdown_button.when_held = shutdown

   # Keep the script running
   while True:
       time.sleep(1)
   ```

3. Save the file (`CTRL+X`, then `Y`, then `Enter`).

#### **Make the script executable**

Run the following command to make the script executable:
```sh
chmod +x shutdown_button.py
```

### **Step 2 (Option B): Enhanced Shutdown Button (With LED Feedback)**

This version adds visual feedback using an LED. The LED will blink while the button is held down for 3 seconds, and the Pi will shut down once the button is released after the hold.

#### **Install required libraries**

If you've already installed `gpiozero`, skip this step.

```sh
sudo apt-get update
sudo apt-get install python3-gpiozero
```

#### **Create the enhanced shutdown script**

1. Create a new Python file:
   ```sh
   nano shutdown_button_with_led.py
   ```

2. Paste the following code:

   ```python
   from gpiozero import Button, LED
   import os
   import time

   # Define GPIO pins for the button and LED
   shutdown_button = Button(26, hold_time=3)
   status_led = LED(17)

   # Function to shutdown the Raspberry Pi
   def shutdown():
       print("Shutdown button pressed and held for 3 seconds")
       status_led.on()  # Turn on the LED just before system shuts down
       os.system("sudo shutdown now")

   # Function to blink the LED while the button is held
   def blink_led():
       print("Button held, blinking LED...")
       for _ in range(6):  # Blink for 3 seconds (6 blinks, each lasting 0.5s)
           status_led.toggle()
           time.sleep(0.5)

   # Configure button behavior
   shutdown_button.when_held = blink_led  # Blink LED while held
   shutdown_button.when_released = shutdown  # Shut down when button is released after 3+ seconds

   # Ensure the LED stays off initially
   status_led.off()

   # Keep the script running
   while True:
       time.sleep(1)
   ```

3. Save the file (`CTRL+X`, then `Y`, then `Enter`).

#### **Make the script executable**

Run the following command:
```sh
chmod +x shutdown_button_with_led.py
```

### **Step 3: Run the Script on Startup**

To ensure that the shutdown button script runs automatically whenever the Pi boots, you can use **rc.local** or create a **systemd service**. These instructions apply to **both the basic and LED-enhanced versions**. Just replace `<script_name.py>` with the appropriate script (`shutdown_button.py` or `shutdown_button_with_led.py`).

#### **Option 1: Using rc.local**

1. Open the `rc.local` file:
   ```sh
   sudo nano /etc/rc.local
   ```

2. Add the following line **before** the `exit 0` line. Replace `<script_path>` with the full path to your script:

   ```sh
   python3 /<script_path>/<script_name.py> &
   ```

3. Save the file and exit.


#### **Option 2: Using systemd**

1. Create a new systemd service file:
   ```sh
   sudo nano /etc/systemd/system/shutdown_button.service
   ```

2. Add the following content:

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

   Replace `/path/to/<script_name.py>` with the full path to the Python script.

3. Save the file and exit (`CTRL+X`, then `Y`, then `Enter`).

4. Enable the service:
   ```sh
   sudo systemctl enable shutdown_button.service
   ```

5. Start the service:
   ```sh
   sudo systemctl start shutdown_button.service
   ```

### **Step 4: Test the Shutdown Button**

To verify that your script and setup work:

1. Press and hold the button for **3 seconds**.
2. If using the LED version:
   - The LED will blink while the button is held down.
   - The Pi will shut down when the button is released after the 3-second hold.
3. Check the status of the service for troubleshooting:
   ```sh
   sudo systemctl status shutdown_button.service
   ```

Now your Pi is configured with a hardware shutdown button, with or without LED feedback. ⚡
