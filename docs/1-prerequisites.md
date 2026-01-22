# Prerequisites

There are prerequisites to safely set up docker and Portaefik on your system as well as dependencies needed for everything to work!

> [!WARNING]
> Keep in mind, you are responsible for your own safety and security. This comes with no guarantees or warranties.

This assumes you are setting up a fresh VPS or server.

<details>
<summary>Don't have a server?</summary>

I recommend either [Hetzner](https://cloud.hetzner.com) or [Linode](https://www.linode.com/).  
If you are not sure what server size to pick, start small with shared cpu servers and scale up as needed. You'll be surprised how far 1vcpu and 1gb of ram will get you!

</details>

> [!NOTE]
> This assumes you know how to use SSH to access your server. If you have no idea what SSH is, either use the integrated terminal on your VPS providers website or go learn SSH first.
>
> If you fall into that category, I still highly recommend playing around to learn, but avoid deploying anything to production with sensitive or valuable data (see the warning above).

## Configure Non-root User

It is important to run docker containers from a non-root user.  
The biggest reason to do this is if someone manages to infiltrate your container and [break out of it](https://www.aquasec.com/cloud-native-academy/container-security/container-escape/) to the system, it is much worse for the attacker to have root access.

While it is best to run Docker in [rootless mode](https://docs.docker.com/engine/security/rootless/), we are using a simpler approach of simply running the regular docker install from a non-root user. If you have the know-how and desire to use rootless mode, feel free to use it, but it may effect setup of Traefik and Portainer.

> [!NOTE]
> This is showing an Ubuntu server setup. While all Linux distributions will be similar, they are not identical.  
> For the best experience, follow this from a fresh Ubuntu Server 24.04 LTS install

### Create Non-root user

You can use any username that you will remember. I use `leaf` since I am too punny for my own good.

Follow all prompts to create the user. **You can leave everything blank/default** other than the password!

```bash
adduser leaf
```

### Add The New User to sudo Group

This lets the user run commands with `sudo`, akin to running something as administrator on [Windows](https://cdn.mos.cms.futurecdn.net/cDmTNwuV7KkQBtjjFrs6na-1200-80.jpg).

```bash
usermod -aG sudo leaf
```

### Switch to new user

```bash
su leaf
```

## Install Dependencies

### Install Commands

This installs docker. It may take a minute with limited feedback:

```bash
# Install docker
sudo curl -fsSL https://get.docker.com | bash
```

You can copy and paste this entire block into your terminal at once. This lets the non-root user use docker commands without needing sudo and updates the system:

```bash
# Allow our non-root user to use docker without sudo
sudo usermod -aG docker leaf
# Install git
sudo apt install git -y
# Install updates
sudo apt update -y
sudo apt upgrade -y
```

### Refresh groups

You need to log out and back in to refresh the user group so that our non-root user can use docker.  
You may need to enter your root password:

```bash
# login to root
su root
```

Log back into the non-root user:

```bash
su leaf
```

Change directory so you are in your non-root users home directory:

```bash
cd ~
```

## Choose Install Type

You can setup your Portaefik with either docker compose or docker swarm. While it is possible to set it up for both, that is not covered in my documentation.

If you have experience with docker, I recommend using docker swarm to gain access to the swarm features like secrets.

If you are new to hosting services, I recommend starting with docker compose.

## Lock It Down

We're done with prerequisites, but not with security.  
When finished with setting up Portaefik, ensure to also follow best practices to set up a firewall, disable SSH, and other security measures.
