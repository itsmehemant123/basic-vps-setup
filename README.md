# VPS Setup

_This is a barebones process for getting started with your fresh VPS. There is a lot of scope for improvements, so Pull Requests are welcome. I did this while setting up my Digital Ocean droplet, hence the abundance of digital ocean docs. Given how good they are, I'm not complaining._

## Basics - Access and Users

References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-an-ubuntu-14-04-vps
- https://www.digitalocean.com/community/questions/how-do-i-add-an-ssh-key-to-an-existing-droplet?answer=18560

Here, we're going to create a new sudo user, and setup ssh-keys.

- While creation of box, add the ssh-key for access via root account.
- Once logged in via root, update & upgrade with:
```bash
apt-get update
apt-get upgrade
```

### Add new sudo user

- Create the [new user](https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-an-ubuntu-14-04-vps) via:

```
adduser hemanth
```

_Set appropriate password for this user. We'll be using this for all subsequent logins, and give sudo._

- Grant `sudo` privileges to this user by editing the suders with:

```
visudo
```

- Search for the line that looks like this:

```
root    ALL=(ALL:ALL) ALL
```

- Below this line, copy the format you see here, changing only the word "root" to reference the new user that you would like to give sudo privileges to:

```
root    ALL=(ALL:ALL) ALL
hemanth ALL=(ALL:ALL) ALL
```

- Or, add the user to `sudo` group by:

```
usermod -aG sudo hemanth
```

