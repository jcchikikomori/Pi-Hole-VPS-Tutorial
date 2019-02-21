# Setting up a Pi-Hole VPN on a VPS

# Introduction

The purpose of this guide is to document the steps I take to set up [Pi-Hole](https://pi-hole.net/) on a VPS. The ultimate goal is to have an ad-blocker that can work with any device connected to the VPS with a VPN connection. 

The software in this tutorial (with the exception of Wireguard) will be installed using Docker containers and `docker-compose`. Using a `docker-compose.yml` file will allow you to quickly deploy your Pi-Hole VPN to any VPS as necessary.

After completing this tutorial you will have:

- Your own recursive DNS resolver using **unbound**
- A **Pi-Hole** accessible from anywhere
- A VPN server using **Wireguard**
- Auto-updating containers using **Watchtower**

## Prerequisites

In order to follow this tutorial you will need to have a VPS with at least 512 MB of memory, although I would personally recommend at least 1 GB if you plan on having a large number of blocklists. This guide assumes that you are using Ubuntu 18.10 and Pi-Hole Version 4.2. Other distros will mostly likely work, but I have only tested the steps covered in this tutorial on Ubuntu 18.10.

Companies like **DigitalOcean** provide [tutorials](https://www.digitalocean.com/docs/droplets/how-to/create/) for creating a VPS on their servers.

## Initial Server Setup

We will be using `ssh` to remotely log into the VPS and configure it. If you are on a Unix-based operating system, it should already be installed. If you are Windows, you will need to install [PuTTY](http://www.putty.org/). Make sure you know your server's IP address and login credentials.

This section essentially covers all of the steps from [**DigitalOcean**'s tutorial for setting up Ubuntu](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04), but with a few differences. We will be creating a specific user, `pi`, that will use for logging into our VPS and running the Docker containers.

### `root` Login

When you have your server's IP address and root passphrase, log into the server as the `root` user

```shell
ssh root@your_server_ip
```

If you are not asked to create a new passphrase, use the `passwd` command to generate a new one. Although we will be disabling password authentication, be sure to create or [generate](http://passwordsgenerator.net/) a secure passphrase anyway.

### Update VPS

We will update and then reboot the VPS to ensure everything is up-to-date and the latest security patches are installed.

Update your VPS (assuming you are using Ubuntu/Debian):

```
sudo apt update && sudo apt upgrade -y
```

Once the updates have been installed, reboot the VPS:

```
sudo reboot
```

### Create user `pi`

Log back into your VPS as `root` and create new user `pi`

```shell
adduser pi
```

Grant root privileges to `pi`

```shell
usermod -aG sudo pi
```

### Public Key Authentication

[Public Key Authentication](https://the.earth.li/~sgtatham/putty/0.55/htmldoc/Chapter8.html) provides an alternative method of identifying yourselve to a remote server and increases the overall security of your server.

If you do not already have an SSH key, you will need to create one on your local computer

```shell
ssh-keygen
```

Save your key in the default file (where `$user` is your user)

```shell
Enter file in which to save the key (/Users/$user/.ssh/id_rsa):
```

Create a secure passphrase. You will need to enter this passphrase each time you utilize your SSH key

Copy the public key from your local machine to your remote server with `ssh-copy-id`

```shell
ssh-copy-id pi@your_server_ip
```

- If you opted to add SSH during the server creation process anyway, this method will not work.

You should repeat these steps for each device you want to access the server, including desktops, laptops, tablets, and mobile phones.

#### Disable Passphrase Authentication

Once you have added SSH keys from all of your devices, we can disable passphrase authentication.

Log into your server as `root`, if you are not already logged in

```shell
ssh root@your_server_ip
```

Open the SSH daemon configuration file

```shell
sudo nano /etc/ssh/ssdh_config
```

- Find the line containing `PasswordAuthentication` and uncomment it by deleting the preceeding `#`. Change it's value to **no**
- Find the line containing `PubkeyAuthentication` and ensure it's value is set to **yes**
- Find the line containing `ChallengeResponseAuthentication` and ensure it's value is set to **no**

Save your changes and close the file

- `CTRL + X`
- `Y`
- `ENTER`

While still logged in as `root`, open a new terminal window and test logging in as `pi` and verify that the public key authentication works

```shell
ssh pi@your_server_ip
```

### Add environment variables

Many Docker containers require similar settings, such as timezones and application directories. To save us from having to type these values multiple times, we will set them as environment variables instead and use them in our `docker-compose.yml` file. We will need to reboot the VPS again after setting these variables.

Log in to your VPS as the `pi` user and open `/etc/environment`:

```
sudo nano /etc/environment
```

Add the following environment variables to the bottom of the file, making changes where necessary.

TODO add instructions for quickly obtaining interface

```
TZ=America/New_York                 # Set this to your timezone
SERVER_IP=127.0.0.1                 # Set this to your VPS's IPv4 Address
SERVER_IPV6=::1                     # Set this to your VPS's IPv6 Address, if using IPv6
DOCKER_DIR=/home/pi/docker
WEB_PASSWORD=myawesomepassphrase    # Your passphrase for the Pi-Hole web interface
INTERFACE=eth0                  # Set to the main interface on your VPS
```

Save your changes and close the file. Then reboot:

```
sudo reboot
```

### (Optional) Install **Mosh**

[**Mosh**](https://mosh.org/), or **mo**bile **sh**ell, is a remote terminal application that allows roaming and intermittent connectivity. It's intended as a replacement for SSH but both can be used on the same server.

```
# Update your sources, if necessary
sudo apt update

# Install mosh
sudo apt install mosh
```

### Set up **`ufw`**

We will set up a basic firewall, [`ufw`](https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29), that will restrict access to certain services on the VPS. Specifically, we want to ensure that only ports needed for our applications. Additional ports can be opened later depending on your specific needs.

We will be opening ports for secure FTP so that `.ovpn` files needed for connecting to our VPN later can be retrieved via a FTP application such as [Filezilla](https://filezilla-project.org/) or [Transmit](https://panic.com/transmit/).

To set up `ufw`, enter the following commands:

TODO: Add rules for Wireguard

```shell
# Apply basic defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Open ports for OpenSSH
sudo ufw allow OpenSSH

# Optionally, allow all access from your IP Address
sudo ufw allow from $yourIPAddress

# Open ports for secure FTP
sudo ufw allow sftp

# Open ports for Mosh if you installed it
sudo ufw allow mosh
```

### Disable `systemd-resolved`

If you are using Ubuntu 18.04 or greater, you will need to disable `systemd-resolved` before installing any software, specifically Pi-Hole.

Run the following commands to disable `systemd-resolved`:

```
sudo systemctl disable --now systemd-resolved.service
sudo rm /etc/resolv.conf
```

Now create your own `resolv.conf` file:

```
sudo nano /etc/resolv.conf
```

And add the following lines:

```
nameserver 127.0.0.1
nameserver 9.9.9.9
```
Save and close the file.

## Set up Docker and `docker-compose`

We will be using [**Docker**](https://www.docker.com/) to run the software covered in this guide in containers. We will install Docker and Docker Compose, add the `pi` user to the Docker group, and then create a Docker Compose configuration file that will be expanded throughout the guide.

### Install Docker

Docker provides a bash script that we can use to automate the entire installation.

Go to your temporary directory:

```
cd /tmp
```

And run the following command to install Docker:

```
curl -fsSL get.docker.com -o get-docker.sh && sh get-docker.sh
```

#### Add `pi` user to Docker Group

Running docker containers requries `sudo` privileges. To avoid needing `sudo` for every command or having to switch to the `root` user, we will add the `pi` user to the `docker` group that was added when we installed docker.

To add the `pi` user to the `docker` group, use the following command:

```shell
sudo usermod -aG docker ${USER}
```

Log out of your VPS and log back in as the `pi` user. `pi` should now be part of the `docker` group. To test this, and to test that Docker has been successfully installed, run the following command:

```
docker run hello-world
```

### Install `docker-compose`

There are multiple options for installing `docker-compose`, including installing it via `pip` and running it in a Docker container itself. We'll be downloading the binary from the GitHub repository and adding executable permissions to it.

Be sure to check the [release page](https://github.com/docker/compose/releases) and ensure you run the following command with the most recent release.

Download `docker-compose`:

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

Apply executable permissions:

```
sudo chmod +x /usr/local/bin/docker-compose
```

## Create `docker-compose.yml` Configuration File

Create a docker folder in `pi`'s home directory:

```
mkdir ~/docker
```

Type in the following command to set the necessary permissions. They will make any new subfolders inherit the permissions from the docker folder. We will be storing configuration folders for our containers in the docker folder, hence why we want such liberal permissions.

```
sudo chmod -R 775 ~/docker
```

Now create the Docker Compose file:

```
nano ~/docker/docker-compose.yml
```

And add the following two lines:

```
version: "3.6"
services:
```

### `unbound`

The first service we will add is `unbound`. There are many Docker containers for `unbound` that can be used. I personally chose [Klutchell](https://github.com/klutchell/unbound)'s because it uses the root DNS servers. Another popular container that you can use is [MathewVance](https://github.com/MatthewVance/unbound-docker)'s; it uses Cloudflare's DNS servers via DNS-over-TLS instead of the root servers.

Add the following lines to your `docker-compose.yml` file, after the `services:` line:

```yaml
version: "3.6"
services:

  unbound:
    container_name: unbound
    image: klutchell/unbound:latest     # or mvance/unbound:latest
    hostname: docker_unbound
    ports:
      - 5053:53/tcp
      - 5053:53/udp
    environment:
      TZ: ${TZ}
    restart: unless-stopped
```

### **Pi-Hole**

The next service we will add is Pi-Hole. The pihole config is a bit more involved and you will most likely want to customize the configuration to meet your specific needs.

I would strongly suggest not changing the value of `network-mode` from `host`. That specific setting is necessary for Pi-Hole to communicate with `unbound`.

Add the following block to your `docker-compose.yml`:

```
pihole:
    container_name: pihole
    image: pihole/pihole:latest
    hostname: pihole
    network_mode: host
    cap_add:
      - NET_ADMIN
    dns: 
      - 127.0.0.1
      - 9.9.9.9
    ports:
      - 53:53/tcp
      - 53:53/udp
      # - 67/udp # Uncomment for DHCP
      - 80:80/tcp
      - 443:443/tcp
    environment:
      TZ: ${TZ}
      WEBPASSWORD: ${WEB_PASSWORD}
      DNS1: 127.0.0.1#5053
      DNS2: "no"
      INTERFACE: ${INTERFACE}
      ServerIP: ${SERVER_IP}
      # ServerIPV6: ${SERVER_IPV6}      # Enable these lines for IPv6
      # IPv6: "yes"
      PROXY_LOCATION: pihole
    volumes:
      - ${DOCKER_DIR}/apps/pihole:/etc/pihole/
      - ${DOCKER_DIR}/apps/dnsmasq.d:/etc/dnsmasq.d/
    restart: unless-stopped
```

---------------------------------------------------------------------------

### (Optional) Configure **Pi-Hole**

**Pi-Hole** allows you to customize what websites you want to block and allows to you whitelist any false positives (e.g., unblocking Netflix or Facebook). **Pi-Hole** developer [WaLLy3K](https://github.com/WaLLy3K) provides a [popular collection of blocklists](https://wally3k.github.io/) that you can add to your own blocklists. Another blocklist collection is provided by [the Block List Project](https://tspprs.com/).

I would also recommend checking out [this GitHub repository](https://github.com/anudeepND/whitelist) that will load commonly whitelisted domains (e.g., Facebook, Instagram, XBox Live) into your Pi-Hole.

Finally, I would suggest following [this guide](https://docs.pi-hole.net/guides/unbound/) from the official Pi-Hole documentation to set up [**unbound**](https://nlnetlabs.nl/projects/unbound/about/) as your own recursive DNS server (rather than using a public DNS server such as Google DNS or Cloudflare). This will help to further increase the privacy of your DNS queries.

## Let's Encrypt

The following section is optional and **requires you to have your own domain name**, but it will configure your Pi-Hole's web interface to use `https` courtesy of [Let's Encrypt](https://letsencrypt.org/) and [Certbot](https://certbot.eff.org/). It can be considered overkill just for Pi-Hole, but it certainly doesn't hurt. First we will acquire the certificate and then we will configure `lighttdp` to automatically redirect any `http` requests to `https`. The steps are based on [this reddit post](https://www.reddit.com/r/pihole/comments/6e2jyr/self_signed_ssl_cert_for_the_admin_login_page/di7ct0b).

### Acquiring the certificate

1. Log into your remote server again either as `root` or with root privileges.
2. Go to this [Certbot page](https://certbot.eff.org/lets-encrypt/ubuntubionic-other) (for Ubuntu 18.04) and following the **Install** commands to install Certbot on your server.
3. Perform a dry run to acquire a certificate for your domain. For example:
    ```bash
    certbot certonly --webroot -w /var/www/html -d example.com --dry-run
    ```
4. If acquiring the certificate was successful, run the same command again without `--dry-run`. For example:
    ```bash
    certbot certonly --webroot -w /var/www/html -d example.com
    ```
5. Edit the file `/etc/lighttpd/conf-available/10-ssl.conf`. Replace `example.com` with your own domain name:
    ```bash
    ssl.pemfile = "/etc/letsencrypt/live/example.com/combined.pem"
    ssl.ca-file = "/etc/letsencrypt/live/example.com/chain.pem"
    ```
6. Run the following commands, replacing `example.com` with your domain name:
    ```bash
    ln -s /etc/lighttpd/conf-available/10-ssl.conf /etc/lighttpd/conf-enabled/10-ssl.conf
    cd /etc/letsencrypt/live/example.com/
    cat privkey.pem cert.pem > combined.pem
    ```
7. Restart `lighttpd`:
    ```bash
    sudo systemctl restart lighttpd
    ```
    `https` should now be enabled on your web interface.
8. Add a cron job to automatically renew the certificate every 90 days. Open `/etc/crontab` and add the following line:

    ```bash
    47 5 * * * root certbot renew --quiet --no-self-upgrade --renew-hook "cat $RENEWED_LINEAGE/privkey.pem $RENEWED_LINEAGE/cert.pem > $RENEWED_LINEAGE/combined.pem;systemctl reload-or-try-restart lighttpd"
    ```

### Configure the Redirect

Open the `lighttpd` configuration file, `/etc/lighttpd/lighttpd.conf`, and add the following block of code:

```bash
compress.cache-dir = "/var/cache/lighttpd/compress/"
compress.filetype = ( "application/javascript", "text/css", "text/html", "text/plain" )

# [add after the syntax above]

# Redirect HTTP to HTTPS
$HTTP["scheme"] == "http" {
    $HTTP["host"] =~ ".*" {
        url.redirect = (".*" => "https://%0$0")
    }
}
```

Restart `lighttpd` again:

```bash
sudo systemctl restart lighttpd
```

Your web interface should now automatically redirect any `http` requests to `https`.


## Sources

- **Pi-Hole**
  - [Official Website](https://pi-hole.net/)
  - [Github](https://github.com/pi-hole/pi-hole)
- **PiVPN**
  - [Official Website](http://www.pivpn.io/)
  - [Github](https://github.com/pivpn/pivpn)
- [Digital Ocean: Initial Server Setup with Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)
- [Secure Password Generator](http://passwordsgenerator.net/)
- [Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.55/htmldoc/Chapter8.html)
- [Mosh](https://mosh.org/)
- [Debian Wiki: Uncomplicated Firewall](https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29)
- [FileZilla](https://filezilla-project.org/)
- [Transmit](https://panic.com/transmit/)
- [Static vs. dynamic IP addresses](https://support.google.com/fiber/answer/3547208?hl=en)
- [The Big Blocklist Collection](https://wally3k.github.io/)
- [Pi-Hole: Commonly Whitelisted Domains](https://discourse.pi-hole.net/t/commonly-whitelisted-domains/212)
- [OpenVPN](https://openvpn.net/)
- [How To Set Up an OpenVPN Server on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04#step-8-adjust-the-server-networking-configuration)
- <https://itchy.nl/raspberry-pi-3-with-openvpn-pihole-dnscrypt>
