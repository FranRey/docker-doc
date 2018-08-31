## NVIDIA Docker2 on CentOS 7

### Step 01.00: Install Docker.

Follow these steps to install Docker CE and NVIDIA Docker 2.0 on your host platform.

#### Step 01.01: Install Docker CE.

```
# Install required packages
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

# Set up the docker stable repository
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# Install the latest version of Docker CE
sudo yum install docker-ce

# Start Docker
sudo systemctl start docker

# Verify that docker is installed correctly by running the hello-world image
sudo docker run hello-world
```

#### Step 01.02: Install Nvidia Docker 2.0

For CentOS 7 install:

```
# If you have nvidia-docker 1.0 installed: we need to remove it and all existing GPU containers
docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
sudo yum remove nvidia-docker

# Add the package repositories
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | \
  sudo tee /etc/yum.repos.d/nvidia-docker.repo

# Install nvidia-docker2 and reload the Docker daemon configuration
sudo yum install -y nvidia-docker2
sudo pkill -SIGHUP dockerd

nvidia-docker registers a new container runtime to the Docker daemon.
You must select the nvidia runtime when using docker run:

# Test nvidia-smi with the latest official CUDA image
sudo systemctl start docker
sudo docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
```

This should give you a similar output:
```
Mon Jun 25 17:12:31 2018
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 396.24                 Driver Version: 396.24                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  TITAN V             Off  | 00000000:01:00.0  On |                  N/A |
| 33%   45C    P8    28W / 250W |    213MiB / 12063MiB |      1%      Default |
+-------------------------------+----------------------+----------------------+
|   1  TITAN V             Off  | 00000000:02:00.0 Off |                  N/A |
| 30%   44C    P8    28W / 250W |      0MiB / 12066MiB |      1%      Default |
+-------------------------------+----------------------+----------------------+
|   2  TITAN V             Off  | 00000000:04:00.0 Off |                  N/A |
| 32%   45C    P8    25W / 250W |      0MiB / 12066MiB |      1%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```

Command Explanation:

-    nvidia-docker – the NVIDIA shim/wrapper that helps setup GPUs with DOCKER
    run – tells nvidia-docker wrapper that you’re going to start (instantiate) a container
        Note that for any command that does not include ‘run’ in it, you can simply use docker, but if you use nvidia-docker the command gets passed through to docker (E.g docker images displays the docker images on your system, nvidia-docker images would also execute and show the same info)
-    –rm – this tells DOCKER that after the command runs, the container should be stopped/removed
        This is a very interesting feature / capability. If you think about it, an entire environment is being created, for nvidia-smi to run and then the container is destroyed. It can be done repeatedly and is very simple and fast.
-    nvidia/cuda – this is the name of an image
        Note that, the first time you run this command, DOCKER will go out and find an image with that name, and download the docker image from the hub.docker.com repository. This will only happen the first time. You could also run docker pull nvidia/cuda before hand to be verbose and separate the steps. This one liner works though.
-    nvidia-smi – this is the command to be run on the container



If you would like to use Docker as a non-root user, you should now consider
adding your user to the "docker" group with something like:

```
  sudo usermod -aG docker your-user
```

Remember to log out and back in for this to take effect!

WARNING: Adding a user to the "docker" group grants the ability to run
         containers which can be used to obtain root privileges on the
         docker host.
         Refer to https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface
         for more information.

#### Step 01.03: Create a user on the host system.

List existing users:
```
less /etc/passwd
```

List existing groups:
```
cat /etc/group
```

To delete an existing user:
```
sudo userdel --remove developer
```

Create a new group: developers

```
export GROUP=developers

sudo groupadd \
  -f \
  -g 65536 \
  $GROUP
```

Create a new user: developer

```
export USER=developer
export USER_COMMENT=Developer
export UID=1001
export GID=1001
export GROUPS=developers

# create user group
sudo groupadd \
  -f \
  -g $GID \
  $USER

# create user and assign groups
sudo useradd \
  -u $UID \
  -g $GID \
  -G $GROUPS \
  -c $USER_COMMENT \
  -m $USER

# change the password
sudo passwd $USER
```

#### Step 01.03: Configure user-namespace remapping.

From the command line:
```
sudo systemctl stop docker.service
```

Specify subuid and subgid

Edit /etc/subuid
```
sudo nano /etc/subuid

developer:1001:1
developer:100100:65536
```

Get the docker group id:
```
getent group docker
docker:x:982:developer
```

Use the obtained gid `982` to set the subgid, for specifying the docker group
for setting the user group, for usernamespace remapping. This will allow files
created on the host to belong to $USER:docker.

Edit /etc/subgid
```
sudo nano /etc/subgid

developer:982:1
developer:100100:65536
```

Edit /etc/docker/daemon.json
```
sudo nano /etc/docker/daemon.json

{
    "dns": ["8.8.8.8", "8.8.4.4"],
    "userns-remap": "developer",
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }

}
```

List available kernels:
```

```

Change default kernel:
```
sudo grub2-set-default 0
```


Add the `namespace.unpriv_enable=1` option to the kernel (vmlinuz*) command line. To do this, use the grubby command as follows:
```
sudo -s
grubby --args="namespace.unpriv_enable=1 user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
```

List all the kernel entries and verify the changes:
```
# grubby --info=ALL


```

Add a value to the user.max_user_namespaces kernel tuneable so it is set permanently as follows:
```
echo "user.max_user_namespaces=15076" >> /etc/sysctl.conf
```

Reboot the system. After the system comes up, check that the kernel options were properly assigned and that the docker service is running with user namespaces enabled.



