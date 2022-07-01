---
layout: post
title: "How to replace Firefox Snap by classical Firefox deb package on Ubuntu 22.04?"
comments: true
taxonomies: 
  tags: [ubuntu, firefox]
---

With Ubuntu 22.04, Canonical started to ship Firefox packaged as a Snap instead of a Deb.

Here are the explanations to replace the Firefox Snap package by the classical deb package provided by the Mozilla team on Ubuntu 22.04.

<!-- more -->

## Uninstall permanently the transitional Firefox deb package

This package replace the Firefox deb package and install the Snap package.
You don't need it anymore.

    sudo apt remove firefox

If you don't remove it, the dev Firefox package you will install later will be overridden again and again by this one.

## Copy your Firefox snap profile

Your Firefox profile is located **inside** the snap install location.
So you have to move it to use it with the new Firefox installation.

    cp -R ~/snap/firefox/.mozilla ~/

## Install permanently the deb package of Firefox

    sudo add-apt-repository ppa:mozillateam/ppa
    sudo apt install -t 'o=LP-PPA-mozillateam' firefox

Then, create this new file:

    sudo nano /etc/apt/preferences.d/firefox

Add the following content:

```
Package: firefox*
Pin: release o=LP-PPA-mozillateam
Pin-Priority: 501
```

This file will prevent Ubuntu from restoring the transitional Firefox package.

## Run Firefox 

Run Firefox and choose your legacy profile:

    firefox --profileManager

You just new to run this command once.

## Remove the snap

Finally, you can remove the Firefox snap package:

    sudo snap remove firefox
