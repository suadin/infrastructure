# Table of Contents

1. [Introduction](#introduction)
2. [Domain Setup](#domain-setup) 
3. [Server Setup](#server-setup)
   * [Teamspeak3](#teamspeak3)
   * [Minecraft](#minecraft)
   * [Docker](#docker)
4. [CI/CD Setup](#cicd-setup)
   * [GitHub](#github)
   * [DockerHub](#dockerhub)
   * [Deployment](#deployment)

## Introduction
As full-stack-developer you need as well to know how to setup a server and how to auto-deploy software on it. Aim is to document the setup. The setup itself is the infrastructure solution for [suadin.de](https://suadin.de) which hosts the [suadin/suadin.de](https://github.com/suadin/suadin.de) website with low budget and high functionality.

## Domain Setup

Domain [suadin.de](http://suadin.de) and Server [81.169.247.92]() are currently managed in [strato](https://www.strato.de/). Domain is not by default directly assigned to Server, small Domain Setup was necessary.

First step documented [here](https://www.strato.de/faq/domains/welche-einstellungen-kann-ich-im-konfigurationsdialog-a-record-vornehmen/) (found only german version), add on DNS properties of Domain the Server IP as [A-Record](https://simple.wikipedia.org/wiki/A_record). Second step documented [here](https://www.strato.com/faq/en_us/product/this-is-how-you-can-set-a-custom-dns-reverse-for-your-ip-addresses/), add on Server the Domain as [DNS-Reverse](https://en.wikipedia.org/wiki/Reverse_DNS_lookup). First step is required to do second step successfully.

## Server Setup

Server has as Operating System (OS) **Ubuntu 18.04 LTS 64bit**. Setup manually, used scripts attached.

Pre-Installations:
* sudo apt-get install screen
* sudo apt-get install nano

Usually all setups have same start and end:
1. login as root
2. apt update && apt upgrade
3. adduser $user_name
4. usermod -aG sudo $user_name
5. su $user_name
6. cd ~
7. **(see sub-captures below)**
8. exit
9. sudo deluser $user_name sudo
10. chown -hR $user_name:$user_name /home/$user_name
11. exit

### Teamspeak3

Source documentation [here](https://www.beruni.de/teamspeak-3-server-auf-strato-server-installieren/):
1. wget https://files.teamspeak-services.com/releases/server/3.9.1/teamspeak3-server_linux_amd64-3.13.2.tar.bz2
2. tar xfvj teamspeak3-server_linux_amd64-3.13.2.tar.bz2
3. rm teamspeak3-server_linux_amd64-3.13.2.tar.bz2
4. cd teamspeak3-server_linux_amd64
5. touch .ts3server_license_accepted
6. chmod +x ts3server_startscript.sh
7. ./ts3server_startscript.sh start
8. sudo chown -hR teamspeak3:teamspeak3 /home/teamspeak3
9. nano /etc/crontab
   * add **@reboot teamspeak3 /home/teamspeak3/teamspeak3-server_linux_amd64/ts3server_startscript.sh**
   * Ctrl+O, Enter, Ctrl+X

### Minecraft

Source documentation [here](https://www.vpsserver.com/community/tutorials/4005/minecraft-spigot-bukkit-server-on-ubuntu/):
1. sudo apt-get install openjdk-8-jdk
2. copy/paste via [WinSCP](https://winscp.net/eng/download.php) existing spigot server
3. if you have no existing spigot server, execute following steps to create new spigot server
   * sudo apt install git
   * wget https://hub.spigotmc.org/jenkins/job/BuildTools/lastSuccessfulBuild/artifact/target/BuildTools.jar
   * java -jar BuildTools.jar
3. nano start.sh
   * while true; do echo "Starting server now!";
   * java -Xms4G -Xmx4G -XX:+UseConcMarkSweepGC -jar spigot-1.16.5.jar
   * echo "Server restarting in 5 seconds! Press control+c to stop!"; sleep 5; done;
4. sudo chmod +x start.sh
5. nano eula.txt
   * eula=true
6. screen ./start.sh
   * Ctrl+A+D
7. nano /etc/crontab
   * add **@reboot minecraft /usr/bin/screen -dmS minecraft-screen /home/minecraft/start.sh**
   * Ctrl+O, Enter, Ctrl+X

### Docker

Source documentation [here](https://www.digitalocean.com/community/tutorials/so-installieren-und-verwenden-sie-docker-auf-ubuntu-18-04-de):
1. sudo apt install apt-transport-https ca-certificates curl software-properties-common
2. curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
3. sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
4. apt-cache policy docker-ce
5. sudo apt install docker-ce
6. sudo systemctl status docker
7. sudo groupadd docker
8. sudo gpasswd -a docker docker
9. sudo service docker restart
10. screen -d -m bash -c "docker run -p 8080:80 docker/getting-started"
    * check [suadin.de:8080](http://suadin.de:8080) shows getting started website for docker

### SSL Create
Source documentations: [docker](https://thomasbandt.com/running-aspnetcore-with-https-in-a-docker-container), [snapd/certbot](https://certbot.eff.org/lets-encrypt/ubuntubionic-other), [pfx](https://www.ssl.com/how-to/create-a-pfx-p12-certificate-file-using-openssl/)
1. sudo apt install snapd
1. sudo apt install fuse
1. sudo snap install core; sudo snap refresh core
1. sudo apt-get remove certbot
1. sudo snap install --classic certbot
1. sudo ln -s /snap/bin/certbot /usr/bin/certbot
1. sudo certbot certonly --webroot
   * Email address: ***
   * Read Terms of Service: Yes
   * Share Email address: NO
   * Domain names: suadin.de
   * Choose webroot: doesn't now, cancel throws error, therefore in step 8 & 9 I try alternative to --webroot
1. stop webserver
   * sudo su docker
   * stop suto-deployment: screen -ls, screen -r <screen-id>, Ctrl+A, K, Y
   * stop container: docker container ls, docker container stop <container-id>
   * exit
1. sudo certbot certonly --standalone, enter domain, expect "Congratulations!"
   * /etc/letsencrypt/live/suadin.de/fullchain.pem
   * /etc/letsencrypt/live/suadin.de/privkey.pem
1. sudo openssl pkcs12 -export -out suadin.de.pfx -inkey /etc/letsencrypt/live/suadin.de/privkey.pem -in /etc/letsencrypt/live/suadin.de/fullchain.pem
    * choose export password
1. prepare *.pfx for docker usage
    * sudo mkdir /home/docker/.aspnet
    * sudo mkdir /home/docker/.aspnet/https
    * sudo cp suadin.de.pfx /home/docker/.aspnet/https/
  
### SSL Manual Renew

1. <details><summary>stop website</summary>
   <p>

   ```sh
   su docker  
   pkill screen
   container_id=$(docker ps -aqf "name=$repo" -aqf "status=running")
   if [ ! -z "$container_id" ]; then
      docker container stop "$container_id"
   fi
   ```
     
   </p>
   </details>
1. `sudo certbot renew`
1. execute last two steps from SSL setup
1. <details><summary>start website</summary>
   <p>

   ```sh
   su docker
   cd ~
   screen ./continuous-deployment.sh
   ```
     
   </p>
   </details>
  
### SSL Auto Renew
  
Usually
  1. add start and stop webservice as pre and post scripts into certbot
  1. create private key with using password
  2. add new private key into website
  
> :warning: But never did it, therefore open task to do setup.
  
### PostgreSQL
  
Source documentations: [postgresql installation](https://www.postgresqltutorial.com/install-postgresql-linux/), [create user](https://www.postgresql.org/docs/8.0/sql-createuser.html), [grant user](https://serverfault.com/questions/240887/whats-the-required-to-make-a-normal-user-can-create-schema-on-postgresql)
  1. sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
  1. wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
  1. sudo apt-get update
  1. sudo apt-get install postgresql
  1. sudo apt-get install postgresql-12
  1. sudo -i -u postgres
  1. psql
     1. CREATE DATABASE suadin;
     1. CREATE USER suadin WITH PASSWORD 'jw8s0F4';
     1. GRANT CREATE ON DATABASE suadin TO suadin;
     1. \q
  1. exit
  1. exit

### pgAdmin4
  
Source documentations: [pgAdmin4 installation](https://www.tecmint.com/install-postgresql-and-pgadmin-in-ubuntu/), [change apache port](https://ubiq.co/tech-blog/how-to-change-port-number-in-apache-in-ubuntu/).
  1. use root user!
  1. cd /etc/postgresql
  1. curl https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo apt-key add
  1. sudo sh -c 'echo "deb https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list && apt update'
  1. sudo apt install pgadmin4
  1. change port of apache, because port 80/443 is already used by website
     1. nano /etc/apache2/ports.conf --> replace 80/443 with 8080/8443
     1. nano /etc/apache2/sites-enabled/000-default.conf --> replace 80 with 8080
     1. sudo systemctl restart apache2 #SystemD
     1. sudo service apache2 restart #SysVInit
  1. sudo /usr/pgadmin4/bin/setup-web.sh

Now pgadmin4 is available with http://81.169.247.92:8080/pgadmin4.

Too long, therefore login to strato, create subdomain db.suadin.de and create external detour to http://81.169.247.92:8080/pgadmin4. Now pgadmin4 is available with http://db.suadin.de.


## CI/CD Setup

[Continuous Integration (CI)](https://en.wikipedia.org/wiki/Continuous_integration) and [Continuous Delivery (CD)](https://en.wikipedia.org/wiki/Continuous_delivery) of own software solutions happens through [GitHub](https://github.com/) and [DockerHub](https://hub.docker.com/). Pull of docker image happens with [deployment script](#deployment) on Server.

Following diagram shows final CI/CD setup:

![alternative text](http://www.plantuml.com/plantuml/png/PS_1IWGn30RWUv_YFmaUnWSOHFQme7Vn0JBJw5RRT4dIbNbxBHramDjV-lrjSZ8dzLPoeDMhuimtplNA6luIfYSy9tzfounhiujXhP7X5OMIO56IzHA6wFPSro_Mpl5UzPiqZaOO5__LqbBU3UvWNfKDgT37iV8uuPNrnjg7oDd0ltb3ITASTpt0aIOnfwuxwAzha_wJE2LXtPT-CzP3kHzdi7pMpM2DOfA7o9Yd-t1YYQta7m00)

### GitHub

GitHub account [suadin](https://github.com/suadin) contains the ci/cd relevant [GIT](https://en.wikipedia.org/wiki/Git) repository [suadin/suadin.de](https://github.com/suadin/suadin.de). All changes on main-branch triggers CI process.

### DockerHub

First connect to GitHub repository with following [trivial steps](https://docs.docker.com/docker-hub/builds/link-source/). Next [setup automated builds](https://docs.docker.com/docker-hub/builds/), choose main branch and link to existing Dockerfile. At least push your code into GitHub, expect DockerHub build runs. If the latest build status shows SUCCESS, then we are done with CI and can start with CD in Deployment capture.

> :warning: **If you get error 'COPY failed: stat /var/lib/docker/tmp/docker-builder...'**: I solved it by remove repo from docker-hub and create new with same name.

### Deployment

Deployment based on following idea:
1. check (and get) frequently for new version with `docker pull`
2. if an old version exists, stop and remove old version
3. run new version

<details><summary>Following script implements CD: /home/docker/continuous-deployment.sh</summary>
<p>

```sh
#!/bin/bash
repo="<repo-name>"
feed="<dockerhub-id>/$repo"
docker_params="-p 80:80 -p 443:443 -e ASPNETCORE_URLS=\"https://+443;http://+80\" -e ASPNETCORE_HTTPS_PORT=443 -e ASPNETCORE_Kestrel__Certificates__Default__Password=\"$cert_password\" -e ASPNETCORE_Kestrel__Certificates__Default__Path=/https/suadin.de.pfx -v ~/.aspnet/https:/https/"
while [ true ]
do
  sleep 1
  docker container prune -f
  docker image prune -a -f
  sleep 1
  container_id=$(docker ps -aqf "name=$repo" -aqf "status=running")
  pull=$(docker pull $feed)
  if [[ $pull == *"Status: Image is up to date for $feed:latest"* ]]; then
    if [ ! -z "$container_id" ]; then
      echo "Nothing to do..."
    else
      # duplicated code, see below. For now ok but potential improvement exists.
      echo "start container with image $feed"
      screen -d -m bash -c "docker run $docker_params --name $repo $feed"
    fi
    sleep 60
  elif [[ $pull == *"Downloaded newer image for $feed:latest"* ]]; then
    echo "New version detected!"
    if [ ! -z "$container_id" ]; then
      echo "stop container $container_id"
      docker container stop "$container_id"
      sleep 1
      docker container prune -f
      docker image prune -a -f
      sleep 1
    fi
    echo "start container with image $feed"
    screen -d -m bash -c "docker run $docker_params --name $repo $feed"
    sleep 60
  else
    echo "Script doesn't work like expected, please verify!"
    sleep 5
  fi
done
```

</p>
</details>

Start script manually as docker user with `screen ./continuous-deployment.sh`. Start script after server reboots by adding with root user CRON job:
* nano /etc/crontab
   * add @reboot docker /usr/bin/screen -dmS continuous-deployment-screen /home/docker/continuous-deployment.sh
   * Ctrl+O, Enter, Ctrl+X
