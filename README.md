# BlueSky
**NOTE: SolarWinds does not provide technical support for BlueSky.** Visit the #solarwinds-msp channel of MacAdmins Slack for unofficial help.

BlueSky establishes and maintains an SSH tunnel initiated by your client’s computer to a BlueSky server. The tunnel allows two connections to come back to the computer from the server: SSH and VNC. The SSH and VNC services on the computer are the ones provided by the Sharing.prefpane.

You use an Admin app to connect via SSH to the BlueSky server and then follow the tunnel back to your client computer. You select which computer by referencing its BlueSky ID as shown in the web admin.

Apps are provided to connect you to remote Terminal (SSH), Screen Sharing (VNC), and File/Folder copying (SCP). You still need to be able to authenticate as a user on the target computer.

Since BlueSky from your client computers is an outgoing connection most SMB networks won’t block it. In enterprise environments, BlueSky can read the proxy configuration in system preferences and send the tunnel through a proxy server.

Read more in the [Wiki](https://github.com/logicnow/BlueSky/wiki)

## Docker setup

This is an attempt at getting BlueSky to run within docker.  This should allow it to be easily setup on a wide variety of systems that support docker containers.

### Environment variables

These variables can be overriden when you run the bluesky docker container.

Variable | Default Value | Note
--- | --- | ---
SERVERFQDN | localhost | Bluesky FQDN
WEBADMINPASS | admin | 
USE_HTTP | 0 | Set to 1 to use HTTP instead of HTTPS
MYSQLSERVER | db | IP/DNS of your MySQL server (docker link to db by default)
MYSQLROOTPASS | admin | 
EMAILALERT | root@localhost | 
SMTP_SERVER | | SMTP Server (Required for email alerts) Port optional
SMTP_AUTH | | SMTP auth user (Required for email alerts)
SMTP_PASS | | SMTP auth pass (Required for email alerts)

### Docker volumes

The following locations are mappable locations within the container.  These will be used for data that needs to persist between runs.

Path | Note
--- | ---
/certs | BlueSky SSH keys
/home/admin/.ssh | ?
/home/bluesky/.ssh | ?
/home/admin/newkeys | ?
/home/bluesky/newkeys | ?
/home/ssl/certs | HTTPS certificate
/home/ssl/private | HTTPS private key
/tmp/pkg | Client install pkg

### Example Setup: Persistant storage

#### MySQL

Because we are also using MySQL within docker in this example setup, we need to setup the local storage first.  In the example below we are mapping the `/var/docker/bluesky/db` directory on the host to have the persistent mysql data.

First create and set permissions on the local folder:
_you may want to modify permissions depending on how you are running docker_
```
sudo mkdir -p /var/docker/bluesky/db
sudo chmod -R 777 /var/docker/bluesky
```

#### BlueSky

Because docker images do not keep their data between runs, we need to map locations (volumes) for persistent storage.  In the example below we are mapping various directories within `/var/docker/bluesky/` on the host to have the persistent key data.

First create and set permissions on the local folder:
_you may want to modify permissions depending on how you are running docker_
```
sudo mkdir -p /var/docker/bluesky/certs
sudo mkdir -p /var/docker/bluesky/admin.ssh
sudo mkdir -p /var/docker/bluesky/admin.newkeys
sudo mkdir -p /var/docker/bluesky/bluesky.ssh
sudo mkdir -p /var/docker/bluesky/bluesky.newkeys
sudo mkdir -p /var/docker/bluesky/pkg
sudo mkdir -p /var/docker/bluesky/ssl-certs
sudo mkdir -p /var/docker/bluesky/ssl-private
sudo chmod -R 777 /var/docker/bluesky
```

### Run BlueSky

Setup MySQL:
_note that currently the root password is embedded in this command_
```
docker run -d --name bluesky_db \
	-v /var/docker/bluesky/db:/var/lib/mysql \
	-e MYSQL_ROOT_HOST=% \
	-e MYSQL_ROOT_PASSWORD=admin \
	mysql/mysql-server:5.7
```

Wait for the MySQL docker container to initialize...

Setup BlueSky:
```
docker run -d --name bluesky \
	--link bluesky_db:db \
	-e SERVERFQDN=bluesky.example.com \
	-v /var/docker/bluesky/certs:/certs \
	-v /var/docker/bluesky/admin.ssh:/home/admin/.ssh \
	-v /var/docker/bluesky/bluesky.ssh:/home/bluesky/.ssh \
	-v /var/docker/bluesky/admin.newkeys:/home/admin/newkeys \
	-v /var/docker/bluesky/bluesky.newkeys:/home/bluesky/newkeys \
	-v /var/docker/bluesky/ssl-certs:/home/ssl/certs \
	-v /var/docker/bluesky/ssl-private:/home/ssl/private \
	-v /var/docker/bluesky/pkg:/tmp/pkg \
	-p 80:80 \
	-p 443:443 \
	-p 3122:22 \
	sphen/bluesky
```

### HTTPS SSL Certificate Setup

If you are opting to use HTTPS within the docker container you should map in valid SSL certificates.  By default with no action, the container will generate a self-signed certificate.  If you have a valid `pem` and `key` file that you would like to use, we expect a few things:
- You are mapping the `/home/ssl/certs` and `/home/ssl/private` volumes
- The pem file to use within certs is named `ssl-cert-snakeoil.pem`
- The key file to use within private is name `ssl-cert-snakeoil.key`

### Upgrading BlueSky Server

To update your BlueSky docker instance it is very simple
```
docker pull sphen/bluesky
docker rm -f BlueSky
```

Then just issue the `docker run` command you originally used to build your specific container!
```
docker run -d --name bluesky <the rest of your arguments...> sphen/bluesky
```

### Troubleshooting

You can get the logs from a container to see if there are any issues:
```
docker logs bluesky
```

You can also shell into the bluesky container if needed:
```
docker exec -it bluesky bash
```

### TODO

~~- Set up persistant storage for the bluesky container.~~
  - ~~This will require alot of changes to get those files within different directories.~~
- ~~Fix SSL if being used in container.~~
  - ~~right now complaining about "/etc/ssl/certs/ssl-cert-snakeoil.pem"~~
  - ~~outline instructions for mapping your own cert.~~
- Add example of Caddy docker container in front of bluesky for auto-generated SSL certificates :)
- ~~Auto-generate client installer~~
- ~~Handle emailHelper setup~~
- SSH Pub Key auth by default - instructions on setting keys
- Bring back fail2ban

### Links

- Auto-build on Docker Hub: https://hub.docker.com/r/sphen/bluesky/
- Forked BlueSky on GitHub: https://github.com/logicnow/BlueSky
