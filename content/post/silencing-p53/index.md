---
aliases: [silencing-p53]
projects: [linux, hacks]
title: Silencing the Lenovo Thinkpad P53 on Linux
authors: [Dainius Jocas]
date: '2021-02-20'
tags: [Linux, Hacks]
categories:
  - Linux
  - Hacks
summary: "A battle story on how I've managed to silence the the Lenovo P53 lapton"
image:
  caption: "[Thinkpad P53](https://www.omgubuntu.co.uk/2019/06/lenovo-thinkpad-p-series-ubuntu-preinstalled)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

## TL;DR

Lenovo Thinkpad P53 is a powerfull laptop but a regular Linux install is just too loud for me. However, it is just a machine and there must be several control knobs that can make the experience better. In this post I'll show my setup that a the laptop is silent most of the time and performant when needed.

## The motivation

To get my daily work done I spin many fat JVMs like Elasticsearch, Kafka, GraalVM native-image tool, etc. To plow through those tasks a year ago I've got a top spec'ed Lenovo Thinkpad P53 laptop:

| Part | Spec |
|-----:|:----|
|CPU | Intel(R) Core(TM) i9-9880H CPU @ 2.30GHz |
|Memory| 128GB (not a mistake here) |
|Graphics Processor | Nvidia Quadro RTX 4000 |

The laptop(!) can be so power hungry that the power charger needs to provide up to 230W (!). Just imagine how loud the fans get when executing computationaly intensive tasks. I simple have to leave the room and close the door until the task is finished.

Anything that is CPU intensive makes the chip hot which spins the fans. By anything here I mean regular web browsing (looking at you JIRA), or a simple conference call (looking at you Google Meet and Zoom). I have some doubts that Lenove done the best job possible with the fans but I leave that on their concience.

Being a powerfull machine does not mean that all cores have to work full speen when working on non-demanding tasks. So, I want to setup the P53 to be a silent Linux machine.

## Prehistory

When I got the laptop the first thing I've done was that I've installed Ubuntu since it is officially supported. In a very short time I've discovered that at that time it was impossible to silence the laptop to a satisfiable level: Linux just had no drivers that could silence all(!) the fans[^1].

The hope was lost: I had to boot up Windows, setup WSL, and setup my environment there. All because several drivers were missing. It's worth to mention that I've set up Windows after ~10 years. It was OEM install with all the updates. However, for some reason to me it was not a surprise that Windows 10 could not properly handle the sleep-wake up cycle (its a laptop after all). But hey, I could disable the Nvidia GPU, then enable it, and everything would be working fine again. I could get stuff done.

However, updating Windows is a thing that one can postpone only that long and when you do let it update with a setup that is a little exotic you might get a broken system that fails to boot up on the the next try. Yeah, sure, then just run the system fix scripts and you can continue working.

Of course, it doesn't help that it turned out that the laptop arrived with a faulty motherboard that caused all kinds of troubles among which the most painful was random shutdowns. Also, sometimes even booting up was a real strugle. I guess that poor machine was telling me to stop torturing it and by failing to work gave me some well deserved leisure time.

One day the machine after a Windows update really got stuck during the reboot. A blank screen with fans maxed out and no reaction to any button or to power cable unpluging. Several google searches away and I've discovered that it was a known problem with Thinkpads. So the motherboard got replaced.

Also, even Windows that has a good driver support from everyone involved has the power consumption and fan control settings that are not all that powerfull after all to use and somehow makes me wonder how the software got accepted to be released.

## The setup

The news somehow came to me that Linux kernel 5.11 has the support for the second fan on some thinkpads. It was a signal to roll up my sleeves and get the P53 fans under my control.

The main pieces are the Linux kernel 5.11 and Kubuntu 21.04 prereleased.

Also, there are a couple helper utilities that I've used:
- Fancontrol GUI
- thinkfan (with GUI)
- CPU Power GUI

## Linux kernel

I've installed 5.11 on Ubuntu 20.10 and it should not come to you as a surprise that Nvidia drivers were broken and install from their website failed. So, failed experiment.

Nothing fancy here just the driver that is able to control the second ThinkPad fan. I've used [Mainline tool](https://ubuntuhandbook.org/index.php/2020/08/mainline-install-latest-kernel-ubuntu-linux-mint/) to install the kernel. 

