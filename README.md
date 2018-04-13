# jupyterhub-deploy-compose

This is my setup for getting jupyterhub to deploy using docker-compose.

It was forked from https://github.com/jupyterhub/jupyterhub-deploy-docker and modifed for my use

## Install

If you would like to repliacte this setup, you need to do the following steps:

1. Create a `secrets/` directory in the root of the repo: `mkdir secrets`
2. Obtain a fully qualified domain name (called `DOMAIN.COM` in this example)
3. Make sure your domain has an `A` record pointing to the IP address of the server
4. Create an SSL certificate using letsencrypt. Here's how I would do it (NOTE: make sure to replace `DOMAIN.COM` with your domain):
```bash
cd ..
git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
./letsencrypt-auto certonly --standalone -d DOMAIN.COM
```
5. Move the SSL cert and key to the `secrets/` directory (NOTE: you will need to replace `DOMAIN.COM` with your domain)
```bash
# only do the cd if you aren't already in this directory
cd ../jupyterhub-docker-compose
sudo cp  /etc/letsencrypt/live/DOMAIN.COM/privkey.pem secrets/jupyterhub.key
sudo cp  /etc/letsencrypt/live/DOMAIN.COM/privkey.pem secrets/jupyterhub.key
```
6. Create a file `userlist` in this directory. This should contain one line for each user you would like to have access to the jupyterhub instance. Each line will contain the github username and, optionally ` admin` if the user should be allowed to administer the jupyterhub instance. Here is an example:
```
$ cat userlist                                                                                                                                                                                                                                                                                                          
sglyon admin
other_user
other_admin admin
```
7. Configure github Oauth. Please follow the instructions [here](https://github.com/jupyterhub/jupyterhub-deploy-docker/tree/96e41209e518bcef839fcb866c7b8ed968c8d736#authenticator-setup). In the end you will need to have created a file `secrets/oauth.env` that has three entries like this:
```
GITHUB_CLIENT_ID=< get this value from github >
GITHUB_CLIENT_SECRET=< get this value from github >
OAUTH_CALLBACK_URL=https://DOMAIN.COM/hub/oauth_callback
```
8. Prepare the single-user notebook image by running `make notebook_image`
9. Build the docker-compose setup by running `make build`
10. Now you are all ready! You just need to do `docker-compose up -d` from this directory to launch the jupyterhub server. If you want to stop it cd to this directory and run `docker-compose down`. 

## Usage

When you first login to jupyterhub you will see a folder named `work`. This directory is mounted in a docker volume and the data is stored on the server's hard drive. Anything done outside of this folder (e.g. alongside it) **will not be persisted across startups of the user's singleuser server**. Thus, for data to be saved long term, it must be saved inside the `work` directory.


## Config

Other settings you can configure:

- You can change the docker image for the single user notebook servers that are spawned by changing the `DOCKER_NOTEBOOK_IMAGE` setting in the file `.env`. This should point to something you can `docker pull`. Please follow the instructions [here](https://github.com/jupyterhub/jupyterhub-deploy-docker/tree/96e41209e518bcef839fcb866c7b8ed968c8d736#how-do-i-specify-the-notebook-server-image-to-spawn-for-users) to make sure the singleuser image is configured properly. After changing this setting be sure to execute `make notebook_image` to update the `jupyterhub-user` on the server. I run the following sequence of commands after changing the `DOCKER_NOTEBOOK_IMAGE` setting:
```bash
make notebook_image
docker-compose down
make build
docker-compose up -d
```

## Updating ssl certs

Every 90 days we will have to update SSL certs. Here's how I did it (starting at the root of this repo).

```shell
docker-compose down
rm secrets/jupyterhub.{key,crt}
cd ../letsencrypt
./letsencrypt-auto certonly --standalone -d DOMAIN.COM
cd ../jupyterhub-deploy-docker
sudo cp /etc/letsencrypt/live/DOMAIN.COM/fullchain.pem secrets/jupyterhub.crt
sudo cp  /etc/letsencrypt/live/DOMAIN.COM/privkey.pem secrets/jupyterhub.key
docker-compose build --no-cache
docker-compose up -d
```
