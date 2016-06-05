DIY Audio Network Player
------------------------

This document contains a list of instructions I followed to configure my ANP.

Hardware:
* [Raspberry Pi 3](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/)
* [HiFiBerry Dac+ pro](https://www.hifiberry.com/dacplus)
* [Edimax EW-7811UTC / AC600](http://www.edimax.com/edimax/merchandise/merchandise_detail/data/edimax/global/wireless_adapters_ac600_dual-band/ew-7811utc)
* [Flirc USB IR receiver](https://flirc.tv/more/flirc-usb)
* [7" IPS Display](https://www.adafruit.com/products/1931)
* Custom build linear power supply with 4 outputs: 3x 5v 2A, 1x 9v 2a
* An aluminium chassis with some CNC work [www.modushop.pl](http://www.modushop.pl/772,galaxy-gx288-3u-panel-3mm-czarny-1gx288n-3u-.html)

Software:
* [Raspbian Jessie Lite](https://www.raspberrypi.org/downloads/raspbian/)
* [MPD](https://www.musicpd.org/)
* [ncmpcpp](http://rybczak.net/ncmpcpp/)
* [C.A.V.A](http://karlstav.github.io/cava/)
* [PiFi-Remote](https://github.com/kowalcj0/PiFi-Remote) a modified subset version of [pi.hifi](https://bitbucket.org/bertrandboichon/pi.hifi) by Bertrand Boichon

btw. I've successfully used [volumio](https://volumio.org/) for a good while,
but I've decided to build something similar but with some extras.



# Flash Raspbian Jessie lie

```
sudo ddrescue -D --force 2016-05-10-raspbian-jessie-lite.img /dev/mmcblk0
```


# Expand filesystem size to the max size of your card

```
sudo raspi-config
```
Choose option `1 Expand Filesystem`


# Enable HiFiBerry DAC+ Pro

[Source](https://www.hifiberry.com/guides/configuring-the-sound-card-in-openelec-with-device-tree-overlays/)


## /boot/config.txt
If you don't want to use your SD card reader then you will have to remount the
`/boot` or `/flash` partition to `rw` mode.
```
mount -o remount,rw /boot
```
or on some distros
```
mount -o remount,rw /flash
```

And then append this to: `/boot/config.txt` or `/flash/config.txt`

```
## enable DAC+ standard/pro
dtoverlay=hifiberry-dacplus
dtdebug=1
```

If you don't want to remount that partion then simply use your SD card reader
on your PC to edit that `config.txt` file


## /etc/asound.conf
The next step is to enable the DAC for ALSA.
Create `/etc/asound.conf` with this code:
```
pcm.!default {
type hw card 0
}
ctl.!default {
type hw card 0
}
```

## Disable on-board audio
Lastly, you can disable on-board audio:
```
sudo vim /etc/modprobe.d/raspi-blacklist.conf
blacklist snd_bcm2835
```

You can also blacklist any other device/module you want.
Simply use the module names listed with `lsmod` command


#Disable terminal/screen blanking

[Source](https://www.raspberrypi.org/forums/viewtopic.php?f=66&t=18200)

```
sudo vim /etc/kbd/config
# set
BLANK_TIME=0
POWERDOWN_TIME=0

# reboot or restart kbd service
sudo /etc/init.d/kbd restart
```


# Get WiFi driver

[Source](https://www.raspberrypi.org/forums/viewtopic.php?f=28&t=102323&p=782878)

Follow this step if you don't want to use the onboard WiFi adapter.

As instructed in the `Source` link download and extract the
`install-wifi` script by `MrEngman`.

```
# detect the device and check if driver is available
sudo ./install-wifi -c

# then install the driver
./install-wifi 8812au
```


# Configure WiFi connection

```
# generate the PSK for your netwrok
wpa_passphrase <network-SSID> <network-Password> > wpa.conf
sudo mv wpa.conf /etc/wpa_supplicant.conf

# use  -Dnl80211 or -Dwext
sudo wpa_supplicant -B -iwlan0 -c/etc/wpa_supplicant.conf -Dnl80211
sudo dhclient wlan0
```

ps. if you're not using the on-board WiFi card then change `wlan0` to `wlan1`
or whatever name system assigned to it.

Once you're happy with your WiFi configuration, then to make it automatically
reconnect after boot-up edit the `/etc/network/interfaces`

If you're using external WiFi USB dongle and this device is known as `wlan1`
 then it should look more or less like this:
```
source-directory /etc/network/interfaces.d

auto lo
iface lo inet loopback

iface eth0 inet manual

auto wlan1
allow-hotplug wlan1
iface wlan1 inet manual
    pre-up wpa_supplicant -B -D nl80211 -i wlan1 -c /etc/wpa_supplicant.conf
    post-down killall -q wpa_supplicant
iface default inet dhcp
```

If you prefer to use on-board WiFi card then

```
source-directory /etc/network/interfaces.d

auto lo
iface lo inet loopback

iface eth0 inet manual

auto wlan0
allow-hotplug wlan0
iface wlan0 inet manual
    pre-up wpa_supplicant -B -D nl80211 -i wlan0 -c /etc/wpa_supplicant.conf
    post-down killall -q wpa_supplicant
iface default inet dhcp
```

You might need to restart the networking service or reboot to get it working.

```
sudo service networking restart
# or
sudo reboot
```

# MPD, ncmpcpp & C.A.V.A Dependencies

This is going to download around 1.5GB of packages so go for a lunch.

```
sudo apt-get install g++ cmake automake autoconf tmux lame flac mpc \
  libtool libfftw3-dev libasound2-dev libncursesw5-dev libpulse-dev \
  libmad0-dev libmpg123-dev libid3tag0-dev \
  libflac-dev libvorbis-dev libopus-dev \
  libadplug-dev libaudiofile-dev libsndfile1-dev libfaad-dev \
  libfluidsynth-dev libgme-dev libmikmod2-dev libmodplug-dev \
  libmpcdec-dev libwavpack-dev libwildmidi-dev \
  libsidplay2-dev libsidutils-dev libresid-builder-dev \
  libavcodec-dev libavformat-dev \
  libmp3lame-dev \
  libsamplerate0-dev libsoxr-dev \
  libbz2-dev libcdio-paranoia-dev libiso9660-dev libmms-dev \
  libzzip-dev \
  libcurl4-gnutls-dev libyajl-dev libexpat-dev \
  libasound2-dev libao-dev libjack-jackd2-dev libopenal-dev \
  libpulse-dev libroar-dev libshout3-dev \
  libmpdclient-dev \
  libnfs-dev libsmbclient-dev \
  libupnp-dev \
  libavahi-client-dev \
  libsqlite3-dev \
  libsystemd-daemon-dev libwrap0-dev \
  libcppunit-dev xmlto \
  libboost-dev libboost-all-dev \
  libicu-dev libavahi-compat-libdnssd-dev
```


# Get all the required repos

```
cd
mkdir git
cd git

git clone https://github.com/MaxKellermann/MPD --depth=1
git clone https://github.com/arybczak/ncmpcpp.git --depth=1
git clone https://github.com/karlstav/cava.git --depth=1
git clone https://github.com/taglib/taglib.git --depth=1
```

# TagLib

Taglib is required by `ncmpcpp` and it's not available via `apt-get install` :(

```
cd ~/git/taglib
cmake -DCMAKE_INSTALL_PREFIX=/usr/local -DCMAKE_BUILD_TYPE=Release .
make
sudo make install
```

# Cava

```
cd ~/git/cava
./autogen.sh
./configure.sh
make
sudo make install
```

# MPD

```
cd ~/git/mpd
./autogen.sh
./configure --enable-pipe-output --enable-soundcloud --enable-vorbis \
			--enable-wave-encoder --enable-curl --enable-smbclient \
			--enable-nfs --enable-cdio-paranoia  --enable-mms \
			--enable-sqlite --enable-id3 --enable-libmpdclient \
			--enable-systemd-daemon --enable-neighbor-plugins --enable-flac \
			--enable-mpc --enable-wavpack --enable-lame-encoder --enable-alsa
make
sudo make install
```


# ncmpcpp

```
cd ~/git/ncmpcpp
./autogen.sh
./configure --with-curl --with-fftw --with-taglib --enable-outputs \
			--enable-visualizer --enable-clock --enable-unicode
make
sudo make install
```


# MPD configuration

This configuration file has 4 outputs defined:
* alsa - to get the sound from from DAC
* mp3 stream - so you can stream your music to your MPD client (ie. MPDroid)
* flac stream - same as above but it will convert the currently playin track to FLAC
* fifo - for ncmpcpp visualizer and/or C.A.V.A.

```
mdkir -p ~/mpd/playlists
vim ~/.mpdconf

music_directory     "/home/mpd/music"
playlist_directory  "/home/mpd/.mpd/playlists"
db_file             "/home/mpd/.mpd/database"
log_file            "/home/mpd/.mpd/log"
pid_file            "/home/mpd/.mpd/pid"
state_file          "/home/mpd/.mpd/state"
sticker_file        "/home/mpd/.mpd/sticker.sql"
user                "mpd"
zeroconf_enabled    "yes"
zeroconf_name       "mpd"

input {
    plugin          "curl"
}

audio_output {
    type            "alsa"
    name            "HiFiBerry dac+ pro"
    device          "hw:0,0"    # optional
    mixer_type      "hardware"  # optional
    mixer_control   "digital"   # optional
}

audio_output {
    type            "httpd"
    name            "mp3 HTTP Stream"
    encoder         "lame"
    port            "8000"
    #quality        "5.0"           # do not define if bitrate is defined
    bitrate         "320"           # do not define if quality is defined
    #compression    "8"
    format          "44100:16:2"
    max_clients     "0"             # optional 0=no limit
    tags            "yes"
}

audio_output {
    type            "httpd"
    name            "FLAC HTTP Stream"
    encoder         "flac"
    port            "9000"
    format          "44100:16:2"
    max_clients     "0"             # optional 0=no limit
    tags            "yes"
}

audio_output {
    type            "fifo"
    name            "fifo"
    path            "/tmp/mpd.fifo"
    format          "44100:16:2"
}

playlist_plugin {
    name            "soundcloud"
    enabled         "true"
    apikey          "your_soundclound_api_key"
}
```


# ncmpcpp config

```
mpd_host = "127.0.0.1"
mpd_port = "6600"
mpd_connection_timeout = "5"
mpd_crossfade_time = "5"

visualizer_fifo_path = "/tmp/mpd.fifo"
visualizer_output_name = "fifo"
visualizer_type = "spectrum" (spectrum/frequency)
visualizer_sync_interval = "10"
visualizer_in_stereo = "yes"
visualizer_look = "+|"

now_playing_prefix = "$b"
now_playing_suffix = "$/b"
playlist_display_mode = "columns" (classic/columns)
autocenter_mode = "yes"
centered_cursor = "yes"

song_status_format = "%t » %a »{ %b » }%y"
progressbar_look = "━■"

browser_playlist_prefix = "$2plist »$9 "
browser_display_mode = "columns" (classic/columns)

song_window_title_format = "{%a - }{%t}{ - %b{ Disc %d}}|{%f}"
search_engine_display_mode = "columns" (classic/columns)
follow_now_playing_lyrics = "yes"
clock_display_seconds = "yes"
display_bitrate = yes

startup_screen = visualizer
```


# PiFi Remote
If you want to add remote cotroller support to your RPi and you have for
example a [Flirc IR USB receiver](https://flirc.tv/more/flirc-usb) then follow
the instructions below.

## Map your IR remote controller codes (buttons) to regular keyboard key presses

Get the `flirc_util` for RPi and record the button mappings

```
wget https://flirc.tv/software/release/gui/linux/Linux_Release.zip
unzip Linux_Release.zip
cd release/rpi/

# Record the button mapping with `flirc_util`
sudo ./flirc_util record escape
sudo ./flirc_util record up
sudo ./flirc_util record left
sudo ./flirc_util record right
sudo ./flirc_util record down
sudo ./flirc_util record enter
sudo ./flirc_util record backspace
sudo ./flirc_util record mute
sudo ./flirc_util record vol_up
sudo ./flirc_util record vol_down
sudo ./flirc_util record play/pause
sudo ./flirc_util record stop
sudo ./flirc_util record fast_forward
sudo ./flirc_util record rewind
sudo ./flirc_util record prev_track
sudo ./flirc_util record next_track
sudo ./flirc_util record pageup
sudo ./flirc_util record pagedown
sudo ./flirc_util record u
sudo ./flirc_util record r
sudo ./flirc_util record x
```

Then install required Python packages and clone the PiFi-Remote repo

```
sudo apt-get install python-pip evdev python-mpd2
cd ~/git/
git clone https://github.com/kowalcj0/PiFi-Remote.git
cd PiFi-Remote
```

Now, you'll have to map the buttons or key codes according to your liking.

```
# open 2 more terminals and connect to your Raspberry Pi
# on the 2nd one start `PiFiRemote.py`
cd ~/git/PiFi-Remote/pifi
sudo ./PiFiRemote.py
# on the 3rd one monitor the PiFi-Remote log:
tail -f /var/log/pifi-remote.log
```

Now let's get back to the 1st terminal and edit the key mappings
`vim pifi/PiFiRemote.py`

Press a button on you IR remote and observe the log.
You should see something like:
```
2016-06-04 23:01:10,155 DEBUG mpd._write_command: Calling MPD close()
2016-06-04 23:01:10,156 INFO mpd.disconnect: Calling MPD disconnect()
2016-06-04 23:01:10,157 INFO PiFiRemote.monitorRemote: Job monitorRemote started
2016-06-04 23:01:13,129 DEBUG PiFiRemote.monitorRemote: event at 1465081273.129032, code 04, type 04, val 458827
2016-06-04 23:01:13,180 DEBUG PiFiRemote.monitorRemote: key event at 1465081273.129032, 104 (KEY_PAGEUP), down
2016-06-04 23:01:13,181 INFO mpd.connect: Calling MPD connect('localhost', 6600, timeout=None)
2016-06-04 23:01:13,182 DEBUG mpd._write_command: Calling MPD consume(0,)
2016-06-04 23:01:13,183 DEBUG mpd._write_command: Calling MPD single(0,)
2016-06-04 23:01:13,184 DEBUG mpd._write_command: Calling MPD status()
2016-06-04 23:01:13,186 DEBUG PiFiRemote.monitorRemote: Status: {'songid': '2', 'playlistlength': '29', 'playlist': '2', 'repeat': '0', 'consume': '0', 'mixrampdb': '0.000000', 'random': '1', 'state': 'stop', 'volume': '83', 'single': '0', 'nextsong': '17', 'song': '1', 'nextsongid': '18'}
2016-06-04 23:01:13,186 DEBUG PiFiRemote.monitorRemote: KeyCode: 104
2016-06-04 23:01:13,187 DEBUG mpd._write_command: Calling MPD play(1,)
2016-06-04 23:01:13,188 DEBUG mpd._write_command: Calling MPD close()
2016-06-04 23:01:13,189 INFO mpd.disconnect: Calling MPD disconnect()
2016-06-04 23:01:13,190 DEBUG PiFiRemote.monitorRemote: synchronization event at 1465081273.129032, SYN_REPORT
2016-06-04 23:01:13,257 DEBUG PiFiRemote.monitorRemote: event at 1465081273.257060, code 04, type 04, val 458827
2016-06-04 23:01:13,969 DEBUG PiFiRemote.monitorRemote: key event at 1465081273.257060, 104 (KEY_PAGEUP), up
2016-06-04 23:01:14,020 DEBUG PiFiRemote.monitorRemote: synchronization event at 1465081273.257060, SYN_REPORT
```

Find the line that says `key event at ...`
Take a note of the key (event) code next to the key name (ie. `KEY_PAGEUP`)
Then use that number to associate it with selected action you want to trigger

In my case I've mapped it to start the song from the begginging (instant reply)
```
elif event.code == 104:
    mpc.play(1)
```


## Install PiFi-Remote service

When you're happy with your button mapping then simply install the package.
```
cd ~/git/PiFi-Remote
sudo ./setup.py install
```
This will create a new service, which state you can manage with standard
```
sudo service pifiremote start|stop|restart
```


# Change the default user name

[Source](http://askubuntu.com/a/317008/551528)

* Log in using your username and password
* Set a password for the "root" account
`sudo passwd root`
* enable SSH logins for root user
```
sudo vi /etc/ssh/sshd_config
# change `PermitRootLogin without-password` to `PermitRootLogin yes`
```
* Restart the `sshd` service
```
sudo service sshd restart
```
* Log out
* Log in using the "root" account and the password you have previously set
* Change the username and the home folder to the new name that you want
```
usermod -l <newname> -d /home/<newname> -m <oldname>
```
* Change the group name to the new name that you want
```
groupmod -n <newgroup> <oldgroup>
```
* Lock the "root" account
```
passwd -l root
```
* Disable SSH logins for root user
```
sudo vi /etc/ssh/sshd_config
# change `PermitRootLogin yes` to `PermitRootLogin no`
```
* If you were using ecryptfs (encrypted home directory). Mount your encrypted directory using ecryptfs-recover-private and edit `<mountpoint>/.ecryptfs/Private.mnt` to reflect your new home directory.
* Log out
* Log back in as the new user


# Change the device network name

* Edit the `/etc/hosts` and change `127.0.1.1	pi` to `127.0.1.1	your_new_amazing_network_name`
* Then edit the `/etc/hostname` and change `pi` to `your_new_amazing_network_name`
* Reboot

From now on, your Pi will be accessible via:
`your_new_amazing_network_name.home` or `your_new_amazing_network_name.local` hostname :)


# Console Autologin

Change 4 occurences of username `pi` to `your_new_username` in `/usr/bin/raspi-config`

I'll use `mpd` as my username

```
sudo vim /usr/bin/raspi-config

#...
"B2 Console Autologin" "Text console, automatically logged in as 'mpd' user" \
#...
"B4 Desktop Autologin" "Desktop GUI, automatically logged in as 'mpd' user" \
#...
sed /etc/inittab -i -e "s/1:2345:respawn:\/bin\/login -f mpd tty1 <\/dev\/tty1 >\/dev\/tty1 2>&1/1:2345:respawn:\/sbin\/getty --noclear 38400 tty1/"
###...
sed /etc/inittab -i -e "s/1:2345:respawn:\/sbin\/getty --noclear 38400 tty1/1:2345:respawn:\/bin\/login -f mpd tty1 <\/dev\/tty1 >\/dev\/tty1 2>&1/"
```

* Then launche the `raspi-config` as `root`
* Choose option `3 Boot Options`
* Choose option `B2 Console Autologin`

Next time you're going to boot up your RPI it will automatically login as that user.

# Start MPD and C.A.V.A on autologin

I don't know why MPD didn't register itself as a new system service, so to start it up
on login simply modify your `.profile`
```
vim ~/.profile
mpd; wait

cava
```
