# Disable UPS beeper (USB) in Synology NAS
Small script to disable / enable the beeper sound from a USB connected UPS in Synology DiskStation Manager.

Based on [this post](https://moshib.in/2019/02/08/disable-ups-beeper-synology.html) from [Moshi's Blog](https://moshib.in).

Some time ago the power went off at 4am for a long time and I had to get up and shutdown the UPS in order to stop the beeping. The abovementioned post describes how to disable it permanently. Still, I did not want it to stop beeping during day time. The next steps and scripts are my solution to disable and enable the beeping during specific parts of the day (in my case between 00h and 09h).

**Important:** DSM updates seem to remove the scripts under /root/ and rollback the upsd.users file to its original state. If you update DSM you have to repeat the process.

## Intro
In most Linux distros there is a set of tools to manage an UPS. Namely:
* `upsc` - the ups client, which can be used to query details about the ups
* `upsd` - the service / daemon
* `upscmd` - command line tool used to send commands and change settings

### Some examples:
List available UPSes:
```shell
user@nas:/$ upsc -l
ups
```

List all variables and values from a specific ups:
```shell
panda@calvin:/$ upsc ups
battery.charge: 100
battery.type: PbAc
device.mfr: EATON
(...)
ups.beeper.status: enabled
(...)
```
View a specific field/variable:
```shell
panda@calvin:/$ upsc ups ups.beeper.status
enabled
```

Both `upsc` and `upscmd` communicate with the `upsd` using a client/server model (TCP port 3493). Unfortunately the upsmcd is not available under Synology DiskStation Manager (DSM) OS. Still, we can connect to the `upsd` using telnet and emulate the commands `upscmd` would send. The protocol is described [here](https://networkupstools.org/docs/developer-guide.chunked/ar01s09.html). In our specific case we just want to perform 3 actions (4 messages):
1. Login
    - `USERNAME <user>`
    - `PWD <pwd>`
2. Enable or disable the beeper
    - `INSTCMD <upsname> <command>`
3. Log out
    - `LOGOUT`


## Howto:

### 1. SSH into the NAS
Activate SSHd under the synology control panel if you haven't done so and SSH into it. Note: under windows you might need a decent console with ssh (git bash?) or to use PuTTY.
```shell
ssh <username>@<nas_ip> -p <ssh_port>
```
### 2. Find user configuration file
The upsd.users file could be located in two places, find the needed one:
```shell
user@nas:/$ find /usr/syno/etc/ups/ /etc/ups/ -name "upsd.users"
(result /path/to/upsd.users)
```

### 3. Add a new user to upsd
Edit the upsd.users file (depending on file location according to the previous step) and add a new user account with permissions to change the beeper status

For file inside _/usr/syno/etc/ups_
```shell
user@nas:/$ sudo vim /usr/syno/etc/ups/upsd.users
Password: <insert your pwd>
```
For file inside _/etc/ups_
```shell
user@nas:/$ sudo vim /etc/ups/upsd.users
Password: <insert your pwd>
```

Be careful with the VIM editor! In case you are not familiar with it:
* Move the cursor down <kbd>&#8595;</kbd> until you find the place where you want to add the new lines
* Press <kbd>I</kbd> to enter the INSERT MODE
* Edit the file as needed
* Press the <kbd>Esc</kbd> Key to leave the edit mode
* Press <kbd>:</kbd> to issue a command
* Write "wq" (write an quit) and hit <kbd>Enter</kbd>

So, edit the upsd.users file and add a new user with privileges to enable/disable the beeper (replace `<upsd_username>` and `<upsd_pwd>` with the desired values):
```shell
    [<upsd_username>]
        password = <upsd_pwd>
        actions = SET
        instcmds = beeper.enable beeper.disable ups.beeper.status
```

### 4. Restart the upsd service
```shell
sudo synosystemctl restart ups-usb
(wait a few seconds)
```

### 5. Create a python script to issue commands
Clone the repository, move the `ups_beeper_control.sh` and `upscmd.py` files to your user's home folder,
Open `upscmd.py` with vim to configure the username(`user`) and password(`pwd`) in lines 5 and 6:
```shell
vim upscmd.py
```
Move this file and `ups_beeper_control.sh` to the right place, e.g.:
```shell
sudo mv ups_beeper_control.sh upscmd.py /root/
```
**Note**: `ups_beeper_control.sh` is used to call the `upscmd.py` script with "beeper.toggle", you could just execute `sudo python2 /root/upscmd.py beeper.toggle`.

### 6. Make the scripts executable
```shell
sudo chmod u+x /root/ups_beeper_control.sh
sudo chmod u+x /root/upscmd.py
```

At this point you can test the scripts:
```shell
user@nas:/$ sudo /root/ups_beeper_control.sh disable
Password:
disable beeper...
USERNAME cmd status: OK
PASSWORD cmd status: OK
INSTCMD cmd status: OK

OK Goodbye

Waiting 5 seconds for UPS to update state...
Beeper disabled.

user@nas:/$ sudo /root/ups_beeper_control.sh curtime
Using current time to set UPS beeper status
enable beeper...
USERNAME cmd status: OK
PASSWORD cmd status: OK
INSTCMD cmd status: OK

OK Goodbye

Waiting 5 seconds for UPS to update state...
Beeper enabled.
```

### 7. Schedule it
Go to the DSM Web interface (Control panel -> Task scheduler) and add the 3 tasks:
1. Scheduled task to enable beeper (daily)
    - user: root
    - shedule: daily at 9am
    - command: `bash /root/ups_beeper_control.sh enable`
    - send email when the script terminates abnormally
2. Triggered task to disable beeper on boot (similar to above)
    - user: root
    - command: `bash /root/ups_beeper_control.sh disable`
3. Triggered task to enable / disable on boot based on the current time
    - command: `bash /root/ups_beeper_control.sh curtime`
4. Etcetera...
