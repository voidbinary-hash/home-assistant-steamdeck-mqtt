🚀 Integrating Steam Deck Battery, Storage, and Installed Games into Home Assistant

This guide explains how to integrate Steam Deck battery, charging status, installed games, and available storage into Home Assistant using the REST API.

🔹 Features

✔ Live battery percentage with a dynamic charging icon
✔ Detects charging or discharging status
✔ Tracks the number of installed Steam games
✔ Monitors available storage space
✔ Automated updates every minute using systemd


---

📌 Prerequisites

Before starting, ensure you have:

✅ Home Assistant Core (not Supervised or OS version)

✅ SSH access to your Steam Deck

✅ A Home Assistant Long-Lived Access Token (from Profile → Create Token)

✅ A Home Assistant entity called sensor.steam_deck_battery



---

🔹 Step 1: Generate a Home Assistant API Token

Your Steam Deck needs an API key to communicate with Home Assistant.

🔹 How to Get Your API Token

1️⃣ Log in to Home Assistant via your web browser.
2️⃣ Click your Profile Picture (bottom-left corner).
3️⃣ Scroll to Long-Lived Access Tokens → Click "Create Token"
4️⃣ Name it (e.g., "Steam Deck Integration")
5️⃣ Copy and save the token (Home Assistant will NOT show it again).

✅ You will need this API token in later steps.


---

🔹 Step 2: Prepare Your Steam Deck

Since Steam Deck runs SteamOS, we need to: 1️⃣ Disable the read-only filesystem
2️⃣ Install necessary tools
3️⃣ Create a script to send data to Home Assistant
4️⃣ Set up an automatic systemd service


---

📌 Step 2.1: Disable Steam Deck's Read-Only Filesystem

sudo steamos-readonly disable

This allows us to install software and create persistent scripts.


---

📌 Step 2.2: Install Required Packages

sudo pacman -S --noconfirm curl

This ensures we can send HTTP requests to Home Assistant.


---

📌 Step 2.3: Create a Battery & Storage Monitoring Script

1️⃣ Create the script file:

nano ~/battery_rest.sh

2️⃣ Paste this script inside:

#!/bin/bash

# Fetch Battery Level & Charging Status
BATTERY_LEVEL=$(cat /sys/class/power_supply/BAT1/capacity)
CHARGING_STATUS=$(cat /sys/class/power_supply/BAT1/status)  # "Charging" or "Discharging"

# Get Number of Installed Steam Games
GAMES_INSTALLED=$(ls -1 ~/Steam/steamapps/common/ | wc -l)

# Get Available Disk Space (in GB)
DISK_SPACE=$(df -h ~ | awk 'NR==2 {print $4}')

# Home Assistant API Information
HA_URL="http://YOUR_HA_IP:8123/api/states/sensor.steam_deck_battery"
HA_TOKEN="Bearer YOUR_LONG_LIVED_ACCESS_TOKEN"

# Convert Charging Status to Boolean
if [ "$CHARGING_STATUS" == "Charging" ]; then
    CHARGING="true"
else
    CHARGING="false"
fi

# Send Data to Home Assistant
curl -X POST "$HA_URL" \
     -H "Authorization: $HA_TOKEN" \
     -H "Content-Type: application/json" \
     -d "{
          \"state\": \"$BATTERY_LEVEL\",
          \"attributes\": {
            \"unit_of_measurement\": \"%\",
            \"device_class\": \"battery\",
            \"friendly_name\": \"Steam Deck Battery\",
            \"charging\": \"$CHARGING\",
            \"installed_games\": \"$GAMES_INSTALLED\",
            \"available_storage\": \"$DISK_SPACE\"
          }
        }"

3️⃣ Save and exit (CTRL + X → Y → ENTER)
4️⃣ Make the script executable:

chmod +x ~/battery_rest.sh

5️⃣ Test it manually:

~/battery_rest.sh

✅ If it works, you should see battery data in Home Assistant → Developer Tools → States.



---

🔹 Step 3: Automate with Systemd

To ensure the script runs every minute, we’ll use a systemd service and timer.

📌 Step 3.1: Create a Systemd Service

1️⃣ Create the service file:

mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/battery_update.service

2️⃣ Paste this inside:

[Unit]
Description=Send Steam Deck Battery Data to Home Assistant
After=network-online.target

[Service]
ExecStart=/bin/bash /home/deck/battery_rest.sh
Restart=on-failure
RestartSec=10s
StandardOutput=journal
StandardError=journal
NoNewPrivileges=true

[Install]
WantedBy=default.target

3️⃣ Save and exit (CTRL + X → Y → ENTER).


---

📌 Step 3.2: Create a Systemd Timer

1️⃣ Create the timer file:

nano ~/.config/systemd/user/battery_update.timer

2️⃣ Paste this inside:

[Unit]
Description=Run Steam Deck Battery Update Every Minute

[Timer]
OnBootSec=1min
OnUnitActiveSec=1min
Unit=battery_update.service

[Install]
WantedBy=timers.target

3️⃣ Save and exit (CTRL + X → Y → ENTER).


---

📌 Step 3.3: Enable and Start the Services

1️⃣ Reload systemd:

systemctl --user daemon-reload

2️⃣ Enable and start the service:

systemctl --user enable --now battery_update.service

3️⃣ Enable and start the timer:

systemctl --user enable --now battery_update.timer

4️⃣ Check if it’s working:

systemctl --user status battery_update.service
systemctl --user list-timers --all

✅ Now, the Steam Deck automatically sends battery data to Home Assistant every minute!


---

🔹 Step 4: Verify and Display in Home Assistant

📌 Step 4.1: Check in Developer Tools

1️⃣ Go to Home Assistant → Developer Tools → States
2️⃣ Search for sensor.steam_deck_battery
3️⃣ Ensure the attributes look correct:

unit_of_measurement: "%"
device_class: battery
friendly_name: Steam Deck Battery
charging: "true" or "false"
installed_games: "60"
available_storage: "269G"


---

📌 Step 4.2: Add to Your Dashboard

✅ Entities Card

type: entities
entities:
  - entity: sensor.steam_deck_battery

✅ Tile Card

type: tile
entity: sensor.steam_deck_battery

✅ Now, the Steam Deck battery icon dynamically updates in Home Assistant!


---

🚀 Final Steps

✔ Steam Deck sends battery, charging, storage, and game updates every minute
✔ Home Assistant dynamically updates the battery icon
✔ Installed games & available storage are also tracked

Now your Steam Deck behaves just like any other Home Assistant battery entity! 🚀🔥


---

📌 Contribute & Feedback

Have any suggestions, improvements, or issues? Feel free to open an issue or submit a pull request!

Let me know if you need any formatting changes before posting to GitHub! 🚀
