---
layout: post
title:  How FRIENDS taught me why to choose my distribution wisely.
date: 2018-03-15 16:40:16
description: "Why to choose your probability distribution wisely"
tags: statistics quirks python bash friends monte-carlo-simulation
categories: data-science machine-learning
---

Like most 90’s kids I am also a crazy fan of the American television sitcom Friends or may be a bit crazier than others though I would leave that to the readers to decide. And like most die hard fans I too have the habit of playing random episodes of Friends while doing other chores. Even though I have watched all the 234 episodes a gazillion times I still enjoy the humour that each character plays in its own style. But deciding which episode to play at random is tedious when you have 234 episode choices to pick from. So I thought wouldn’t it be cool if I let my computer choose which episode to play at random for me.

Aha! What an opportunity to put my skills to good use. The task was pretty simple, choose a random episode from a random season and play it in VLC media player. So I researched on how to choose random file from a random folder in command line and implemented that solution as a small bash script using `sort` command


```
#!/bin/bash
dir="../Friends"
season=`ls ${dir} | sort -R | head -1`
ep=`ls "${dir}/${season}/"*.mkv | sort -R | head -1`
/Applications/VLC.app/Contents/MacOS/VLC "$ep"
```

and set its alias to `friends` so that I could play a random episode just by typing `friends` in the command line.
I was happy that a random friends episode was just a couple of keystrokes away. And I started enjoying episodes from this self written script but my happiness didn’t last long. I observed that the more episodes I played the more they kept repeating which, I thought, ought to happen because under the hood `sort` method throws 234 faced die corresponding to each episode to play and you will sometimes get same number more than once but it wasn’t always the case. There were some episodes that were repeating quite too often and some not at all.

So to understand what’s happening here let’s recall some high school probability. If you throw a unbiased die there is an equal chance of getting a number from 1 to 6 because all the numbers have the same probability of 1/6 i.e. ideally you will get every number once in 6 throws. So it has a uniform distribution. However, if you have a biased die it may not be the case. This made me suspicious about the working of the script I implemented.
To investigate this I visualised the output of the script by running it a million times and recording the output and found something unexpected.

*For data nerds out there this process of running an experiment multiple times is called Montè Carlo simulations*

|<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-8-1.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>|
| :-: |
| Bar plot: Episode vs. Frequency of episodes |


Apparently the 234 faced die that the sort method of the script used was not completely unbiased as I suspected earlier. As you can see it has a non uniform distribution across episodes which explains why it ended up playing some episodes more often than others.

So, I had to throw away the whole script and think of an alternate solution starting from the underlying distribution. Since I didn’t know how to generate a uniform distribution from command line that’s where Python came to the rescue. I wrote another script, this time in Python, to generate a uniform distribution and pick a number randomly from that distribution which was like choosing a number from an *unbiased* 234 faced die.

```
#!/usr/bin/env python
import pandas as pd
import numpy as np
import subprocess
data = pd.read_csv('../Friends/frnds.csv')
p = int(np.random.uniform(0,234))
episode = data.iloc[p,:][1]
p = subprocess.Popen(["/Applications/VLC.app/Contents/MacOS/VLC", episode], stdout=subprocess.PIPE)
p.communicate()
```

This worked like a charm and gave the following distribution

| <div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-8-2.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>|
| :-: |
| Bar plot: Episode vs. Frequency of episodes |

which seemed pretty uniform in a million trials. Thanks to the uniform distribution life was sweet again and that’s how I learnt why it’s important to choose your distribution wisely.
