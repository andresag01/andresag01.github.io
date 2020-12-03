---
layout: post
title:  "Managing Audio in i3 with PulseAudio"
date:   2020-12-03 11:21:09 +0100
---

I recently switched to i3 after using GNOME for a few years. Unlike many window
managers, i3 is very bare-bones and its default installation does not include
functionality to manage audio through a graphical interface like you would in
GNOME. I fixed the problem by modifying my i3 configuration instead of
installing a graphical application such as `gnome-control-center`. Here is how
I did it using
[PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/#:~:text=PulseAudio%20is%20a%20sound%20system,your%20application%20and%20your%20hardware.).

Lets look at the easy part first: mute audio and increase/decrease volume. With
a bit of Googling, you can actually find a lot of suggestions for how to do
this. I chose to do it through i3 by mapping the F10, F11 and F12 keys to
PulseAudio commands that mute, decrease volume and increase volume
respectively. I added the following lines to my i3 config file:

```
bindsym $mod+F10 exec pactl set-sink-mute @DEFAULT_SINK@ toggle # Mute
bindsym $mod+F11 exec pactl set-sink-volume @DEFAULT_SINK@ -5%  # Up
bindsym $mod+F12 exec pactl set-sink-volume @DEFAULT_SINK@ +5%  # Down
```

The volume is increased/decrease by 5% for each time that the commands are
issued. A small matter with PulseAudio is that you can actuall set the volume
above 100%!

A slightly harder problem I had was to changing the output device. For example,
my computer has two output devices for audio: a speaker and the headphones. I
wanted an easy and quick way to change the output audio device (or sink in
PulseAudio terminology) with a keystroke (or two). So I mapped the F9 key to
execute a bash script which does just that. Here is the relevant line in my i3
configuration:

```
bindsym $mod+F9  exec ~/.config/i3/toggle_sink.sh
```

The script relies on PulseAudio commands and is relatively simple. It does
three things:

1. Query a list of possible output sinks.
1. Update the current `DEFAULT_SINK` to the next available output sink in the
previously queried list.
1. Redirect all currently playing audio streams to the new `DEFAULT_SINK`.

The current `DEFAULT_SINK` is the sink that the mute and increase/decrease
audio commands will affect. Also, simply changing the `DEFAULT_SINK` does not
cause the currently playing audio to be automatically redirected to that sink.

Whenever the script is executed, all sound is redirected to the next sink in
the list. So repeatedly pressing $Mod+F9 effectively lets me cycle through the
audio output devices with a couple of keystrokes!

```bash
#! /usr/bin/env bash

set -eu

# Get the ID for the current DEFAULT_SINK
defaultSink=$(pactl info | grep "Default Sink: " | awk '{ print $3 }')

# Query the list of all available sinks
sinks=()
i=0
while read line; do
    index=$(echo $line | awk '{ print $1 }')
    sinks[${#sinks[@]}]=$index

    # Find the current DEFAULT_SINK
    if grep -q "$defaultSink" <<< "$line"; then
        defaultIndex=$index
        defaultPos=$i
    fi

    i=$(( $i + 1 ))
done <<< "$(pactl list sinks short)"

# Compute the ID of the new DEFAULT_SINK
newDefaultPos=$(( ($defaultPos + 1) % ${#sinks[@]} ))
newDefaultSink=${sinks[$newDefaultPos]}

# Update the DEFAULT_SINK
pacmd set-default-sink $newDefaultSink

# Move all current playing streams to the new DEFAULT_SINK
while read stream; do
    # Check whether there is a stream playing in the first place
    if [ -z "$stream" ]; then
        break
    fi

    streamId=$(echo $stream | awk '{ print $1 }')
    pactl move-sink-input $streamId @DEFAULT_SINK@
done <<< "$(pactl list short sink-inputs)"
```
