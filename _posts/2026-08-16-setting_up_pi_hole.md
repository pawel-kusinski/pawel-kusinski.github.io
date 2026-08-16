---
title: "Setting Up Pi-hole"
excerpt: "Notes on setting up Pi-hole on a Raspberry Pi, from installing Raspberry Pi OS to configuring a static IP address and accessing the Pi-hole dashboard."
categories:
  - Misc
tags:
  - pi-hole
  - raspberry-pi
---

## Introduction
[Pi-hole](https://pi-hole.net/) is ad- and tracker-blocking software that can be installed on
a single host in the network. All traffic goes through Pi-hole,
allowing it to block unwanted traffic without the need to
install ad- and tracker-blocking software on individual devices.

In this post, I collected my notes on how to get it up and running
on a Raspberry Pi.

## Hardware Used

1. Raspberry Pi. It can be any PC that can run Linux. Raspberry Pi
is a good choice because of its small form factor. I have a
Raspberry Pi 3 Model B that has been collecting dust for years. Now
it's time to make use of it.

2. MicroSD Card. A minimum of 32 GB of storage is recommended.

## Install Operating System
Install Raspberry Pi OS on the microSD card:
1. [Download](https://www.raspberrypi.com/software/) and install Raspberry Pi Imager.
I downloaded it as an AppImage. To start the Raspberry Pi Imager AppImage, run the following commands:
```bash
chmod +x imager_2.0.10_amd64.AppImage
sudo ./imager_2.0.10_amd64.AppImage
```
2. Using Raspberry Pi Imager, install Raspberry Pi OS on the microSD card by following the steps in [this guide](https://www.raspberrypi.com/documentation/computers/getting-started.html#imager-install).
* I installed Raspberry Pi OS **Full**, as my microSD card is large enough and I want to make sure all recommended software is already installed. This variant of the OS can be selected by choosing "Raspberry Pi OS (other)" on the "Choose Operating System" screen of the Imager app.
* I chose non-5GHz WiFi, as I wasn't sure whether my Raspberry Pi supports 5GHz WiFi.
* Make sure an SSH connection is set up via Imager, so that we can SSH into the Raspberry Pi later. Choose public key authentication. If a public key does not exist on the host machine from which we SSH into the Raspberry Pi, it can be generated as follows:
```bash
ssh-keygen -t ed25519 -C "your comment here"
```
After the OS is installed, insert the microSD card into the Raspberry Pi, and then power it up.

## Access Raspberry Pi remotely
When the Raspberry Pi is powered up, SSH into it using the command below:
```bash
$ ssh <user>@<ip_address>
```
`<user>` is the username created in the "Customisation: Choose username" step in the Imager app.
The easiest way to get the `<ip_address>` is to log into the router's control panel and look for
a new device that appeared on the network. This can usually be found by looking for the hostname that was
created in the Imager app as well.

## Set static IP address
A static IP address is required by Pi-hole. The static address should be set on the Raspberry Pi. I did that using `nmtui` and following the steps in this [tutorial](https://raspberry.tips/en/raspberrypi-einsteiger/raspberry-pi-static-ip-address).

The same IP address must be reserved, or removed from the DHCP pool, in the router settings. The way this can be done
depends on the router manufacturer. Usually, however, you can access your router by entering [https://192.168.1.1/](https://192.168.1.1/) and then change this setting using the web GUI.

After applying the changes, reboot the Raspberry Pi:
```bash
sudo reboot
```

## Install and Setup Pi-hole
After the Raspberry Pi has rebooted, log in and run the following command:
```bash
sudo curl -sSL https://install.pi-hole.net | bash
```
More details on [Pi-hole's website](https://github.com/pi-hole/pi-hole/#one-step-automated-install)

In the router settings, this time change the DNS and set its IP address to the Raspberry Pi's IP address.

## Access Pi-hole Dashboard
Go to [http://192.168.1.173/admin/](http://192.168.1.173/admin/) in the web browser.
Done!