First ensure you have an ISO USB boot of the Ubuntu 20.04 distro

Turn off the CF-C2 and insert your USB Boot disk

Next boot into the Panasonic BIOS and turn off UEFI  boot (from boot tab)

Navigate to exist tab and choose the USB Device in the options to escape from boot

The machine will restart and should boot from the USB

The Ubuntu setup should be straightforward and most things, including wifi, wwan,bluetooth, screen rotation, touch, should be setup as part of the install.

You will need to manually deal with the following:

Sound not working:

Fire up Terminal (Cntrl-Alt-T)
enter the following commands:

```
sudo apt update
sudo apt upgrade
sudo pico /usr/share/pulseaudio/alsa-mixer/paths/analog-output-speaker.conf
```

In the analog-output-speaker.conf file:

Change the [Element Headphone] section to:

```
switch=off
volume=merge
override-map.1 = all
override-map.2 = all-left,all-right
```

change the [Element Speaker] section to:

```
required-any = any
switch = mute
volume = off
```

Then save and exit terminal and reboot. Post reboot sound should work.

Next lets fix the brightness shortcuts which are not working:

Fire up the terminal and create a bash script called redirect-brightness in /usr/local/bin

```
sudo pico /usr/local/bin/redirect-brightness
```

copy and paste the below into the pico editor and save:

```
#!/bin/bash


# NAME: redirect-brightness
# PATH: /usr/local/bin
# DESC: Redirect to correct driver when Ubuntu is adjusting the wrong
#       /sys/class/DRIVER_NAME/brightness

# DATE: June 13, 2018. Modified June 14, 2018.

# NOTE: Written for Ubuntu question:
#       https://askubuntu.com/q/1045624/307523

WatchDriver="/sys/class/backlight/panasonic"
PatchDriver="/sys/class/backlight/intel_backlight"

# Must be running as sudo
if [[ $(id -u) != 0 ]]; then
    echo >&2 "Root access required. Use: 'sudo redirect-brightness'"
    exit 1
fi

# inotifywait required
type inotifywait >/dev/null 2>&1 || \
    { echo >&2 "'inotifywait' required but it's not installed.  Aborting."; \
      echo >&2 "Use 'sudo apt install inotify-tools' to install it.'"; \
      exit 1; }

# Was right watch driver directory name setup correctly?
if [[ ! -d $WatchDriver ]]; then
    echo >&2 "Watch directory: '$WatchDriver'"; \
    echo >&2 "does not exist. Did you spell it correctly? Aborting.'"; \
    exit 1;
fi

# Was right patch driver directory name setup correctly?
if [[ ! -d $PatchDriver ]]; then
    echo >&2 "Redirect to directory: '$PatchDriver'"; \
    echo >&2 "does not exist. Did you spell it correctly? Aborting.'"; \
    exit 1;
fi

# Get maximum brightness values
WatchMax=$(cat $WatchDriver/max_brightness)
PatchMax=$(cat $PatchDriver/max_brightness)

# PARM: 1="-l" or "--log-file" then write each step to log file.
fLogFile=false
if [[ $1 == "-l" ]] || [[ $1 == "--log-file" ]]; then
    fLogFile=true
    LogFile=/tmp/redirect-brightness.log
    echo redirect-brightness LOG FILE > $LogFile
    echo WatchMax: $WatchMax PatchMax: $PatchMax >> $LogFile
fi

SetBrightness () {
    # Calculate watch current percentage
    WatchAct=$(cat $WatchDriver/actual_brightness)
    WatchPer=$(( WatchAct * 100 / WatchMax ))
    [[ $fLogFile == true ]] && echo WatchAct: $WatchAct WatchPer: $WatchPer >> $LogFile
    # Reverse engineer patch brightness to set
    PatchAct=$(( PatchMax * WatchPer / 100 ))
    echo $PatchAct | sudo tee $PatchDriver/brightness
    [[ $fLogFile == true ]] && echo PatchAct: $PatchAct >> $LogFile
}

# When machine boots, set brightness to last saved value
SetBrightness

# Wait forever for user to press Fn keys adjusting brightness up/down.
while (true); do
    inotifywait --event modify $WatchDriver/actual_brightness
    [[ $fLogFile == true ]] && \
        echo "Processing modify event in $WatchDriver/actual_brightness" >> $LogFile
    SetBrightness
done
```

Now make it executable:

```
chmod a+x /usr/local/bin/redirect-brightness
```

now run the script from the command line using

```
redirect-brightness -l
```

You should find the Fn-F1 and Fn-F2 controls work and brightness can be turned up and turned down

End the test with Cntrl-C

Now lets create a systemd service to start it after boot:

```
 pico /etc/systemd/system/redirect-brightness.service
 ```
Add the following and save
```
[Unit]
Description=Redirect Backlight Brightness from one driver to another

[Service]
ExecStart=/usr/local/bin/redirect-brightness
1
[Install]
WantedBy=basic.

```

Install, enable and start the service:

```
sudo chmod 664 /etc/systemd/system/redirect-brightness.service
sudo systemctl daemon-reload
sudo systemctl enable redirect-brightness.service
sudo systemctl start redirect-brightness.service
```

All done !


