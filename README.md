# OSSN Docker

This repository contains the Docker setup for [Open Source Social Network (OSSN)](https://www.opensource-socialnetwork.org/), enabling you to quickly build and run OSSN in containers.

## Prerequisites

* Docker installed ([Get Docker](https://docs.docker.com/get-docker/))
* Docker Compose installed ([Get Docker Compose](https://docs.docker.com/compose/install/))


## Setup & Usage

### 1. Clone this repository

```
git clone https://github.com/opensource-socialnetwork/docker.git
cd docker
```

### 2. Build and start the containers

This will build the web container and start both web and database containers.

```
docker-compose up --build -d
```

Make sure port 80 allowed by ufw
```
sudo ufw allow 80/tcp
sudo ufw reload
 ```
- The OSSN web app will be available at `http://localhost` (or your server IP) or visit FQDN that points to your server IP example http://yourwebsite.com/
- MySQL database is preconfigured for OSSN with environment variables set in `docker-compose.yml`
- Follow the instructions and leave the pre-filled fields unchanged.

### 3. Stop the containers

```
docker-compose down
```

This stops and removes containers but **does not remove data volumes**.

### 4. Remove containers and volumes (clean start)

To remove containers **and** persistent data (database and OSSN files), run:

```
docker-compose down -v
```

> ⚠️ Warning: This deletes all your OSSN data!

### 5. Access the running web container

To enter the web container shell for debugging or file edits:

```
docker-compose exec web /bin/bash
```

### 6. Common container management commands

* **View logs**
  ```
  docker-compose logs -f
  ```
* **Rebuild container after Dockerfile changes**
  ```
  docker-compose build web
  ```
  or without cached
  ```
  docker-compose build --no-cache
  ```
* **Restart containers**
  ```
  docker-compose restart
  ```
## Notes

* The database hostname for OSSN is `db` (matching service name in Docker Compose).
* Persistent data is stored in Docker volumes (`db_data` for MySQL, `ossn_data` for OSSN files).
* To enable HTTPS, additional Apache configuration and SSL certificates are needed.

## Troubleshooting

* If you encounter database connection errors, ensure containers are running and OSSN connects using hostname `db`.
* For clean installs, remove volumes with `docker-compose down -v` to reset data.
* Check logs for errors: `docker-compose logs -f`

Happy Open Source Social Networking! 🎉
