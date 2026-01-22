# Setup

> [!WARNING]
> Ensure that you have all [prerequisites](./1-prerequisites.md) setup before following these steps

## Clone Repo

This copies the GitHub repo to your system:

```bash
git clone https://github.com/liamdmcgarrigle/portaefik.git
```

Then change into the new directory:

```bash
cd portaefik
```

## Create and Edit `.env` file

### Copy the `.env.example`

There is an example .env (`.env.example`) in the repo. You can copy it to make adding the env variables easier.

From the `~/portaefik` directory, run:

```bash
cp .env.example .env # copies contents of .env.example to new .env file
```

### Generate Password Hash

Generate a password hash for Traefik dashboard. This is required to access your Traefik dashboard:

```bash
echo "HASHED_TRAEFIK_PASSWORD='$(openssl passwd -apr1 your-secure-password)'"  >> .env
```

### Edit Your Env Vars

Change env values in the new `.env` file.
Also, delete the placeholder `HASHED_TRAEFIK_PASSWORD=hashed_password_here`:

```bash
nano .env # opens new .env file to edit
```

### Add DNS Records

> [!CAUTION]
> The ssl certs will fail to generate if the dns records aren't added and propagated before starting the containers.
>
> If this happens to you, wait for it to propagate then restart the container

Add DNS A records pointing the correct domain names to the IP addresses.

> [!WARNING]
> If using Cloudflare, go into security settings and ensure ssl mode is set to `Full (strict)`. Otherwise you will likely get a too many redirects error.

## Optionally Use Docker Swarm

You can use docker swarm to enable extra features, most notably docker secrets.

<details>
<summary>Docker Swarm Setup</summary>

**Initialize Swarm**  
If you want to use docker swarm, initialize it once with this command:

```bash
docker swarm init
```

**Create Network**  
With Swarm we need to create the proxy network for Traefik manually.
Do that with this command:

```bash
docker network create --driver overlay --attachable proxy
```

**Create Portainer Agent Secret**  
Now create a random portainer agent token and save it to docker secrets:

```bash
openssl rand -base64 32 | docker secret create portainer_agent_secret -
```

**Swarm vs Compose**  
With swarm, instead of `docker compose up -d`, we use `docker stack deploy -c docker-compose.yaml stack-name`

> [!NOTE]
> There are some slight incompatibilities between docker compose and stack. Most simple compose files will work without modification. Please see the docker docs for details

Then when we want to bring the service down, we need to use `docker stack rm stack-name` instead of `docker compose down`

**Start Portaefik With Swarm**  
Swarm doesn't automatically get envs from `.env` so we need to manually export them:

```bash
set -a      # allows envs to be exported beyond shell env
source .env # export our vars from .env
set +a      # restricts other vars from being exported

docker stack deploy -c docker-swarm.yaml portaefik # Starts Portaefik
```

</details>

## Start Container

> [!WARNING]
> If using docker swarm, use the instructions in the section above instead of these docker compose instructions

Start the container with the command below from the `~/portaefik` directory:

```bash
docker compose up -d
```

## Access Portainer and Traefik Sites

After a minute or two of your SSL generating, your sites will be available at the domains you specified in the `.env` file.

## Exposing Services With Traefik

Now we can set up services directly in the Portainer web UI.

Using Portainer is out of scope of these docs, [Portainer has their own great docs](https://docs.portainer.io/user/docker). I will specifically focus on details related to Portaefik.

### Add Required Labels

Add these labels to any container you want to expose through Traefik:

<details>
<summary>Swarm Labels</summary>

With docker swarm the labels are different, you need to place them under a deploy tag like this:

```yaml
deploy:
    labels:
        # Enable Traefik for this container
        - "traefik.enable=true"
        # Route traffic based on domain name
        - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
        # Specify entrypoint
        - "traefik.http.routers.${APP_NAME}.entrypoints=websecure"
        # Enable automatic SSL
        - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
        # Port your application runs on
        - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
```

The rest of the details shown are the same for swarm and compose.

</details>

**Docker Compose Labels:**

```yaml
labels:
    # Enable Traefik for this container
    - "traefik.enable=true"
    # Route traffic based on domain name
    - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
    # Specify entrypoint
    - "traefik.http.routers.${APP_NAME}.entrypoints=websecure"
    # Enable automatic SSL
    - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
    # Port your application runs on
    - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
```

### Specify "proxy" Network

Add network to container (under container config):

```yaml
networks:
  - proxy
```

Declare network (at root level):

```yaml
networks:
  proxy:
    external: true
```

It needs to be `external: true` since the network was defined in an external config, that being the one in our traefik compose file.

To see full examples, see the [Usage - Compose](./3-usage-compose.md) or [Usage - Swarm](./4-usage-swarm.md) pages.
