---
layout: post
title:  "Convert an Existing WordPress Site to Docker"
date:   2015-07-18 13:33:17 -0600
categories: wordpress docker
---

I had a "classic" install of WordPress, Apache/PHP, and Mysql installed on the root drive, that I wanted to convert to Docker containers.

### Why?

**Install**

Installing is easy. Installing and starting publicly available images is a single `docker run` command.

**Maintain**

With a little bit of planning updating your apps is a simple process.

- `docker pull` the latest image.
- `docker rm -f` the current container (remember to use volumes on data you need to save).
- `docker run` with the same options and you have the latest app.

**Scale**

In the off chance that someone besides my Mom (hi Mom!) reads my blog, scaling containerized apps is a breeze.

- Add another host.
- `docker run` your app with the same options.
- Add a load balancer.

Note: These commands assume you are running as root.

## Saving Your WordPress Content

### Dump Your Database

This will dump the WordPress database to a sql file. You will need your MySQL server hostname, user, and password. If you don't remember what that is, check the `wp-config.php` file in your current WordPress install.

Enter your WordPress DB user password when prompted.

```
mysqldump -h [hostname] -p -u [user] [database] > ~/wordpress.sql
```

### Backup the wp-content directory

All the stuff you need to save is in the wp-content file in your WordPress install.

```
tar cvzf ~/wp-content.tar.gz ./wp-content
```

## Docker all the Things

### Install Docker

Run the install script from <a href="http://www.docker.com">http://www.docker.com</a>:

```
wget -qO- https://get.docker.com/ | sh
```

### Install MySQL Container

**Make a storage directory**

Make a directory to persistently store the DB files. This will make it easier to update the Docker Image later.

```
mkdir -p /data/mysql
```

**Install/Start a MySQL container**

This is really as simple as:

```
docker run -d \
--volume=/data/mysql:/var/lib/mysql \
--restart=always \
--publish=172.17.42.1:3306:3306 \
--env='MYSQL_USER=wp-user' \
--env='MYSQL_PASSWORD=myAwesomePassword' \
--env='MYSQL_DATABASE=wordpress' \
--env='MYSQL_ROOT_PASSWORD=myAwesomeRootPassword' \
--name=mysql \
mysql:5.6
```

**What's going on here?**

- `docker run` - Start a Container from an Image. If that image is not already downloaded, it will try to download it from the specified repository.
- `-d` - Run the Container as a Daemon (in the background).
- `--volume /data/mysql:/var/lib/mysql` - Mount the local directory `/data/mysql` in the Container at /var/lib/mysql.
- `--restart=always` - Tell docker to restart the container on boot and anytime it dies.
- `--publish 172.17.42.1:3306:3306` - Here we are telling docker to map port 3306 in the container to the special internal docker0 ip address 172.17.42.1. This prevent external access but allow the local host and any local docker containers to communicate with MySQL.
- `--env 'MYSQL_USER=wp-user'` - Use env to pass parameters in to your container. In this case we are passing in the default user, database, and passwords. Good Images will document the available environmental variables.
- `--name mysql` - This is the friendly name you can set for easy control of the container.
- `mysql:5.6` - This is the Image Name followed by the Image Tag. This is calling the registry.hub.docker.com default `library/mysql` image. This and other public images can be found at <a href="https://registry.hub.docker.com/">https://registry.hub.docker.com/</a>

**ADVANCED: Why not use `--link`?**

The problem with linking containers is, if the "Linked To" container (mysql) is restated for any reason it will get a new IP address. Any Linking containers (wordpress) will need to be restarted to pick up that new IP address. This really becomes an issue if you have multiple containers linked or nested dependent containers. I believe that linking causes more issues then it solves. Its better to treat all of your containers as if they were on separate hosts.

### Import the DB

Import that .sql file you saved earlier. Use the IP and credentials you specified in the previous step.

```
mysql -h 172.17.42.1 -p -u wp-user wordpress < ~/wordpress.sql
```

### Install WordPress

**Extract Your wp-content Backup**

Just like the MySQL Container you will need persistent storage for the WordPress content. Create a directory and extract the tarball. You may need to update the owner/group so the WordPress container can modify the data.

```
mkdir -p /data/wordpress
cd /data/wordpress
tar xvzf ~/wp-content.tar.gz
chown -R www-data:www-data ./wp-content
```

**Install/Start the WordPress Container**

Just like MySQL we are using the public WordPress Image. Set the env variables with the IP and credentials you defined when you set up the MySQL container. This container is bound to port 80 on all of the host IP addresses.

```
docker run -d \
--restart=always \
--publish=80:80 \
--volume=/data/wordpress/wp-content:/var/www/html/wp-content \
--env='WORDPRESS_DB_HOST=172.17.42.1' \
--env='WORDPRESS_DB_USER=wp-user' \
--env='WORDPRESS_DB_PASSWORD=myAwesomePassword' \
--env='WORDPRESS_DB_NAME=wordpress' \
--name=wordpress \
wordpress:latest
```

### Check docker

At this point we should be done. You can check to see if the docker containers are running.

```
# docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                        NAMES
685247608fff        wordpress:latest    "/entrypoint.sh apac   3 weeks ago         Up About an hour    0.0.0.0:80->80/tcp           wordpress
75b5732cabdc        mysql:5.6           "/entrypoint.sh mysq   3 weeks ago         Up About an hour    172.17.42.1:3306->3306/tcp   mysql
```

Browse to port 80 on your server and you should see your site.

### Troubleshooting Tip

If something has gone wrong you can check the logs from your containers.

```
docker logs -f <container>
```
