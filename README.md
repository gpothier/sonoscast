sonoscast
=========

Out of the box, a Sonos system is not able to play (the audio of) YouTube videos. How to achieve this? If you have a server (or a VM running Linux) that is always turned on in your house, here is a rough tutorial on how to set this up using only free software. Press the Send to TV button on your phone's YouTube app, and your Sonos system will play the audio track of the video!

The idea is to have the YouTube audio play on your very own internet radio station, and have the Sonos tune to it. Here is the sketch of the solution:

1. Cast receiver. The "Send to TV" button of the phone's YouTube app uses the DIAL (or Cast Receiver) API to instruct your TV to play the selected YouTube content. The leapcast software (https://github.com/dz0ny/leapcast) emulates a Chromecast device, which understands the DIAL API. Install it on your server, and you will be able to use the Send to TV button to play the video on your server.

2. Internet radio. Use IceCast2 and Darkice to convert the audio output of your server into an Internet radio station the Sonos system can tune to. Whenever you send a video to your server with the Send to TV button, the radio station will broadcast the audio track of the video.

3. Monitor. A simple python script (below) monitors the audio output of the server, and instructs the Sonos system to tune to your radio station whenever it detects audio activity (using the SoCo remote control software - https://github.com/SoCo/SoCo).

This is a bit convoluted (and uses a bit a CPU on your server during playback), but it works quite well!

For now this will a rather rough tutorial for those who know their way around a Linux server, I hope others will step in to make it more understandable to the masses (maybe package it?). Also, I might be forgetting some steps as I'm writing this mostly from memory; please post a comment here in case something is missing.

Ideally, you should set this up on a virtual machine so that you don't stream random audio through your radio station (like system notification sounds, etc.). If you have never created a virtual machine, VirtualBox (https://www.virtualbox.org/) is probably the easiest option. As far as I am concerned, I did it with KVM. In the remaining of the tutorial I will refer to this virtual machine as just "the server".

The first step is to install Ubuntu Server 14.04 on your server. Please go ahead and google it if you don't know how to do this. In the remaining of this tutorial, I will assume the administrator's user name is "gpothier" (yeah...), but feel free to choose anything you like ("sonos" sounds about right). Don't forget to change every instance of gpothier to the correct user name. 

Next step is to figure out how to connect to the server in a stable way. In most cases, once the machine obtains an IP address from your network's DHCP server, it should always get that address (DHCP servers usually remember the IP last assigned to a given MAC address); otherwise, you can configure the server to have a fixed IP address. Of course if you have a DNS server in your network (as in my case), you are all set. In the following I will use sonos-helper as the hostname of the server; replace it with your servers's IP address (or hostname if you have a DNS).

Install required packaged software:

    sudo apt-get install xorg node npm darkice icecast2 virtualenvwrapper virtualenv python-pip python-twisted-web python2.7-dev chromium-browser python-setuptools python-dev build-essential git-core git

This should prompt you to configure Icecast2; use "sonos" for the password everywhere.

Setup virtualenv (I'm not familiar with that, so maybe it is not the best way to do it, but leapcast seems to need it):

    sudo easy_install pip
    sudo pip install virtualenv
    sudo pip install virtualenvwrapper
    mkdir ~/virtualenvs 
    echo "export WORKON_HOME=~/virtualenvs" >> ~/.bashrc
    echo "source /usr/local/bin/virtualenvwrapper.sh" >> ~/.bashrc 
    echo "export PIP_VIRTUALENV_BASE=~/virtualenvs" >> ~/.bashrc 
    source ~/.bashrc 

Install remaining software:

Leapcast (Cast Receiver)

    cd
    git clone https://github.com/dz0ny/leapcast.git
    cd leapcast
    mkvirtualenv leapcast
    pip install .

SoCo (Sonos remote control)

    cd
    git clone https://github.com/SoCo/SoCo.git

At the time of this writing there is a slight problem with SoCo, see https://github.com/SoCo/SoCo/issues/201. In the soco/core.py file, you have to replace this (end of the discover function, currently around line 61):

    response, _, _ = select.select([_sock], [], [], timeout)
    # Only Zone Players will respond, given the value of ST in the
    # PLAYER_SEARCH message. It doesn't matter what response they make. All
    # we care about is the IP address
    if response:
        _, addr = _sock.recvfrom(1024)
        # Now we have an IP, we can build a SoCo instance and query that player
        # for the topology to find the other players. It is much more efficient
        # to rely upon the Zone Player's ability to find the others, than to
        # wait for query responses from them ourselves.
        zone = config.SOCO_CLASS(addr[0])
        if include_invisible:
            return zone.all_zones
        else:
            return zone.visible_zones
    else:
        return None


by this:

    while True:
        response, _, _ = select.select([_sock], [], [], timeout)
        # Only Zone Players will respond, given the value of ST in the
        # PLAYER_SEARCH message. It doesn't matter what response they make. All
        # we care about is the IP address
        if response:
            data, addr = _sock.recvfrom(1024)
            if not "Sonos" in data:
                continue
            
            # Now we have an IP, we can build a SoCo instance and query that player
            # for the topology to find the other players. It is much more efficient
            # to rely upon the Zone Player's ability to find the others, than to
            # wait for query responses from them ourselves.
            zone = config.SOCO_CLASS(addr[0])
            if include_invisible:
                return zone.all_zones
            else:
                return zone.visible_zones
        else:
            return None
    else:
        return None

(ensure the indentation is right, if you copy/paste from above you probably have to add four spaces at the beginning of each line in your editor so that it remains at the same position as the original code)

python-pulseaudio (Python interface to the pulseaudio sound system)

    cd
    git clone https://github.com/Valodim/python-pulseaudio.git

Configure Darkice

Edit /etc/darkice.cfg so that it contains the following:

    [general]
    duration        = 0      # duration in s, 0 forever
    bufferSecs      = 1      # buffer, in seconds
    reconnect       = yes    # reconnect if disconnected
    
    [input]
    device          = pulse
    sampleRate      = 44100  # sample rate 11025, 22050 or 44100
    bitsPerSample   = 16     # bits
    channel         = 2      # 2 = stereo
    
    [icecast2-0]
    bitrateMode     = vbr       # variable bit rate (`cbr' constant, `abr' average)
    quality         = 1.0       # 1.0 is best quality
    format          = mp3       # format. Choose `vorbis' for OGG Vorbis
    bitrate         = 256       # bitrate
    server          = localhost # or IP
    port            = 8000      # port for IceCast2 access
    password        = sonos    # source password to the IceCast2 server
    mountPoint      = sh.mp3  # mount point on the IceCast2 server .mp3 or .ogg
    name            = sonos-helper

Feel free to modify the mountPoint, this is what will appear in Sonos when you tune in to your radio. If you change it here, remember to change it everywhere else. The name parameter (sonos-helper here) does not seem to be relevant.

Monitor

The monitor script is the program that constantly monitors the audio output of the server, and instructs the Sonos system to tune in to the Icecast2 stream when something is playing.

Download it script into your home directory:

    cd
    wget https://raw.githubusercontent.com/gpothier/sonoscast/master/monitor.py
    
Don't forget to replace sonos-helper in RADIO_URI by your server's IP address or host name.        

Important: you may have to edit this line:

    SINK_NAME = 'auto_null'

It is the name assigned by pulseaudio to your audio output and depends on your hardware. Try running the monitor by hand:

    cd
    PYTHONPATH=SoCo:python-pulseaudio python monitor.py

If the output looks like this:

    Pulseaudio connection ready...
    ------------------------------------------------------------
    index: 0
    name: auto_null
    description: Built-in Audio Analog Stereo

then you are all set: the important thing is that the name line matches the SINK_NAME. If you get another name, you have to use that name as the SINK_NAME (eg. alsa_output.pci-0000_00_1b.0.analog-stereo).

Finally, set up the system so that all the stuff starts up at boot. Edit /etc/rc.local and put this before the "exit 0" line:

    /usr/bin/X &
    su gpothier -l -c darkice &
    su gpothier -l -c "DISPLAY=:0 /home/gpothier/virtualenvs/leapcast/bin/leapcast" &
    su gpothier -l -c "PYTHONPATH=/home/gpothier/SoCo:/home/gpothier/python-pulseaudio python /home/gpothier/monitor.py" &

Don't forget to replace gpothier by the your username everywhere.

Yeah, this is not the cleanest way to launch services at boot... 

Reboot, and you should be done! Press the Send to TV button in your phone's YouTube app, you should now see an entry called "leapcast". Select it, wait for the connection to establish, and your should hear the audio on your Sonos system.


