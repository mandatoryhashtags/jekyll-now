---
layout: post
title: Magento 2 Cloud Docker
---

## Magento Cloud Docker Development

While at Magento Imagine 2019 this year I was excited to sit in on a lab for the new Cloud docker local development environment. While the lab was a little bit of a disaster due to some technical difficulties the team at Adobe/Magento were able to cover the basics for someone who has a deep knowledge of docker. 

I decided to put together this blog post to cover setting up this environment for people who do not have a deep knowledge of Docker. I will be covering initial setup of a vanilla Magento 2 site using the Magento Template as well as setting up an existing site.

This environment takes full advantage of the robust ECE tools rolled out this year for public access. Enough talking let's get to work!

I will assume you are on a MacBook but this will work on Linux, Mac and Windows. On Linux it will be the fastest, on Windows I would suggest using VirtualBox.

### Install Docker

1. Install Docker
	2. Find it here: https://docs.docker.com/install/

### Configure Docker

1. We want to make some changes to our docker configuration once it is done. We do that with the "Whale" icon at the top of your screen.

2. In Preferences you want to make a few changes
	3. Ensure under "Disk" the disk ends in ".raw"
	4. Under advanced set "CPU" to 4
	5. Under advanced set "Memory" to 8 GB
		6. This can be lower if you have memory issues
	7. Under Advanced set "Swap" to 2 GB
	8. When finished it will restart docker

### Install dependencies

* Install PHP

```
brew install php@7.2
```

* Install Composer
 
```
brew install composer
``` 

* Install Docker-Sync
	* This can be replaced with other options but it was recommended by Magento

```
brew install docker-sync
```

### Start Process

Now we need to go through some prelim. steps to get started.

* Create directory structure
* I do this from home (~)

```
mkdir www
```

* Shut down apache if it is running to free ports

```
sudo apachectl stop
```

* Clone the Magento Cloud Template project


```
git clone https://github.com/magento/magento-cloud magento2.docker
```

* You will need your own Oauth token and credentials for Magento for this step. If you do not have them contact your IT department or Magento for them as that is outside the scope of this Blog post


* Access the project directory

```
cd magento2.docker
```

* Run Composer install to get the Magento file system

```
composer install
```

* Start docker-sync
	* docker-sync.yml is required for this process and composer install came with it
	* This will take a bit

```
docker-sync start
```

* Now we need to build the docker configuration files
	* Magento gave us an easy way to do this based on each Cloud projects .magento.app.yml file

```
# This is for production mode explained later
./vendor/bin/ece-tools docker:build
```

* You might get the following error:

```
There are some service versions which are not supported by current Magento version:
Magento 2.3.1.0 does not support version "3.0" for service "redis".Service version should satisfy "~3.2.0 || ~4.0.0 || ~5.0.0" constraint.
Do you want to continue?[y/N]
```

* If you do just say N and then use the following steps

```
cd .magento
```

* Edit services.yml with you favorite editor (or VI like me) and change the redis version to 3.2 then proceed back up a directory

```
cd ..
./vendor/bin/ece-tools docker:build
```

* Now you have all the docker configuration files needed for a production spin up of the server
	* Lets spin up the server
	* Depending on connection speed if this is your very first time this will take a while
	* It is much faster on subsequent builds


```
# Spin up the container
docker-compose up -d
```

* Now lets run the cloud-build this is precisely the same way it is when code is deployed to a cloud server so it will run through all the same steps

```
# Run the build command
docker-compose run build cloud-build
```

* Now lets run a deployment

```
# Run the deploy command
docker-compose run deploy cloud-deploy
```

* Now we want to set the caching mechanism and lock it

```
docker-compose run deploy magento-command config:set system/full_page_cache/caching_application 2 --lock-env
```

* And last but not least we want to run the familiar cache clear

```
docker-compose run deploy magento-command cache:clea
```

* Now we have a fully deployed site before we can hit it we need to configure our hosts file

```
echo "127.0.0.1 magento2.docker" | sudo tee -a /etc/hosts
```

* Navigate to http://magento2.docker/ you should see the familiar Luma home page

Congrautlations you just setup and created a Docker Cloud Local environment.

When done working on the site run these two commands:

```
docker-compose stop
docker-sync stop
```

#### Cool stuff bro but how do I develop on this?

So some things glossed over above are some specific commands provided to work with the environment. Additionally, how to work with an already existing site. Also what is the difference in Production and Development?

##### Production
This is where all files are static, it is a read-only file system and local changes have no effect on the server. This is very fast, everything is locked and is great for doing a final pass on QA items or when reviewing another developers work. 

* Protip - Have your developers include a dump of their DB if it is not too big and when reviwing code you can import their DB and run this in production mode

##### Development
This setting is where you will do the lions share of your work. It is where you will write code and the docker-sync system will share the files across the docker instance from your local. We will be setting this up next on an already existing site. 


### Setup an already existing site for development
* This assumes you have already completed the previous steps

##### Highlevel outline of steps
1. Clone down your repository
2. Get a DB dump of the cloud environment
	1. I suggest magento-cloud db:dump for this
3. Create config files for composer
	1. Use the build process from above
	2. Create a docker-sync.yml file


* Clone your repository from the cloud

```
git clone [[[url given from cloud dashboard]] [[local path]]
```

* Run composer install

```
composer install
```

* Create docker-sync file

```
vi docker-sync.yml
```

* With these contents

```
version: 2
syncs:
  magento:
    src: './'
    sync_excludes: ['.git', '.idea']
```

* Start docker-sync

```
docker-sync start
```

* Next we want to move the copy of our database where it will be added to the build of the box

```
mv [[dump_file]].sql your_path/docker/mysql/docker-entrypoint-initdb.d
```

* Now we want to build our docker compose files for Developer mode
	* There is a chance that "mode" will be an unknown flag depeneding on the version of ece-tools if so you need to update ECE tools
	* composer require magento/ece-tools ~2002.0.18
	
```
./vendor/bin/ece-tools docker:build --mode developer
```

* Now we need to prepare the environment variables

```
cp docker/config.php.dist docker/config.php
./vendor/bin/ece-tools docker:config:convert

```

* Now lets start the environment

```
# Docker compose in detached mode
docker-compose up -d
# Skip the build step and go straight to deploy
docker-compose run deploy cloud-deploy
# Set magento to developer mode
docker-compose run deploy magento-command deploy:mode:set developer
# Set caching
docker-compose run deploy magento-command config:set system/full_page_cache/caching_application 2 --lock-env
# Clean the cache
docker-compose run deploy magento-command cache:clean
```

* Now hit your environment
* As before when you are done working on this site use the following


```
docker-compose stop
docker-sync stop
```

### Final thoughts
This development approach will allow your team and/or you to work with Magento 2 development the same way it works in the cloud. This will allow a much faster turn around time for catching bugs or issues in development.


### Helpful Links
* https://devdocs.magento.com/guides/v2.3/cloud/docker/docker-quick-reference.html
* https://devdocs.magento.com/guides/v2.3/cloud/docker/docker-database.html
* https://devdocs.magento.com/guides/v2.3/cloud/docker/docker-config.html
* https://devdocs.magento.com/guides/v2.3/cloud/docker/docker-development-debug.html
