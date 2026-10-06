# Table of Contents
- [Setup Guide](#raspberry-pi-4-setup)
    - [1 Initial Setup (from a fresh install via Raspberry Pi Imager)](#1-initial-setup-from-a-fresh-install-via-raspberry-pi-imager)
    - [2 SSH into the Pi from Another Computer](#2-ssh-into-the-pi-from-another-computer)
    - [3 Optimize Boot Time](#3-optimize-boot-time)
    - [4 Verify Network Connectivity (Before Upgrading Packages)](#4-verify-network-connectivity-before-upgrading-packages)
    - [5 Download Required Packages](#5-download-required-packages)
    - [6 Set Up the Git Repo](#6-set-up-the-git-repo)
    - [7 Set Up the Network (Wi-Fi Access Point)](#7-set-up-the-network-wi-fi-access-point)
    - [8 Auto-Redirect Port 80 to the App (Port 4000)](#8-auto-redirect-port-80-to-the-app-port-4000)
    - [9 Set Up Auto-Start on Boot](#9-set-up-auto-start-on-boot)
    - [10 Note: AP vs. Wi-Fi Receiver Mode](#10-note-ap-vs-wi-fi-receiver-mode)
- [Additional Notes](#additional-notes)



# Raspberry Pi 4 Setup

Using Raspberry Pi Lite OS for fast boot time in case of downtime.
> A UPS is included, but fast boot time and proper recovery steps are still essential.

## 1 Initial Setup (from a fresh install via Raspberry Pi Imager)

If ssh does not work, you may need to connect the Pi to an external screen and keyboard to enable it.
1. Connect the Pi to an external screen (micro HDMI) and keyboard.
2. Log in with the username and password created during Imager setup.
3. Enable and start SSH:
   ```bash
   sudo systemctl enable ssh
   sudo systemctl start ssh
   ```

## 2 SSH into the Pi from Another Computer

1. On Raspberry Pi 4 and below (check your model first), the USB-C port is power-only — you'll need an additional Ethernet cable to connect.

## 3 Optimize Boot Time

1. Open the quick system settings GUI to connect to a WiFi network for update purposes:
   ```bash
   sudo raspi-config
   ```
2. Disable unneeded services for a faster boot:
   ```bash
   sudo systemctl disable NetworkManager-wait-online.service
   sudo systemctl mask NetworkManager-wait-online.service
   sudo systemctl disable cloud-init-local.service
   sudo systemctl disable cloud-init-main.service
   sudo systemctl disable cloud-config.service
   sudo systemctl disable cloud-final.service
   sudo systemctl disable apt-daily-upgrade.service
   sudo systemctl disable apt-daily.service
   ```
3. Note: boot time will still be around 20 seconds.

## 4 Verify Network Connectivity (Before Upgrading Packages)
 
`apt full-upgrade` requires a working internet connection — check this first if you hit connection errors.
 
1. Check if Wi-Fi is scanning networks:
```bash
   sudo iwlist wlan0 scan | grep ESSID
```
2. If nothing comes back (Wi-Fi is down), bring it back up:
```bash
   sudo rfkill unblock wifi
   sudo ip link set wlan0 up
   sudo nmcli radio wifi on
   sudo systemctl restart wpa_supplicant
   sudo systemctl restart NetworkManager
```
3. Re-run the scan command from step 1 to confirm networks now appear, then proceed.

## 5 Download Required Packages

1. Update and install system packages:
   ```bash
   sudo apt full-upgrade
   sudo apt install git nodejs npm curl hostapd dnsmasq dhcpcd5 iptables nginx -y
   sudo npm install yarn -g
   ```
2. Install nvm and Node LTS:
   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
   source ~/.bashrc
   nvm install --lts
   nvm use node
   ```

## 6 Set Up the Git Repo

1. Create a directory and clone the repo:
   ```bash
   mkdir apps
   cd apps
   git clone https://github.com/shipshape-marine/bosun.git bosun
   cd bosun
   yarn install
   yarn start:all
   ```
2. Force-stop the process once the initial build completes.

3. Copy '.env' into backend

## 7 Set Up the Network (Wi-Fi Access Point)

1. Stop the AP-related services before configuring them:
   ```bash
   sudo systemctl stop hostapd
   sudo systemctl stop dnsmasq
   ```
2. Mark `wlan0` as unmanaged by NetworkManager. Edit:
   ```bash
   sudo nano /etc/NetworkManager/conf.d/unmanaged.conf
   ```
   Add:
   ```ini
   [keyfile]
   unmanaged-devices=interface-name:wlan0
   ```
3. Set a static IP for `wlan0`. Edit:
   ```bash
   sudo nano /etc/dhcpcd.conf
   ```
   Add:
   ```ini
   interface wlan0
   static ip_address=192.168.4.1/24
   nohook wpa_supplicant
   ```
4. Restart the dhcpcd service:
   ```bash
   sudo systemctl restart dhcpcd
   ```
5. Configure hostapd. Edit:
   ```bash
   sudo nano /etc/hostapd/hostapd.conf
   ```
   Add:
   ```ini
   interface=wlan0
   driver=nl80211

   ssid=Proa_II

   hw_mode=g
   channel=6

   country_code=SG

   wmm_enabled=1

   auth_algs=1
   ignore_broadcast_ssid=0

   wpa=2
   wpa_passphrase=password

   wpa_key_mgmt=WPA-PSK
   rsn_pairwise=CCMP
   ```
6. Point the hostapd daemon at that config. Edit:
   ```bash
   sudo nano /etc/default/hostapd
   ```
   Add:
   ```ini
   DAEMON_CONF="/etc/hostapd/hostapd.conf"
   ```
7. Back up and replace the dnsmasq config:
   ```bash
   sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
   sudo nano /etc/dnsmasq.conf
   ```
   Add:
   ```ini
   interface=wlan0

   dhcp-range=192.168.4.2,192.168.4.100,255.255.255.0,24h

   dhcp-option=3,192.168.4.1
   dhcp-option=6,192.168.4.1

   address=/#/192.168.4.1
   ```
8. Unmask and enable the AP services, then restart networking and start them:
   ```bash
   sudo systemctl unmask hostapd
   sudo systemctl enable hostapd
   sudo systemctl enable dnsmasq

   sudo systemctl restart wpa_supplicant
   sudo systemctl restart NetworkManager
   sudo systemctl start hostapd
   sudo systemctl start dnsmasq
   ```
9. Verify `wlan0` is running in AP mode:
    ```bash
    iw dev wlan0 info
    ```
    If the type isn't `AP`, run `sudo reboot` and check again.

10. Give access to user to run nmcli over Node
    ```bash
    sudo nano /etc/polkit-1/rules.d/50-nmcli.rules
    ```
    Add:
    ```javascript
    polkit.addRule(function(action, subject) {
        if (action.id.indexOf("org.freedesktop.NetworkManager.") === 0 &&
            subject.user == "admin") {
            return polkit.Result.YES;
        }
    });
    ```
11. Restart polkit:
    ```bash
    sudo systemctl restart polkit
    ```


## 8 Auto-Redirect Port 80 to the App (Port 4000)

1. Create the redirect service:
   ```bash
   sudo nano /etc/nginx/sites-available/solarproa
   ```
   Add:
   ```ini
   server {
    listen 80;
    server_name solarproa.local;

    location / {
        proxy_pass http://127.0.0.1:4000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
   }
   ```
2. Enable and start it:
   ```bash
   sudo ln -s /etc/nginx/sites-available/solarproa /etc/nginx/sites-enabled/
   ```

3. Test it:
   ```bash
   sudo nginx -t
   ```

4. Restart nginx service
   ```bash
   sudo systemctl restart nginx
   ```

   When connected to the same network as the raspberry pi, go to http://solarproa.local

## 9 Set Up Auto-Start on Boot

1. Create the app service:
   ```bash
   sudo nano /etc/systemd/system/bosun.service
   ```
   Add:
   ```ini
    [Unit]
    Description=Bosun
    After=network-online.target
    Wants=network-online.target

    [Service]
    Type=simple
    User=admin
    WorkingDirectory=/home/admin/apps/bosun/backend

    Environment=NODE_ENV=production
    Environment=NVM_DIR=/home/admin/.nvm

    ExecStart=/bin/bash -c 'source "$NVM_DIR/nvm.sh" && nvm use default && exec yarn start'

    Restart=always
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
   ```
2. Enable and start it:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable bosun
   sudo systemctl start bosun
   ```
3. Check the logs:
   ```bash
   journalctl -u bosun -f
   ```

## 10 Note: AP vs. Wi-Fi Receiver Mode

The Wi-Fi chip can only act as **either** an access point **or** a receiver at one time — switch back to receiving, or use an external Wi-Fi adapter.

1. Check the new Wi-Fi receiver's interface name (should be `wlan1`).
2. Connect to a network on that interface:
   ```bash
   sudo nmcli device wifi connect "YourSSID" password "YourPassword" ifname wlan1
   ```

# Additional Notes

- While raspberry pi 4 is used as the server, the setup can be ran on any windows / linux machine, as long as the master esp32 node can be physically connected to the machine