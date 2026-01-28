---
title: 'Dockerised REDCap Installation for Research Data Management'
date: 2024-08-14
permalink: /posts/2024/08/dockerised-redcap/
tags:
  - docker
  - redcap
  - linux
---

Running REDCap locally on your PC  shouldn’t be fragile or painful. In this post, I walk through how I Dockerised REDCap to create a portable, reproducible, and easy-to-maintain research data platform on my laptop.

## Pre-requisite
1. Docker and docker composed installed. Follow the offical [documentation](https://docs.docker.com/engine/install/) for installation guide
1. A valid licence from the REDCap [consortium](https://projectredcap.org/about/consortium/).
1. [Git](https://git-scm.com/install/linux) installed
1. Access to the redcap source files zipped

## Steps
* First, lets create a working directory to put everything in one place.

```bash
mkdir redcap-project
cd redcap-project/

```

* Andy123 has this nice repository with a docker compose file for configuring redcap inside docker. Let clone the project using the following git command

```bash
git clone git@github.com:123andy/redcap-docker-compose.git .

```

* We need to configure the redcap application to our own preference. To do this we need to edit the .env file localated in the path **rdc** directory. We can make a copy of the .env-example file given to set our configurations
```bash
cd rdc
cp .env-example .env
nano .env
```
Read the settings defined in the file. The follow are key to ensure redcap works properly
* **id** variable should be set to your user assigned id on your computer. You can run the following shell to get your id

```bash
id -u
```

* Set where redcap web app should be runned from, by default it is the root / directory. you can specify a different path  
REDCAP_WEBROOT_PATH = /redcap/

* To secure your installation, you need to specify a salt value for cryptographic functions within redcap.You can generate one [here](https://generate-random.org/salts)
REDCAP_SALT=MbtCG+xIfgn/fAY/F5xcwQKU1YadtnSo
* Set a strong password for the following database parameters. You can use lastpass to generate strong passwords and store them securely inside the [lastpass](https://www.lastpass.com/) vault


  - MYSQL_PORT=3306
  - MYSQL_ROOT_PASSWORD=Strong Password
  - MYSQL_DATABASE=redcap
  - MYSQL_USER=redcap
  - MYSQL_PASSWORD=Strong password

* Now **lets Go Build the Image**.  This might take some time. Coffee break!

```bash
cd rdc
docker compose build
```

If everything runned successfully, you shouldn't get any errors. If you make changes to the docker-compose.yml file, I strongly recommend you run docker compose build --no-cache

```
docker compose build --no-cache
```

the above command will ensure the docker images are rebuild from scratch.

* Verify you have the image build by using the following docker command

```
docker images -a
```

* With the docker images build successfully, it's time we spin some container off of them. The first time the containers are runned may take some time.

```
cd rdc
docker compose up -d
```

* You can view the logs from the container using the command

```
docker compose logs -f
```

* To test if everything worked as expected, visit **http://localhost** or **http://127.0.0.1** to verify if the webbrowser is working.

* You can change how you access the web app by using  a custom domain name like redcap.local. To achieve this, edit your hosts file and create a mapping for localhost as shown below

```bash
sudo nano /etc/hosts
127.0.0.1   redcap.local
```

You should be able to access redcap via [http://redcap.local](http://redcap.local)

* With the web app running, it's about time you get a copy of the redcap source zip file. You can get it [here](https://redcap.vumc.org/community/custom/download.php) if you're a member of the consortium. and use it to complete the redcap installation.

In the part 2 of the post, we will walk through a typical redcap upgrade using our dockerised environment