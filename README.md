# WordPress + MariaDB + phpMyAdmin (Docker)

A minimal, official-images-only stack for running WordPress with a MariaDB
database and phpMyAdmin, managed by Docker Compose.

Three containers, one private bridge network, two published ports:

```
                ┌──────────────────────────────────────────┐
                │              Docker host                 │
                │                                          │
  :8080  ──────▶│  wordpress  ─────────┐                   │
                │  (Apache/PHP)        │                   │
                │                      ▼                   │
                │  phpmyadmin ─────▶  db (MariaDB)         │
  :8081  ──────▶│  (web UI)      (internal only, no host)  │
                │                                          │
                └──────────────────────────────────────────┘
```

| Container   | Image                      | Purpose               | Host port   |
| ----------- | -------------------------- | --------------------- | ----------- |
| `db`        | `mariadb:11.4`             | Database              | not exposed |
| `wordpress` | `wordpress:php8.3-apache`  | WordPress site        | `8080`      |
| `phpmyadmin`| `phpmyadmin:5.2-apache`    | Database web UI       | `8081`      |

MariaDB is reachable only from the internal Docker network. Only WordPress and
phpMyAdmin are published on the host.

## Project structure

```
.
├── docker-compose.yml   # the whole stack
├── .env.example         # configuration template (committed)
├── .env                 # your local configuration (NOT committed)
├── .gitignore
└── README.md
```

## Requirements

- A Linux server (or any host with Docker) with:
  - Docker Engine 24+ and Docker Compose v2 (`docker compose version`)
  - Ports `8080` and `8081` free (or other ports chosen in `.env`)
- Access to Docker:
  - as `root`, or
  - as a user in the `docker` group, or
  - with `sudo` before every `docker compose` command.

Install Docker on Ubuntu/Debian:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"   # log out and back in afterwards
```

Verify:

```bash
docker --version
docker compose version
```

## Step-by-step deployment

### 1. Get the project onto the server

```bash
git clone https://github.com/<your-account>/<your-repo>.git
cd <your-repo>
```

If you copied the files manually (without git), just `cd` into the folder that
contains `docker-compose.yml`.

### 2. Create your configuration file

Linux/macOS:

```bash
cp .env.example .env
```

Windows (PowerShell):

```powershell
Copy-Item .env.example .env
```

### 3. Specify server address, ports, logins and passwords

Open `.env` in any editor and set your values:

```dotenv
SERVER_HOST=203.0.113.10
WORDPRESS_PORT=8080
PHPMYADMIN_PORT=8081
WORDPRESS_URL=http://203.0.113.10:8080

DB_NAME=wordpress
DB_USER=master
DB_PASSWORD=Qwerty123
DB_ROOT_PASSWORD=Qwerty123
```

Rules:

- `WORDPRESS_URL` must be a **full URL that starts with `http://` or
  `https://`**. It is written directly into `WP_HOME` / `WP_SITEURL`. A value
  like `192.168.1.100:8080` (without a scheme) is treated as a relative path
  and produces broken, duplicated redirect URLs.
- It must match how users open the site:
  - port 8080: `http://192.168.1.100:8080`
  - port 80: `http://192.168.1.100` (no `:80`)
  - domain + HTTPS (reverse proxy): `https://example.com`
- No trailing slash in `WORDPRESS_URL`.
- If you change `WORDPRESS_PORT`, update `WORDPRESS_URL` accordingly.
- `SERVER_HOST` is only referenced in this documentation to build the access
  URLs; the containers do not read it. Editing `SERVER_HOST` alone does not
  change the site address — always update `WORDPRESS_URL` too.
- `.env` is ignored by git, so secrets are never committed.

Verify the generated WordPress configuration before starting:

```bash
docker compose config | grep -A2 WORDPRESS_CONFIG_EXTRA
```

The output must contain the full URL with the scheme, for example
`define('WP_HOME', 'http://192.168.1.100:8080');`.

### 4. Start the stack

```bash
docker compose up -d
```

On a fresh host Compose builds nothing (all images are pulled from Docker Hub),
so this usually takes 1-2 minutes. Check the result:

```bash
docker compose ps
docker compose logs -f wordpress
```

Wait until `wordpress-db` is `healthy` and the WordPress container is `running`.

### 5. Open WordPress and finish the installation

Open the address from `WORDPRESS_URL`, for example:

