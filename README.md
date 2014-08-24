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

Create monitor.sh in your home directory with the following content. This is the program that constantly monitors the audio output of the server, and instructs the Sonos system to tune in to the Icecast2 stream when something is playing. It mostly comes from http://freshfoo.com/blog/pulseaudio_monitoring.
  
    import sys
    from Queue import Queue
    from ctypes import POINTER, c_ubyte, c_void_p, c_ulong, cast
    
    # From https://github.com/Valodim/python-pulseaudio
    from pulseaudio.lib_pulseaudio import *
    
    import soco
    
    # edit to match your sink
    SINK_NAME = 'auto_null'
    METER_RATE = 10
    MAX_SAMPLE_VALUE = 127
    DISPLAY_SCALE = 2
    MAX_SPACES = MAX_SAMPLE_VALUE >> DISPLAY_SCALE
    RADIO_URI = "x-rincon-mp3radio://sonos-helper:8000/sh.mp3"
    
    class PeakMonitor(object):
    
        def __init__(self, sink_name, rate):
            self.sink_name = sink_name
            self.rate = rate
    
            # Wrap callback methods in appropriate ctypefunc instances so
            # that the Pulseaudio C API can call them
            self._context_notify_cb = pa_context_notify_cb_t(self.context_notify_cb)
            self._sink_info_cb = pa_sink_info_cb_t(self.sink_info_cb)
            self._stream_read_cb = pa_stream_request_cb_t(self.stream_read_cb)
    
            # stream_read_cb() puts peak samples into this Queue instance
            self._samples = Queue()
    
            # Create the mainloop thread and set our context_notify_cb
            # method to be called when there's updates relating to the
            # connection to Pulseaudio
            _mainloop = pa_threaded_mainloop_new()
            _mainloop_api = pa_threaded_mainloop_get_api(_mainloop)
            context = pa_context_new(_mainloop_api, 'peak_demo')
            pa_context_set_state_callback(context, self._context_notify_cb, None)
            pa_context_connect(context, None, 0, None)
            pa_threaded_mainloop_start(_mainloop)
    
        def __iter__(self):
            while True:
                yield self._samples.get()
    
        def context_notify_cb(self, context, _):
            state = pa_context_get_state(context)
    
            if state == PA_CONTEXT_READY:
                print "Pulseaudio connection ready..."
                # Connected to Pulseaudio. Now request that sink_info_cb
                # be called with information about the available sinks.
                o = pa_context_get_sink_info_list(context, self._sink_info_cb, None)
                pa_operation_unref(o)
    
            elif state == PA_CONTEXT_FAILED :
                print "Connection failed"
    
            elif state == PA_CONTEXT_TERMINATED:
                print "Connection terminated"
    
        def sink_info_cb(self, context, sink_info_p, _, __):
            if not sink_info_p:
                return
    
            sink_info = sink_info_p.contents
            print '-'* 60
            print 'index:', sink_info.index
            print 'name:', sink_info.name
            print 'description:', sink_info.description
    
            if sink_info.name == self.sink_name:
                # Found the sink we want to monitor for peak levels.
                # Tell PA to call stream_read_cb with peak samples.
                print
                print 'setting up peak recording using', sink_info.monitor_source_name
                print
                samplespec = pa_sample_spec()
                samplespec.channels = 1
                samplespec.format = PA_SAMPLE_U8
                samplespec.rate = self.rate
    
                pa_stream = pa_stream_new(context, "peak detect demo", samplespec, None)
                pa_stream_set_read_callback(pa_stream,
                                            self._stream_read_cb,
                                            sink_info.index)
                pa_stream_connect_record(pa_stream,
                                         sink_info.monitor_source_name,
                                         None,
                                         PA_STREAM_PEAK_DETECT)
    
        def stream_read_cb(self, stream, length, index_incr):
            data = c_void_p()
            pa_stream_peek(stream, data, c_ulong(length))
            data = cast(data, POINTER(c_ubyte))
            for i in xrange(length):
                # When PA_SAMPLE_U8 is used, samples values range from 128
                # to 255 because the underlying audio data is signed but
                # it doesn't make sense to return signed peaks.
                self._samples.put(data[i] - 128)
            pa_stream_drop(stream)
    
    def main():
        zones = list(soco.discover())
        zones.sort(key = lambda zone: zone.ip_address)
        
        zone = zones[0]
        print "Sonos zone: %s" % zone.player_name
    
        monitor = PeakMonitor(SINK_NAME, METER_RATE)
        
        sounding = False
        blank_time = 0
        
        for sample in monitor:
            if sample > 0:
                blank_time = 0
                if not sounding:
                    sounding = True
                    print "Now sounding"
                    track = zone.get_current_track_info()
                    transport = zone.get_current_transport_info()
                    if transport["current_transport_state"] == "STOPPED" or track["uri"] != RADIO_URI:
                        print "Tuning in"
                        zone.play_uri(RADIO_URI)
            else:
                blank_time += 1
                if sounding and blank_time > 5*METER_RATE:
                    print "Not sounding anymore"
                    sounding = False
                    #zone.next()
    
    if __name__ == '__main__':
        main()

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


