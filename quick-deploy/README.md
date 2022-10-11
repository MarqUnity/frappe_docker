# Getting Started With Quick Deploy

Possibly suitable for small scale deployments. 

- Currently does not include nginx. Can be used behind a load balancer like AWS Application Load Balancer that includes SSL etc. Nginx still needs to be added. see https://github.com/frappe/frappe_docker/issues/858
- For production deployments please see this guide: https://github.com/frappe/frappe_docker/blob/main/docs/single-server-example.md
- For setting up conatiners for development please see this guide: https://github.com/frappe/frappe_docker/blob/Quick_Start/development/README.md

## Prerequisites

In order to start you need to satisfy the following prerequisites:

- Docker
- docker-compose
- user added to docker group

It is recommended you allocate at least 4GB of RAM to docker:

- [Instructions for Windows](https://docs.docker.com/docker-for-windows/#resources)
- [Instructions for macOS](https://docs.docker.com/docker-for-mac/#resources)

## Bootstrap Containers for quick deployment

Clone and change directory to frappe_docker directory

```shell
git clone https://github.com/frappe/frappe_docker.git
cd frappe_docker
```

Copy example quick deploy config from `quick-deploy-container-example` to `.quick-deploy-container`

```shell
cp -R quick-deploy-container-example .quick-deploy-container
```

## Setup Deployment 

Open `.quick-deploy-container/docker-compose.yml` with a text editor and replace "123" to set your database password.

- The `quick-deploy` directory is ignored by git. It is mounted and available inside the container. Create all your benches (installations of bench, the tool that manages frappe) inside this directory.
- Node v14 and v10 are installed. Check with `nvm ls`. Node v14 is used by default.

### Start containers

Ensure you have the latest version of the required containers.
```shell
docker pull redis:alpine
docker pull frappe/bench:latest
```


## Docker Compose V1

```shell
docker-compose -f .quick-deploy-container/docker-compose.yml up -d
```

## Docker Compose V2

```shell
docker compose -f .quick-deploy-container/docker-compose.yml up -d
```

you can run the above command without `-d` to view logs

The environment is now running inside docker. To run bench commands we need to enter the interactive shell with the following command:
```shell
docker exec -e "TERM=xterm-256color" -w /workspace/quick-deploy -it frappe bash
```

### Setup first bench

Run the following commands in the terminal inside the container.

```shell
bench init --skip-redis-config-generation frappe-bench
cd frappe-bench
```

For *develop* version use the `--frappe-branch` option.
```shell
bench init --skip-redis-config-generation --frappe-branch develop frappe-bench
cd frappe-bench
```

For your own fork of frappe use the `--frappe-path` option.
```shell
bench init --skip-redis-config-generation --frappe-path https://github.com/MyFork/frappe.git --frappe-branch develop frappe-bench
cd frappe-bench
```
For *version 13* use Python 3.9 by passing option to `bench init` command,

```shell
bench init --skip-redis-config-generation --frappe-branch version-13 --python python3.9 frappe-bench
cd frappe-bench
```

For *version 12* use Python 3.7 by passing option to `bench init` command,

```shell
bench init --skip-redis-config-generation --frappe-branch version-12 --python python3.7 frappe-bench
cd frappe-bench
```

### Setup hosts

We need to tell bench to use the right containers instead of localhost. Run the following commands inside the container:

```shell
bench set-config -g db_host mariadb
bench set-config -g redis_cache redis://redis-cache:6379
bench set-config -g redis_queue redis://redis-queue:6379
bench set-config -g redis_socketio redis://redis-socketio:6379
```

For any reason the above commands fail, set the values in `common_site_config.json` manually.

```json
{
  "db_host": "mariadb",
  "redis_cache": "redis://redis-cache:6379",
  "redis_queue": "redis://redis-queue:6379",
  "redis_socketio": "redis://redis-socketio:6379"
}
```

### Create a new site with bench

You can create a new site with the following command:

```shell
bench new-site sitename --no-mariadb-socket
```

sitename must end with .localhost for trying deployments locally, otherwise use the your intended fully quilified domain name.

for example:

```shell
bench new-site myerp.exmaple.com --no-mariadb-socket
```

The same command can be run non-interactively as well:

```shell
bench new-site myerp.exmaple.com --mariadb-root-password 123 --admin-password admin --no-mariadb-socket
```

The command will ask the MariaDB root password. The default root password you created earlier.
This will create a new site and a `myerp.exmaple.com` directory under `frappe-bench/sites`.
The option `--no-mariadb-socket` will configure site's database credentials to work with docker.
You may need to configure your system /etc/hosts if you're on Linux, Mac, or its Windows equivalent.

### Install an app

To install an app we need to fetch it from the appropriate git repo, then install in on the appropriate site:

You can check [VSCode container remote extension documentation](https://code.visualstudio.com/docs/remote/containers#_sharing-git-credentials-with-your-container) regarding git credential sharing.

To install custom app

```shell
# --branch is optional, use it to point to branch on custom app repository
bench get-app --branch version-12 https://github.com/myusername/myapp
bench --site mysite.localhost install-app myapp
```

At the time of this writing, the Payments app has been factored out of the Version 14 ERPNext app and is now a separate app. ERPNext will not install without it, however, so we need to specify `--resolve-deps` command line switch to install it.

```shell
bench get-app --branch version-14 --resolve-deps erpnext
bench --site mysite.localhost install-app erpnext
```

To install ERPNext (from the version-13 branch):

```shell
bench get-app --branch version-13 erpnext
bench --site mysite.localhost install-app erpnext
```

Note: Both frappe and erpnext must be on branch with same name. e.g. version-14

### Start Frappe without debugging

Execute following command from the `frappe-bench` directory.

```shell
bench start
```

You can now login with user `Administrator` and the password you choose when creating the site.
Your website will now be accessible at location [mysite.localhost:8000](http://mysite.localhost:8000)
Note: To start bench with debugger refer section for debugging.



## Use additional services during development

Add any service that is needed for development in the `.quick-deploy-container/docker-compose.yml` then rebuild and reopen in quick-deploy-container.

e.g.

```yaml
...
services:
 ...
  postgresql:
    image: postgres:11.8
    environment:
      POSTGRES_PASSWORD: 123
    volumes:
      - postgresql-data:/var/lib/postgresql/data
    ports:
      - 5432:5432

volumes:
  ...
  postgresql-data:
```

Access the service by service name from the `frappe` container. The above service will be accessible via hostname `postgresql`. If ports are published on to host, access it via `localhost:5432`.

