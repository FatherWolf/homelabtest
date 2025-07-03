# Self-Hosted Media Server and Aggregation

Make sure to review everything here and if you have any issues please submit it as an issue. Also, we are more than open to any suggests or edits. Also, checkout the [Servarr Docker Setup](https://wiki.servarr.com/docker-guide) for more details on installing the stack.


## Navigation
* [Apps](https://github.com/TechHutTV/homelab/tree/main/apps)
* [Home Assistant](https://github.com/TechHutTV/homelab/tree/main/homeassistant)
* [__Media Server__](https://github.com/TechHutTV/homelab/tree/main/media)
  - [Companion Video](#companion-video)
    * [Updates Since Video Publish](#updates-since-video-publish)
  - [Media Server](#media-server)
    * [Jellyfin](https://github.com/TechHutTV/homelab/tree/main/media/jellyfin)
    * [Plex](https://github.com/TechHutTV/homelab/tree/main/media/plex)
  - [Data Directory](#data-directory)
    * [Folder Mapping](#folder-mapping)
    * [Network Share](#network-share)
  - [User Permissions](#user-permissions)
  - [Docker Compose and .env](#docker-compose-and-env)
  - [Download Clients](#download-clients)
    * [NZBGet](#nzbget)
      + [NZBGet Login Credentials](#nzbget-login-credentials)
      + [Download Directories Mapping](#nzbget-download-directories)
      + [Fix "directory does not appear" error in Sonarr/Radarr](#fix-directory-does-not-appear-to-exist-inside-the-container-error)
    * [qBittorrent](#qbittorrent)
      + [qBittorrent Login Credentials](#qbittorrent-login-credentials)
      + [Download Directories Mapping](#qbittorrent-download-directories)
  - [*arr Apps](#arr-apps)
* [Server Monitoring](https://github.com/TechHutTV/homelab/tree/main/monitoring)
* [Surveillance System](https://github.com/TechHutTV/homelab/tree/main/surveillance)
* [Storage](https://github.com/TechHutTV/homelab/tree/main/storage)
* [Proxy Management](https://github.com/TechHutTV/homelab/tree/main/proxy)

## Companion Video
```
# Updated video coming soon
[![alt text](image url)](video link)
```
### Updates Since Video Publish
* Added [ytdl-sub](https://ytdl-sub.readthedocs.io/en/latest/) to the `compose.yaml`. Remove if unwanted.

## Media Server
Media Servers have their own guides! Check the link below and it will take you to the folder for the guides.

- [Jellyfin](https://github.com/TechHutTV/homelab/tree/main/media/jellyfin)
- [Plex](https://github.com/TechHutTV/homelab/tree/main/media/plex)

## Data Directory
### Folder Mapping
It's good practice to give all containers the same access to the same root directory or share. This is why all containers in the compose file have the bind volume mount `/data:/data`. It makes everything easier, plus passing in two volumes such as the commonly suggested `/tv`, `/movies`, and `/downloads` makes them look like two different file systems, even if they are a single file system outside the container. See my current setup below.
```
data
├── books
├── downloads
│   ├── qbittorrent
│   │   ├── completed
│   │   ├── incomplete
│   │   └── torrents
│   └── nzbget
│       ├── completed
│       ├── intermediate
│       ├── nzb
│       ├── queue
│       └── tmp
├── movies
├── music
├── shows
└── youtube
```
Here is a easy command to create the download directory scheme. Run within the `/data` directory.
```bash
mkdir -p downloads/qbittorrent/{completed,incomplete,torrents} && mkdir -p downloads/nzbget/{completed,intermediate,nzb,queue,tmp}
```

### Network Share
I generally install Docker on the same LXC that I have my media server on as well as all my data. This, however, is [not recommended by Proxmox](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_pct). Going forward you should create a separate VM for all your docker containers and mount the data directory we created in the [storage guide](https://github.com/TechHutTV/homelab/tree/main/storage) with the share. You can also use this method if you're using a separate share on another machine running something like Unraid or TrueNAS.

Within the VM install `cifs-utils`
```bash
sudo apt install cifs-utils
```
Now, edit the `fstab` file and add the following lines editing them to match your information:
```bash
sudo nano /etc/fstab
```
```
//10.0.0.100/data /data cifs uid=1000,gid=1000,username=user,password=password,iocharset=utf8 0 0
```
Storing the user credentials within this file isn't the best idea. Check out [this question](https://unix.stackexchange.com/questions/178187/how-to-edit-etc-fstab-properly-for-network-drive) on Stack Exchange to learn more.

Now reload the configuration and mount the shares with the following commands.
```bash
sudo systemctl daemon-reload
sudo mount -a
```

## User Permissions
Using bind mounts (`path/to/config:/config`) may lead to permission conflicts between the host operating system and the container. To avoid this problem, you can specify the user ID (`PUID`) and group ID (`PGID`) to use within some of the containers. This will give your user permissions to read and write configuration files, etc.

In the compose file I use `PUID=1000` and `PGID=1000`, as those are generally the default IDs in most Linux systems, but depending on your setup you may need to change this.

```bash
id your_user
```
This command will return something like the following:
```
uid=1000(your_user) gid=1000(your_user) groups=1000(your_user),27(sudo),24(cdrom),30(dip),46(plugdev),108(lxd)
```
If you are using a network share mounted though `/etc/fstab` match the permissions there. Learn more above.

If you run into errors after creating all the folders you can assign the permissions using `chown`. For example:
```bash
sudo chown -R 1000:1000 /data
```
Also, I like to store all my Docker configurations in a root `/docker` directory on my Linux system. These can go wherever you prefer whether that be your home directory or somewhere else. Do note, many Docker apps may have issues if you're trying to store you Docker configurations in a SMB network share.
```bash
mkdir /docker
sudo chown -R 1000:1000 /docker
```
## Docker Compose and .env
Navigate to the directory you want to spin up the servarr stack in. I run mine from `/docker/servarr` but you can run it from anywhere you'd like such as `/home/user/docker/servarr`. Then download the `compose.yaml` and `.env` files from this repo.
```bash
wget https://github.com/TechHutTV/homelab/raw/refs/heads/main/media/compose.yaml && wget https://github.com/TechHutTV/homelab/raw/refs/heads/main/media/.env
```
Most of our editing is going to be done in the `.env` file. Here you change your `UID` and `GID`, timezone, and can make edits to the `compose.yaml` file such as the mount point locations, for example, if you are using something other than `/data:/data` or even changing the docker network IP addresses for your services.

## Download Clients

### NZBGet

#### NZBGet Login Credentials 
The default credentials for NZBGet are a username of `nzbget` and a password of `tegbzn6789`. It's strongly recommended to change these default credentials for security reasons. This can be done under _Settings > SECURITY_, then change the ControlUsername and ControlPassword.

#### NZBGet Download Directories
If following the `/data:/data` directory scheme and used the command to setup the download directories open the qBittorent Web UI and do under _Settings > PATHS_ and change the paths.

_MainDir:_ `/data/downloads/nzbget`

_DestDir:_ `${MainDir}/completed`

_InterDir:_ `${MainDir}/intermediate`

And keep everything else as is.

#### Fix directory does not appear to exist inside the container error
This error may appear within Sonarr and Radarr. Once NZBGet is setup go to settings and under **INCOMING NZBS** change the **AppendCategoryDir** to **No**. This will prevent some potential mapping issues and save on unnecessary directories.

### qBittorrent

#### qBittorrent Login Credentials
When you first launch qBittorrent it will generate a random password. To find this password you can view the logs to see what the password is.
```bash
docker container logs qbittorrent
```
Now, go to your settings and setup a new username and password under _WebUI > Authentication_.

#### Qbittorrent Download Directories
If following the `/data:/data` directory scheme and used the command to setup the download directories open the qBittorent Web UI and do under _Settings > Downloads_ and change the paths.

_Default Save Path:_ `/data/downloads/qbittorrent/completed`

_Keep incomplete torrents in:_ `/data/downloads/qbittorrent/incomplete`

_Copy .torrent files to:_ `/data/downloads/qbittorrent/torrents`


## *arr Apps

When connecting your *arr applications be sure to use the new configured IP addresses in the `servarrnetwork`. We will soon update this section with more text documentation.
