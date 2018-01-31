---
layout: post
title: Multi-room audio with MPD, Snapcast and Raspberry Pis
---

![header]({{ site.url }}/assets/images/2018-01-16/IMGP1554.jpg)

While having our bathroom and sauna renovated, I wanted to be able to listen
to music in sauna. Getting a speaker and its cable in place was straightforward,
but after that the setup became more complex. Some amplifier with bluetooth audio
or Spotify streaming would have been simple, but for bluetooth you'd still
need the music somewhere, and I don't want a Spotify subscription.

And of course you'd want the same music to play inside and outside the sauna.
So, I had an excuse to build multi-room audio with inexpensive components for
two rooms.

### Software

Raspberry Pis were an easy choice to build around, so everything has to work
in Raspbian Linux.

[Music Player Daemon](https://www.musicpd.org/) (MPD) is used for playing music,
it has a local playlist that can be changed with clients so that the device
can keep playing music even if no controllers are connected to it. If
you *do* have that Spotify subscription, [Mopidy](https://www.mopidy.com/)
is a MPD compatible server with Spotify support through extensions.

[Soundirok](http://www.kvibes.de/en/soundirok/) is a MPD client for iOS devices.
It supports also loading cover images with HTTP, so I have
[nginx](https://www.nginx.com/resources/wiki/) set up for that purpose also.
On macOS, I use [ncmpcpp](https://rybczak.net/ncmpcpp/).

Multi-room audio is achieved with [Snapcast](https://github.com/badaix/snapcast).
Unfortunately it doesn't have an iOS client software available, so controlling
different rooms' volumes separately is not easily possible at the moment.

[ALSA](https://www.alsa-project.org/) is used for controlling the sound cards.
It's not that simple to configure, but then again, it's quite flexible when
you do get the configuration in place.

For sorting out the music library, I use [beets](http://beets.io/), but explaining
that pipeline would be a topic for another blog post. Here I assume that the music
just appears to the correct place.

### Hardware

![multiroom audio diagram]({{ site.url }}/assets/images/2018-01-16/multiroom-audio.svg)

I have devices in two rooms: living room and sauna. `rpi-olohuone` in living room
is the server device `rpi-kph` in sauna is a client device.

![Living room server setup]({{ site.url }}/assets/images/2018-01-16/IMGP1579.jpg)
The server setup has Raspberry Pi model 1 B, but the actual model doesn't really
matter as everything could also be done with a Pi Zero also. Music is played with
[NuForce &mu;DAC-2](https://www.optoma.com/ap-nuforce/product/%CE%BCdac3/) USB
sound card (aka DAC). Files are stored in a Seagate HDD.

&mu;DAC and HDD are powered through USB, and at least the HDD requires more power
than what Raspberry is able to give, so I have a powered TP-Link USB hub in the middle.

![Sauna client setup]({{ site.url }}/assets/images/2018-01-16/IMGP1559.jpg)
Sauna client is a Raspberry Pi Zero with a
[RedBear IoT pHAT](https://github.com/redbear/IoT_pHAT) for WiFi connectivity and
an external antenna for added range. The case was not optimal, but does its job.

Adding pHAT required soldering both on the Pi Zero and on the pHAT itself.
There is also a model Pi Zero W with WiFi included, and with that you don't need
the pHAT at all.

Selecting the sound card required a bit more thought as I had to have a DAC and
amplifier both. After some googling, I ended up with
[Topping VX1](http://www.tpdz.net/en/products/vx1/), an USB-DAC *and* amplifier
in the same device, which should also work in Linux!

### Configuration

Configuration is done with Ansible and is available in my
[raspberry-ansible](https://github.com/rhietala/raspberry-ansible) repo. Everything
described here is included in two playbooks:
[`bootstrap.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/bootstrap.yml) and
[`music.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/music.yml)

### Configuration: External HDD

[`bootstrap.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/bootstrap.yml#L13-L16) playbook:

```{% raw %}
- include_role:
    name: external-hdd
  when:
  - exthdd is defined
{% endraw %}```

[`host_vars/rpi-olohuone.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/host_vars/rpi-olohuone.yml#L3-L5):

```{% raw %}
exthdd: "/dev/sda1"
exthdd_fstype: "vfat"
exthdd_mountpoint: "/mnt/piikiekko"
{% endraw %}```

[`roles/external-hdd/tasks/main.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/roles/external-hdd/tasks/main.yml):

```{% raw %}
- name: mount device
  mount:
    src: "{{ exthdd }}"
    path: "{{ exthdd_mountpoint }}"
    fstype: "{{ exthdd_fstype }}"
    state: mounted
    opts: "umask=000" # allow writes from all users
{% endraw %}```

Setting up the external HDD is quite simple with Ansible, you need only the
[mount](http://docs.ansible.com/ansible/latest/mount_module.html) module.


You can find out which device the external hdd is with `lsblk` and `blkid`
(here `/dev/sda1`):

```{% raw %}
pi@rpi-olohuone:~ $ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0 931.5G  0 disk
`-sda1        8:1    0 931.5G  0 part /mnt/piikiekko
mmcblk0     179:0    0  14.4G  0 disk
|-mmcblk0p1 179:1    0  41.5M  0 part /boot
`-mmcblk0p2 179:2    0  14.4G  0 part /

pi@rpi-olohuone:~ $ blkid
/dev/mmcblk0p1: LABEL="boot" UUID="CDD4-B453" TYPE="vfat" PARTUUID="dba1b453-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="72bfc10d-73ec-4d9e-a54a-1cc507ee7ed2" TYPE="ext4" PARTUUID="dba1b453-02"
/dev/sda1: LABEL="PIIKIEKKO" UUID="8001-1AF6" TYPE="vfat"
{% endraw %}```

See also [External storage configuration](https://www.raspberrypi.org/documentation/configuration/external-storage.md)
on official Raspberry Pi documentation.

### Configuration: RedBear IoT pHAT

[`bootstrap.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/bootstrap.yml#L9-L12) playbook:

```{% raw %}
- include_role:
    name: wifi
  when:
    - enable_wifi is defined and enable_wifi == True
{% endraw %}```

[`host_vars/rpi-kph.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/host_vars/rpi-kph.yml#L2):

```{% raw %}
enable_wifi: True
{% endraw %}```

`group_vars/pi/vault.yml` (file with secret variables, this can be edited with
`ansible-vault edit` and the file in my github repo cannot be used, you'll have
to overwrite it with your own):

```{% raw %}
wifi_network: <network name>
wifi_password: <password>
{% endraw %}```

[`roles/external-hdd/tasks/main.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/roles/external-hdd/tasks/main.yml):

```{% raw %}
- name: Add wifi configuration
  blockinfile:
    dest: "/etc/wpa_supplicant/wpa_supplicant.conf"
    block: |
      network={
      	ssid="{{ wifi_network }}"
      	psk="{{ wifi_password }}"
      	key_mgmt=WPA-PSK
      }
  when: ansible_wlan0
  notify: restart machine
{% endraw %}```

Setting up the RedBear IoT pHAT is also really simple, Linux kernel notices the
device itself and you just have to setup the correct WiFi network parameters.

See also [RedBear's installation instructions](https://github.com/redbear/IoT_pHAT).

### Configuration: Sound cards / ALSA

ALSA configuration was probably the hardest part to get working, especially with
the Topping VX1 sound card.

[`music.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/music.yml#L5)
playbook, relevant parts:

```{% raw %}
- hosts: music-server
  roles:
    - alsa

- hosts: music-client
  roles:
    - alsa
{% endraw %}```

[`host_vars/rpi-olohuone.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/host_vars/rpi-olohuone.yml#L8-L17):

```{% raw %}
asound_conf: |
  pcm.!default {
    type hw
    card 1
  }

  ctl.!default {
    type hw
    card 1
  }
{% endraw %}```

Living room ALSA configuration is fairly simple, set the card number 1
(NuForce &mu;DAC) as the default. Card numbers can be printed with `aplay -l`:

```{% raw %}
pi@rpi-olohuone:~ $ sudo aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: ALSA [bcm2835 ALSA], device 0: bcm2835 ALSA [bcm2835 ALSA]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
card 0: ALSA [bcm2835 ALSA], device 1: bcm2835 ALSA [bcm2835 IEC958/HDMI]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: N2 [NuForce µDAC 2], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: N2 [NuForce µDAC 2], device 1: USB Audio [USB Audio #1]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
{% endraw %}```

[`host_vars/rpi-olohuone.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/host_vars/rpi-olohuone.yml#L8-L17):

```{% raw %}
asound_conf: |
  pcm.!default makemono

  pcm.makemono {
    type route
    slave.pcm "sysdefault:VX1"
    ttable {
      0.0 1    # in-channel 0, out-channel 0, 100% volume
      1.0 1    # in-channel 1, out-channel 0, 100% volume
    }
  }
{% endraw %}```

There is only one speaker in sauna, so I'd want both the channels to be mixed into
mono. Luckily there was [a snippet for this](https://superuser.com/a/155601).
`sysdefault:VX1` comes from `aplay -L` output, chosen with trial and error:

```{% raw %}
pi@rpi-kph:~ $ sudo aplay -L
null
    Discard all samples (playback) or generate zero samples (capture)
makemono
sysdefault:CARD=ALSA
    bcm2835 ALSA, bcm2835 ALSA
    Default Audio Device

...

sysdefault:CARD=VX1
    VX1, USB Audio
    Default Audio Device

...
{% endraw %}```

[`roles/alsa/tasks/main.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/roles/alsa/tasks/main.yml):

```{% raw %}
- name: install alsa-utils
  apt:
    name: "{{ item }}"
  with_items:
    - "alsa-utils"

- name: add snd_bcm2835 kernel module
  modprobe:
    name: "snd_bcm2835"

- name: add snd_bcm2835 kernel module to be loaded on boot
  lineinfile:
    path: "/etc/modules"
    line: "snd_bcm2835"

- name: setup alsa default sound card
  copy:
    content: "{{ asound_conf }}" # device-dependent
    dest: "/etc/asound.conf"
  notify:
    - reboot
    - wait for reboot
{% endraw %}```

The task configures also the Raspberry Pi onboard sound card (`snd_bcm2835`) into
use, even though neither of these current devices use it. The installed package
`alsa-utils` can be used to test the configuration:

```{% raw %}
pi@rpi-kph:~ $ sudo speaker-test -c 2

speaker-test 1.1.3

Playback device is default
Stream parameters are 48000Hz, S16_LE, 2 channels
Using 16 octaves of pink noise
Rate set to 48000Hz (requested 48000Hz)
Buffer size range from 1024 to 8192
Period size range from 511 to 513
Using max buffer size 8192
Periods = 4
was set period_size = 512
was set buffer_size = 8192
 0 - Front Left
 1 - Front Left
{% endraw %}```

should output noise on both channels in turns (and from the same speaker in sauna).

See also [Alsa Opensrc Org: .asoundrc](https://alsa.opensrc.org/Asoundrc) for
troubleshooting help.

### Configuration: Snapcast

Snapcast has to be configured so that music player, MPD sends audio to Snapcast server,
which will forward the audio to all connected clients. In order to get the audio
synchronized, also the device with MPD has to play music through Snapcast client.

[`music.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/music.yml) playbook, relevant parts:

```{% raw %}
- hosts: music-server
  roles:
    - snapcast-server
    - snapcast-client

- hosts: music-client
  roles:
    - snapcast-client
{% endraw %}```

[`roles/snapcast-server/tasks/main.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/roles/snapcast-server/tasks/main.yml)

```{% raw %}
- name: install snapserver
  apt:
    deb: "https://github.com/badaix/snapcast/releases/download/v0.12.0/snapserver_0.12.0_armhf.deb"

- name: open ports in firewall
  ufw:
    rule: "allow"
    port: "{{ item }}"
  with_items:
    - 1704
    - 1705
{% endraw %}```

Snapcast server is not available from Debian or Raspbian Aptitude repos, but luckily an
Aptitude package is available in Github. The built package is different for different
processor families, `_armhf` is the correct for Raspberry Pi, others are listed in
the [releases page](https://github.com/badaix/snapcast/releases/latest).

Snapcast server requires ports 1704 and 1705 to be open, but otherwise the default
configuration doesn't need any adjustments.

[`roles/snapcast-client/tasks/main.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/roles/snapcast-client/tasks/main.yml)

```{% raw %}
- name: install snapclient
  apt:
    deb: "https://github.com/badaix/snapcast/releases/download/v0.12.0/snapclient_0.12.0_armhf.deb"

- name: configure snapcast server address
  lineinfile:
    path: "/etc/default/snapclient"
    regexp: "^SNAPCLIENT_OPTS"
    line: "SNAPCLIENT_OPTS=\"{{ snapclient_opts }}\""
  notify: restart snapcast client
{% endraw %}```

[`host_vars/rpi-olohuone.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/host_vars/rpi-olohuone.yml#L18-L19)

```{% raw %}
snapcast_server: "192.168.1.249"
snapclient_opts: "--host {{ snapcast_server }}"
{% endraw %}```

[`host_vars/rpi-kph.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/host_vars/rpi-kph.yml#L16-L17)

```{% raw %}
snapcast_server: "192.168.1.249"
snapclient_opts: "--host {{ snapcast_server }} --soundcard makemono"
{% endraw %}```

Snapcast client is installed from the same place as server. It doesn't require that
much configuration either, only the server IP address on both clients, and
additionally the custom sound card `makemono` on `rpi-kph` (it's supposed to be
the default, don't know why it didn't work with snapclient without specifying it
like this).

Troubleshooting Snapcast server can be started with `systemctl`, working output
is something like this:

```{% raw %}
pi@rpi-olohuone:~ $ sudo systemctl -l status snapserver.service
● snapserver.service - Snapcast server
   Loaded: loaded (/lib/systemd/system/snapserver.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-01-21 00:30:27 EET; 1 weeks 3 days ago
  Process: 464 ExecStart=/usr/bin/snapserver -d $USER_OPTS $SNAPSERVER_OPTS (code=exited, status=0/SUCCESS)
 Main PID: 472 (snapserver)
   CGroup: /system.slice/snapserver.service
           └─472 /usr/bin/snapserver -d --user snapserver:snapserver

Jan 21 00:30:26 rpi-olohuone systemd[1]: Starting Snapcast server...
Jan 21 00:30:27 rpi-olohuone snapserver[464]: Settings file: "/var/lib/snapserver/server.json"
Jan 21 00:30:27 rpi-olohuone snapserver[464]: 2018-01-21 00-30-27 [Notice] Settings file: "/var/lib/snapserver/server.json"
Jan 21 00:30:27 rpi-olohuone snapserver[472]: daemon started
Jan 21 00:30:27 rpi-olohuone systemd[1]: Started Snapcast server.
Jan 21 01:26:56 rpi-olohuone snapserver[472]: StreamServer::NewConnection: ::ffff:192.168.1.249
Jan 21 01:27:34 rpi-olohuone snapserver[472]: StreamServer::NewConnection: ::ffff:192.168.1.3
{% endraw %}```

And Snapcast client the same way:

```{% raw %}
pi@rpi-kph:~ $ sudo systemctl status -l snapclient.service
● snapclient.service - Snapcast client
   Loaded: loaded (/lib/systemd/system/snapclient.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-01-21 01:27:34 EET; 1 weeks 3 days ago
  Process: 1155 ExecStart=/usr/bin/snapclient -d $USER_OPTS $SNAPCLIENT_OPTS (code=exited, status=0/SUCCESS)
 Main PID: 1156 (snapclient)
   CGroup: /system.slice/snapclient.service
           └─1156 /usr/bin/snapclient -d --user snapclient:audio --host 192.168.1.249 --soundcard makemono

Jan 21 01:27:34 rpi-kph systemd[1]: Starting Snapcast client...
Jan 21 01:27:34 rpi-kph snapclient[1156]: daemon started
Jan 21 01:27:34 rpi-kph systemd[1]: Started Snapcast client.
Jan 21 01:27:34 rpi-kph snapclient[1156]: Connected to 192.168.1.249
{% endraw %}```

### Configuration: MPD

[`music.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/music.yml) playbook, relevant parts:

```{% raw %}
- hosts: music-server
  roles:
    - mpd-server
{% endraw %}```

[`roles/mpd-master/tasks/main.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/roles/mpd-master/tasks/main.yml):

```{% raw %}
- name: install mpd, nginx, mpc
  apt:
    name: "{{ item }}"
  with_items:
    - "mpd"
    - "nginx"
    - "mpc"

- name: copy mpd configuration
  template:
    src: "mpd.conf.j2"
    dest: "/etc/mpd.conf"
  notify: restart mpd

- name: copy nginx cover art configuration
  template:
    src: "mpd-cover-art.conf.j2"
    dest: "/etc/nginx/sites-available/mpd-cover-art.conf"
  notify: restart nginx

- name: symlink nginx cover art configuration
  file:
    src: "/etc/nginx/sites-available/mpd-cover-art.conf"
    dest: "/etc/nginx/sites-enabled/mpd-cover-art.conf"
    state: "link"
  notify: restart nginx

- name: open ports in firewall
  ufw:
    rule: "allow"
    port: "{{ item }}"
  with_items:
    - "http"
    - 6600
{% endraw %}```

MPD role installs and configures the MPD server (of course), and also nginx
HTTP server for serving albums' cover art images. MPD controls use port 6600.

[mpc](https://www.musicpd.org/clients/mpc/) is a command-line client for MPD.
It is not used in these Ansible scripts, but comes in very handy when you add
music to the library and the MPD database has to be updated.

[`roles/mpd-master/templates/mpd.conf.j2`](https://github.com/rhietala/raspberry-ansible/blob/master/roles/mpd-master/templates/mpd.conf.j2):

```{% raw %}
music_directory    "{{ music_dir }}"
playlist_directory "/var/lib/mpd/playlists"
db_file            "/var/lib/mpd/tag_cache"
log_file           "/var/log/mpd/mpd.log"
pid_file           "/run/mpd/pid"
state_file         "/var/lib/mpd/state"
sticker_file       "/var/lib/mpd/sticker.sql"

user               "mpd"
bind_to_address    "0.0.0.0"
filesystem_charset "UTF-8"
id3v1_encoding     "UTF-8"

input {
  plugin "curl"
}

#audio_output {
#  type  "alsa"
#  name  "My ALSA Device"
#}

audio_output {
  type       "fifo"
  name       "snapcast"
  path       "/tmp/snapfifo"
  format     "48000:16:2"
  mixer_type "software"
}
{% endraw %}```

MPD configuration is fairly default. `audio_output` is the interesting part:
MPD plays music to `/tmp/snapfifo` stream, from where the Snapcast server reads it.

[`host_vars/rpi-olohuone.yml`](https://github.com/rhietala/raspberry-ansible/blob/master/host_vars/rpi-olohuone.yml#L6-L7)

```{% raw %}
mpd_server: True
music_dir: "{{ exthdd_mountpoint }}/musiikkia"
{% endraw %}```

Variables are also straightforward, define this device as MPD server, and specify
where the music is located.

MPD database can be updated with `mpc`:

```{% raw %}
pi@rpi-olohuone:~ $ mpc update
Updating DB (#1) ...
volume: 82%   repeat: off   random: off   single: off   consume: off
{% endraw %}```

and queried, queued and played:

```{% raw %}
pi@rpi-olohuone:~ $ mpc search album highway
Bob Dylan/Highway 61 Revisited/01 Like a Rolling Stone.flac
Bob Dylan/Highway 61 Revisited/02 Tombstone Blues.flac
...
pi@rpi-olohuone:~ $ mpc add "Bob Dylan/Highway 61 Revisited/01 Like a Rolling Stone.flac"
pi@rpi-olohuone:~ $ mpc play
Bob Dylan - Like a Rolling Stone
[playing] #1/1   0:00/6:13 (0%)
volume: 82%   repeat: off   random: off   single: off   consume: off
{% endraw %}```

Although you should definitely use some other client such as
[Soundirok](http://www.kvibes.de/en/soundirok/) for iOS
or [ncmpcpp](https://rybczak.net/ncmpcpp/) for macOS/Linux.

Troubleshooting can be started with `systemctl` again:

```{% raw %}
pi@rpi-olohuone:~ $ sudo systemctl status -l mpd.service
● mpd.service - Music Player Daemon
   Loaded: loaded (/lib/systemd/system/mpd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-01-21 00:30:27 EET; 1 weeks 3 days ago
     Docs: man:mpd(1)
           man:mpd.conf(5)
           file:///usr/share/doc/mpd/user-manual.html
 Main PID: 469 (mpd)
   CGroup: /system.slice/mpd.service
           └─469 /usr/bin/mpd --no-daemon

Jan 21 00:30:27 rpi-olohuone systemd[1]: Started Music Player Daemon.
{% endraw %}```

### Configuration: Soundirok

For Soundirok, MPD server has to be added under *Settings* &rarr; *Devices*:

![Soundirok configuration]({{ site.url }}/assets/images/2018-01-16/IMG_5924.PNG)

If I remember correctly, Soundirok should start synchronizing the MPD music library
to the client. It can be done manually by clicking the top bar device name
("Olohuone" in my case) and selecting *Refresh Soundirok database*.

### Closing remarks

I was planning that the sauna devices could be turned on and off with the light
switch, and there are power sockets to enable this. However, even though
Topping VX1 features include *Auto turn on / turn off synchronously with your PC
(Only in USB mode)*, I haven't figured out how to do this. Might be that this only
works in Windows.

When the devices are powered on (or get electricity), I still have to press a button
in VX1 to wake it up. It would be easier if this wasn't necessary as the device
isn't in a very easily accessible place. As a workaround, I've had it turned on
most of the time.

Other than that, the setup works really well.

You can leave questions and comments to [Reddit](https://www.reddit.com/r/raspberry_pi/comments/7udrdm/multiroom_audio_with_mpd_snapcast_and_raspberry/).

### More photos

![client in the ceiling]({{ site.url }}/assets/images/2018-01-16/IMGP1575.jpg)
Sauna devices are in the bathroom ceiling.

![speaker below sauna benches]({{ site.url }}/assets/images/2018-01-16/IMGP1570.jpg)
Speaker is under the sauna benches. The speaker is a weather-proof
[Bower & Wilkins AM-1](http://www.bowers-wilkins.eu/Speakers/Installation-Speakers/Outdoor-Marine/AM-1.html) and it sounds really good.
