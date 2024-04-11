---
sidebar_position: 90
---

# Podman deploy with quadlets

You can deploy Immich on Podman using quadlets.

Here are some sample rootless quadlet container files that can be placed in /etc/containers/systemd/users/${ID} where ID is the uid of whatever your rootless user is.

Please note you'll need :z or :Z for selinux enabled hosts.

If you are doing hardware acceleration with NVIDIA make sure you follow the immich documentation including the appropriate NVIDIA Container Device Interface (https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html) and adding the device (AddDevice=nvidia.com/gpu=0).

# Sample Quadlet Files

It is recommended to run these rootless, and as such the ideal place is likely /etc/containers/systemd/users/${UID}

## Universal

immich-database.container
```bash
[Unit]
Description=Immich Database
Requires=immich-redis.service

[Container]
AutoUpdate=registry
EnvironmentFile=${location_of_env_file}
Image=registry.hub.docker.com/tensorchord/pgvecto-rs:pg16-v0.2.1
Label=registry
Network=slirp4netns:port_handler=slirp4netns
PublishPort=5432:5432
Volume=${host_database_directory}:/var/lib/postgresql/data:z
Volume=/etc/localtime:/etc/localtime:ro

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

immich-redis.container
```bash
[Unit]
Description=Immich Redis

[Container]
AutoUpdate=registry
Image=registry.hub.docker.com/library/redis:6.2-alpine@sha256:51d6c56749a4243096327e3fb964a48ed92254357108449cb6e23999c37773c5
Label=registry
Network=slirp4netns:port_handler=slirp4netns
PublishPort=6379:6379
Timezone=America/Montreal

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

immich-server.container
```bash
[Unit]
Description=Immich Server
Requires=immich-redis.service immich-database.service

[Container]
AutoUpdate=registry
EnvironmentFile=${location_of_env_file}
Image=ghcr.io/immich-app/immich-server:release
Label=registry
Network=slirp4netns:port_handler=slirp4netns
Exec=start.sh immich
PublishPort=3000:3000
PublishPort=3001:3001
Volume=${host_upload_directory}:/usr/src/app/upload
Volume=/etc/localtime:/etc/localtime:ro

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

## No Hardware Acceleration

immich-microservices.container
```bash
[Unit]
Description=Immich Microservices
Requires=immich-redis.service immich-database.service

[Container]
AutoUpdate=registry
EnvironmentFile=${location_of_env_file}
Image=ghcr.io/immich-app/immich-server:release
Label=registry
Network=slirp4netns:port_handler=slirp4netns
PublishPort=3002:3002
Volume=${host_upload_directory}:/usr/src/app/upload:z
Volume=/etc/localtime:/etc/localtime:ro
Exec=start.sh microservices

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

immich-ml.container
```bash

[Unit]
Description=Immich Machine Learning
Requires=immich-redis.service immich-database.service

[Container]
AutoUpdate=registry
EnvironmentFile=${location_of_env_file}
Image=ghcr.io/immich-app/immich-machine-learning:release
Label=registry
Network=slirp4netns:port_handler=slirp4netns
PublishPort=3003:3003
Volume=${cache_directory}:/cache:z
Volume=/etc/localtime:/etc/localtime:ro

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

## Hardware Acceleration - NVIDIA CUDA

immich-microservices.container
```bash
[Unit]
Description=Immich Microservices
Requires=mnt-data01.mount immich-redis.service immich-database.service

[Container]
AddDevice=nvidia.com/gpu=0
AutoUpdate=registry
EnvironmentFile=/mnt/data01/immich-app/.env
Image=ghcr.io/immich-app/immich-server:release
Label=registry
Network=slirp4netns:port_handler=slirp4netns
PublishPort=3002:3002
Volume=/mnt/data01/uploads:/usr/src/app/upload:z
Volume=/etc/localtime:/etc/localtime:ro
Exec=start.sh microservices

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```

immich-ml.container
```bash
[Unit]
Description=Immich Machine Learning
Requires=mnt-data01.mount immich-redis.service immich-database.service

[Container]
AddDevice=nvidia.com/gpu=0
AutoUpdate=registry
EnvironmentFile=/mnt/data01/immich-app/.env
Image=ghcr.io/immich-app/immich-machine-learning:release-cuda
Label=registry
Network=slirp4netns:port_handler=slirp4netns
PublishPort=3003:3003
Volume=/mnt/data01/model-cache:/cache:z
Volume=/etc/localtime:/etc/localtime:ro

[Service]
Restart=always

[Install]
WantedBy=multi-user.target default.target
```
