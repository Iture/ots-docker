# Docker setup for OpenTAKServer
--- 
Docker setup for [OpenTAKServer](https://github.com/brian7704/OpenTAKServer) (OTS) is yet another open source TAK Server for ATAK, iTAK, and WinTAK

### *****************************************************************
### NOT READY FOR PRODUCTION YET
### *****************************************************************

###
###


## First boot
```Shell
# Download / Pull
git clone git@github.com:milsimdk/ots-docker.git && cd ots-docker

# Start OTS
make up

# Check logs
make logs
```

WebUI is available on https://localhost

## Problems you can have
- [Got permission denied while trying to connect to the Docker daemon socket](#docker-daemon-socket-denied)
- [Folder permissions](#folder-permissions)

### Docker daemon socket denied
```shell
sudo usermod -aG docker $USER
newgrp docker
```

### Folder permissions
If you get a `Permission denied` error it might because of folder permissions \
To fix it add your user id and group id in `compose.override.yaml`
```shell
id
cp compose.override.yaml-example compose.override.yaml

# Change line 4 so it matches your folder owner id and group id 
```


## Config changes 
You can change config options by using 'environment' in the compose.override.yaml file \
Options name must have the prefix `DOCKER_` or else they are ignored! \
You can overwrite all settings this way so watch out!!!

It's also possible to just change them in the `persistent/ots/config.yml` file

## Security Considerations

### Network Isolation (Optional)

Starting from version 3.0, you can enable network segmentation to improve security:

```bash
# .env
NETWORK_SECURITY_MODE=segmented
```

This separates internet-facing services from the database and message broker, providing an additional security layer.

For detailed configuration and testing guide, see [Network Security in CLAUDE.md](./CLAUDE.md#network-security).

### Required Ports for TAK Clients

The following ports must be open in your firewall for TAK clients to connect:

```bash
# Essential TAK ports
ufw allow 80/tcp        # HTTP WebUI
ufw allow 443/tcp       # HTTPS WebUI
ufw allow 8080/tcp      # TAK API (HTTP)
ufw allow 8443/tcp      # TAK API (HTTPS + mTLS)
ufw allow 8446/tcp      # Certificate enrollment
ufw allow 8088/tcp      # CoT streaming (TCP)
ufw allow 8089/tcp      # CoT streaming (SSL/TLS)
ufw allow 8883/tcp      # MQTT / Meshtastic

# Optional: Video streaming (if using MediaMTX)
ufw allow 8554/tcp      # RTSP video
ufw allow 8888/tcp      # HLS video streaming

# Restrict everything else
ufw default deny incoming
ufw enable
```

### Development Debug Ports

For development and debugging, you can expose database and message broker ports:

```bash
# Use the development override
docker compose -f compose.yaml -f compose.dev.yaml up -d

# Now accessible from your network:
# PostgreSQL: 5432 (pgAdmin, DBeaver, psql)
# RabbitMQ AMQP: 5672 (message queue testing)
# RabbitMQ MQTT: 1883 (mosquitto_pub/sub)
# RabbitMQ Management UI: 15672 (monitoring dashboard)
```

**WARNING**: These ports expose services on all network interfaces. Use only in development, never in production!

## Whats supported for now
 - [x] Tak server
 - [x] Rabbitmq 
   - [x] MQTT
 - [x] WebUI
 - [x] Meshtastic
 - [x] MediaMTX
 - [ ] Mumble
 - [ ] SSL / Let's Encrypt

## Requirements
 - Docker must be installed
 - Docker compose v2 is used
 - Only tested locally on my Macbook (arm64), but should work on most Linux operating systems
 - Custom OpenTakServer docker images used
   - [OpenTakServer](https://github.com/milsimdk/ots-docker-image/pkgs/container/ots-docker-image)
   - [OpenTakServer-WebUI](https://github.com/milsimdk/ots-ui-docker-image/pkgs/container/ots-ui-docker-image)

## How to use the MakeFile
```shell
# Show help
make
```

## Thanks
  - [Brian](https://github.com/brian7704) for creating [OpenTAKServer](https://github.com/brian7704/OpenTAKServer)