- Logout, and add existing key to this user from local [via](https://www.digitalocean.com/community/questions/how-do-i-add-an-ssh-key-to-an-existing-droplet?answer=18560):

_for linux_
```
ssh-copy-id user@123.45.56.78
```
_for macOS_
```
cat ~/.ssh/id_rsa.pub | ssh hemanth@123.45.56.78 "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"
```

_This means you can start `ssh`ing into the VPS by using the key instead of the password._

- Optionally, disable password login to root via editing the `sshd_config`:

```
nano /etc/ssh/sshd_config
```

- Set the `PasswordAuthentication` in the file to `no` to disable root login via password.

- Restart `ssh` with:

```
sudo systemctl restart ssh
```

### Delete User

- Delete user via:

```
deluser newuser
```

## Setting ssh-conf on local for this VPS

- Edit your local ssh config by:

```
nano ~/.ssh/config
```

- Add the following block at the end:

```
Host <short_name>
    HostName <ip_add_of_vps_or_domain>
    User hemanth
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 120
```

_Where, `id_rsa` is the key which was copied over in the previous step._

- Now, going foward, to ssh into this box, type:

```
ssh <short_name>
```

## Installing zsh, Oh my ZSH, and powerlevel9k for user

References for this section:
- https://github.com/robbyrussell/oh-my-zsh/wiki/Installing-ZSH
- https://github.com/bhilburn/powerlevel9k/wiki/Install-Instructions

_We'll be installing [another shell](https://github.com/robbyrussell/oh-my-zsh/wiki/Installing-ZSH)_ on the VPS along with a theme.

- First, install ZSH by logging into the VPS with `hemanth`, and then:

```
[sudo] apt install zsh
```

- Test if it works by running:

```
zsh --version
```

- Set it to default shell for your user by:

```
chsh -s $(which zsh)
```

- Logout and login to configure zsh.

- Now, install Oh my ZSH via:

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

- Logout and log back in to your new shell!

- Finally, install powerlevel9k theme [via](https://github.com/bhilburn/powerlevel9k/wiki/Install-Instructions):

```
git clone https://github.com/bhilburn/powerlevel9k.git ~/.oh-my-zsh/custom/themes/powerlevel9k
```

- Now, enable this theme, and configure your theme by editing the `.zshrc` via:

```
nano ~/.zshrc
```

- Once inside, change `ZSH_THEME="robbyrussell` to `ZSH_THEME="powerlevel9k/powerlevel9k"`.

- Add the following to the bottom of the file:

```
POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(context anaconda dir rbenv docker_machine vcs)
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(status root_indicator background_jobs)
POWERLEVEL9K_ANACONDA_LEFT_DELIMITER="("
POWERLEVEL9K_ANACONDA_RIGHT_DELIMITER=")"
POWERLEVEL9K_ANACONDA_BACKGROUND="red"
POWERLEVEL9K_ANACONDA_FOREGROUND="black"
```

- Add more plugins if you want (_they will slow down the shell start, be warned_):

```
plugins=(
  git
  docker
  go
  node
  python
  pip
  sudo
)
```

- Here are a few common `env vars` for programming languages:

```
PATH="$HOME/bin:$HOME/.local/bin:$PATH"
export GOPATH=$HOME/work
export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin

export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"

# added by Anaconda3 installer
export PATH="/home/hemanth/anaconda3/bin:$PATH"
```

_Don't set all of them now. Add them as you install the respective environment/runtime._

- Logout and log back in for the new shell experience!

## Installing netdata for perf monitoring
References for this section:
- https://github.com/firehol/netdata/wiki/Installation

We're going to install netdata, a nifty server dashboard.

- Install [netdata](https://github.com/firehol/netdata/wiki/Installation) by running:

```
bash <(curl -Ss https://my-netdata.io/kickstart.sh) 
```
_Do not run with sudo!_

- Once finished, it installs as a service, which starts up at boot automatically. It runs on port `19999`.

- We'll later set it behind nginx along with a subdomain and a certificate. For now, its accessible via `<ip_address_of_vps_or_domain>:19999`.


_You may notice that the entropy dips pretty low. We're going to fix that next!_

## Installing haveged for entropy
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-setup-additional-entropy-for-cloud-servers-using-haveged

- Install [haveged](https://www.digitalocean.com/community/tutorials/how-to-setup-additional-entropy-for-cloud-servers-using-haveged) via:

```
[sudo] apt-get install haveged
```

_If you have the netdata dashboard open, you will be able to see the entropy shoot up!_

### From this point onwards, better start taking snapshots of your VPS so that you can revert if something goes wrong.

## Installing Docker and Docker Compose
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04
- https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-16-04

Docker is super useful for creating and deploying containers. We'll be installing both [Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-16-04) and [Docker Compose](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-16-04) on the VPS.

- First, add the GPG key for the official Docker repository to the system by:

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

- Add the docker repository to APT by:

```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

- Update the package database by running:

```
sudo apt-get update
```

- Ensure you're able to install docker from the docker repo by running:

```
apt-cache policy docker-ce
```
_See docker link for more details about this step_


- Install docker by running:

```
sudo apt-get install -y docker-ce
```

- Docker should be enabled, and set to run on boot. Check status by running:

```
sudo systemctl status docker
```

### Adding user to docker group

_Currently, to run docker, sudo is required. Instead of using sudo, we'll add the user to the `docker` group._

- Add logged in user to `docker` by:

```
sudo usermod -aG docker ${USER}
```

- Now, either logout and log back in, or switch user again with:

```
su - ${USER}
```

- Check to see if user has the `docker` group by running:

```
id -nG
```
_You should see the `docker` group_

- You can add other users to the `docker` group by running:

```
sudo usermod -aG docker username
```

### Installing Docker Compose

- Download and run the current docker compose release by:

```
sudo curl -L https://github.com/docker/compose/releases/download/1.19.0-rc2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```
_As of writing this document, the latest release of docker is `1.19.0-rc2`._

- Set the permissions by:

```
sudo chmod +x /usr/local/bin/docker-compose
```

- Verify the installation by running:

```
docker-compose --version
```

_You should be seeing the installed version of docker compose._

## Installing Nginx
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04
- https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-with-ssl-as-a-reverse-proxy-for-jenkins
- https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-as-a-web-server-and-reverse-proxy-for-apache-on-one-ubuntu-16-04-server

Nginx functions pretty well as a web server, reverse proxy, and a load balancer.

- Install nginx with:

```
sudo apt-get install nginx
```

- Check if nginx is installed with:

```
nginx -v
```

_It should show the installed version of nginx._

## Setup a firewall

References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-16-04
- https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04

We'll use `ufw`, a simple firewall which serves our requirements. It is bundled with ubuntu.

- If not installed, install it with:

```
[sudo] apt install ufw
```

- Set the rules to default so we're all on the same page:

```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

- Allow ssh access by:

```
sudo ufw allow ssh
```

- Allow Nginx HTTP traffic with:

```
sudo ufw allow 'Nginx Full'
sudo ufw delete allow 'Nginx HTTP'
```

- Enable it by:

```
sudo ufw enable
```

_This activates ufw, and makes sure it start on boot._

- Check current rules with:

```
sudo ufw status verbose
```

_At this point, you should no longer be able to access netdata from <ip_address_of_vps_or_domain>:19999. This is expected, as ufw is blocking connections._

- Add incoming connections on port `3306` and `27017` for mysql and mongodb by:

```
sudo ufw allow 3306
sudo ufw allow 27017
```

_Instead of allowing connections from any IP Address, ufw allows rules which specify incoming IP Address. That is a much safer alternative._

- Reload ufw by:

```
sudo ufw reload
```

## Installing Mysql
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-set-up-a-remote-database-to-optimize-site-performance-with-mysql

We'll install [Mysql server](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-remote-database-to-optimize-site-performance-with-mysql) and secure it.

- Start by updating the package cache with:

```
sudo apt-get update
```

- Install mysql with:

```
sudo apt-get install mysql-server
```

- Run the install script with:

```
mysqld --initialize
```

_You can specify the data directory here, or use the default._

- Secure the database by running the script:

```
sudo mysql_secure_installation
```

### Creating a remote user

- Connect to the instance with:

```
mysql -u root -p
```

- Once logged in, create a user who can remote it by:

```
create user 'user'@'%' identified by 'password';
```

_Change the `%` with `localhost` if you do not want remote login for the created user._

- Grant privileges on tables by:

```
grant all privileges on *.* to 'user'@'%';
```

_This is optional._

- This user doesn't have `grant` privileges. Instead of the above command, run this to give that too:

```
grant all privileges on *.* to 'user'@'%' with grant option;
```

- Finally, flush privileges with:

```
flush privileges;
```

### Configuring mysql for remote access

- Enable remote login by editing the configuration file:

```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

- Within the `[mysqld]` section, look for `bind-address`, and replace `127.0.0.1` with either your vps ip or `0.0.0.0` to enable access from anywhere. It should look like this:

```
bind-address		= 0.0.0.0
```

_This lets users remote from anywhere. You might not want to set it to `0.0.0.0`. Rather, set it to the IP address of the VPS itself._

- [OPTIONAL] Add the following within `[mysqld]` to disable dns resolution:

```
skip-host-cache
skip-name-resolve
```

- Restart the mysql service with:

```
sudo service mysql restart
```

- Check status with:

```
sudo service mysql status
```

_By default, mysql runs on port 3306_

## Installing Mongodb
References for this section:
- https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/
- https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-mongodb-on-ubuntu-16-04

We'll start installing mongodb, a popular nosql database. 

- Import the public key for the package repository by:

```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2930ADAE8CAF5059EE73BB4B58712A2291FA4AD5
```

- Create a list file for mongo (ubuntu 16.04) by:

```
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.6.list
```

_Check the appropriate version [here](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/#create-a-list-file-for-mongodb) for your os version._

- Update the packages by running:

```
sudo apt-get update
```

- Install mongodb with:

```
sudo apt-get install -y mongodb-org
```

- Enable mongo startup on boot with:

```
sudo systemctl enable mongod
```

### Setting up an admin user

- Start the mongo with:

```
sudo systemctl start mongod
```

- Start the mongo cli with:

```
mongo
```

- Add an admin user by running the following:

```
use admin

db.createUser(
  {
    user: "admin_user",
    pwd: "admin_password",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
```

- Exit out of the shell with:

```
exit
```

### Enable authentication

- Edit the mongo config file with:

```
sudo nano /etc/mongod.conf
```

- In the #security section, we'll remove the hash in front of security to enable the stanza. Then we'll add the authorization setting. When we're done, the lines should look like the excerpt below:

```
security:
  authorization: "enabled"
```

_Note that the “security” line has no spaces at the beginning, and the “authorization” line must be indented with two spaces._

- Restart mongo with:

```
sudo systemctl restart mongod
```

- Check status with:

```
sudo systemctl status mongod
```

- Now, you can only connect to `mongo` with username and password. Connect to it with:

```
mongo -u admin_user -p --authenticationDatabase admin
```

_By default, mongo runs on port 27017._

## Installing Redis cache
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-redis-on-ubuntu-16-04
- https://www.rosehosting.com/blog/how-to-install-configure-and-use-redis-on-ubuntu-16-04/

We'll install a redis cache, and protect it with a password.

- Update the package cache with:

```
sudo apt-get update
```

- Install redis with:

```
sudo apt-get install redis-server
```

### Configuring Redis 

- Edit the configuration file with:

```
sudo nano /etc/redis/redis.conf
```

- Uncomment `requirepass`, and add a password. It should look like so:

```
requirepass redis_password
```

- Save, and exit.

- Restart and enable start on boot with:

```
sudo systemctl restart redis-server.service
sudo systemctl enable redis-server.service
```

## Installing Anaconda
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-install-the-anaconda-python-distribution-on-ubuntu-16-04 

We'll start installing programming environments starting with [anaconda](https://www.digitalocean.com/community/tutorials/how-to-install-the-anaconda-python-distribution-on-ubuntu-16-04).

- Switch to `tmp`:

```
cd /tmp
```

- Download anaconda installation script via:

```
curl -O https://repo.continuum.io/archive/Anaconda3-5.0.1-Linux-x86_64.sh
```

_As of writing this document, the latest version is `5.0.1`._

- Run the downloaded script with:

```
bash Anaconda3-5.0.1-Linux-x86_64.sh
```

_This should start installing anaconda at your desired location._

- The installation may ask to update the PATH variable with the anaconda location. Say yes. However, it updates `.bashrc`, where as we're using `.zshrc` thanks to zsh. So, copy the following into `~/.zshrc`.


```
# added by Anaconda3 installer
export PATH="/home/user/anaconda3/bin:$PATH"
```

_This may change according to the your configuration. Copy the corresponding lines from `~/.bashrc` to `~/.zshrc`._

- Now, you can either logout and log back in to see the changes, or source the updated `~/.zshrc` by:

```
source ~/.bashrc
```

### Updating anaconda

- Update existing anaconda installation by:

```
conda update conda
```
_Which updates the package listings._

```
conda update anaconda
```
_Which updates anaconda._

### Uninstalling anaconda

- To uninstall anaconda, get the uninstall package by:

```
conda install anaconda-clean
```

- Now, uninstall anaconda by running:

```
anaconda-clean
```

- After it is finished, you can delete the directory created by:

```
rm -rf ~/anaconda3
```
_This may change according to your configuration._

- Finally, update your `~/.bashrc` and `~/.zshrc` by removing the following line:

```
# added by Anaconda3 4.2.0 installer
export PATH="/home/user/anaconda3/bin:$PATH"
```

## Installing NodeJS
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-16-04
- https://github.com/tj/n
- https://github.com/sindresorhus/guides/blob/master/npm-global-without-sudo.md

We'll install nodejs, along with `n` for managing multiple versions of nodejs.

- Update the package cache by:

```
sudo apt-get update
```

- Install nodejs with:

```
sudo apt-get install nodejs
```

- Install nodejs's package manager npm with:

```
sudo apt-get install npm
```

- Make sure this entry is in the `~/.zshrc`:

```
PATH="$HOME/bin:$HOME/.local/bin:$PATH"
```

- Now, install `n` with:

```
npm install -g n
```

- Finally, you can update to latest version of node with `n` by:

```
sudo n latest
```


## Install certbot for issuing letsencrypt certs
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04
- https://github.com/certbot/certbot/issues/5405
- https://certbot.eff.org/docs/intro.html#how-to-run-the-client
- https://certbot.eff.org/
- https://certbot.eff.org/docs/install.html

Certbot is a tool by the people at letsencrypt which makes it super easy to create certificates. 

- Add the repository:

```
sudo add-apt-repository ppa:certbot/certbot
```

- Update the package cache:

```
sudo apt-get update
```

- Install the certbot with nginx plugin with:

```
sudo apt-get install python-certbot-nginx
```

_As of writing this document, there are issues with tls-sni-01 authentication mechanism with nginx, so the following is optional. This issue was fixed with 0.21.x version of certbot, but it is not yet available via the PPA. By the time you're reading this, it should be available via PPA, so no need to continue with the next few steps._

- Download certbot-auto with:

```
wget https://dl.eff.org/certbot-auto
```

- Set permissions with:

```
chmod a+x ./certbot-auto
```

- Test the bot with:

```
./certbot-auto --help
```

_This downloads and sets up all the dependencies, and then the bot._

- Issue certificate for host (_example.com_) with:

_If you're using `certbot`_
```
sudo certbot --nginx -d example.com
```

_If you're using `certbot-auto`_
```
sudo ./certbot-auto --nginx -d example.com
```

## Adding Nginx rule for web applications
References for this section:
- https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04
- https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-as-a-web-server-and-reverse-proxy-for-apache-on-one-ubuntu-16-04-server
- https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-with-ssl-as-a-reverse-proxy-for-jenkins

We'll finally start adding entries in nginx for each web application we're hosting. 

### Adding rules for netdata

_We'll keep adding entries in `/etc/nginx/sites-enabled/default` for each web application. However, you can remove `default` entirely, and instead add separate file for each web application._

- Edit the virtual hosts by editing the config file by:

```
sudo nano /etc/nginx/sites-enabled/default
```

- Make sure you have the `dhparams.pem` in `/etc/ssl` folder.

- Create folders for logging, and give `www-data` owner perms.

```
sudo mkdir /var/log/nginx/netdata                           
sudo touch /var/log/nginx/netdata/access.log                
sudo touch /var/log/nginx/netdata/error.log                 
sudo chown -R www-data:www-data /var/log/nginx/netdata
```

- Add the following entry at the end of this file for netdata:

```
# netdata configuration
server {
	listen 80;
	server_name netdata.example.com;
	
	return 301 https://$host$request_uri;
}

server {
	listen 443 ssl;
	server_name netdata.example.com;

	location / {
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;

		proxy_pass http://localhost:19999;
		proxy_read_timeout 90;

		proxy_redirect http://localhost:19999 https://netdata.example.com;
	}

	access_log /var/log/nginx/netdata/access.log;
	error_log /var/log/nginx/netdata/error.log;

	ssl_dhparam /etc/ssl/dhparams.pem;
	ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';

	ssl_prefer_server_ciphers on;

}
```

_Here, replace all instances of `netdata.example.com` with your own `subdomain.domain`._

- Issue certificate for this host with:

_If you're using `certbot`_
```
sudo certbot --nginx -d netdata.example.com
```

_If you're using `certbot-auto`_
```
sudo ./certbot-auto --nginx -d netdata.example.com
```

- Restart nginx with:

```
sudo service nginx restart
```

_Here, we're issuing a separate certificate for this subdomain. However, its much better to issue a single certificate for multiple subdomains by adding more `-d subdomain.domain`. This helps with the letsencrypt rate limits if you have many subdomains._

_Another thing to remember, letsencrypt still doesn't support wildcard certificates. So, be careful while issuing certificates. Try to bundle them._

- Now your `netdata.example.com` is ready. Navigate to that address in a browser, and see it in action. You can test the configuration by going to https://www.ssllabs.com/ssltest/analyze.html?d=netdata.example.com as suggested by certbot.