```
http://192.168.1.100:8080
```

You will see the WordPress "Welcome" screen. Complete the 5-minute install:

| Field            | Value              |
| ---------------- | ------------------ |
| Site Title       | anything you like  |
| Username         | `master`           |
| Password         | `Qwerty123`        |
| Your Email       | your email address |

The WordPress administrator account is created here (it is not stored in
`.env`). The default `master` / `Qwerty123` is what the project uses by
default; change it later in **Users → Profile**.

### 6. Open phpMyAdmin

```
http://192.168.1.100:8081
```

Log in with:

| Field    | Value        |
| -------- | ------------ |
| Username | `master`     |
| Password | `Qwerty123`  |

The server/host field is filled automatically (`db`). The MariaDB `root`
account (`DB_ROOT_PASSWORD`) can also be used if needed.

## Configuration reference (`.env`)

| Variable            | Default                          | Description                                          |
| ------------------- | -------------------------------- | ---------------------------------------------------- |
| `SERVER_HOST`       | `YOUR_SERVER_IP_OR_DOMAIN`       | Server IP/domain (documentation only; containers do not read it) |
| `WORDPRESS_PORT`    | `8080`                           | Host port for the site                               |
| `PHPMYADMIN_PORT`   | `8081`                           | Host port for phpMyAdmin                             |
| `WORDPRESS_URL`     | `http://YOUR_SERVER_IP_OR_DOMAIN:8080` | Full site URL incl. `http://`/`https://`; becomes `WP_HOME` / `WP_SITEURL` |
| `DB_NAME`           | `wordpress`                      | Database name                                        |
| `DB_USER`           | `master`                         | Database user (also phpMyAdmin login)                |
| `DB_PASSWORD`       | `Qwerty123`                      | Password for `DB_USER`                               |
| `DB_ROOT_PASSWORD`  | `Qwerty123`                      | MariaDB root password                                |

`DB_*` values are applied only when the database volume is initialized for the
first time. Changing them later requires an SQL update (see below).

Apply `.env` changes by recreating the containers:

```bash
docker compose up -d
```

## Common commands

```bash
docker compose ps                 # status of containers
docker compose logs -f            # follow logs of all containers
docker compose logs -f db         # follow database logs
docker compose restart            # restart all containers
docker compose stop               # stop without removing
docker compose down               # remove containers (data is kept)
docker compose down -v            # remove containers AND volumes (data loss!)
```

## Using port 80

Set in `.env`:

```dotenv
WORDPRESS_PORT=80
WORDPRESS_URL=http://203.0.113.10
```

Then `docker compose up -d`.

On Linux the Docker daemon (running as root) binds port 80, so the command
usually works as-is. If you run Docker in rootless mode or get a
"permission denied" error, either run with `sudo` or lower the unprivileged
port limit:

```bash
sudo sysctl net.ipv4.ip_unprivileged_port_start=80
```

Make sure port 80 is not already used by another web server:

```bash
sudo ss -ltnp | grep ':80 '
```

## Changing the database password on an existing installation

The MariaDB entrypoint only creates users on the first start (empty volume).
If the stack is already running and you want new credentials:

1. Log in to the database as root:

   ```bash
   docker compose exec db sh -c 'exec mariadb -uroot -p"$MARIADB_ROOT_PASSWORD"'
   ```

   This works in Bash, Zsh and PowerShell.

2. Check which root accounts exist, then change the passwords:

   ```sql
   SELECT User, Host FROM mysql.user WHERE User IN ('master', 'root');
   ALTER USER 'master'@'%' IDENTIFIED BY 'NewUserPassword';
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewRootPassword';
   -- if the query also listed root@'%', change it too:
   -- ALTER USER 'root'@'%' IDENTIFIED BY 'NewRootPassword';
   FLUSH PRIVILEGES;
   EXIT;
   ```

3. Update `DB_PASSWORD` / `DB_ROOT_PASSWORD` in `.env`.
4. Apply:

   ```bash
   docker compose up -d
   ```

## Backup and restore

### Database

Backup:

```bash
docker compose exec db sh -c 'exec mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --all-databases > /tmp/backup.sql'
docker compose cp db:/tmp/backup.sql ./backup.sql
```

Restore:

```bash
docker compose cp ./backup.sql db:/tmp/restore.sql
docker compose exec db sh -c 'exec mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" < /tmp/restore.sql'
```

