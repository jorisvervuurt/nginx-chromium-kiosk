# nginx-chromium-kiosk 
The instructions below allow you to setup a kiosk using a Raspberry Pi and a Pimoroni HyperPixel XP 4.0 Square Touch display. The Pi will silently boot into the OS. A local NGINX server will then be started on port `8080`, serving files from the `/var/www/nginx-chromium-kiosk` directory. Once NGINX has started, Chromium is launched in kiosk mode.

**CAUTION:** it is assumed that these instructions are ***not*** used on an existing installation!

## Requirements
- [Raspberry Pi](https://www.raspberrypi.com)
- [Pimoroni HyperPixel XP 4.0 Square Touch](https://shop.pimoroni.com/products/hyperpixel-4-square?variant=30138251444307) display
- An SD card flashed using [Raspberry Pi Imager](https://www.raspberrypi.com/software/):
    - Choose Rasberry Pi OS Lite (you can find it under the `Raspberry Pi OS (other)` menu; use the 64-bit version if you have a Raspberry Pi that supports it)
    - Enable SSH access
    - Choose a username and set a password
    - Configure Wifi if you're not using Ethernet

*Note:* if you see `[USERNAME]` (the username as configured in Raspberry Pi Imager) and/or `[IP_ADDRESS]` (the IP address of the Pi) placeholders in the commands below, replace them with the actual values.

## Step one: configure the kernel
1. Connect the display to the Pi, insert the SD card and boot the Pi<br>
    *Note:* if you're booting the device for the first time, it will reboot once or twice while it sets everything up; be patient!

2. SSH into the Pi:
    ```
    $ ssh [USERNAME]@[IP_ADDRESS]
    ```

3. Update the OS:
    ```
    $ sudo apt update && sudo apt full-upgrade
    ```

4. Edit the `/boot/cmdline.txt` file and add the following:<br>
    ```
    noatime logo.nologo quiet vt.global_cursor_default=0 consoleblank=1
    ```

5. Edit the `/boot/config.txt` file:
    1. Add the following at the end of the file:
        ```
        # Disable splash screen
        disable_splash=1

        # Set GPU memory to 128 MB
        gpu_mem=128

        # Enable HyperPixel XP 4.0 Square Touch display
        dtoverlay=vc4-kms-dpi-hyperpixel4sq
        ```
    2. Optionally, disable Bluetooth by adding the following at the end of the file:
        ```
        # Disable Bluetooth
        dtoverlay=disable-bt
        ```
    3. Optionally, disable Wifi by adding the following at the end of the file:
        ```
        # Disable Wifi
        dtoverlay=disable-wifi
        ```

## Step two: disable unnecessary `systemd` services
If you chose to disable Bluetooth in the previous step, we can now also disable the related `systemd` services:

```
$ sudo systemctl disable hciuart
$ sudo systemctl disable bluetooth
```

## Step three: software installation
1. Install Git, X11, Openbox, NGINX and Chromium:
    ```
    $ sudo apt install git xserver-xorg x11-xserver-utils xinit xloadimage openbox nginx chromium-browser
    ```

2. Stop the NGINX `systemd` service from starting automatically:
    ```
    $ sudo systemctl stop nginx
    $ sudo systemctl disable nginx
    ```

3. Create a dedicated user/group and add yourself to that group:
   ```
   $ sudo useradd nginx-chromium-kiosk -r -U -m
   $ sudo usermod -a -G nginx-chromium-kiosk [USERNAME]
   $ newgrp nginx-chromium-kiosk
   ```

4. Edit `/etc/systemd/system/getty@tty1.service.d/autologin.conf` and replace it's contents with the following:
    ```
    [Service]
    ExecStart=
    ExecStart=-/sbin/agetty --autologin nginx-chromium-kiosk %I $TERM
    ```

5. Clone this repository and navigate to it:
    ```
    $ git clone https://github.com/jorisvervuurt/nginx-chromium-kiosk.git nginx-chromium-kiosk
    $ cd nginx-chromium-kiosk
    ```

6. Configure NGINX:<br>
    1. Edit `/etc/nginx/nginx.conf` and update the value of the `user` option:
        ```
        user nginx-chromium-kiosk;
        ```

    2. Create the directory where files will be served from:
        ```
        $ sudo mkdir /var/www/nginx-chromium-kiosk
        $ sudo chown -R nginx-chromium-kiosk: /var/www/nginx-chromium-kiosk
        $ sudo chmod g+srwx /var/www/nginx-chromium-kiosk
        ```

    2. Enable the site:
        ```
        $ sudo rm -f /etc/nginx/sites-enabled/default
        $ cp -R nginx/www/* /var/www/nginx-chromium-kiosk
        $ sudo cp nginx/nginx-chromium-kiosk /etc/nginx/sites-available/nginx-chromium-kiosk
        $ sudo ln -s /etc/nginx/sites-available/nginx-chromium-kiosk /etc/nginx/sites-enabled/nginx-chromium-kiosk
        ```

7. Install the `systemd` service:
    ```
    $ sudo cp systemd/nginx-chromium-kiosk.service /etc/systemd/system/nginx-chromium-kiosk.service
    $ sudo cp sudoers/nginx-chromium-kiosk /etc/sudoers.d/nginx-chromium-kiosk
    ```


8. Configure Openbox:
    ```
    $ sudo mkdir -p /home/nginx-chromium-kiosk/.config/openbox
    $ sudo cp openbox/background.png /home/nginx-chromium-kiosk/.config/openbox/background.png
    $ sudo cp openbox/autostart /home/nginx-chromium-kiosk/.config/openbox/autostart
    $ sudo cp bash/.bash_profile /home/nginx-chromium-kiosk/.bash_profile
    $ sudo chown -R nginx-chromium-kiosk: /home/nginx-chromium-kiosk/.config /home/nginx-chromium-kiosk/.bash_profile
    ```

9. Cleanup
    ```
    $ cd ..
    $ rm -R -f nginx-chromium-kiosk
    ``` 

## Final steps

1. Reboot your Pi:
    ```
    $ sudo reboot now
    ```

2. After the Pi has finished booting, you should see a welcome page being displayed on the display.

3. Using an SFTP-client of choice, connect to the Pi and upload your files to the `/var/www/nginx-chromium-kiosk` directory. Make sure to overwrite the existing `index.html` file.

4. Once you're done uploading, restart the `systemd` service:
    ```
    $ sudo systemctl restart nginx-chromium-kiosk
    ```

## Optional extra steps

### Setup a firewall
By default, the files served by NGINX will also be accessible by other devices in the network (via `http://[IP_ADDRESS]:8080`). If you don't want this, the easiest way to prevent this is to setup a firewall.

We'll use [UFW](https://manpages.ubuntu.com/manpages/bionic/en/man8/ufw.8.html) for this, as it's super easy to setup.

1. Install UFW:
    ```
    $ sudo apt install ufw
    ```

2. By default, UFW will block all incoming connections and allow all outgoing connections.<br>
    To prevent us from losing remote access, we need to add an allow rule for SSH:
    ```
    $ sudo ufw allow 22/tcp
    ```

3. Finish by enabling the firewall:
    ```
    $ sudo ufw enable
    ```