Reload the docker service in systemd to activate changes to the service configuration:
```
sudo systemctl daemon-reload
```
If you need to restart the docker service itself, enter the following command:

```
sudo systemctl restart docker
```
or
```
sudo dockerd --userns-remap="developer:developer"
```
if you didn't make an entry in daemon.json

The Docker Engine applies the same user namespace remapping rules to all containers, regardless of who runs a container or who executes a command within a container.

Check if the new directory 1001.982 is created. This is the subordinate user ID file user by user-namespaces.

```
sudo ls -la /var/lib/docker

total 0
drwx--x--x. 1 root      root      194 Jun 30 10:37 .
drwxr-xr-x. 1 root      root      790 Jun 25 20:41 ..
drwx------. 1 developer docker    158 Jun 30 10:45 1001.982
drwx------. 1 root      root       20 Jun 25 20:41 btrfs
drwx------. 1 root      root       20 Jun 25 20:41 builder
drwx--x--x. 1 root      root       12 Jun 25 20:41 containerd
drwx------. 1 root      root      512 Jun 26 21:34 containers
drwx------. 1 root      root       10 Jun 25 20:41 image
drwxr-x---. 1 root      root       10 Jun 25 20:41 network
drwx------. 1 root      root       20 Jun 25 20:41 plugins
drwx------. 1 root      root        0 Jun 26 20:01 runtimes
drwx------. 1 root      root        0 Jun 25 20:41 swarm
drwx------. 1 root      root        0 Jun 26 21:27 tmp
drwx------. 1 root      root        0 Jun 25 20:41 trust
drwx------. 1 root      root       22 Jun 25 20:41 volumes
```

Check if everything is working as expected.

```
sudo docker run --rm -it alpine /bin/sh

Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
ff3a5c916c92: Pull complete
Digest: sha256:e1871801d30885a610511c867de0d6baca7ed4e6a2573d506bbec7fd3b03873f
Status: Downloaded newer image for alpine:latest
```

Type the following command in the docker container
```
/ # ping ubuntu.com
```

On the host:
```
ps xua | grep ping

developer     2851  0.0  0.0 602280 16656 ?        Sl   11:32   0:02 /usr/libexec/gsd-housekeeping
developer    20882  0.0  0.0   1512     4 pts/0    S+   16:20   0:00 ping ubuntu.com
developer    20886  0.0  0.0 112720  2216 pts/1    S+   16:20   0:00 grep --color=auto ping
```

The ping command started in the container is now being run by user "developer".


If you want to start a container without using User-Namspaces, you can override the configuration when you start a container by using the command-line flag `--userns=host`. This will start up a container in the "normal" way, i.e. owned by root.

Refer the following links for more details:
- [Docker and NVIDIA-docker on your workstation: Setup User Namespaces  - Dr Donald Kinghorn - Peget Systems](https://www.pugetsystems.com/labs/hpc/Docker-and-NVIDIA-docker-on-your-workstation-Setup-User-Namespaces-906/)
- [Configuring User Namespace Remapping - Oracle Container Runtime for Docker](https://docs.oracle.com/cd/E52668_01/E87205/html/ol-docker-userns-remap.html)
- [Docker runtime execution options - Docker Docs](https://docs.docker.com/engine/reference/commandline/dockerd/#docker-runtime-execution-options)

#### Step 01.04: Configure volumes

Ensure that the Dockefile contains the following entries for defining volume mount-points:
```
# Define mountable directories.
VOLUME ["/backup", "/data", "/project", "/tool"]

# Define working directory.
WORKDIR /project
```

Create folders for the volume mount-points on the host:
```
export USER=developer
export GROUP=developer

mkdir -p \
  /home/${USER}/mount/backup \
  /home/${USER}/mount/data \
  /home/${USER}/mount/project \
  /home/${USER}/mount/tool
```

Run docker and mount the volumes:
```
export USER=developer
export GROUP=developer

sudo docker run -it \
  -p 5901:5901 \
  -e USER=${USER} \
  -e GROUP=${GROUP} \
  -v /home/${USER}/mount/backup:/backup \
  -v /home/${USER}/mount/data:/data \
  -v /home/${USER}/mount/project:/project \
  -v /home/${USER}/mount/tool:/tool \
  --runtime=nvidia \
  nvidia/cuda/cudnn/ubuntu-desktop:9.2-7-16.04

```

### Step 02.00: Re-install docker.

#### Step 02.01: Uninstall docker-ce on CentOS 7.

```bash
sudo yum remove docker-ce
sudo rm -rf /var/lib/docker
```

#### Step 02.02: Install docker-ce on CentOS 7.

```bash
sudo yum install docker-ce
sudo yum install nvidia-docker2
```


### Step 03.00: Install a remote desktop client.

#### Step 03.01: Install TigerVNC on CentOS 7.

```
sudo yum install tigervnc
```



## Gists

Usernamespace Remapping

1. [Setup User Namespaces for docker on RHEL/Centos 7.3  - Docker_User_NameSpaces_setup.md - dpneumo - Github](https://gist.github.com/dpneumo/279d6bc5dcbe5609cfcb8ec48499701a)

## Related links

1. [Linux List All Users In The System - nixCraft](https://www.cyberciti.biz/faq/linux-list-users-command/)

2. [Getting Started with Containers - User Namespace Options - RedHat Linux Atomic Host 7](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html-single/getting_started_with_containers/index#user_namespaces_options)

3. [CentOS / RHEL 7 : How to Modify GRUB2 Arguments with grubby](https://www.thegeekdiary.com/centos-rhel-7-how-to-modify-grub2-arguments-with-grubby/)
