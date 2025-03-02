🚀 Full Guide: Integrating Steam Deck Battery, Storage, and Installed Games into Home Assistant



This guide will walk you through integrating Steam Deck battery, charging status, installed games, and available storage into Home Assistant.



🔹 Prerequisites



Before starting, ensure you have:

✅ Home Assistant

✅ SSH access to your Steam Deck

✅ A Home Assistant Long-Lived Access Token (from Profile → Create Token)

✅ A Home Assistant entity called sensor.steam_deck_battery



🔹 Step 1: Generate a Home Assistant API Token



Your Steam Deck needs an API key to communicate with Home Assistant.



📌 How to Get Your API Token



1️⃣ Log in to Home Assistant via your web browser.

2️⃣ Click your Profile Picture (bottom-left corner).

3️⃣ Scroll to Long-Lived Access Tokens → Click "Create Token"

4️⃣ Name it (e.g., "Steam Deck Integration")

5️⃣ Copy and save the token (Home Assistant will NOT show it again).



🔹 Step 2: Prepare Your Steam Deck



Since Steam Deck runs SteamOS, we need to:

1️⃣ Disable the read-only filesystem

2️⃣ Install necessary tools

3️⃣ Create a script to send data to Home Assistant

4️⃣ Set up an automatic systemd service



📌 Step 2.1: Disable Steam Deck's Read-Only Filesystem

Run the following command:

sudo steamos-readonly disable

📌 Step 2.2: Install Required Packages

Run:

sudo pacman -S --noconfirm curl

📌 Step 2.3: Create a Battery & Storage Monitoring Script

Create the script file:

nano ~/battery_rest.sh



#!/bin/bash



# Fetch Battery Level & Charging Status

BATTERY_LEVEL=$(cat /sys/class/power_supply/BAT1/capacity)

CHARGING_STATUS=$(cat /sys/class/power_supply/BAT1/status)  # "Charging" or "Discharging"

GAMES_INSTALLED=$(ls -1 ~/Steam/steamapps/common/ | wc -l)

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

curl -X POST "$HA_URL"      -H "Authorization: $HA_TOKEN"      -H "Content-Type: application/json"      -d "{

          "state": "$BATTERY_LEVEL",

          "attributes": {

            "unit_of_measurement": "%",

            "device_class": "battery",

            "friendly_name": "Steam Deck Battery",

            "charging": "$CHARGING",

            "installed_games": "$GAMES_INSTALLED",

            "available_storage": "$DISK_SPACE"

          }

        }"



🔹 Step 3: Automate with Systemd

📌 Step 3.1: Create a Systemd Service

Create the service file:

nano ~/.config/systemd/user/battery_update.service



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



📌 Step 3.2: Create a Systemd Timer

Create the timer file:

nano ~/.config/systemd/user/battery_update.timer



[Unit]

Description=Run Steam Deck Battery Update Every Minute



[Timer]

OnBootSec=1min

OnUnitActiveSec=1min

Unit=battery_update.service



[Install]

WantedBy=timers.target



📌 Step 3.3: Enable and Start the Services

Run the following commands:



systemctl --user daemon-reload

systemctl --user enable --now battery_update.service

systemctl --user enable --now battery_update.timer

systemctl --user status battery_update.service

systemctl --user list-timers --all



🔹 Step 4: Verify and Display in Home Assistant

Go to Developer Tools → States and search for `sensor.steam_deck_battery`.

📌 Step 4.2: Add to Your Dashboard



✅ Entities Card:

type: entities

entities:

  - entity: sensor.steam_deck_battery



✅ Tile Card:

type: tile

entity: sensor.steam_deck_battery



If you have issues you can try updating the Home Assistant yaml by adding this near the bottom



📌 Optional: Update configuration.yaml



If needed, you can manually define the battery sensor using a template:



Sensor:

-	Platform: template

    Sensors:

      Steam_deck_battery:

        Friendly_name: “Steam Deck Battery”

        Value_template: “{{ states(‘sensor.steam_deck_battery’) }}”

        Unit_of_measurement: “%”

        Device_class: battery

        Attributes:

          Charging: “{{ state_attr(‘sensor.steam_deck_battery’, ‘charging’) }}”

          Installed_games: “{{ state_attr(‘sensor.steam_deck_battery’, ‘installed_games’) }}”

          Available_storage: “{{ state_attr(‘sensor.steam_deck_battery’, ‘available_storage’) }}”





---

🚀 Final Steps

✔ Steam Deck sends battery, charging, storage, and game updates every minute

✔ Home Assistant dynamically updates the battery icon

✔ Installed games & available storage are also tracked
