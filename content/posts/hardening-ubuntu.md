---
title: "Hardening Ubuntu"
date: 2020-11-01T17:26:57+02:00
tags:
  - ubuntu
  - ufw
---

This is a short list, largely borrowed from [digitalocean](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04)

Log in with your provided root credentials
```bash
ssh root@123.123.123.123
```
Create a user (just hit enter and skip the info if you want)
```bash
adduser username
```
Enable the user to use sudo
```bash
usermod -aG sudo username
```
Disable root SSH login
```bash
nano /etc/ssh/sshd_config
```
Modify the file like below. Pick a high port number between 1024 and 34627. Hit ctrl-x, 'Y' and finally hit enter to save.
```bash  
PermitRootLogin no  
Port 12345
```
Finally reboot and log in with the new settings
```bash
reboot  
ssh username@123.123.123.123 -p 12345
```
From now on we might need to use 'sudo' to run certain commands.
You might want to consider using a ssh key instead. As it is more secure.  
[TODO] Insert that info.

If you do not use a external firewall you might want to use a software based one.  
Check digital oceans [guide](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04) for more details.  
Here are some example commands

```bash
   sudo ufw allow 12345  
   sudo ufw default deny incoming  
   sudo ufw default allow outgoing  
   sudo ufw enable  
   sudo ufw status

   sudo ufw deny  
   sudo ufw status numbered  
   sudo ufw delete 2  
   sudo ufw delete allow OpenSSH  
   sudo ufw status verbose  
   sudo ufw reset
```
Lets reboot and check if everything is working with 
```bash
sudo reboot
```

Update everything
```bash
sudo apt-get update 
sudo apt-get upgrade  
sudo apt-get dist-upgrade
```