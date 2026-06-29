---
layout: page
title: Local Voice Assistant
description: An Alexa-equivalent running entirely on the Raspberry Pi cluster — no cloud, no audio leaving the house.
img: assets/img/pi-cluster-boards.jpg
importance: 1
category: hardware
---

I wanted an Alexa without Amazon. This is one: the whole assistant — wake word, speech
recognition, a 4-billion-parameter language model, and text-to-speech — runs on the
four-node Raspberry Pi 4 cluster, with no cloud and no audio leaving the house.

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.html path="assets/img/pi-cluster-boards.jpg" title="The four Raspberry Pi 4 nodes" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-4 mt-3 mt-md-0">
        {% include figure.html path="assets/img/pi-cluster-rack.jpg" title="The full rack with cooling and switch" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The cluster the assistant runs on: four Raspberry Pi 4 (8GB) boards, two cooling fans, and a gigabit switch.
</div>

## What it does
Answers questions, gives live weather, sets timers, looks things up on Wikipedia, and
describes what a camera sees — all locally. One node runs the wake word, speech
recognition, and vision; one runs the language model; one does text-to-speech; the
fourth is the cluster control plane and NAS.

## The interesting part
A custom "ok computer" wake word that kept firing on "ok" alone, and what fixing it
taught me about training data and shortcut learning.

[Read the full writeup →]({{ '/blog/2026/building-a-fully-local-voice-assistant-on-a-raspberry-pi-cluster/' | relative_url }})
