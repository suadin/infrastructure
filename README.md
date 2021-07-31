# Table of Contents

1. [**Introduction**](#introduction)
1. [**Domain**](#domain) -> [Server](#server) | [Google](#google)
1. [**Hosting**](#hosting) -> [Docker](#docker) | [Teamspeak](#teamspeak) | [Minecraft](#minecraft) | [PostgreSQL](#postgresql) | [pgAdmin4](#pgadmin4)
1. [**Security**](#security) -> [SSL-Certificate](#ssl-certificate)
1. [**Website**](#website-deployment) -> [GitHub-Repository](#github-repository) | [DockerHub-Repository](#dockerhub-repository) | [GitHub-Actions](#github-actions) | [Deployment-Script](#deployment-script)

# Introduction
As full-stack-developer you need as well to know how to host software. For example infrastructure of https://suadin.de [details](https://github.com/suadin/website) but as well other software like [Teamspeak3](#teamspeak3) or [Minecraft](#minecraft). Aim of documentation is to host these software with low budget and high functionality.

# Domain

## Server

Domain [suadin.de](http://suadin.de) and Server [81.169.247.92](http://suadin.de) are currently managed in [strato](https://www.strato.de/). Need setup to connect both:
1. add on Domain for DNS properties the Server IP as [A-Record](https://simple.wikipedia.org/wiki/A_record) [[source](https://www.strato.de/faq/domains/welche-einstellungen-kann-ich-im-konfigurationsdialog-a-record-vornehmen/)]
1. add on Server the Domain as [DNS-Reverse](https://en.wikipedia.org/wiki/Reverse_DNS_lookup) [[source](https://www.strato.com/faq/en_us/product/this-is-how-you-can-set-a-custom-dns-reverse-for-your-ip-addresses/)]

## Google

1. Login to your google account, goto [domain verification](https://console.cloud.google.com/apis/credentials/domainverification), add domain [suadin.de](http://suadin.de)
1. proof domain ownership: select strato, create subdomain `CNAME-Target`, configure [CNAME-Record](https://www.strato.de/faq/domains/wie-kann-ich-bei-strato-meine-dns-eintraege-verwalten/) with `CNAME-Target`

# Hosting

Server has Operating System (OS) **Ubuntu 18.04 LTS 64bit**. Pre-Installs: `sudo apt-get install screen`, `sudo apt-get install nano`

## Template

1. login as root, `apt update`, `apt upgrade`, `adduser $user_name`, `usermod -aG sudo $user_name`, `su $user_name`, `cd ~`
1. main steps on next capters
1. `exit`, `sudo deluser $user_name sudo`, `chown -hR $user_name:$user_name /home/$user_name`, `exit`

## Docker

1. follow step 1 of guide [[source](https://www.digitalocean.com/community/tutorials/so-installieren-und-verwenden-sie-docker-auf-ubuntu-18-04-de)]
1. follow step 2 but with slidely different commands: `sudo gpasswd -a docker docker`, `sudo service docker restart`
1. start docker in screen: `screen -d -m bash -c "docker run -p 8080:80 docker/getting-started"`, check: http://suadin.de:8080

## Teamspeak

1. download/unpack/accept-license with guide [[source](https://www.arubacloud.com/tutorial/how-to-install-and-configure-a-teamspeak-server-on-ubuntu-20-04.aspx)]
1. add into autostart: `nano /etc/crontab` -> add `@reboot teamspeak3 /home/teamspeak3/teamspeak3-server_linux_amd64/ts3server_startscript.sh`
3. grant/start script: `chmod +x ts3server_startscript.sh`, `sudo chown -hR teamspeak3:teamspeak3 /home/teamspeak3`, `./ts3server_startscript.sh start`

## Minecraft

1. follow guide to setup new world [[source](https://www.vpsserver.com/community/tutorials/4005/minecraft-spigot-bukkit-server-on-ubuntu/)]
1. overwrite files with existing world via [WinSCP](https://winscp.net/eng/download.php)
1. add into autostart: `nano /etc/crontab` -> add `@reboot minecraft /usr/bin/screen -dmS minecraft-screen /home/minecraft/start.sh`

## PostgreSQL
  
1. folow install guide [[source](https://www.postgresqltutorial.com/install-postgresql-linux/)]
1. [create](https://www.postgresql.org/docs/8.0/sql-createuser.html)/[grant](https://serverfault.com/questions/240887/whats-the-required-to-make-a-normal-user-can-create-schema-on-postgresql) database/user: `sudo -i -u postgres`, `psql`, `CREATE DATABASE suadin;`, `CREATE USER suadin WITH PASSWORD 'jw8s0F4';`, `GRANT CREATE ON DATABASE suadin TO suadin;`, `\q`, `exit`, `exit` 

## pgAdmin4

1. use root user and follow guide for install pgAdmin4, scroll a bit down [[source](https://www.tecmint.com/install-postgresql-and-pgadmin-in-ubuntu/)]
1. change apache port to have no conflict with suadin.de [[source](https://ubiq.co/tech-blog/how-to-change-port-number-in-apache-in-ubuntu/)]
1. start pgAdmin4: `sudo /usr/pgadmin4/bin/setup-web.sh`
1. create subdomain https://db.suadin.de with external detour http://81.169.247.92:8080/pgadmin4 

# Security

## SSL-Certificate

### Create-Certificate

1. follow certbot guide until step 6, use on step 7 webroot with domain [suadin.de](https://suadin.de) [[source](https://certbot.eff.org/lets-encrypt/ubuntubionic-other)]
1. stop [deployment script](#deployment-script), `sudo certbot certonly --standalone` with domain [suadin.de](https://suadin.de)
   * result: `/etc/letsencrypt/live/suadin.de/fullchain.pem` & `/etc/letsencrypt/live/suadin.de/privkey.pem`
1. convert `privkey.pem` with `fullchain.pem` to `suadin.de.pfx`, choose password [[source](https://www.ssl.com/how-to/create-a-pfx-p12-certificate-file-using-openssl/)]
1. prepare *.pfx for docker: `sudo cp suadin.de.pfx /home/docker/.aspnet/https/` (create missing folder) [[source](https://thomasbandt.com/running-aspnetcore-with-https-in-a-docker-container)]
  
### Renew-Manual-Certificate

1. stop website with deployment script
   ```sh
   su docker  
   pkill screen
   container_id=$(docker ps -aqf "name=$repo" -aqf "status=running")
   if [ ! -z "$container_id" ]; then
      docker container stop "$container_id"
   fi
   ```
1. `sudo certbot renew`
1. execute last two steps from [Create-Certificate](#create-certificate)
1. start deployment scrip
   ```sh
   su docker
   cd ~
   screen ./continuous-deployment.sh
   ```
  
### Renew-Auto-Certificate

1. follow step 9 of certobot guide [[source](https://certbot.eff.org/lets-encrypt/ubuntubionic-other)]
1. `pre/haproxy.sh` contains first scriptblock from [Renew-Manual-Certificate](#renew-manual-certificate)
1. `sudo certbot renew` is executed automatically between pre and post
3. `post/haproxy.sh` contains
   1. execute last two steps from [Create-Certificate](#create-certificate)
   1. last scriptblock from [Renew-Manual-Certificate](#renew-manual-certificate)
  
> :warning: But never did it, therefore open task to do setup.

# Website-Deployment

[Continuous Integration (CI)](https://en.wikipedia.org/wiki/Continuous_integration) and [Continuous Delivery (CD)](https://en.wikipedia.org/wiki/Continuous_delivery) of website happens through [GitHub](https://github.com/) and [DockerHub](https://hub.docker.com/). Pull of docker image happens with [deployment script](#deployment) on Server.

![alternative text](http://www.plantuml.com/plantuml/png/PS_1IWGn30RWUv_YFmaUnWSOHFQme7Vn0JBJw5RRT4dIbNbxBHramDjV-lrjSZ8dzLPoeDMhuimtplNA6luIfYSy9tzfounhiujXhP7X5OMIO56IzHA6wFPSro_Mpl5UzPiqZaOO5__LqbBU3UvWNfKDgT37iV8uuPNrnjg7oDd0ltb3ITASTpt0aIOnfwuxwAzha_wJE2LXtPT-CzP3kHzdi7pMpM2DOfA7o9Yd-t1YYQta7m00)

## GitHub-Repository

GitHub account [suadin](https://github.com/suadin) contains [GIT](https://en.wikipedia.org/wiki/Git) repository of [website](https://github.com/suadin/website).

## DockerHub-Repository

1. connect to GitHub repository [source](https://docs.docker.com/docker-hub/builds/link-source/), choose main branch and link to existing Dockerfile
1. expect push into GitHub triggers DockerHub build run

> :warning: **If you get error 'COPY failed: stat /var/lib/docker/tmp/docker-builder...'**: I solved it by remove repo from docker-hub and create new with same name.

> :warning: **DockerHub force you to Upgrade your account if you need a connection to github**: Solved that by using GitHub Actions to push images to DockerHub.

## GitHub-Actions

1. put DockerHub secrets into GitHub [[source](https://docs.github.com/en/actions/reference/encrypted-secrets)]
1. follow guide to push docker image into DockerHub & GitHub Packages [[source](https://docs.github.com/en/actions/guides/publishing-docker-images)]
1. do slidely changes on last step [[source](https://www.docker.com/blog/docker-v2-github-action-is-now-ga/)]
   1. add on both `build & publish` steps below context `file: src/Server/Dockerfile`
   1. replace hashed versions `docker/build-push-action@ad44023a93711e3deb337508980b4b5e9bcdc5dc` with major versions `docker/build-push-action@v2`

> :information_source: Change to GitHub Packages to remove DockerHub dependency
  
## Deployment-Script

```sh
#!/bin/bash
repo="<repo-name>"
feed="<dockerhub-id>/$repo:main"
docker_params="-p 80:80 -p 443:443 -e ASPNETCORE_URLS=\"https://+443;http://+80\" -e ASPNETCORE_HTTPS_PORT=443 -e ASPNETCORE_Kestrel__Certificates__Default__Password=\"$cert_password\" -e ASPNETCORE_Kestrel__Certificates__Default__Path=/https/suadin.de.pfx -v ~/.aspnet/https:/https/"
while [ true ]
do
  sleep 1
  docker container prune -f
  docker image prune -a -f
  sleep 1
  container_id=$(docker ps -aqf "name=$repo" -aqf "status=running")
  pull=$(docker pull $feed)
  if [[ $pull == *"Status: Image is up to date for $feed"* ]]; then
    if [ ! -z "$container_id" ]; then
      echo "Nothing to do..."
    else
      # duplicated code, see below. For now ok but potential improvement exists.
      echo "start container with image $feed"
      screen -d -m bash -c "docker run $docker_params --name $repo $feed"
    fi
    sleep 60
  elif [[ $pull == *"Downloaded newer image for $feed"* ]]; then
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
    secrets="--env-file secrets.txt"
    screen -d -m bash -c "docker run $docker_params $secrets --name $repo $feed"
    sleep 60
  else
    echo "Script doesn't work like expected, please verify!"
    sleep 5
  fi
done
```
Comments:
* `while [ true ]` & `sleep 60`: minutely endless loop
* `sleep 1` before/after prune necessary to avoid error messages
* `container_id=$(docker ps -aqf "name=$repo" -aqf "status=running")`: take container id for potential stop and prune
* `pull=$(docker pull $feed)`: pull spam to detect potential changes
* `docker_params="-p 80:80 -p 443:443 ...`: configure ports, specially https with certificate binding [[details](#ssl-certificate)]
* `secrets="--env-file secrets.txt"`: for pass secrets into website [[source](https://www.baeldung.com/ops/docker-container-environment-variables)]

Run:
* Start script manually: `screen ./continuous-deployment.sh`
* Start after server reboot with CRON job: `nano /etc/crontab` -> add `@reboot docker /usr/bin/screen -dmS continuous-deployment-screen /home/docker/continuous-deployment.sh`
