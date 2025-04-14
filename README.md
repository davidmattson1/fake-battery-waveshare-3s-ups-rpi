# Waveshare 3S UPS Battery Emulator for Raspberry Pi (BETA)

This project emulates a battery-backed power supply on Raspberry Pi using the Waveshare 3S UPS HAT and the INA219 current/voltage sensor. It uses a custom kernel module to fake battery and AC power status so that desktop environments (e.g., GNOME, LXDE) report realistic battery states.

---
- Currently, I have only tested this on Ubuntu 24.10 on a Raspberry pi 5. This very well could work on other things that can read i2c from python.

---

## Features
- Reports battery capacity and AC online status using INA219 readings
- Integrates with `/sys/class/power_supply` and UPower
- Provides a `/dev/fake_battery` interface to update battery state
- Automatically loads on boot via systemd and `/etc/modules-load.d`

---

## Known Issues
- Sometimes it will drop down in percentage and go back up based on python calculation
- Sometimes it will repeat the 20% battery health warnings because of above issue
- When the Waveshare 3s UPS is plugged in the battery percentage shows higher than what the battery is currently charged to

---

### 🔌 UPS Module 3S Pinout & Raspberry Pi 5 GPIO Mapping
#### Based on instructions on: https://www.waveshare.com/wiki/UPS_Module_3S
Ensure that you have i2c turned on in the config.txt/raspi-config per the link above.


| UPS Module Pin | Function           | Raspberry Pi 5 GPIO Pin | Notes                                                                 |
|----------------|--------------------|--------------------------|-----------------------------------------------------------------------|
| 5V             | 5V Output          | Pin 2 or Pin 4           | Provides regulated 5V power output. Can power the Pi via GPIO.        |
| 3.3V           | 3.3V Output        | Pin 1 or Pin 17          | Provides regulated 3.3V power output.                                 |
| SDA            | I2C Data           | Pin 3 (GPIO 2)           | I2C data line for communication.                                     |
| SCL            | I2C Clock          | Pin 5 (GPIO 3)           | I2C clock line for communication.                                    |
| GND            | Ground             | Pin 6, 9, 14, 20, etc.   | Common ground connection. Multiple GND pins are available on the Pi. |



## 1. Clone the Repository
```bash
git clone https://github.com/davidmattson1/fake-battery-waveshare-3s-ups-rpi.git
cd fake-battery-waveshare-3s-ups-rpi
```

---

## 2. Install Kernel Headers and Build Tools
Update and install dependencies:
```bash
sudo apt update
sudo apt install -y build-essential python3 python3-pip
```
Then install headers for your current kernel:
```bash
sudo apt install linux-headers-$(uname -r)
```
If the above fails, find your header version with:
```bash
apt search linux-headers | grep $(uname -r | cut -d'-' -f1-2)
```

---

## 3. Build and Install the Kernel Module
```bash
make
sudo mkdir -p /lib/modules/$(uname -r)/extra
sudo cp fake_battery.ko /lib/modules/$(uname -r)/extra/
sudo depmod
sudo modprobe fake_battery
echo fake_battery | sudo tee /etc/modules-load.d/fake_battery.conf
```
This installs the `fake_battery` kernel module globally and ensures it loads at boot.

---

## 4. Install and Configure the INA219 Monitoring Script
Copy the INA219 script:
```bash
sudo cp I2CSCRIPT/ina219.py /usr/local/bin/ina219.py
sudo chmod +x /usr/local/bin/ina219.py
```

---

## 5. Create the `ups-module` Systemd Service
Create the service definition:
```bash
echo "[Unit]
Description=Waveshare UPS Monitor
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/ina219.py
Restart=always

[Install]
WantedBy=multi-user.target" | sudo tee /etc/systemd/system/ups-module.service
```
Enable and start the service:
```bash
sudo systemctl enable --now ups-module.service
```

---

## 6. Reboot and Verify
```bash
sudo reboot
```
After reboot, confirm functionality:

It will take a moment to start the systemd service and may show 100% initially
```bash
cat /sys/class/power_supply/BAT0/*
```
You should see battery state and charging status update based on load conditions.

---




## 🛠️ Troubleshooting

### 🔎 Service Not Updating Battery Status?

If your battery always shows 100% or the system doesn’t detect `/sys/class/power_supply/BAT0`, try the following:

---

### 1. Run the Monitoring Script Manually

This is a great way to see real-time output and catch Python errors:

```bash
sudo python3 /usr/local/bin/ina219.py
```

If the script is working, you should see debug output or at least no errors. Use `Ctrl+C` to exit.

---

### 2. Check If the Fake Battery Module Is Loaded

```bash
ls /sys/class/power_supply/
```

Expected output:

```text
BAT0  AC0
```

If `BAT0` is missing, reload the kernel module:

```bash
sudo modprobe fake_battery
```

---

### 3. Check Service Logs

View logs for the INA219 monitor service:

```bash
journalctl -u ups-module.service
```

Look for any Python exceptions or startup failures.

---

### 4. Check I2C Communication

Ensure I2C is enabled:

```bash
grep -i i2c /boot/firmware/config.txt
```

You should see:

```text
dtparam=i2c_arm=on
```

Scan for I2C devices (you should see `0x40` or `0x42`):

```bash
sudo i2cdetect -y 1
```

---

### 5. Watch the Battery Data in Real Time

```bash
watch -n 2 cat /sys/class/power_supply/BAT0/capacity
```

Or view everything in one go:

```bash
for f in /sys/class/power_supply/BAT0/*; do echo -n "$(basename "$f"): "; cat "$f"; done
```

---

Still stuck? Try running the script manually with debug logging, or check for multiple BAT devices being registered.

## Credits
- Kernel module adapted from [linux-fake-battery-module](https://github.com/hoelzro/linux-fake-battery-module) by @hoelzro.
- Customized for Waveshare 3S UPS and Raspberry Pi 5 by [@davidmattson1](https://github.com/davidmattson1).
