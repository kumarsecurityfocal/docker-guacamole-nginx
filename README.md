# Guacamole with NGINX Proxy Manager Setup using Docker

This guide outlines the steps to set up **Apache Guacamole** and **NGINX Proxy Manager** using Docker and Docker Compose, with MariaDB as the database backend.

---

## **1. Prerequisites**

Ensure the following are installed on your system:

- Docker
- Docker Compose plugin

You can verify the installations with:

```bash
docker --version
docker compose version
```

---

## **2. Clone the Repository or Create a Setup Directory**

Create a directory for the setup:

```bash
mkdir -p ~/guacamole-setup
cd ~/guacamole-setup
```

---

## **3. Create the Docker Compose File**

1. Create a `docker-compose.yml` file:

   ```bash
   nano docker-compose.yml
   ```

2. Add the following content:

   ```yaml
   services:
     # Guacamole Backend Proxy
     guacd:
       image: guacamole/guacd:latest
       container_name: guacd
       restart: always
       ports:
         - "4822:4822"

     # Guacamole Web Application
     guacamole:
       image: guacamole/guacamole:latest
       container_name: guacamole
       restart: always
       depends_on:
         - guacd
         - guacamole-db
       environment:
         GUACD_HOSTNAME: guacd
         MYSQL_HOSTNAME: guacamole-db
         MYSQL_DATABASE: guacamole_db
         MYSQL_USER: guacamole_user
         MYSQL_PASSWORD: guac_password
         MYSQL_ROOT_PASSWORD: root_password
       ports:
         - "8080:8080"

     # Guacamole Database (MariaDB)
     guacamole-db:
       image: mariadb:latest
       container_name: guacamole-db
       restart: always
       environment:
         MYSQL_ROOT_PASSWORD: "root_password"
         MYSQL_DATABASE: "guacamole_db"
         MYSQL_USER: "guacamole_user"
         MYSQL_PASSWORD: "guac_password"
       volumes:
         - ./mysql-guacamole:/var/lib/mysql

     # NGINX Proxy Manager (Application)
     npm-app:
       image: jc21/nginx-proxy-manager:latest
       container_name: npm-app
       restart: always
       depends_on:
         - npm-db
       ports:
         - "80:80"
         - "81:81"
         - "443:443"
       environment:
         DB_MYSQL_HOST: "npm-db"
         DB_MYSQL_PORT: 3306
         DB_MYSQL_USER: "npm"
         DB_MYSQL_PASSWORD: "npm_password"
         DB_MYSQL_NAME: "npm"
       volumes:
         - ./npm-data:/data
         - ./npm-letsencrypt:/etc/letsencrypt

     # NGINX Proxy Manager (Database)
     npm-db:
       image: mariadb:latest
       container_name: npm-db
       restart: always
       environment:
         MYSQL_ROOT_PASSWORD: "root_password"
         MYSQL_DATABASE: "npm"
         MYSQL_USER: "npm"
         MYSQL_PASSWORD: "npm_password"
       volumes:
         - ./mysql-npm:/var/lib/mysql
   ```

3. Replace `root_password`, `guac_password`, and `npm_password` with secure passwords.

---

## **4. Start the Containers**

Run the following command to start all the containers:

```bash
docker compose up -d
```

Verify that all containers are running:

```bash
docker ps
```

---

## **5. Initialize the Guacamole Database**

Install ****`mysql`**** Client in the MariaDB Container**

1. Access the `guacamole-db` container:

   ```bash
   docker exec -it guacamole-db bash
   ```

2. Install the `mysql` client:

   ```bash
   apt-get update
   apt-get install mysql-client -y
   ```

3. Exit the container:

   ```bash
   exit
   ```

4. Import the database schema:

   ```bash
   docker exec -i guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
   docker exec -i guacamole-db mysql -u root -proot_password guacamole_db < initdb.sql
   ```

## **6. Access the Applications**

### **NGINX Proxy Manager**

- URL: `http://<server-ip>:81`
- Default credentials:
  - **Email**: `admin@example.com`
  - **Password**: `changeme`

### **Guacamole**

- URL: `http://<server-ip>:8080`
- Default credentials:
  - **Username**: `guacadmin`
  - **Password**: `guacadmin`

---

## **7. Configure NGINX Proxy Manager for Guacamole**

1. Log in to NGINX Proxy Manager at `http://<server-ip>:81`.
2. Add a new **Proxy Host**:
   - **Domain Name**: `yourdomain.com` (or subdomain like `guac.yourdomain.com`).
   - **Forward Hostname/IP**: `guacamole`.
   - **Forward Port**: `8080`.
   - Enable **Websockets Support**.
3. Add an **SSL Certificate**:
   - Choose **Let's Encrypt**.
   - Enter your email address and agree to the terms.
4. Save and apply the settings.

---

## **8. Verify the Setup**

1. Access Guacamole through the domain configured in NGINX Proxy Manager (e.g., `https://guac.yourdomain.com`).
2. Ensure everything is working, including SSL and proxy configurations.

---

## **9. Optional Enhancements**

- **Backups**:
  - Regularly back up the `./mysql-guacamole` and `./mysql-npm` directories.
- **Security**:
  - Enable a firewall (e.g., `ufw`) and allow only necessary ports (80, 443).
  - Regularly update Docker images using:
    ```bash
    docker compose pull
    docker compose up -d
    ```

---

## **10. Troubleshooting**

### **Check Logs**

If something doesn’t work, check the logs for individual containers:

```bash
docker logs guacd
docker logs guacamole
docker logs npm-app
docker logs npm-db
docker logs guacamole-db
```

### **Common Issues**

- **Database Initialization Failed**: Ensure the database is running and accessible.
- **NGINX Proxy Manager Login Issues**: Reset credentials or check the logs of the `npm-app` container.

---

This completes the setup of Apache Guacamole and NGINX Proxy Manager using Docker!