## Kubuntu 21.04

When all hope was lost and I was about to go back to Windows for another half a year I got the last thing to try: why not upgrade Kubuntu to 21.04 which is not even in BETA? It would come with newer kernel and most likely with Nvidia driver support.

It came with kernel 5.10 and Nvidia drivers worked. Unfortunately, the second fan was not detected. Fail.

The last hope was to ask Mainline to install kernel 5.11 and pray that Nvidia drivers are fine with it. Install, reboot, and voila: Nvidia driver works, external screen is detected (though only HDMI), and most importantly both fans are detected and controllable! Hurray!

### Fancontrol

[This](https://github.com/Maldela/fancontrol-gui) is a GUI utility that allows you to set up fan profiles by setting two dots: for example, the temperature at which to start spin the fans and when to max them out.

![Fancontrol in KDE System Setting](fancontrol-system-settings.png)

Note: I've installed it from source code.
Note: there are standalone GUI but I like the integretion into the KDE System Settings.

### Thinkfan

Check the set up instructions [here](https://gist.github.com/Yatoom/1c80b8afe7fa47a938d3b667ce234559).

There is even the GUI for the [thinkfan](https://github.com/scientifichackers/thinkfan-control-gui). This will give a window like this:

![thinkfan GUI](thinkfan-gui.png)

Note the two lines that starts with `Fan`. It means that two thinkpad fans are detected and controled.

Note that with thinkfan it is up to you to write the settings into the file.

### CPU Power GUI

It is a little utility that allows to set the CPU clock limits:
![CPU frequency settings](cpupower-gui.png)

Install from the official repositories.

```bash
sudo apt install cpupower-gui
```

The trick here is that high CPU frequency leads to lots of heat which in turn spins the fans. When the frequency is limited not that much of the heat is created which prevents fans from kicking in. I've noticed that the minimal frequency (800 MHz) is enough to get my work done. The temperature rarely goes up more than 60 degrees celsius.

Of course the machine is noticeably slower, but hey it still a high end CPU and can get some work done. The slowdown is most noticeable when starting up an app. For example, IntelliJ is fast enough.

### GreenWithEnvy

For the Nvidia fans the [GreenWithEnvy](https://gitlab.com/leinardi/gwe) looks promissing but I guess that the Quadro cards are not supported.

## Summary

A year long story. Given the current setup the tasks that are not super demanding can be done on a quiet machine where the only things I hear are my keyboard and the little Marshall Emberton speaker playing my favourite [Rammstein](https://www.youtube.com/watch?v=ZkW-K5RQdzo) [tunes](https://www.youtube.com/watch?v=NeQM1c-XCDc). And when my crazy performance test is setup, I allow the fans to spin at full speed and get the stuff done as fast as it can.


## P.S. Nvidia and Linux

Yeah, a laptop with an Nvidia GPU should be a red flag that prevents installing Linux in the first place. Of course, there are drivers and stuff, but in general good luck to have them properly working. However, Ubuntu gives you the setup that is worked out. 

Also, there are open source drivers called Nouveu but they have problems of their own.

What is worse with this P53 is that ALL video outputs are attached to the discrete GPU (who in their right mind would do it?). This means that if drivers are not properly set-up Linux will able to output only 1 FPS while mouse and some other windows works without any lag. 

Sure, things like [Optimus](https://www.nvidia.com/en-us/geforce/technologies/optimus/technology/) [reverse PRIME](https://forums.developer.nvidia.com/t/the-all-new-outputsink-feature-aka-reverse-prime/129828) but I can only wish you good luck setting up these. 

In summary, I wish there was a modification of the laptop that had only the indegrated graphics.

## P.S.S. Setup on the table

The P53 is a big and fat laptop. Since most of the time I work from home, I don't need to move or touch it that often. Thefore, it can stand on the table as a triangle (see a picture). Having the laptop in such a position allows air to circulate better. Also, in this way it takes less space on the table. On the other hand, I can't use the laptop screen.

![My Table setup](thinkpad-setup.jpg)

## References

- 

## Footnotes

[^1]: I've tried several other distros but had no luck.
