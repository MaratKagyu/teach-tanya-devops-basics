# Setup base environment (Debian)
Usually when setting up a new server on the internet, you receive
an email with **IP address** and **root password**

## 1. Login to the server
```bash

ssh root@192.168.2.149
# and enter the password you received
```

## 2. Create a new user
Create a new User:
```bash

adduser tatiana
# Fill all the requested data and the password 
```

Setup sudo:
```bash

# Install SUDO if it isn't there in the first place
apt-get update
apt-get install sudo
 
# Add the user to passwordless sudoers
echo "tatiana  ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/tatiana

# Switch to you user, and try sudo command
su - tatiana
sudo -s

# If you get elevated to root, then everything is fine

```

## 3. Setup SSH
Obtain a public SSH key and save it at `~/.ssh/authorized_keys`
```bash
# Switch to your user
su - tatiana

# Create ~/.ssh if it doesn't exist
mkdir ~/.ssh
nano ~/.ssh/authorized_keys
```
Disconnect from the server and try logging in as you user 
```bash
# On your localhost
ssh tatiana@192.168.2.149
```
**_IMPORTANT!!!_** Please make sure you logged in **_without password_**.

**Disable login with password**
```bash
sudo nano /etc/ssh/ssh_config
# Change "PasswordAuthentication" to "no"
# Restart sshd
sudo service sshd restart
```

## 4. Fix warning about the locale
```bash
# Reconfigure locales
sudo dpkg-reconfigure locales

# Erase /etc/default/locale
sudo -s
echo "" > /etc/default/locale
echo "LC_CTYPE=\"en_US.UTF-8\"" >> /etc/default/locale
echo "LC_ALL=\"en_US.UTF-8\"" >> /etc/default/locale
echo "LANG=\"en_US.UTF-8\"" >> /etc/default/locale
echo "LANGUAGE=\"en_US:en\"" >> /etc/default/locale

# Exit the machine and login back again
```

