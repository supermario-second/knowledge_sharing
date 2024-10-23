# first step: install docker

- reference this website
- <https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/>
- follow these commands:

> delete your docker if you installed before

```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do apt-get remove $pkg; done
```

> install dependencies

```
apt-get update
apt-get install ca-certificates curl gnupg
```

> add these thing to your mirror
>
> ```
> install -m 0755 -d /etc/apt/keyrings
> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
> sudo chmod a+r /etc/apt/keyrings/docker.gpg
> echo \
>   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
>   "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
>   tee /etc/apt/sources.list.d/docker.list > /dev/null
> ```
>
> then install docker

```
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

# use docker compose to start pi-hole

- create a file named daemon.json under /etc/docker/

```
sudo nano /etc/docker daemon.json 
```

- enter these content in it, below the mirrors i have tested, they are good now, but i am not sure if it can be useful in the future

```
{
"registry-mirrors": [
"https://dockerhub.icu",
"https://docker.chenby.cn",
"https://docker.1panel.live",
"https://docker.awsl9527.cn",
"https://docker.anyhub.us.kg",
"https://dhub.kubesre.xyz"
]
}
```

- navigate to your work space, where your want to start your docker container, for example:

```
cd /home/username/Desktop/pi-hole
```

::: info
if you don't have this folder, you can create one, don't worry you can create you workspace at any where in your system, if you want to create a new folder you can use this command

:::

```
mkdir /home/username/Downloads/pi-hole2
```

- create a new file name it to compose.yaml

```
nano compose.yaml
```

- enter these content in it

```
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    # For DHCP it is recommended to remove these ports and instead add: 
    network_mode: "host"
    dns:
      - 127.0.0.1
    environment:
      TZ: 'Asia/Shanghai'
      WEBPASSWORD: 'pihole-1'
      WEB_PORT: 8573
    # Volumes store your data between container upgrades
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    #   https://github.com/pi-hole/docker-pi-hole#note-on-capabilities
    restart: unless-stopped # Recommended but not required (DHCP needs NET_ADMIN)
```

- use docker compose to start your pi-hole

```
sudo docker compose up
```

::: warn
normally your container will start by itself, but if your system has application listening on port 53, you will get the message during the start process, you should follow the below process, if not, you can jump to "pi-hole configuration"

normally, port 53 will be used by 'systemd-resoled', but, it's an important software on ubuntu, so you cannot stop it, but you can make some modifications, 

:::

- enter the folder 

```
cd /etc/systemd/
```

- edit the content of resolved.conf

```
nano resolved.conf
```

- modify the parameter below ( just remove the # symbol before DNSStubListener and set its value to no

```
#Cache=no-negative
#CacheFromLocalhost=no
DNSStubListener=no
#DNSStubListenerExtra=
```

- Use this command to restart your docker

```
sudo systemctl restart docker
```

- until now every thing will be OK, you Pi-hole will running in your docker

# configure your pi-hole

- after starting your pi hole without issues, you can visit it though localhost:8573/admin, or your_ip_address:8573/admin, then you will see the dialog below, 

::: info
enter the password,  it should be 'pihole-1' that was configured in your docker compose file

:::

![image.png](.attachments.724/image.png)

- after entering you will see the page like this 

![image (2).png](.attachments.724/image%20%282%29.png)

- find the local DNS ->DNS Rrecords, click it

![image (3).png](.attachments.724/image%20%283%29.png)

- in the domain area enter your domain that is only belong to you, such as: home.home or home.local or example.com
- after entering, then click add

  ![image (5).png](.attachments.724/image%20%285%29.png)
- Now you already have a domain which is just belong to yourself.
- But you still need to do something else to complete the configuration.
- navigate to the setting dialog and click the DNS button, here you can define your up stream DNS server if you have one, if you don't you also can use the google DNS serve as your up stream server

::: warn
In the interface area, the most important thing is to choose '**Allow only local requests'** That means only the devices on the same network as your server have the right to access

:::

![image (6).png](.attachments.724/image%20%286%29.png)

- for now you have completed your local DNS serve setting, additionally pi-hole also has many other functionalities for you to discover

#  configure your computer network

- since your DNS(pi-hole) server is running on your computer, ever it uses docker
- next you need to set you whole home network to use your DNS( pi-hole) server
- first set you host computer ip address, net-mask, gateway
- second, on your computer set the DNS server address to it self( the ip address of your computer, which runs pi-hole on it)

![image (7).png](.attachments.724/image%20%287%29.png)

# configure your reverse proxy use caddy

you cloud find some references from here 

<https://caddyserver.com/docs/install#debian-ubuntu-raspbian>

Follow these commands to complete the installation process.

```
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

After installing, navigate to the Caddy folder and edit the content in the Caddyfile

```
sudo nano /etc/caddy/Caddyfile
```

delete all the exiting content and paste these code in it

for example 'home.local' is the domain that your defined in pi-hole before

and here we assume port 81 is your nextcloud server's port (for now we haven't installed it yet ) 

```
http://home.local {
    redir https://home.local{uri}
}

https://home.local:443 {
    tls /etc/caddy/ssl.crt /etc/caddy/ssl.key
    reverse_proxy localhost:81
    header Strict-Transport-Security "max-age=15552000; includeSubDomains"
}
```

now we need to make ssl for our predefined domain with the command blow

```
openssl req -x509 -out ssl.crt -keyout ssl.key -newkey rsa:2048 -days 3650 -nodes -sha256
```

::: info
here is the explanation of the command

 req -x509: Generates a self-signed X.509 certificate
\-out ssl.crt: Specifies the output file for the certificate
\-keyout ssl.key: Specifies the output file for the private key
\-newkey rsa:2048: Generates a new 2048-bit RSA key pair
\-days 3650: Sets the validity period of the certificate to 10 years (3650 days)
\-nodes: Generates the private key without a passphrase
\-sha256: Uses the SHA-256 hashing algorithm

:::

then flow the prompts, complete the generation process

after that, if the process is correctly completed.,  you will see three files stored in the caddy folder like this 

![image (8).png](.attachments.724/image%20%288%29.png)

restart your caddy 

```
sudo systemctl restart caddy
```

Now you have completed the installation and the configuration process of caddy

# install nextcloud and configure it

you can reference this website

<https://github.com/nextcloud-snap/nextcloud-snap/tree/master>

install nextcloud with snap use this command

```
sudo snap install nextcloud
```

wait...until it complete

after installing, if nextcloud be installed correctly then you can visit it though localhost:80 or your ip_address:80, but we haven't complete the deployment process

 run this command, modify the port to the port that defined in our caddy file, if you didn't change it, it should be 81

```
sudo snap set nextcloud ports.http=81 ports.https=444
```

then add your reverse proxy server address to the nextcloud trusted proxy array, that means only this reverse proxy server can access your nextcloud, for example add 'home.local' into it

```
 sudo nextcloud.occ config:system:set overwritehost --value="home.local"
```

```
sudo nextcloud.occ config:system:set overwriteprotocol --value="https"
```

navigate to this folder 

```
/var/snap/nextcloud/current/nextcloud/config/
```

edit this file, add the content to the last line, set your ip address to replace 'your_ip_address"

::: info
when you edit this file it has the same effective with the nextcloud.occ ctl command, you also can complete the above steps in this file

:::

```
nano config.php
```

```
'trusted_proxies' => 
  array (
    0 => 'your_computer_ip_address',
  ),
```

::: warn
remember the structure of this file, when you open it, it must be like this, with an example ip address here

:::

![image (9).png](.attachments.724/image%20%289%29.png)

you have completed, have fun.