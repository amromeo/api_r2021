# Training Environment for API lecture

This documents the steps to set up an RStudio Server Pro training instance for this session.

The goal is to create an RSP instance reachable at <https://api-r.cloud>. We will use the `rsp-train` Docker image and deploy to AWS, as detailed [here](https://github.com/skadauke/rsp-train).

## Local testing

```bash
# Replace with valid license
export RSP_LICENSE=XXXX-XXXX-XXXX-XXXX-XXXX-XXXX-XXXX

docker run --privileged -it \
    -p 8787:8787 \
    -e USER_PREFIX=apir \
    -e N_USERS=100 \
    -e PW_SEED=12345 \
    -e GH_REPO=https://github.com/amromeo/api_r2021 \
    -e R_PACKAGES=rmarkdown,shiny,DT,flexdashboard \
    -e RSP_LICENSE=$RSP_LICENSE \
    -v "$PWD/rsp-train/custom-login":/etc/rstudio \
    skadauke/rsp-train
```

### Deploy to AWS

- Registered `api-r.cloud` using AWS Route 53
- Created public `c5a.4xlarge` instance ($0.41/hr), with security groups configured to allow HTTP, HTTPS, and SSH
- Configured A record sending `api-r.cloud` to the public IP of the instance

```bash
ssh ubuntu@api-r.cloud
```

```bash
sudo apt -y update &&
sudo apt -y upgrade &&

sudo ufw allow OpenSSH &&
yes | sudo ufw enable
```

```bash
sudo apt -y install nginx &&
sudo ufw allow 'Nginx Full' &&
sudo systemctl start nginx &&
sudo systemctl enable nginx
```

```
MY_DOMAIN=uams.r-training.cloud &&

sudo snap install --classic certbot &&
sudo ln -s /snap/bin/certbot /usr/bin/certbot &&

sudo certbot --nginx -d $MY_DOMAIN --non-interactive --agree-tos -m webmaster@$MY_DOMAIN
```

```
sudo apt update &&
sudo apt -y install apt-transport-https ca-certificates curl gnupg lsb-release &&

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg &&

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null &&

sudo apt update &&
sudo apt -y install docker-ce docker-ce-cli containerd.io
```

```bash
# Replace with valid license
export RSP_LICENSE=XXXX-XXXX-XXXX-XXXX-XXXX-XXXX-XXXX &&

GH_REPO=https://github.com/amromeo/api_r2021 &&

sudo rm -rf r-training-lecture-uams-2021 &&

git clone $GH_REPO &&

cd api_r2021 &&

sudo docker run --privileged -it \
    --detach-keys "ctrl-a" \
    --restart unless-stopped \
    -p 8787:8787 -p 5559:5559 \
    -e USER_PREFIX=apir \
    -e N_USERS=100 \
    -e PW_SEED=12345 \
    -e GH_REPO=$GH_REPO \
    -e R_PACKAGES=rmarkdown,shiny,DT,flexdashboard \
    -e RSP_LICENSE=$RSP_LICENSE \
    -v "$PWD/rsp-train/custom-login":/etc/rstudio \
    skadauke/rsp-train
```

Detach the container by pressing **Ctrl+A**.

Inside the `/etc/nginx/sites-available/default` file, replace the following lines:

```
location / {
        # First attempt to serve request as file, then
        # as directory, then fall back to displaying a 404.
        try_files $uri $uri/ =404;
}
```

With the following:

```
location / {
    proxy_pass http://localhost:8787;
    proxy_redirect http://localhost:8787/ $scheme://$http_host/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_read_timeout 20d;
}
```

Add the following lines right after `http {` in `/etc/nginx/nginx.conf`:

```
        map $http_upgrade $connection_upgrade {
            default upgrade;
            ''      close;
        }

        server_names_hash_bucket_size 64;
```

Test the configuration and restart Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```