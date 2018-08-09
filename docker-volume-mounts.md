# Docker Volume Mounts

This technote describes how to create docker volume mounts.

## Local directory mounts

To illustrate a local directory mount, we can create some directories under
`~/mount`:
```
└── ~/mount
  ├── backup
  ├── project
  └── data
```

You can mount these folders as bind-mounts.
```
mkdir -p \
~/mount/backup \
~/mount/data \
~/mount/project
```

Create an infrastructure server container and bind-mount local directories as volumes:
```
docker run -ti \
  -v ~/mount/backup:/backup \
  -v ~/mount/data:/data \
  -v ~/mount/project:/project \
  --name server \
  --rm \
  alpine:3.8 \
  sh
```

You can access these volumes from the infrastructure container using the `--volumes-from server` runtime argument:
```
docker run -ti \
  --volumes-from server \
  --name client \
  --rm \
  alpine:3.8 \
  sh
```


## Technotes

01. [How To Share Data between Docker Containers  - Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-share-data-between-docker-containers)

02. [How to share Docker volumes across hosts - JAXEnter](https://jaxenter.com/how-to-share-docker-volumes-across-hosts-119602.html)

03. [Why Containers Miss a Major Mark: Solving Persistent Data in Docker - Storage OS](https://storageos.com/why-containers-miss-a-major-mark-solving-persistent-data-in-docker/)
