# How to record and archive Twitch Livestreams using StreamLink and Google Workspace.
## Introduction
In this guide, we'll explain how to record any Twitch live stream in real-time and automatically upload to the Cloud (Google Workspace) using your Debian server. It allows you to be able to save live streams from your favorite streamers without having to worry about deleted VODs or muted VODs due to DMCA. Google Workspace is commonly used for business and enterprise applications but it's perfect for uploading and storing large files.

## Prerequisites
- You will need a Debian Linux VPS/Server with root or regular, non-root user with sudo privileges configured on your server.
- You will need a Google Workspace account with an active subscription (Business Standard, Business Plus or Enterprise).


### **Step 1 - Setup Google Workspace**

1. Setup Google Apps/G-Suite Account:
   - Login to [Google Developer Console](https://console.developers.google.com) using your Google Workspace Credentials.
   - Create a New Project


  - *Enable Google Drive API:*
     - Main Menu on Top Left > APIs and Services > Enable    APIs and Services > Search for Google Drive API > Enable


   - *Create a New Service Account for Rclone:*
     - Main Menu > IAM & Admin > Service Accounts > Create Service Account > Fill in desired name > Create and Continue > Done
     - **Make sure to copy the OAuth 2 Client ID, it will be used later on.**


   - *Export your Google Account JSON Key:*
     - Actions > Manage keys > Create New Keys > Select JSON > Save the Key.

2. Allowing API access to Google Drive:
  - Login to [Google Workspace Admin Console](https://admin.google.com)
  - Go to Security > Scroll down to API Controls > Manage Domain wide Delegation > Add new API Client:
    - Paste your OAuth 2 Client ID into Client ID Field
    - Add the following to OAuth scopes
       `https://www.googleapis.com/auth/drive`
  - Click on Authorize

### **Step 2 - Install Docker**

  In order to install Docker, you need to setup the official Docker Repository.

  1. Update the `apt` package repository:
  ```
  sudo apt updates
  ```
  2. Install packages to allow `apt` to use a repository over HTTPS:
  ```
  sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
  ```
  3. Add Docker's GPG key:
  ```
  curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  ```
  4. Add the stable repository:
  ```
  echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  ```
  5. Update the `apt` repository and install the *latest* Docker packages:
  ```
  sudo apt-get update
  sudo apt-get install docker-ce docker-ce-cli containerd.io
  ```

### **Step 3 - Setup Rclone**
We will use Rclone to mount Google Drive as Local Storage.

1. Upload your JSON file to your server.

    - Windows: Use [WinSCP](https://winscp.net)

    - Linux:
    ```
    scp *.json username@destination_ip:/
    ```

2. Create a Default Directory for VODs and Docker
```
sudo mkdir -p /data/docker /data/VODs
```
3. Install Rclone & Mount Dependencies:
```
sudo apt install rclone fuse
```
4. Configure Rclone:
```
rclone config
```
  - Enter `n` to create new configuration
  - Enter `gdrive` for name
  - Select the number that corresponds to Google Drive.
  - Press *Enter* for both `client_id` & `client_secret`
  - Enter `1` for scope
  - Press *Enter* for `root_folder_id`
  - Enter `/*.json` for `service_account_file`


5. Mount Google Drive as local filesystem
```
rclone mount --daemon --vfs-cache-mode full --drive-impersonate user@domain.com gdrive:data /data
```
  **Replace `user@domain.com` with your Google Workspace Email Address**


### **Step 4 - Setup Docker**
We will use Docker to monitor Twitch and save them to mounted filesystem.

1. Setup Docker to use Google Workspace as Default Directory
  - Stop Docker service
```
sudo systemctl stop docker
```
  - Install nano text editor
```
sudo apt install nano
```
  - Use nano to create/edit default Docker JSON file 
```
sudo nano /etc/docker/daemon.json
```
  - Copy the following into `/etc/docker/daemon.json`

```
{
   "data-root": "/data/docker"
}
```

  *Use `CTRL-X` to exit, `Y` to save, `ENTER` to write changes*


  - Start Docker service
```
sudo systemctl start docker
```
### **Step 5 - Create Docker Containers**
1. Create Docker Containers for each Twitch Stream
```
docker create --name TwitchUsername --restart unless-stopped -v /data/VODs/TwitchUsernameVOD:/home/download -e streamLink='twitch.tv/TwitchUsername' -e streamQuality='best' -e streamName='TwitchUsername' -e streamOptions='--twitch-disable-hosting --twitch-disable-ads' -e uid='0' -e gid='0' lauwarm/streamlink-recorder
```
2. Run Docker Containers
```
docker start TwitchUsername
```
  **Replace `TwitchUsername` with your desired Twitch username**

  *If you want to record multiple streams, repeat [Step 5](https://github.com/concpt/twitcharchiver#step-5---create-docker-containers).*
