# Overview

This is an experiment/Lab to practice deploying microservice application a local ubuntu VM. The `docker-compose.yml `file aggregates backend images required for the microserve api to work. Backend has two APIs - `microserve-admin` and `microserve-main`

Also supporting services : `rabbit`, `postgres` and `mysql` are also created with this `docker-compose` in the production server

### Related repositories:

- [microserve-main](https://github.com/sudiptob2/microserve-main)
- [microserve-main-front](https://github.com/sudiptob2/microserve-main-front)
- [microserve-admin](https://github.com/sudiptob2/microserve-admin)
- [microserve-admin-front](https://github.com/sudiptob2/microserve-admin-front)

![Alt text](images/system-architecture.jpg?raw=true "System Architecture")

# Setting Up a Ubuntu VM

## Multipass setup

We will use multipass to create a virtual machine in our host machine. In this case my host machine is **ubuntu-20.0**

- Install multipass
- Spin up a VM

**commands**

```
sudo snap install multipass

multipass launch --name demo-server
```

This will create a lightweight VM with Ubuntu OS installed in it.

## NGINX setup

At this point let's install **NGINX** and **lynx - a command line browser**

```
multipass exec demo-server bash
sudo apt update
sudo apt install nginx
sudo apt instll lynx
```

After installation NGINX will be started automatically. Check the status of NGINX using the following command

```
sudo systemctl status nginx
```

If NGINX is not active start the NGINX using the following command.

```
sudo systemctl start nginx
```

Now if you visit `lynx http:localhost:80` you will see the default nginx page in this command line browser

In the host machine terminal use `multipass list` command to list down all multipass instances. Your output will look something like following.

```
Name                    State             IPv4             Image
primary                 Suspended         --               Ubuntu 20.04 LTS
demo-server             Running           10.79.154.253    Ubuntu 20.04 LTS

```

copy the IPV4 address of the demo-server and visit the IP address from Chrome browser in your host. You will be able to view the default NGINX page served in the browser.

We have two front end website as you can see in the system design diagram. Let's create to fake domain aliases in our host machine for them.

I will use _microserve.com_ and _admin.microserve.com_ for main and admin website respectively.

```
sudo vim /etc/hosts

add the following lines

...
10.79.154.253 admin.microserve.com
10.79.154.253 microserve.com
...

```

Now if you just type [http://microserve.com](http://microserve.com) it will be redirected to the ip address of the VM and show the default NGINX page at this point.

# Setup Dokcer and docker-compose

As our backend applications will be running using docker we have to install docker and docker compose. Just follow the official docker instruction to install docker and docker compose in the VM.

- [docker installation](https://docs.docker.com/engine/install/ubuntu/)
- [must follow post installation guide](https://docs.docker.com/engine/install/linux-postinstall/)
- [docker-compose installation](https://docs.docker.com/compose/install/)

# Setup GitHub SSH

Our codes are in GitHub. so you have to set up an ssh to pull codes from the GitHub. Otherwise, you can copy codes from host to multipass VM using `multipass transfer` command. We will use both ways in this guide.

Here is a nice blog on how to set up ssh in GitHub. [setup ssh in GitHub](https://www.inmotionhosting.com/support/server/ssh/how-to-add-ssh-keys-to-your-github-account/)

Now pull this repository into your VM.

# Spin up the backend

To run the backend you have to go to the repository folder. Now create a `.env` file form the `.env.template` file

Then just use following command to run the docker compose.

```
docker-compose up -d
```

see the status of the containers using following command

```
docker ps
```

# Get the front-end files

We can pull the front-end code from the github directly into the server. But as I said earlier, we will use `multipass transfer command` to move files from host computer to the server.

Pull both front-end repo in your host machine. Now run the following commands to generate production build for the angular and react based frontends.

### Admin frontend(React)

First put appropriate env variables in the .env file. Base url in this case will be _admin.microserve.com_

```
docker-compose up -d
```

```
docker-compose exec web npm run build
```

make zip of the build folder

```
tar -czvf build-admin.tar.gz build
```

Transfer files to the multipass server

```
multipass transfer  build-admin.tar.gz demo-server:build-admin.tar.gz
```

Now in the server, unzip the files

```
tar -xf build-admin.tar.gz
```

Finally move the files into the `/var/www/admin-front` directory

```
sudo cp -r build/. /var/www/admin-front/
```

Similarly, we have to build and move the build folder to `/var/www/main-front` directory for the angular based main app.

# Setting up nginx configurations

First, go to `/etc/nginx/sites-available` directory. Here we will add two configuration file. On for our admin panel website another for main website. First remove the `default` configuration file placed there. Next, create a file `microserve.com` and paste the following configuration there.

```
server {
    listen       80;
    server_name  microserve.com www.microserve.com;

    location /api/ {
	    proxy_set_header HOST $host;
	    proxy_set_header X-Forwarded-Proto $scheme;
    	    proxy_set_header X-Real-IP $remote_addr;
    	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	    proxy_pass http://127.0.0.1:8001;
    }


    location / {
        root   /var/www/main-front;
        try_files $uri $uri/ /index.html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /var/www/error;
    }

}


```

create another file `admin.microserve.com` and paste the following configuration there.

```
server {
    listen       80;
    server_name  admin.microserve.com  www.admin.microserve.com;

    location / {
        root   /var/www/admin-front;
        try_files $uri $uri/ /index.html;
        index  index.html index.htm;
    }

    location /api/ {
	    proxy_set_header HOST $host;
	    proxy_set_header X-Forwarded-Proto $scheme;
    	    proxy_set_header X-Real-IP $remote_addr;
    	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	    proxy_pass http://127.0.0.1:8000;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /var/www/error;
    }

}

```

Now to enable these two configuration file we have to create a `sys link` with the folder `sites-enabled`

```
sudo ln -s /etc/nginx/sites-available/microserve.com /etc/nginx/sites-enabled/microserve.com
```

```
sudo ln -s /etc/nginx/sites-available/admin.microserve.com /etc/nginx/sites-enabled/admin.microserve.com
```

Finally, we have to reload the NGINX so that the changes can take effect.

```
nginx -s reload
```

Now if you visit `microserve.com` and `admin.microserve.com` you will be able to see the desired webpages.

However, the websites don't use **https** protocol. Let's install a self-signed **TLS** certificate so that we can use **https**

Follow [this blog](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu-18-04) to install a self-signed certificate.

The final NGINX config files will look like something as follows.

_microserve.com_

```
server {

    listen 443 ssl;
    listen [::]:443 ssl;
    include snippets/self-signed.conf;
    include snippets/ssl-params.conf;

    server_name  microserve.com www.microserve.com;

    location /api/ {
	    proxy_set_header HOST $host;
	    proxy_set_header X-Forwarded-Proto $scheme;
    	    proxy_set_header X-Real-IP $remote_addr;
    	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

	    proxy_pass http://127.0.0.1:8001;
    }


    location / {
        root   /var/www/main-front;
        try_files $uri $uri/ /index.html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /var/www/error;
    }

}

server {
    listen 80;
    listen [::]:80;
    server_name  microserve.com www.microserve.com;
    return 302 https://$server_name$request_uri;
}

```

_admin.microserve.com_

```
server {

    listen 443 ssl;
    listen [::]:443 ssl;
    include snippets/self-signed.conf;
    include snippets/ssl-params.conf;

    server_name  admin.microserve.com  www.admin.microserve.com;

    location / {
        root   /var/www/admin-front;
        try_files $uri $uri/ /index.html;
        index  index.html index.htm;
    }

    location /api/ {
	    proxy_set_header HOST $host;
	    proxy_set_header X-Forwarded-Proto $scheme;
    	    proxy_set_header X-Real-IP $remote_addr;
    	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

	    proxy_pass http://127.0.0.1:8000;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /var/www/error;
    }

}

server {
    listen 80;
    listen [::]:80;
    server_name  admin.microserve.com www.admin.microserve.com;
    return 302 https://$server_name$request_uri;
}

```
