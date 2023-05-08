# Presentation

> [Like Prometheus, but for logs!](https://grafana.com/docs/loki/latest/)

Unlike other logging systems, Loki is built around the idea of only indexing metadata about your logs: labels (just like Prometheus labels). Log data itself is then compressed and stored in chunks in object stores such as S3 or GCS, or even locally on the filesystem. A small index and highly compressed chunks simplifies the operation and significantly lowers the cost of Loki.

[Loki features](https://grafana.com/docs/loki/latest/fundamentals/overview/#loki-features):

- Efficient memory usage for indexing the logs
- Multi-tenancy
- LogQL, Loki’s query language
- Scalability
- Flexibility

There are like 3 deployment modes, I'm using something like the monolithic type:

![Example image](https://grafana.com/docs/loki/latest/fundamentals/architecture/deployment-modes/monolithic-mode.png)  

__________________________________________

Ideas gathered from:

- <https://youtu.be/h_GGd7HfKQ8>
- <https://docs.technotim.live/posts/grafana-loki/>
- <https://github.com/a-konar/arvindkonar/tree/main/loki/docker>

__________________________________________

## 1. Installing docker

Installing general dependencies:

```bash
apt-get update
apt-get install -y ca-certificates curl gnupg lsb-release
```

Installing docker:

```bash
# Adding docker repos
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installing Docker
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable the docker daemon
systemctl enable --now docker

# Running docker without sudo IF YOU ARE NOT ROOT
usermod -aG docker $USER
```

__________________________________________

## 2. Running

The whole idea of this repo it's to have a ready, [production level and scalable Grafana-Loki deployment](https://grafana.com/docs/loki/latest/fundamentals/architecture/deployment-modes/#simple-scalable-deployment-mode)

So if you wanna just that:

```bash
git clone https://github.com/kCyborg/grafana-loki.git && \
cd grafana-loki && \
docker compose up -d
```

And you will have something like:

```bash
[+] Running 7/7
 ✔ Network grafana-loki_loki                                   Created                                    0.1s
 ✔ Container grafana-loki-grafana-1                            Started                                    1.4s
 ✔ Container grafana-loki-minio-1                              Started                                    1.4s
 ✔ Container grafana-loki-promtail-1                           Started                                    1.4s
 ✔ Container grafana-loki-loki_write-1                         Started                                    2.0s
 ✔ Container grafana-loki-loki_read-1                          Started                                    2.1s
 ✔ Container grafana-loki-gateway-1                            Started                                    2.6s
```

Once the containers started, just head to your host IP using the 3000 port: <http://$IP:3000>. If it's your first time, then the credentials will be the default ones:

- user: `admin`
- pass: `admin`

It will ask you to change the default password and then you can proceed adding new promtail clients.

__________________________________________

## 3. Docker compose

The main inspiration was the [official docx](https://grafana.com/docs/loki/latest/installation/docker/), but it lacks some features I added:

- Basic (and customizable) authentication, using nginx container.
- Use [Min.IO](https://min.io/) as storage solution.
- Scalable deployment, meaning more than one loki container with write and read separated roles.

This chapter will explain the proccess to manually create the docker compose and I will try to explain (not so) detailled the options I used.

> NOTE: The following commands should be send to the host where Docker is running

Create the necesary folders:

```bash
mkdir -p ~/logs/grafana && \
mkdir ~/logs/loki && \
mkdir ~/logs/promtail && \
mkdir ~/logs/minio && \
mkdir ~/logs/nginx && \
cd ~/logs
```

### 3.1 Changing API authentication username and password

The first step it's create and nginx username/password. This seeks to give the Loki API some authentication, otherwise the data exposed by the API will be publicly accesible if we have our Loki deployment on any public IP. Just write in the command line:

```bash
username=loki
password=loki
printf "$username:$(openssl passwd -apr1 $password)\n" > nginx/.htpasswd
```

### 3.2 Changing Min.io credentials

Min.io credentials will be only accesible wihtin containers in a local network but, if we want, we can change MINIO credentials:

```bash
MINIO_ACCESS_KEY=superstrong
MINIO_SECRET_KEY=accesskey
echo "MINIO_ACCESS_KEY=superstrong" > .env
echo "MINIO_SECRET_KEY=accesskey" >> .env
```

If we change Min.io credentials we will need to also tweak the `./loki/loki-config.yml` by:

> NOTE: You can take a close look into the `loki-config.yml` [here](https://github.com/kCyborg/grafana-loki/blob/main/loki/loki-config.yml).

```bash
sed -i "s/access_key_id: \w\+/access_key_id: $MINIO_ACCESS_KEY/" ./loki/loki-config.yml
sed -i "s/secret_access_key: \w\+/secret_access_key: $MINIO_SECRET_KEY/" ./loki/loki-config.yml
```

> NOTE: You can take a closer look into the technical stuff of the `schema_config` option [here](https://grafana.com/docs/loki/latest/operations/storage/boltdb-shipper/#compactor)

Now we need to do the same thing, but for **promtail** now:

### 3.3 Changing local Promtail crentials

Promtail is the client that will send sever's logs to Loki throug Loki's API, and in order to access the API to push the logs it will need authentication. By default it use my own credentials, but their are not secure at all, so in order to change it:

```bash
sed -i "s/loki:loki/$username:$password/" promtail/promtail-config.yml
```

> NOTE: You can take a close look into promtail.yml configuration file [here](https://github.com/kCyborg/grafana-loki/blob/main/promtail/promtail-config.yml)

### 3.4 The docker compose file

Altough you can check [here the docker-compose file](https://github.com/kCyborg/grafana-loki/blob/main/docker-compose.yml) I will put it in here and I will let it in the one line bash command if you want to copy it:

```bash
cat <<EOF > docker-compose.yml
---
version: "3"

networks:
  loki:

services:
  loki_read:
    image: grafana/loki:latest
    command: "-config.file=/etc/loki/loki-config.yml -target=read"
    user: "0"
    ports:
      - 3100
      - 7946
      - 9095
    volumes:
      - ./loki:/etc/loki
    restart: unless-stopped
    depends_on:
      - minio
    networks: &loki-dns
      loki:
        aliases:
          - loki

  loki_write:
    image: grafana/loki:latest
    command: "-config.file=/etc/loki/loki-config.yml -target=write"
    ports:
      - 3100
      - 7946
      - 9095
    volumes:
      - ./loki:/etc/loki
    depends_on:
      - minio
    networks:
      <<: *loki-dns

  minio:
    user: "0"
    image: minio/minio
    entrypoint:
      - sh
      - -euc
      - |
        mkdir -p /data/loki-data && \
        mkdir -p /data/loki-ruler && \
        minio server /data
    environment:
      - MINIO_ACCESS_KEY=${MINIO_ACCESS_KEY}
      - MINIO_SECRET_KEY=${MINIO_SECRET_KEY}
      - MINIO_PROMETHEUS_AUTH_TYPE=public
      - MINIO_UPDATE=off
    ports:
      - 9000
    volumes:
      - ./minio:/data
    networks:
      - loki

  promtail:
    image: grafana/promtail:2.4.0
    volumes:
      - /var/log:/var/log
      - ~/logs/promtail:/etc/promtail
    # ports:
    #   - "1514:1514" # this is only needed if you are going to send syslogs
    restart: unless-stopped
    command: -config.file=/etc/promtail/promtail-config.yml
    networks:
      - loki

  grafana:
    image: grafana/grafana:latest
    user: "0"
    volumes:
    - ./grafana:/var/lib/grafana
    ports:
      - "3000:3000"
    restart: unless-stopped
    ###
    entrypoint:
      - sh
      - -euc
      - |
        mkdir -p /etc/grafana/provisioning/datasources
        cat <<EOF > /etc/grafana/provisioning/datasources/ds.yaml
        apiVersion: 1
        datasources:
        - name: Loki
          type: loki
          access: proxy
          url: http://gateway:3100
          orgId: 1
          basicAuth: true
          basicAuthUser: loki
          secureJsonData:
            basicAuthPassword: loki
          isDefault: true
          version: 1
          editable: true
        EOF
        /run.sh
    ###
    networks:
      - loki

  gateway:
    image: nginx:latest
    depends_on:
      - loki_read
      - loki_write
    ports:
      - "3100:3100"
    networks:
      - loki
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/.htpasswd:/etc/nginx/.htpasswd
EOF
```

__________________________________________

## 4. Configuring a promtail client using Ansible

In this very repo you will find an ansible playbook called `play.yaml`. This will install the promtail client and configure it to point to your Loki server

In order to use the playbook I created you will need to prepare your host:

```bash
apt install -y build-essential sshpass python3-dev python3-virtualenv 
```

Then activate a python virtual environment with:

```bash
# within the folder where you downloaded the repo
cd ansible
virtualenv ansiblevenv
source ansiblevenv/bin/activate
```

Now install the requeriments:

```bash
pip3 install -r requeriments.txt
```

Modify the `inventory` file:

```bash
[loki_clients]
123.456.789.101
```

And then just run:

```bash
source ansiblevenv/bin/activate
ansible-playbook -i play.yaml
```

And thus you will have a client in your Loki.

__________________________________________

## 5. Adding some easy rules with `ufw`

This is the easist part:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22
ufw allow 3000
ufw allow 3200
ufw allow 8092
ufw enable
```

__________________________________________

## To do list

- Automatize the ansible playbook (right now it use the default credential for the API authentication, this should change)
- Add a nice custom dashboard
- Add some rules
