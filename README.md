# home-assistant-steamdeck-mqtt
Create Battery Entity for Steam Deck in Home Assistant


Prerequisites for This Guide

This guide is specifically for Home Assistant Core (the version installed manually on Linux, not the Home Assistant OS or Supervised versions). Before proceeding, ensure you have the following:

✅ Home Assistant Core Installed – Running on a Linux server or Raspberry Pi.
✅ SSH Access to Home Assistant & Steam Deck – Required for command-line setup.
✅ Basic Linux Knowledge – Comfort with editing configuration files and using nano.
✅ MQTT Broker (Mosquitto) – Installed on the Home Assistant server.
✅ Steam Deck in Desktop Mode – Used for configuring battery monitoring. Battery monitoring works in gaming mode as well. 

I dont believe the updates will continue pushing to home assistant while the device is asleep. 

If you're using Home Assistant OS (Hass.io) or Supervised, the MQTT setup might differ slightly, as the add-on store provides an easier way to install Mosquitto.



🔹 Step 1: Install and Configure Mosquitto MQTT Broker on Home Assistant Server

The MQTT broker is the central hub where the Steam Deck will send battery data, and Home Assistant will read it.

1️⃣ Install Mosquitto on the Home Assistant Server via SSH

Run:

sudo apt update && sudo apt install mosquitto mosquitto-clients -y

Enable and start the service:

sudo systemctl enable mosquitto
sudo systemctl start mosquitto

2️⃣ Set Up an MQTT User (Optional)

If you want authentication:

sudo mosquitto_passwd -c /etc/mosquitto/passwd mqttuser

Restart Mosquitto:

sudo systemctl restart mosquitto

3️⃣ Configure Mosquitto

Edit the Mosquitto config file:

sudo nano /etc/mosquitto/mosquitto.conf

Ensure it includes:

listener 1883
password_file /etc/mosquitto/passwd
allow_anonymous false

Restart Mosquitto:

sudo systemctl restart mosquitto


---

🔹 Step 2: Set Up Home Assistant to Use MQTT

1️⃣ Install the MQTT Integration

1. Open Home Assistant.


2. Go to Settings → Devices & Services → Add Integration.


3. Search for MQTT and select it.


4. Configure:

Broker: <Home Assistant Server IP>

Port: 1883

Username: mqttuser

Password: mqttpassword



5. Click Submit.



📖 Setting Up MQTT on the Steam Deck for Home Assistant Integration

This guide explains how to install Mosquitto, send battery data to Home Assistant, and automate updates using cron on the Steam Deck. It also includes steps to unlock and re-lock the Steam Deck’s filesystem.


---

🔹 Step 1: Unlock the Steam Deck Filesystem

By default, the Steam Deck’s filesystem is read-only, preventing software installations. To allow installations, you must temporarily disable read-only mode:

sudo steamos-readonly disable

This allows you to install necessary packages like Mosquitto and Cron.

You can either ssh into your deck or do the next piece directly in the terminal in desktop mode. 

---

🔹 Step 2: Install Mosquitto and Cron

Now that the filesystem is writable, install Mosquitto (for sending MQTT messages) and Cron (to automate updates).

1️⃣ Open a terminal in Desktop Mode.
2️⃣ Install Mosquitto (MQTT client tools):

sudo pacman -S mosquitto

3️⃣ Install Cron (cronie) to schedule automatic updates:

sudo pacman -S cronie

4️⃣ Enable and start the cron service:

sudo systemctl enable cronie
sudo systemctl start cronie

5️⃣ Verify installation:

mosquitto_pub --help
systemctl status cronie

✅ If "active (running)" appears for Cron, it’s working.

❌ If Cron is not running, restart it:

sudo systemctl restart cronie



---

🔹 Step 3: Create a Script to Send Battery Data

The Steam Deck stores battery information in /sys/class/power_supply/. This script will read the battery percentage and send it to Home Assistant via MQTT.

1️⃣ Find the battery device name:

ls /sys/class/power_supply/

Expected output:

ACAD  BAT1

If you see BAT1, the battery percentage is stored at:

/sys/class/power_supply/BAT1/capacity

2️⃣ Create the battery monitoring script:

nano ~/battery_mqtt.sh

3️⃣ Add the following content (replace <Home Assistant IP>, mqttuser, and mqttpassword):

#!/bin/bash
BATTERY_LEVEL=$(cat /sys/class/power_supply/BAT1/capacity)
mosquitto_pub -h <Home Assistant IP> -t "steamdeck/battery" -m "$BATTERY_LEVEL" -u "mqttuser" -P "mqttpassword"

4️⃣ Save and exit (CTRL + X → Y → ENTER).

5️⃣ Make the script executable:

chmod +x ~/battery_mqtt.sh

6️⃣ Manually test the script:

~/battery_mqtt.sh

If no errors appear, check Settings → Devices & Services → MQTT → Listen to a Topic in Home Assistant to confirm the message was received.



---

🔹 Step 4: Automate Battery Updates with Cron

Now, set up cron to run the script automatically every minute.

1️⃣ Edit the crontab file:

crontab -e

2️⃣ Add this line at the bottom (this runs the script every minute):

* * * * * ~/battery_mqtt.sh

3️⃣ Save and exit (CTRL + X → Y → ENTER).

4️⃣ Verify that cron saved the job:

crontab -l

Expected output:

* * * * * ~/battery_mqtt.sh

5️⃣ Check if cron is running:

systemctl status cronie

✅ If "active (running)" appears, cron is working.

❌ If it's not running, restart it:

sudo systemctl restart cronie



---

🔹 Step 5: Re-Lock the Steam Deck Filesystem

Once cron is installed and running, re-enable the read-only mode to protect the system:

sudo steamos-readonly enable


---
