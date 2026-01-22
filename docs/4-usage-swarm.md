# Usage - Docker Swarm

## Simple Example

<details>
<summary>Swarm Labels Note</summary>

If you set up with swarm the labels are different, you need to place them under a deploy tag like this:

```yaml
deploy:
    labels:
        ...
```

The rest of the details shown are the same.

</details>

Here's a full docker-compose.yaml example using Uptime Kuma:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    volumes:
      - uptime-kuma:/app/data
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
        - "traefik.http.routers.${APP_NAME}.entrypoints=websecure"
        - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
        - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
    networks:
      - proxy
    restart: always

volumes:
  uptime-kuma:

networks:
  proxy:
    external: true
```

Then you need to define the env vars `APP_DOMAIN`, `APP_PORT`, and `APP_NAME` like:

```bash
# what you set in the dns record. Do not include http(s)://
APP_DOMAIN=whatsup.yourDomain.com

# must be what your app actually uses
APP_PORT=3001

# any name unique in your traefik config
APP_NAME=uptime
```

## Docker Swarm

You can use docker swarm to add extra features in Portainer. Swarm adds features such as:

- Native secret management
- Multi node deployments
- Blue-green deployments (0 downtime updates)

> [!NOTE]
> To get set up with swarm, you need to make changes in the terminal of the server hosting Portaefik. Follow the steps under the swarm setup dropdown found in [Setup](./2-setup.md#optionally-use-docker-swarm).

### Compose to Swarm

It is easy to simply use your existing compose setup as a swarm. You can have it run the exact same way but with the features of swarm, most notably secret management. You don't need multiple nodes to run swarm.

## Docker Secrets

Docker secrets are unfortunately only available in swarm mode.

Docker secrets are file based. Meaning when you use a secret like you're supposed to:

```yaml
environment:
    DATABASE_PASSWORD_FILE: /run/secrets/db_password
    API_KEY_FILE: /run/secrets/api_key
```

It passes the file that contains the secret to the environment. It does it this way for security and does not natively support passing in text, just the file.

That's great for the services that accept envs as a file (the convention is to add `_FILE` to the end of the env name), not all services accept files and require text. For those instances, we will use a workaround.

### Text-Only Workaround

> [!WARNING]
> This is a workaround to bypass a security feature. While our workaround may be _MUCH_ more secure than storing the tokens in plain text, please be aware you are responsible for your own security practices

Let's say there is an env var we need to pass called `API_KEY` and it doesn't support file input.  
This is not an officially supported use for docker secrets, but we can make it work.

> [!NOTE]
> These are some advanced docker concepts. I will be skimming the surface as to not be extremely verbose in my explanations.

#### Abstract

**App Entrypoint**  
Each container runs from an image. We can think of the image as a packaged up operating system with everything needed to run the service in the container.

Docker images are created with something called `Dockerfile`s. Where instructions put the image together, something like this:

```Dockerfile
# FROM says what operating system to base our image on
FROM alpine

# Can use RUN to install dependencies and run commands
RUN apk add python3

# CMD is what's important for us
# This tells the container what to run once it starts to start the application.
# We need to know what this command runs for our workaround
CMD ["python3", "app.py"]

# Sometimes there will be an ENTRYPOINT instead of a CMD
# for our purpose, this is the same as CMD
ENTRYPOINT ["/path"]
```

For the service we are running, we need to find their `CMD` or `ENTRYPOINT` that runs in the Dockerfile.

There are two ways we can get this. One is by finding the file named `Dockerfile` in the source code.

**Find Entrypoint With CLI**  
Another way is by using the `docker inspect` command. This will return the `CMD` and `ENTRYPOINT` if it exists. If a `CMD` exists, that is what to use. Otherwise, use the entrypoint.

```bash
docker inspect --format='cmd: {{.Config.Cmd}} entrypoint: {{.Config.Entrypoint}}' image-id
```

To find the image IDs, run `docker image ls`

We need to know the command/entrypoint that starts the app because we are going to change the entry point of the container to a script to extract the secrets from a file. We then have to manually start the app in our script or **it will never start**!

**The Script**  
The script will read the secret file and save it in the env by running `export API_KEY="$$(cat /run/secrets/api_key)"` for each key. The command `cat` simply reads out the file.

**All of it Together**  
Then we add a script under an `entrypoint` key in the yaml. That way our code runs once the container starts instead of the app. Read the inline comments in the code block _(lines starting with #)_ for more details.

```yaml
your-service:
    image: image-name:latest
    restart: unless-stopped
    secrets:
      - api_key
    entrypoint: >
      /bin/sh -c '
        # This puts the content of the secret file to an env var `API_KEY` in the containers environment
        export API_KEY="$$(cat /run/secrets/api_key)" && # add && at the end of each line except for the last

        # Put CMD or ENTRYPOINT from the Dockerfile here after `exec`
        # this manually starts the app after we expose the secrets
        exec ./entrypoint.sh
      '
