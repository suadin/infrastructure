# Table of Contents

1. [Domain Setup](#domain-setup) 
2. [Server Setup](#server-setup)
   * [Teamspeak3](#teamspeak3)
   * [Minecraft](#minecraft)
   * [Docker](#docker)
3. [CI/CD Setup](#cicd-setup)
   * [GitHub](#github)
   * [DockerHub](#dockerhub)
   * [Deployment](#deployment)

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
   * follow original documentation for new setup
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

### SSL
Source documentations: [docker](https://thomasbandt.com/running-aspnetcore-with-https-in-a-docker-container), [snapd/certbot](https://certbot.eff.org/lets-encrypt/ubuntubionic-other), [pfx](https://www.ssl.com/how-to/create-a-pfx-p12-certificate-file-using-openssl/)
1. sudo apt install snapd
2. sudo apt install fuse
3. sudo snap install core; sudo snap refresh core
4. sudo apt-get remove certbot
5. sudo snap install --classic certbot
6. sudo ln -s /snap/bin/certbot /usr/bin/certbot
7. sudo certbot certonly --webroot
   * Email address: ***
   * Read Terms of Service: Yes
   * Share Email address: NO
   * Domain names: suadin.de, www.suadin.de
   * Choose webroot: doesn't now, cancel throws error, therefore in step 8 & 9 I try alternative to --webroot
8. stop webserver
   * sudo su docker
   * stop suto-deployment: screen -ls, screen -r <screen-id>, Ctrl+A, K, Y
   * stop container: docker container ls, docker container stop <container-id>
   * exit
9. sudo certbot certonly --standalone, enter domain, expect "Congratulations!"
   * /etc/letsencrypt/live/suadin.de/fullchain.pem
   * /etc/letsencrypt/live/suadin.de/privkey.pem
10. sudo openssl pkcs12 -export -out suadin.de.pfx -inkey /etc/letsencrypt/live/suadin.de/privkey.pem -in /etc/letsencrypt/live/suadin.de/fullchain.pem
    * choose export password
11. prepare *.pfx for docker usage
    * sudo mkdir /home/docker/.aspnet
    * sudo mkdir /home/docker/.aspnet/https
    * sudo cp suadin.de.pfx /home/docker/.aspnet/https/

## CI/CD Setup

[Continuous Integration (CI)](https://en.wikipedia.org/wiki/Continuous_integration) and [Continuous Delivery (CD)](https://en.wikipedia.org/wiki/Continuous_delivery) of own websites and webapis happens through [GitHub](https://github.com/), [DockerHub](https://hub.docker.com/). Pull of docker image happens with [deployment script](#deployment) on Server.

Following diagram shows final CI/CD setup:

![alternative text](http://www.plantuml.com/plantuml/png/PS_1IWGn30RWUv_YFmaUnWSOHFQme7Vn0JBJw5RRT4dIbNbxBHramDjV-lrjSZ8dzLPoeDMhuimtplNA6luIfYSy9tzfounhiujXhP7X5OMIO56IzHA6wFPSro_Mpl5UzPiqZaOO5__LqbBU3UvWNfKDgT37iV8uuPNrnjg7oDd0ltb3ITASTpt0aIOnfwuxwAzha_wJE2LXtPT-CzP3kHzdi7pMpM2DOfA7o9Yd-t1YYQta7m00)

### GitHub

GitHub account [suadin](https://github.com/suadin) contains following relevant [GIT](https://en.wikipedia.org/wiki/Git) repositories:
* [Infrastructure](https://github.com/suadin/Infrastructure): contains documentation of infrastructure with relevant scripts and configs
* [Example Website](https://github.com/suadin/Example): very simple website, currently realized for CI/CD
* [SukiG Website](https://github.com/suadin/SukiG): main website written with [Blazor](https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor)

Currently we have no branching model, all code changes happens on main branch.

### DockerHub

First connect to GitHub repository with following [trivial steps](https://docs.docker.com/docker-hub/builds/link-source/). Next [setup automated builds](https://docs.docker.com/docker-hub/builds/), choose main branch and link to existing Dockerfile. At least push your code into GitHub, expect DockerHub build runs. If the latest build status shows SUCCESS, then we are done with CI and can start with CD in Deployment capture.

### Deployment

Deployment based on following idea:
1. check (and get) frequently for new version with `docker pull`
2. if an old version exists, stop and remove old version
3. run new version

<details><summary>Following script implements steps above: /home/docker/continuous-deployment.sh</summary>
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

Start script manually as docker user with `screen ./continuous-deployment.sh`. Ensure running script after server reboots by adding with root user CRON job:
* nano /etc/crontab
   * add @reboot docker /usr/bin/screen -dmS continuous-deployment-screen /home/docker/continuous-deployment.sh
   * Ctrl+O, Enter, Ctrl+X
