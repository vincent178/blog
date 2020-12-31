---
title: "Archlinux on G14"
date: 2020-08-04T03:36:04+08:00
draft: true
---

sudo pacman-mirrors -c China -i

sudo pacman -Syyu

sudo pacman -S neovim ranger neofetch

$ sudo bash -c "echo blacklist nouveau > /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
$ sudo bash -c "echo options nouveau modeset=0 >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf"


