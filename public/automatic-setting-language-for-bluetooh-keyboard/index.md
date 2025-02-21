---
title: 'Zipando arquivos com java'
date: '2021-04-13'
spoiler: "Aprenda a zippar multiplos arquivos e pastas utilizando java."
---

# Tutorial: How to Automatically Switch Keyboard Layout When Your Bluetooth Device Connects

This guide will configure a `udev` rule to trigger a script that changes the keyboard layout when a specific Bluetooth device is connected.

## Step 1: Get the MAC Address of the Bluetooth Device

To get the MAC address of your Bluetooth device:

1. **List paired devices**:

   ```bash
   bluetoothctl paired-devices
   ```

   The output will display all paired Bluetooth devices with their MAC addresses in the following format:

   ```plaintext
   Device XX:XX:XX:XX:XX:XX Device_Name
   ```

   Note the MAC address of the device you want to use.

## Step 2: Create the `set_keyboard_layout.sh` Script

Create a script named `set_keyboard_layout.sh` in `/usr/local/bin/`. This script will detect the Bluetooth device and adjust the keyboard layout.

1. **Create and open the script**:

   ```bash
   sudo nano /usr/local/bin/set_keyboard_layout.sh
   ```

2. **Script content**:

   ```bash
   #!/bin/bash

   # Bluetooth keyboard MAC address
   KEYBOARD_MAC="DE:B9:A2:85:8D:B4"

   # Log file name with timestamp
   LOG_FILE="/tmp/keyboard_layout_$(date +'%Y-%m-%d_%H-%M-%S-%3N').log"

   # Log function with timestamp
   log_message() {
       echo "[$(date +'%Y-%m-%d %H:%M:%S.%3N')] $1" >> "$LOG_FILE"
   }

   # Start logging
   log_message "Starting keyboard layout configuration script"

   # Enable access to X server
   log_message "Enabling access to X server..."
   xhost + >> "$LOG_FILE" 2>&1
   log_message "Access to X server enabled."

   # Set DISPLAY variable
   export DISPLAY=:1
   log_message "DISPLAY set to $DISPLAY"

   # Get device name by MAC
   log_message "Getting device name for MAC: $KEYBOARD_MAC"
   DEVICE_NAME=$(bluetoothctl info "$KEYBOARD_MAC" | grep "Alias" | awk -F ': ' '{print $2}')

   if [ -z "$DEVICE_NAME" ]; then
       log_message "Bluetooth device not found or disconnected. Exiting."
       exit 0
   else
       log_message "Device found: $DEVICE_NAME"
   fi

   # Attempt to get device ID up to 10 times
   for i in {1..10}; do
       log_message "Attempt $i to retrieve device ID for name: $DEVICE_NAME"
       DEVICE_ID=$(xinput list | grep -i "$DEVICE_NAME" | grep -o 'id=[0-9]*' | grep -o '[0-9]*')
       
       if [ -n "$DEVICE_ID" ]; then
           log_message "Device ID obtained: $DEVICE_ID"
           break
       else
           log_message "Device ID not found. Retrying in 1 second..."
           sleep 1
       fi
   done

   # Check if device ID was found after 10 attempts
   if [ -z "$DEVICE_ID" ]; then
       log_message "Device ID not found after 10 attempts. Exiting."
       exit 0
   fi

   # Check if keyboard is connected and set layout
   {
       log_message "Checking keyboard connection..."
       if bluetoothctl info "$KEYBOARD_MAC" | grep -q "Connected: yes"; then
           log_message "Keyboard connected. Setting layout to English."
           setxkbmap -device "$DEVICE_ID" us && log_message "Layout set to English successfully." || log_message "Error setting layout to English."
       else
           log_message "Keyboard disconnected. Setting default layout to Portuguese (BR)."
           setxkbmap -device "$DEVICE_ID" br && log_message "Layout set to Portuguese successfully." || log_message "Error setting layout to Portuguese."
       fi
   } >> "$LOG_FILE" 2>&1

   log_message "Script finished"
   ```

3. **Make the script executable**:

   ```bash
   sudo chmod +x /usr/local/bin/set_keyboard_layout.sh
   ```

## Step 3: Create the `udev` Rule

Configure a `udev` rule to run the script when a Bluetooth device is added.

1. **Create the rule file**:

   ```bash
   sudo nano /etc/udev/rules.d/99-keyboard-layout.rules
   ```

2. **Add the `udev` rule**:

   ```plaintext
   ACTION=="add", SUBSYSTEM=="bluetooth", RUN+="/usr/local/bin/set_keyboard_layout.sh"
   ```

3. **Reload `udev` rules**:

   ```bash
   sudo udevadm control --reload
   ```

4. **Test the rule**

   Connect the Bluetooth device and check the log file at `/tmp/keyboard_layout_YYYY-MM-DD_HH-MM-SS-SSS.log` to confirm the script was executed.

---

This setup will automatically change the keyboard layout when the specified Bluetooth device connects, with events logged in detail.