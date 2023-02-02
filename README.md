![HASS + Android](docs/images/HA_Android.png)
[Home Assistant](https://www.home-assistant.io/) installation on a Google Nexus 7 tablet. A similar setup process can be followed for any Android device.

# Credits
All credit goes to @alexanderhale and https://www.reddit.com/user/ihifidt250/
This is just a temporary repository until updates are made and a better process is found.
this is a fork of @alexanderhale repo with the LineageOS and Nexus 7 references removed + an updated install process.

## Prerequisites
- Hardware: Android Device (Phone, Tablet, TV, anything that can install/sideload apps)
- Tools: a computer that can run Android adb (If termux is to be sideloaded)

# Home Assistant Installation
Now that the operating system on the tablet has been swapped to LineageOS, Home Assistant (and related programs) can be installed.

## Install F-Droid
[F-Droid](https://www.f-droid.org/) is a repository for free apps, analogous to the Google Play store. Head to F-Droid's website and download the latest .apk file. Once downloaded, a pop-up will appear - hit `Install`.

## Install Termux
Most of the Home Assistant installation work will be done on the command line. The most robust UNIX console emulator for Android is Termux. To get Termux, head to F-Droid and search for [Termux](https://f-droid.org/en/packages/com.termux), download it, and install it. While you're here, download and install [Termux:API](https://f-droid.org/en/packages/com.termux.api/) and [Termux:Boot](https://f-droid.org/en/packages/com.termux.boot) too - we'll need them later.

## Install Packages
Open up the Termux app. Notice that your home directory is `/data/data/com.termux/files/home/`, and the `etc` directory (where packages will be installed) is at `/data/data/com.termux/files/usr/etc/`. This is different than a standard UNIX operating system.

The package manager in Termux is `pkg`, which is a wrapper around `apt`. You can also use the `apt` command directly, but there's usually no need to do so.


Install the necessary packages with these commands (confirming with `y` when requested):
```bash
pkg upgrade

pkg install python nano make rust libcrypt libffi libjpeg-turbo binutils

# creates a virtual environment (optional, but recommended)
python -m venv hass

source hass/bin/activate
#
pip install --upgrade pip

pip install wheel

pip install tzdata

pip install maturin

MATHLIB=m pip install numpy

MATHLIB=m pip install PyTurboJPEG==1.6.7

export RUSTFLAGS="-C lto=n"

export CARGO_BUILD_TARGET="$(rustc -Vv | grep "host" | awk '{print $2}')"

export CRYPTOGRAPHY_DONT_BUILD_RUST=1
```

## Install Home Assistant
The prerequisites are now in place to install the Home Assistant package:
```bash
pip install homeassistant==2022.12.0
# Due to a bug Only versions up to 2022.12.2 can be installed
#Guide will be updated once a solution is found.
```

## Run
Everything is now in place! With your virtual environment activated, execute `hass -v`. During this first startup, keep an eye on the output for error messages, which might indicate that something has been configured incorrectly.

If the startup succeeds, head to `localhost:8123` in a browser on the tablet - the Home Assistant homepage should appear!

For more information, here is [a Medium post](https://lucacesarano.medium.com/install-home-assistant-hass-on-android-no-root-fb65b2341126) describing this process.

# Addressing Setup
Home Assistant is now accessible from other devices which are also connected to your home network. To access Home Assistant, find your tablet's network IP address (something like `http://192.168.2.xyz`) by running the `ipconfig` command in Termux, or by logging in to your router's admin console (usually at `http://192.168.2.1`) and finding your tablet in the list of devices. Then, navigate to that address from another device using port 8123: `http://192.168.2.xyz:8123`. The Home Assistant interface should load, the same way it does for `http://localhost:8123` on your tablet.

This setup works, but it is not robust. What if your router re-assigns your tablet a new IP address? What if you want to access your Home Assistant setup when you're away from home? What if you want to connect Home Assistant to an external provider like Google Assistant? These hurdles will be addressed in this section.

## Static IP Address
By default, your tablet gets assigned an IP address by your router in a process called [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol). The 'D' in 'DHCP' stands for 'Dynamic', meaning your tablet's assigned IP address could get reassigned at any time.

To assign a _static_ IP address, go into the WiFi settings on your tablet, then tap & hold your home network. Select Manage Network Settings > Show Advanced Options. Change the IP Setting from DHCP to Static, and enter the IP address that you want your tablet to have. If the Gateway doesn't fill in automatically, enter the same value as your IP address, replacing the last section (after the last `.`) with `1`.

[<img src="docs/images/static_ip.png" alt="Static IP Settings" width="600"/>](https://i0.wp.com/thedroidreview.com/wp-content/uploads/2016/03/Set-Static-IP-on-Android.png?fit=900%2C714&ssl=1)

All set! Your tablet will now always receive the same IP address when connected to your home network. This guide will refer to your tablet's `http://192.168.2.xyz` IP address as the "internal network IP address".

## Port Forwarding
The configuration above works only when conneceted to your home network, because your home router knows to route requests at `http://192.168.2.xyz` to your tablet. Try it yourself: disconnect your phone from WiFi, connect to data, and navigate to your tablet's internal network IP address. Nothing loads!

Let's fix that. Just like your tablet has an IP address, your entire home network also has an IP address. Think of your router as the "front door" of an apartment building - there is a number outside the door telling the mail carrier to deliver mail to the concierge (router) of the building, then it is the concierge's (router's) job to distribute the mail to the appropriate apartment number (device, like your tablet) in the building.

[![Internal vs External IP Address](docs/images/internal_external_ip.png)](https://troypoint.com/wp-content/uploads/2019/08/internal-vs-external-ip-address-diagram-600x351.png)

To access Home Assistant from outside your home network, we therefore need to know the "building number" (external IP address of your home network) and "apartment number" (port) to which we should send our requests.

To find the IP address of your home network, look in your router's admin console, or you can go to [whatismyipaddress.com](https://whatismyipaddress.com/) and look for the IPv4 field.

To route requests to the correct device, we need to set up port forwarding. Find the port forwarding configuration area in your router's admin console and set up a new port forward with the following:
- External port: 8123
- Internal port: 8123
- Internal address: `http://192.168.2.xyz`

When your router receives a request to `http://<external-ip-address>:8123`, it will now send it to `http://192.168.2.xyz:8123` - the address where our Home Assistant installation is running! Test it out on your phone (while still connected to mobile data).

Great! Home Assistant is accessible from outside your home network. However, two issues remain:
- The digits of the external IP address are hard to remember, and some external services (like Google Assistant) require a proper hostname.
- We are accessing Home Assistant via an unencrypted `http://` connection, meaning an intruder could see your username and password.

We'll resolve those two issues in the next sections.

## DNS Routing
When you want to make a Google search, you don't navigate to `172.217.1.4` - you navigate to `google.com`. This mapping of human-friendly names to IP addresses is called a "Domain Name System" (DNS).

Registering a domain name - say, `myhomeassistant.org` - means that you're reserving the right to route traffic arriving at `myhomeassistant.org` to a web server owned by you. Most domain names cost money to reserve, but there are some services like [DuckDNS](https://www.duckdns.org/) and [No-IP](https://www.noip.com/) which offer free domain names. These services make money with their "premium" offerings, which are not required to complete this guide.

One additional feature of domain name services is "dynamic" DNS. The IP address of your router is provided by your internet service provider (ISP), and is liable to change at any time. A dynamic DNS provider takes this change in stride, automatically sending traffic to the updated IP address whenever it changes.

To set up your domain name, create a [No-IP](https://www.noip.com) account. Create whatever hostname you'd like - make it something memorable! We'll call it `hass.noip.org` for simplicity in this guide. Set "IP / Target" to the external IP address of your network, then save the hostname.

After waiting a few hours for the new DNS record to propogate through name servers (or up to 2 days, according to No-IP's documentation), navigating to `http://hass.noip.org:8123` should now bring up your Home Assistant instance. Success!

If you prefer to use DuckDNS or another provider to reserve your hostname, that's no problem - the steps above should work the same.

One common gotcha: if `http://hass.noip.org:8123` does not load for you when connected to your local network, but does work when connected to mobile data, it's likely because your router does not support NAT loopback (also known as NAT Hairpinning). See [Reverse Proxy](#reverse-proxy) below.

## SSL Certificate
When navigating to `http://hass.noip.org` in the previous step, a notice appeared on the left side of your address bar, saying something like "Not Secure!", with an open padlock logo.

<img src="docs/images/http_insecure.png" alt="HTTP Insecure Warning" width="400"/>

This happens because an `http` connection is _unencrypted_ - traffic to and from the server is in plain text, and could be read by anybody intercepting the traffic via a "man in the middle" attack. An `https` connection (where the `s` stands for "secure") is a better option.

To allow connections via `https`, your server first needs a TLS/SSL certificate, which:
1. Specifies the owner of the certificate, and includes the domain that content signed with this certificate should be coming from (preventing another server from spoofing this server). 
2. Provides a public key, which anybody wishing to communicate with a server can use to encrypt their traffic.

The certificate is issued by a Certificate Authority, which is an authoratitive body that has been trusted to issue certificates. Just like with domains, there are free and paid Certificate Authorities from which certificates can be issued. We'll be using [LetsEncrypt](https://letsencrypt.org/), which is a free option.

To receive a certificate from LetsEncrypt, you must prove that you "control" the domain - that is, traffic to `hass.noip.org` goes to your server, and not somebody else's. This prevents you from generating a certificate for a server that does not belong to you.

LetsEncrypt provides the handy certbot tool for proving you own a domain, but it is not available in Termux, so some extra steps are required. 

#### Option 1 - Complete the HTTP-01 Challenge Elsewhere
1. Set up a new port forward on your router, which routes traffic to port `80` of your external IP address to port `80` of your computer's internal IP address.
2. Install [Apache](https://httpd.apache.org/) on your computer.
     - Don't start it, though! Certbot will take care of starting and stopping Apache.
3. Run [certbot](https://certbot.eff.org/) in standalone mode to complete the HTTP-01 challenge.
     - Certificate creation: `certbot --standalone --preferred-challenges http -d hass.noip.org`
     - Future renewals: `certbot renew`. Certificates must be renewed every 90 days.
4. Remove the port forward on your router.

#### Option 2 - Complete the DNS-01 Challenge
See [this blog post](https://www.splitbrain.org/blog/2017-08/10-homeassistant_duckdns_letsencrypt) for instructions.

Once the certificate files are generated (`fullchain.pem` and `privkey.pem`), transfer them to your tablet via ADB and place them somewhere accessible. Update your Home Assistant configuration with the paths to the files.

```yaml
http:
  ssl_certificate: /path/to/fullchain.pem
  ssl_key: /path/to/privkey.pem
```

More information details on this process are available in this [instructional blog post](https://community.home-assistant.io/t/installing-tls-ssl-using-lets-encrypt/196975).

## Reverse Proxy
Most ISP-provided routers don't support NAT loopback, which means that the domain name configured above will give a certificate error when accessing it from within the home network. That certificate error can be ignored in the browser, but not in the Home Assistant companion app.

There are a few options to get around this:
- **New Router**: the most obvious solution, but it is not free, and might not be possible depending on your ISP.
- **Split-Brain DNS**: setting up a DNS server which handles the routing of traffic on your home network. This could be a good option, but personally I'm wary of relying on the Nexus 7 tablet for DNS resolution of all my home traffic.
- **Reverse Proxy**: the approach used in this guide.

A [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) is a layer placed in front of a web server to handle incoming traffic. When behind a reverse proxy, all traffic arriving at the web server is coming from one source - the proxy - simplifying the configuration required at the web server.

We'll be using [NGINX](https://www.nginx.com/) as our proxy server. Install it on the tablet with `pkg install nginx`. We'll also need OpenSSL: `pkg install openssl`.

Navigate to `~/../usr/etc/nginx/ssl/`, then execute `openssl dhparam -out dhparams.pem 2048`. You may need to execute this command as root - if that's the case, you can prepend `su` to that command, or execute it in a root shell via ADB.

Create the file `~/../usr/etc/nginx/nginx.conf`, then enter the config below. Replace `hass.noip.org` with your domain name.

```
worker_processes 	1;

events {
	worker_connections 	1024;
}

http {
	include 		mime.types;
	default_type	application/octet-stream;

	sendfile	on;

	keepalive_timeout	65;

	map $http_upgrade $connection_upgrade {
		default 	upgrade;
		''			close;
	}

	server {
		server_name		hass.noip.org;

		listen [::]:8080 default_server ipv6only=off;
		return 301 https://$host$request_uri;
	}

	server {
		server_name 	hass.noip.org;

		ssl_certificate			/data/data/com.termux/files/home/certificates/fullchain.pem;
		ssl_certificate_key 	/data/data/com.termux/files/home/certificates/privkey.pem;
		ssl_dhparam				/data/data/com.termux/files/usr/etc/nginx/ssl/dhparams.pem;

		listen [::]:8443 ssl default_server ipv6only=off http2;
		add_header Strict-Transport-Security "max-age=31536000; includeSubdomains";

		ssl_protocols TLSv1.2;
		ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4"
		ssl_prefer_server_ciphers on;
		ssl_session_cache shared:SSL:10m;

		proxy_buffering off;

		location / {
			proxy_pass http://127.0.0.1:8123;
			proxy_set_header Host $host;
			proxy_redirect http:// https://;
			proxy_http_version 1.1;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection $connection_upgrade
		}
	}
}
```

All traffic to Home Assistant should be routed through the proxy - to configure this, head to `~/.homeassistant/configuration.yml` and change the `http` section:

```yaml
http:
	server_host: 127.0.0.1  # optional: only permit traffic from localhost (i.e. from the reverse proxy)
	use_x_forwarded_for: true
	trusted_proxies: 127.0.0.1
```

Finally, in your router, remove the port forwarding rule at port 8123 and add two new rules:
- for HTTP traffic: external port: 80, internal port: 80, internal IP address: your tablet
- for HTTPS traffic: external port: 443, internal port: 443, internal IP address: your tablet

Start nginx by running `nginx` in Termux. Home Assistant should now be accessible via `https://hass.noip.org` from outside your network and `http://192.186.2.xyz:8123` from your internal network. In the Home Assistant companion app, you can add those two addresses and it will automatically transition between them.

For additional information, [here's a guide](https://community.home-assistant.io/t/reverse-proxy-using-nginx/196954) on the subject.

## Remote SSH Access
Let's set up SSH access to the tablet so that you can access the command line remotely.

Here are the steps:
1. On the Termux command line, install OpenSSH: `pkg install openssh`.
2. On the Termux command line, generate an OpenSSH key pair: `ssh-keygen`
	- This will will create `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`.
	- On the tablet, place the contents of `~/.ssh/id_rsa.pub` in `~/.ssh/authorized_keys`, change the permissions of `~/.ssh/authorized_keys` to 600, and ensure the permissions of `~/.ssh` are 700.
	- Copy `~/.ssh/id_rsa` to the computer via ADB, and add it to your SSH client of choice.
3. On the Termux command line, execute `sshd`.

You should now be able to access the tablet from your computer: `ssh 192.168.2.xyz -p 8022`.

To access the tablet via SSH from outside your home network, add a new entry in your router's port forwarding settings that routes incoming traffic at port 22 to your tablet's IP address at port 8022. Notice that this allows SSH access via the default port: `ssh hass.noip.org`.

Some formatting issues may come up in the terminal when accessing the tablet via SSH. Executing `export TERM=xterm-256color` in your remote terminal (after logging in via SSH) should resolve those issues.

For more information, check out [this article](https://glow.li/posts/run-an-ssh-server-on-your-android-with-termux/). 

## Startup at Reboot
If the tablet ever restarts, a sequence of startup commands can be put in place so that Home Assistant starts up automatically upon reboot. Install the Termux:boot application from Google Play or F-Droid. Once installed, launch the app once - this will configure Termux to launch when the system starts after a reboot.

The sequence of commands to execute on startup can be placed in a file in `~/.termux/boot/`. Here's an example boot sequence:
```bash
#! /data/data/com.termux/files/usr/bin/sh

# prevent system from sleeping
termux-wake-lock

# enable remote access
sshd

# start reverse proxy (in the background by default)
nginx

# start MQTT in the background
mosquitto -d

# start home assistant in the background
source ~/hass/bin/activate
hass --daemon
deactivate

# start node-red
node-red
```

# Configuring Home Assistant
The setup of Home Assistant on the Nexus 7 tablet is now complete! Everything beyond the steps above is "normal" Home Assistant configuration. Refer to the [Home Assistant documentation](https://www.home-assistant.io/docs/) and [community forum](https://community.home-assistant.io/) as resources.

The remainder of this guide will cover the installation of extensions to Home Assistant, because there are some particularities about the Nexus 7 tablet.

# Node-RED Installation
[Node-RED](https://nodered.org/) is a visual automation suite, in which you can create automations beyond the complexity available with the basic Home Assistant tools. As the name suggests, Node-RED is build upon NodeJS.

[AppDaemon](https://github.com/AppDaemon/appdaemon) is a similar tool which allows for the creation of automations in code, but there are several issues with the compatibility of AppDaemon on Termux in LineageOS. Feel free to give the installation of AppDaemon a whirl, but this guide will use Node-RED.

Install the required packages on the tablet, then start Node-RED.
```bash
pkg install nodejs
npm install node-red
npm install node-red-contrib-home-assistant-websocket
node-red
```

See the [Node-RED documentation](https://nodered.org/docs/getting-started/android) or [this guide](https://dev.to/anthrogan/running-node-red-on-android-4fgi) to troubleshoot any issues installation.

## Connect Home Assistant to Node-RED
When using Home Assistant Core, the Node-RED connection must be installed manually from [this repository](https://github.com/zachowj/hass-node-red). In case the process changes with newer versions of the connection, I have not copied the setup instructions here. See the README in that repository for instructions.

After installing the connection, an Access Token must be created in Home Assistant to allow Node-RED access. Follow the instructions in [this guide](https://zachowj.github.io/node-red-contrib-home-assistant-websocket/guide/#generate-access-token), namely:
- Open Home Assistant in your web browser or in the companion app.
- Click your username in the sidebar.
- Scroll down to the Long-Lived Access Token section, and create a token.
- Copy the token so you can use it in the next step.

## Connect Node-RED to Home Assistant
The default port for Node-RED is 1880. If you want to access Node-RED from outside your home network then you can set up a port-forward, but this isn't recommended because Node-RED doesn't require a username or password. Access Node-RED by navigating to `http://192.168.2.xyz:1880`.

To configure Node-RED to recognize your Home Assistant server, place any Home Assistant node on the working area and enter its edit menu. Next to `Server`, click the edit pencil to add a new server, and follow the instructions in the UI. See [this post](https://zachowj.github.io/node-red-contrib-home-assistant-websocket/guide/) for more detailed instructions.

All set! You can now create complex automation flows in Node-RED.

# Google Assistant integration with Home Assistant
All of the elements are now in place to complete the integration of Google Assistant with Home Assistant. The process is well-documented and liable to change if Google changes its authentication procedure, so I will not re-write it. Follow the "Manual Setup" section of the [Google Assistant integration documentation](https://www.home-assistant.io/integrations/google_assistant/).

Once the setup is complete, all your Google Home devices will appear in Home Assistant, and you can start using them for automations.

# Mosquitto MQTT Installation
[MQTT](https://en.wikipedia.org/wiki/MQTT) is a messaging protocol for IoT devices. It's useful for passing around the data from sensors: for example, sensors that monitor a [laundry machine](https://www.home-assistant.io/blog/2015/08/26/laundry-automation-with-moteino-mqtt-and-home-assistant/) to sound an alarm when the laundry is complete. 

The [Mosquitto MQTT broker](https://mosquitto.org/) is compatible with Home Assistant, and lightweight enough to install on the same Nexus 7 tablet that is running Home Assistant. Install it with `pkg install mosquitto`, then you can start the broker with the `mosquitto` command. To hook it up with Home Assistant, add the following to `configuration.yaml`:

```yaml
mqtt:
  broker: 127.0.0.1
```

You can now configure your sensors to pass their data to the IP address of your tablet. For more configuration information, see the [Home Assistant page on MQTT](https://www.home-assistant.io/integrations/mqtt/).

# Fin
