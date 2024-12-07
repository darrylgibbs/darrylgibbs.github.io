---
title: How to Install Invoice Ninja V5 in Docker Compose
author: darryl gibbs
date: 2021-04-07 -03:00
tags: [invoice ninja, selfhosted, docker]
categories: [homelab, tutorial]
---
Maybe you are like me and have a small business that requires an invoicing solution. You could pay the $10-15 to have a provider host the service for you (such as [FreshBooks](https://www.freshbooks.com/)) or you could do it yourself with great Open Source software! Thankfully [Invoice Ninja](https://www.invoiceninja.com/) has a great app that they can host for you (for a fee) or, if you’re adventurous (or cheap… depending on your perspective) you could host it for yourself!

**Cavaet:** Self-hosting Open Source software generally gives you access to ‘Premium’ tier features. Invoice Ninja is no different. However, if you should want their watermark removed from your invoices, you will need to pay them [$30 a year for that priviledge](https://www.invoiceninja.com/faqs/does-the-self-host-version-include-all-proenterprise-advanced-features/). But honestly, that’s not unfair.

**Dev’s need to eat too!**

I have adapted this tutorial from the official tutorial on the [Invoice Ninja Dockerfile Github page](https://github.com/invoiceninja/dockerfiles). Personally, I get confused by these tutorials that just assume you’re a rocket scientist, and so I failed at this about 25 times before I got it right. Although that was partially because I wasn’t reading this correctly.

Finally, Invoice Ninja can be hosted on any x86 (Desktop PC or Server) or ARM (SBC such as a Raspberry Pi) device.

This tutorial will focus on the x86 platform, and using Ubuntu Server. Obviously, you could use other distros, but you will need to adjust package managers etc as necessary to work with your chosen distro.

If you are interested in installing this software in the traditional way, please check out my post on this [here](https://dgibbs.me/posts/2021/03/how-to-install-invoiceninja-on-ubuntu-server/).


## Prerequisites

I’m going to assume that you already have the following:

  * A Linux based server up and running and fully updated.

  * Both [Docker](https://docs.docker.com/engine/install/ubuntu/) and [Docker Compose](https://docs.docker.com/compose/install/) installed, and the current user has been added to the `docker` group (meaning: we don’t need to use `sudo` before issuing docker commands).

  * Optional: A domain pointed at the server.

Also know that I will not be setting up SSL as I will be using a reverse-proxy on my server. I suggest you do the same if this is on a public facing server.

## Getting Started

Install the apps we will need to download and extract the Invoice Ninja files.

```sh
sudo apt install wget \
unzip \
git -y
```

## Installation

At this point, where you place your Docker Compose files is really up to you. I place mine in my home folder. Each app/service gets it’s own folder. So let’s create that.

```sh
mkdir invoiceninja
cd invoiceninja
```

And now let’s clone the git repo that holds all the files necessary (including the `docker-compose.yml` file) we need to get this puppy working.

```sh
git clone https://github.com/invoiceninja/dockerfiles.git
cd dockerfiles
```

Now, we need to generate an `APP_KEY` that will later be used in an `env` file. The output of this command will create a key that starts with `base64...`, and you will need to make note of the **entire output**.

If the command looks like it’s hanging, it’s because this will actually download the Invoice Ninja container in order to generate the key. The good news is that later when we spin it up, we’ll already have the container downloaded!

```sh
docker run --rm -it invoiceninja/invoiceninja php artisan key:generate --show
```

**Critical step incoming!!!**

Now we need to apply folder permissions to this container. Failure to do so will cause an error in the `app` container saying that ‘app permissions are wrong’, thus breaking the whole app.

```sh
chmod 755 docker/app/public
sudo chown -R 1500:1500 docker/app
```

## Editing configs - ENV

Let’s start with the `env` file. At this point we should be in the `dockerfiles` folder.

```sh
nano env
```

At this point, the below `env` is the stock version from the Invoice Ninja `git`. Note, that I have removed everything below the `#4 env vars` as this is no longer relevant.

```sh
APP_URL=http://in.localhost:8003/
APP_KEY=<insert your generated key in here>
APP_DEBUG=true
MULTI_DB_ENABLED=false
DB_HOST1=db
DB_PORT1=3306
DB_USERNAME1=ninja
DB_PASSWORD1=ninja
DB_DATABASE1=ninja
MYSQL_ROOT_PASSWORD=ninjaAdm1nPassword
MYSQL_USER=ninja
MYSQL_PASSWORD=ninja
MYSQL_DATABASE=ninja

#this is a system variable please do not remove
IS_DOCKER=true

PHANTOMJS_PDF_GENERATION=false
```

It goes without saying that you should **ABSOLUTELY** change the password variables, namely:

  * `DB_PASSWORD1`

  * `MYSQL_ROOT_PASSWORD`

  * `MYSQL_PASSWORD`

If you plan on generating PDFs of invoices, change the below variable from `false` to `true`. Although, personally I still can’t get the PDFs to work properly.

```sh
PHANTOMJS_PDF_GENERATION=false
```

Next, the `APP_URL` variable needs to be updated to whatever your domain is.

```sh
APP_URL=http://invoice.example.com
...
```

Next, we need to update the `APP_KEY` variable with the key we generated earlier.

```sh
APP_KEY=base64...
```

So, we should now have something like this.

```sh
APP_URL=http://invoice.example.com
APP_KEY=base64abcdefghijklmn..
APP_DEBUG=true
MULTI_DB_ENABLED=false
DB_HOST1=db
DB_PORT1=3306
DB_USERNAME1=ninja
DB_PASSWORD1=ninjaPASSWORDthatNOoneCOULDguess
DB_DATABASE1=ninja
MYSQL_ROOT_PASSWORD=ninjaAdm1nPassword
MYSQL_USER=ninja
MYSQL_PASSWORD=ninjaPASSWORDthatNOoneCOULDguess
MYSQL_DATABASE=ninja

#this is a system variable please do not remove
IS_DOCKER=true

PHANTOMJS_PDF_GENERATION=true
```

Now, save all the changes and exit.


## Editing configs - docker-compose.yml

The changes here are more subjective and ‘to your use case’, rather than necessary, but I’ll point out a few that I think are worth your time.

### container_name:

This variable currently doesn’t exist for each service, but I think its best to add these as custom values so when looking at your list of docker services, they are easier to identify. So add the below to each service.

```sh
services:
  server:
    ...
    container_name: invninja_server
    ...


  app:
    ...
    container_name: invninja_app
    ...


  db:
    ...
    container_name: invninja_db
    ...
```

### ports:

Currently the `docker-compose.yml` has port 80 exposed for the `server`. If this is fine for your purposes, then ignore this step. If you need to use a different port, then change according. For example:

```sh
services:
  server:
    ...
    ports: 
      - "10000:80"
    ...
```

Now, save all the changes and exit.

## Fire it up!!!

Since these files are in the same folder, ensure that you are still in the `../invoiceninja/dockerfiles` folder, and start the container.

```sh
docker-compose up -d
```

**Cool your jets!!**

**IF** you receive a `SQLSTATE[42S02]` exception complaint in your browser and a metric crap ton of tables, head back to the command line and run the following command.

```sh
docker-compose exec app php artisan migrate
```

I had this issue **everytime** I ran an install. This really should be in the documentation, but this lovely user [mentioned](https://github.com/invoiceninja/dockerfiles/issues/235) it for us.

**Continuing setup!**

Wait about 30s to a 1 minute before opening the URL in the browser as it takes a while for everything to get ready. But once that’s done, open your browser of choice and type in the URL that you specified in the `env` file earlier.

```sh
http://invoice.example.com
```

Now just fill in all the relevant info (remember to pay attention to the `db port`) and also update the `db host` value to `db`. Essentially, this should be the same as whatever the database server is named in the `docker-compose.yml` file. In our case, as it should be for everyone unless you changed it, is `db`.

That should be it!

## Post Install Setup & Possible Issues

### Reverse-Proxy Issues

I personally use [Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM) as my reverse-proxy of choice.

However, when trying to login into my new Invoice Ninja install via the browser, all goes to hell when I enter my domain `invoice.example.com`. It’s as if all the CSS has been removed. This happens on the HTTP and HTTPS versions.

However, if I entered the actual IP address and port number that the container is mapped to, for example, `http://12.345.67.890:1000`, well now all of a sudden everything is sunshine and roses.

So, complete the setup this way, and then try login again via the domain name (with or without HTTPS), and all shall be good.


### Is your interface is having a whinge??

#### APP_DEBUG

Once into my interface, and entering the company name currency of choice, I was met with a `debug` issue. Essentially, we now need to go back to the command line, open the `env` file and set the `APP_DEBUG` value to `false`, as seen below.

```sh
APP_DEBUG=false
```

I orginally tried doing this before firing up the container, but the whole thing refused to work. This is probably why it was set to **true** in the first place.

Now restart the container.

**Cronjob exclamations**

You will notice shortly after intial setup an exclamation in the bottom left corner of the screen. Ironically, after having done the above, and waiting a couple of minutes, just refresh the browser and that will go away.

Very hi-tech solution.

## Final Thoughts

I’m still needing to get to grips with this software for myself, but by all accounts it’s a great! Hopefully all these quirks that I found (and hopefully solved for the long-term) will help you who is reading this, so you can avoid the 3 week slog I’ve gone through.