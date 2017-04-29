# Setup of GitLab Docker on Synology DSM

- [Introduction](#introduction)
- [GitLab](#sameersbndocker-gitlab)
    - [Environment Variables](#environment-variables)   
    - [Port Settings](#port-settings)
    - [Volume](#volume)
    - [Links](#links)
    - [Update Warning](#update-warning)
- [GitLab Runner](#sameersbngitlab-ci-multi-runner)
    - [Environment Variables](#environment-variables-1)   
    - [Volume](#volume-1)
    - [Passwordless Sudoer](#passwordless-sudoer)
- [GitLab Container Registry](#registry)
    - [Environment Variables](#environment-variables-2)   
    - [Volume](#volume-2)
    - [Links](#links-1)
- [HTTPS](#https)
    - [Reverse Proxy](#reverse-proxy)
    - [Certificate](#certificate)
- [Port Forwarding](#port-forwarding)


## Introduction

This tutorial explains how to install a complete GitLab environment, including GitLab Runner and GitLab Container Registry, with the Synology DiskStation Manager (DSM).
Each GitLab module can be installed with the help of Docker container and will receive it's own subdomain.

## sameersbn/docker-gitlab

GitLab itself can be installed via the Package Center. The package is called **Docker GitLab**.
Because it is maintained by Synology itself, it should install without any problem.
If they are not already installed, this package will also install the Docker package as well as the MariaDB package.
If the Docker package was not installed before, a shared folder has to be created for it. In this tutorial the shared folder is called `/docker`.
Once the installation is finished, stop the newly installed package with the help of the Package Center.
This will make the Docker environment variables, port settings, volume mounts and links of the Docker containers editable.
Now open the Docker package and start editing the **synology_gitlab** container.

### Environment Variables

| Environment Variable | Value |
| -------------------- | ----- |
| SSL_REGISTRY_CERT_PATH | /certs/registry.crt |
| SSL_REGISTRY_KEY_PATH | /certs/registry.key |
| GITLAB_REGISTRY_KEY_PATH | /certs/registry-auth.key |
| GITLAB_REGISTRY_CERT_PATH | /certs/registry-auth.crt |
| GITLAB_REGISTRY_API_URL | http://registry:5555 |
| GITLAB_REGISTRY_PORT | 443 |
| GITLAB_REGISTRY_HOST | hub.your_diskstation_url.com |
| GITLAB_REGISTRY_ENABLED | true |
| IMAP_ENABLED | true |
| IMAP_HOST | imap.gmail.com |
| IMAP_PORT | 993 |
| IMAP_USER | your_gmail_user_name |
| IMAP_PASS | your_gmail_password |
| GITLAB_INCOMING_EMAIL_ADDRESS | your_gmail_user_name+%{key}@gmail.com |
| OAUTH_GITHUB_APP_SECRET | your_github_app_secret |
| OAUTH_GITHUB_APP_KEY | your_github_app_key |
| OAUTH_BITBUCKET_APP_SECRET | your_bitbucket_app_secret |
| OAUTH_BITBUCKET_APP_KEY | your_bitbucket_app_key |
| GITLAB_HOST | git.your_diskstation_url.com |
| GITLAB_PORT | 443 |
| GITLAB_SSH_PORT | 30001 |
| GITLAB_EMAIL | your_notification@email_address.com |
| DB_TYPE | mysql |
| DB_HOST | 172.17.0.1 |
| DB_NAME | gitlab |
| DB_USER | gitlab |
| DB_PASS | your_very_long_secure_key_1 |
| GITLAB_SECRETS_OTP_KEY_BASE | your_very_long_secure_key_2 |
| GITLAB_SECRETS_DB_KEY_BASE | your_very_long_secure_key_3 |
| GITLAB_SECRETS_SECRET_KEY_BASE | your_very_long_secure_key_4 |
| SMTP_ENABLED | true |
| SMTP_DOMAIN | www.gmail.com |
| SMTP_HOST | smtp.gmail.com |
| SMTP_PORT | 587 |
| SMTP_USER | your_gmail_user_name |
| SMTP_PASS | your_gmail_password |
| SMPT_OPENSSL_VERIFY_MODE | none |

### Port Settings

| Local Port | Container Port | Type |
| ---------- | -------------- | ---- |
| 30001 | 22 | tcp |
| 30000 | 80 | tcp |

### Volume

| File/Folder | Mount Path | Type |
| ----------- | ---------- | ---- |
| /docker/gitlab_registry/certs | /certs | rw |
| /docker/gitlab | /home/git/data | rw |

### Links

| Container Name | Alias |
| -------------- | ----- |
| synology_gitlab_redis | redisio |

### Update Warning

An update of the Docker GitLab package will revert all environment variable, volume mount, port and link changes made to the **synology_gitlab** and **synology_redis** container.

## sameersbn/gitlab-ci-multi-runner

The GitLab Runner Docker image can be downloaded from Docker Hub alias Registry inside of the Synology Docker package.
Search for **sameersbn/gitlab-ci-multi-runner**.

Because this package is not maintained from Synology directly, one steps have to be made manually first.
To successfully launch a GitLab Runner, you have to create the volume that is later mounted into the Docker.
Open File Station and create the folder `/docker/gitlab_runner`.

After that open the Docker package again, open the **Image** tab and launch an instance of the **gitlab-ci-multi-runner**.
While doing so configure the environment variables and volumes within the advanced settings.

### Environment Variables

| Environment Variable | Value |
| -------------------- | ----- |
| RUNNER_EXECUTOR | shell |
| RUNNER_DESCRIPTION | GitLabRunner |
| RUNNER_TOKEN | your_gitlab_runner_token |
| CI_SERVER_URL | https://git.your_diskstation_url.com:443/ci |

### Volume

| File/Folder | Mount Path | Type |
| ----------- | ---------- | ---- |
| /docker/gitlab_runner | /home/gitlab_ci_multi_runner/data | rw |

### Passwordless Sudoer

If the deployment instructions in the `gitlab-ci.yml` files require sudo permission, follow these instructions.

1. SSH into the Synology.
2. Discover the container ID of the runner.
```
sudo docker ps | grep 'sameersbn/gitlab-ci-multi-runner:latest' | awk '{print $1}'
```
3. Enter the docker container.
```
sudo docker exec -it DOCKER_ID bash
```
4. Make the gitlab_ci_multi_user passwordless sudoer.
```
adduser gitlab_ci_multi_user sudo
echo "gitlab_ci_multi_user ALL=NOPASSWD: ALL" > /etc/sudoers.d/gitlab_ci_multi_user
```

## registry

The official Docker Registry container can be downloaded from Docker Hub alias Registry inside of the Synology Docker package. Search for **registry**.
To prepare the launch of the container, you first have to setup the volumes that are mounted into the container.
Open File Station and create the following folders:

 - `/docker/gitlab_registry`
 - `/docker/gitlab_registry/registry`
 - `/docker/gitlab_registry/certs`

Next a self signed certificate will be created with OpenSSL. To do so SSH into your Synology DiskStation.

1. Open the certificate folder.
```
cd /docker/gitlab_registry/certs
```
2. Generate a private key and sign request for the private key.
```
openssl req -nodes -newkey rsa:4096 -keyout registry-auth.key -out registry-auth.csr -subj "/CN=hub.your_diskstation_url.com"
```
3. Sign your created privated key.
```
openssl x509 -in registry-auth.csr -out registry-auth.crt -req -signkey registry-auth.key -days 3650
```

After that open the Docker package, launch a **registry** container and configure the environment variables, volume mounts and and links like explained below.

### Environment Variables

| Environment Variable | Value |
| -------------------- | ----- |
| REGISTRY_STORAGE_DELETE_ENABLED | true |
| REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE | /certs/registry-auth.crt |
| REGISTRY_AUTH_TOKEN_ISSUER | gitlab-issuer |
| REGISTRY_AUTH_TOKEN_SERVICE | container_registry |
| REGISTRY_AUTH_TOKEN_REALM | https://git.your_diskstation_url.com:443/jwt/auth |
| REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY | /registry |
| REGISTRY_LOG_LEVEL | info |

### Volume

| File/Folder | Mount Path | Type |
| ----------- | ---------- | ---- |
| /docker/gitlab_registry/certs | /certs | rw |
| /docker/gitlab_registry/registry | /registry | rw |

### Links

| Local Port | Container Port | Type |
| ---------- | -------------- | ---- |
| 5555 | 5000 | tcp |

## HTTPS

Synology's Reverse Proxy service and *Let's Encrypt* can be used to secure the connction to GitLab and the registry Docker over HTTPS.

### Reverse Proxy

Create two new rules like the following:

| Description | Source Protocol | Source Hostname | Source Port | Destination Protocol | Destination Hostname | Destination Port |
| ----------- | --------------- | --------------- | ----------- | -------------------- | -------------------- | ---------------- |
| GitLab | HTTPS | git.your_diskstation_url.com | 443 | HTTP | localhost | 30000 |
| GitLab Registry | HTTPS | hub.your_diskstation_url.com | 443 | HTTP | localhost | 5555 |

### Certificate

If you don't have already a certificate, create a *Let's Encrypt* certificate with the domain name **your_diskstation_url.com** and alternative names **git.your_diskstation_url.com;hub.your_diskstation_url.com** in the Certificate section.
After that configure them to be used for the services **git.your_diskstation_url.com** and **hub.your_diskstation_url.com**.

## Port Forwarding

If GitLab is running behind a firewall, for example behind a router, port forwarding need to be configured inside the router.

| Service | Port | Protocol |
| ------- | ---- | -------- |
| GitLab HTTP | 80 | tcp |
| GitLab HTTPS | 443 | tcp |
| GitLab SSH | 30001 | tcp |