```

### Why This Secrets Approach?

We use an inline shell script to handle text-based secrets because:

- **Visibility**: Secret handling is transparent in your compose files
- **Portability**: No external dependencies or additional services
- **Simplicity**: Everything managed through Portainer's UI
- **Compatibility**: Works with any service, even those that don't support file-based secrets

Alternative approaches like HashiCorp Vault or AWS Secrets Manager would:

- Require additional infrastructure
- Break our single-management-interface principle
- Require secrets to manage the secrets manager itself

## Docker Swarm Example (with text secret)

Let's use [Qdrant](https://qdrant.tech/) as an example. It is a self-hostable vector database software.

For Qdrant, we need to pass in an environment variable `QDRANT__SERVICE__API_KEY` and it does not accept an alternative with the `_FILE` postfix, making it a good example to show the text secret workaround.

First we'll pull in the latest qdrant image:

```bash
docker pull qdrant/qdrant:latest
```

Then we'll get the image's ID:

```bash
docker image ls
# Returns:
# REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
# qdrant/qdrant            latest    95c0f7f24ffe   5 weeks ago     191MB
```

Now we'll inspect the image to find the command to manually run the app:

```bash
docker inspect --format='cmd: {{.Config.Cmd}} entrypoint: {{.Config.Entrypoint}}' 95c0f7f24ffe
# Returns:
# cmd: [./entrypoint.sh] entrypoint: []
```

Now we can see that the `CMD` is `./entrypoint.sh`.
If you are working mostly in Portainer, it might be easier to just find the Docker file.
For example, Qdrant's Dockerfile can be found [here](https://github.com/qdrant/qdrant/blob/530430fac2a3ca872504f276d2c91a5c91f43fa0/Dockerfile#L220).

The `CMD` or `ENTRYPOINT` can usually be found at the bottom of the file. And we can see that the CMD lines up with what our docker inspect output was.

In Portainer you need to go to the secrets tab and add in a secret named `qdrant_api_key` with the content you want to be your api key.
I suggest generating a random key in your console like this:

```bash
openssl rand -base64 32
# Returns something like:
# DvJlD5CpThRJoDTrPxtR4z3gCkMFbHV51ee5en8iC5Q=
```

Now we'll adapt the [Qdrant docker compose](https://qdrant.tech/documentation/guides/installation/#docker-compose) template to work for us. As they have it, it won't work in Docker Swarm so we need to make some changes.

```yaml
services:
  qdrant:
    image: qdrant/qdrant:latest
    restart: unless-stopped
    ports:
      - "6333:6333"
      - "6334:6334"
    secrets:
      - qdrant_api_key
    entrypoint: >
      /bin/sh -c '
        export QDRANT__SERVICE__API_KEY="$$(cat /run/secrets/qdrant_api_key)" &&

        exec ./entrypoint.sh
      '
    volumes:
      - qdrant_data:/qdrant/storage
    deploy:
        labels:
            - "traefik.enable=true"
            - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
            - "traefik.http.routers.${APP_NAME}.entrypoints=websecure"
            - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
            - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
    networks:
      - proxy

secrets:
  qdrant_api_key:
    external: true

volumes:
  qdrant_data:

networks:
  proxy:
    external: true
```

Then if we start this stack with these env vars:

> [!NOTE]
> Ensure the domain is pointing to your IP address before running the container.
> This will also work even if you aren't using Portaefik, just remove the labels and proxy network.

```bash
APP_NAME=qdrant
APP_DOMAIN=qdrant.yourdomain.com
APP_PORT=6333
```

Then once you deploy your stack, qdrant will be available at `https://APP_DOMAIN/dashboard` or `http://your-ip-address:6333/dashboard`

## Docker Swarm Example (with file secret)

To show an example of a service we can host that supports the secret file we will use PostgreSQL, a very popular SQL database.

```yaml
services:
  db:
    image: postgres:latest
    environment:
      POSTGRES_USER: ${USER}
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
      POSTGRES_DB: ${DB_NAME}
    ports:
      - 5432:5432
    secrets:
      - postgres_password

secrets:
  postgres_password:
    external: true
```

Then add the env vars:

```bash
USER=username
DB_NAME=demo_db
```

And that's all that's needed!

## Another Example: n8n

[n8n](https://n8n.io/) is a workflow automation tool. It needs some extra environment variables to work properly behind a reverse proxy:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_HOST=${APP_DOMAIN}
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${APP_DOMAIN}/
      - GENERIC_TIMEZONE=${TIMEZONE}
      - TZ=${TIMEZONE}
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
        - "traefik.http.routers.${APP_NAME}.entrypoints=websecure"
        - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
        - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
    networks:
      - proxy

volumes:
  n8n_data:

networks:
  proxy:
    external: true
```

Environment variables:

```bash
APP_DOMAIN=n8n.yourDomain.com
APP_PORT=5678
APP_NAME=n8n
TIMEZONE=America/New_York  # https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
```
