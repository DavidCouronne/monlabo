## 1. Installer Docker sur Ubuntu

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```



## 2. Installer Dockge

```bash
sudo mkdir -p /opt/dockge/data
cd /opt/dockge
sudo curl -fsSL -o compose.yaml https://raw.githubusercontent.com/louislam/dockge/master/compose.yaml
sudo docker compose up -d
```

Dockge sera accessible sur `http://IP_VM:5001`.

## 3. Stack MeTube dans Dockge



```yaml
version: "3.8"
services:
  metube:
    image: ghcr.io/alexta69/metube
    container_name: metube
    restart: unless-stopped
    ports:
      - "8081:8081"
    volumes:
      - /mnt/nas/medias:/downloads
    environment:
      - UID=1000
      - GID=1000
```