### WordPress files

Backup:

```bash
docker compose exec wordpress tar czf /tmp/wp_data.tar.gz -C /var/www/html .
docker compose cp wordpress:/tmp/wp_data.tar.gz ./wp_data.tar.gz
```

Restore:

```bash
docker compose cp ./wp_data.tar.gz wordpress:/tmp/wp_data.tar.gz
docker compose exec wordpress tar xzf /tmp/wp_data.tar.gz -C /var/www/html
```

The `docker compose cp` approach works identically in Bash, Zsh and
PowerShell, and avoids shell encoding problems with binary/UTF-8 streams.

## Updating WordPress and images

```bash
docker compose pull
docker compose up -d
```

WordPress can also be updated from **Dashboard → Updates** inside the admin
panel. Database data and site files live in named volumes and survive updates.

## Troubleshooting

**Port already in use**

```bash
docker compose ps
sudo ss -ltnp | grep -E ':(8080|8081) '
```

Change `WORDPRESS_PORT` / `PHPMYADMIN_PORT` in `.env` and run
`docker compose up -d`.

**"Error establishing a database connection"**

The database may still be starting. Run `docker compose logs db` and wait for
"ready for connections". On the very first start this can take 20-40 seconds;
WordPress will recover after a page reload.

**phpMyAdmin says "Cannot log in to the MySQL server"**

- Use server `db` (the default).
- Check that the username/password match `DB_USER` / `DB_PASSWORD`.
- If you changed `.env` after the first start, the old credentials are still
  active in the database — see the password-change section above.

**Redirect URL repeats the host, e.g. `http://192.168.1.100:8080/192.168.1.100:8080/wp-admin/...`**

`WORDPRESS_URL` is missing the scheme or is otherwise malformed (for example
`192.168.1.100:8080` instead of `http://192.168.1.100:8080`). Without
`http://`/`https://` WordPress treats the value as a relative path and
duplicates it in every generated link.

1. Set the full URL in `.env` on the server:

   ```dotenv
   SERVER_HOST=192.168.1.100
   WORDPRESS_PORT=8080
   WORDPRESS_URL=http://192.168.1.100:8080
   ```

2. Verify Compose interpolation:

   ```bash
   docker compose config | grep -A2 WORDPRESS_CONFIG_EXTRA
   ```

3. Recreate the container and check the generated config:

   ```bash
   docker compose up -d --force-recreate wordpress
   docker compose exec wordpress grep -E "WP_HOME|WP_SITEURL" wp-config.php
   ```

4. Open the site in a private window — browsers cache the old 301 redirects.

5. If the site was already installed with a broken URL, also fix the values
   stored in the database via phpMyAdmin:

   ```sql
   UPDATE wp_options SET option_value='http://192.168.1.100:8080'
   WHERE option_name IN ('siteurl','home');
   ```

**Site redirects to the wrong address**

`WORDPRESS_URL` does not match the URL used in the browser. Fix `.env`, then
`docker compose up -d` and clear the browser cache.

**Permission errors on volumes**

The official images manage ownership of the named volumes themselves. Do not
mount host directories owned by another user into `/var/www/html` unless you
also fix permissions.

**Containers keep restarting**

```bash
docker compose logs --tail=100 wordpress
docker compose logs --tail=100 db
```

**Firewall blocks access**

Open the published ports on the host, for example with UFW:

```bash
sudo ufw allow 8080/tcp
sudo ufw allow 8081/tcp
```

## Security notes

This stack is intentionally simple and serves HTTP. Before exposing it to the
internet:

- Change the default passwords (`master` / `Qwerty123`, root password).
- Restrict phpMyAdmin with a firewall or an SSH tunnel. It is published on
  all interfaces by default; prefer allowing port 8081 only from trusted IPs:

  ```bash
  sudo ufw allow from 203.0.113.5 to any port 8081 proto tcp
  ```

  Or reach it through an SSH tunnel without opening the port at all:
  `ssh -L 8081:127.0.0.1:8081 user@server`.
- Do not publish MariaDB port 3306 to the host.
- Use HTTPS in production: put Nginx, Caddy or Traefik in front of the stack,
  set `WORDPRESS_URL=https://example.com`, and forward traffic to
  `127.0.0.1:8080`.
- Keep Docker and the images up to date (`docker compose pull`).

## License

No license is provided. Add one if you plan to redistribute the project.
