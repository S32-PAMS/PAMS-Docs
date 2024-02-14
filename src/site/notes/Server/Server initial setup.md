---
{"dg-publish":true,"permalink":"/server/server-initial-setup/","tags":["archi","middleware","software"]}
---

> [!abstract] Server setup
> This manual guides the initial setup of the software server for PAMS (Personnel and Asset Monitoring System), which can run both *locally* and *remotely* on a cloud instance. The server hosts various components of the PAMS [[Architecture\|Architecture]], including middleware and software components necessary for tracking and visualising location data of tagged objects.


### Operating system

> Linux-based OS (e.g. Ubuntu, CentOS) is recommended for their robustness and support for Docker and other dependencies

All commands are meant to be run on a Ubuntu or Debian-based system. Ensure your system has a static IP address if running in a local network, for easy access and configuration.

On first set-up, update and get C build-essentials:

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl gcc make build-essential cmake git software-properties-common net-tools
```

#### (Optional) Firewall

A simple firewall can be set up to control incoming and outgoing network traffic. `UFW` (Uncomplicated Firewall) is a user-friendly interface for managing `iptables` and is recommended for simplifying firewall configuration.

```bash
sudo apt-get install -y ufw
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
... take note to allow more ports that you setup later
```

These additional steps will improve the initial security posture. These are basic examples, and the system can be complemented by more sophisticated firewall options as you deem fit.

### Docker

> Required for running containerised applications such as the [[Server/MQTT Broker\|MQTT Broker]], [[Rust Bridge\|Rust Bridge]], [[Apache Kafka\|Apache Kafka]], and [[MongoDB\|MongoDB]].
> 
> This section provides step-by-step instructions for installing Docker on an Ubuntu-based system and setting up Docker authentication with GPG credentials. It ensures a secure and optimised Docker environment for PAMS deployment.

#### Installation

This installation requires administrative access (`sudo`) to the system.

To install docker, refer to the [documentation](https://docs.docker.com/engine/install/ubuntu/), but TLDR:

Remove old versions that might be installed on the system:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Install packages necessary for adding a new repository over HTTPS:

```bash
sudo apt-get install ca-certificates curl gnupg
```

Add Docker's official GPG Key, to ensure that the software you are installing is authenticated and trustworthy:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

To install Docker from its official repository, add it to your system's (`apt`) sources list:

```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update your package index and install Docker Engine, CLI, and other necessary plugins:

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Docker authentication setup

For a secure Docker operation, generate a GPG key.

Create a file named `gen_key` with the GPG Key configuration:

```bash
cat >gen_key <<EOF
  %echo Generating a basic OpenPGP key
  Key-Type: DSA
  Key-Length: 1024
  Subkey-Type: ELG-E
  Subkey-Length: 1024
  Name-Real: PAMS
  Name-Email: capstonepams@gmail.com
  Expire-Date: 0
  Passphrase: capstonepams
  %commit
  %echo done
EOF
```

> [!note]
> The values in the fields should be changed to more secure values, according to your application instance. These are the default values we have used for our prototype.

Generate the GPG Key using the configuration:

```bash
gpg --batch --generate-key gen_key
```


Retrieve and note down the generated key ID for later use:

```bash
filename=$(ls -t ~/.gnupg/openpgp-revocs.d | head -1)
f=$(echo $filename | cut -d'.' -f1)
echo $f
```

Initialise the password store using the generated key ID:

```bash
pass init $f
```

After generating the key, remove the `gen_key` file for security:

```bash
rm gen_key
```

You have successfully installed Docker and configured Docker authentication with GPG credentials on your system. This setup ensures a secure environment for deploying and managing Dockerised applications for the PAMS project.