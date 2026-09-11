# I am currently locking down SSH (Secure Shell) on Linux box - Turn off root login & lock account after 3rd bad password, then prove that it worked!
---
## I am going to use Kali for this but I'm not really going to mess with my KALI settings so Docker it is.
---
## Docker Installation 
```
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker
```
Then in kali terminal put **systemctl status docker** and it will give you the systemd service logs and you will see **Active: active (running)** in green coloring.
systemd is the core system & service manager for Linux OS. systemd -> System daemon 
---
## Alpine OS - Just a few MB download. 
```
sudo docker run -dit --name VulBuild alpine
sudo docker exec -it arcbuild sh
```
First Command -> It will download the alpine image .iso and create a container called VulBuild and start it and leave it running. -d is the operator that let you run it in the background.

Second Command -> Basically go into the VulBuild container that is already running and open a shell.
---
# What will it look like
```
┌──(kali㉿kali)-[~/Desktop/Docker]
└─$ sudo docker run -dit --name VulBuild alpine
[sudo] password for kali: 
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
55afa1ecc21d: Pull complete 
56dceff11b33: Download complete 
f5124fb579e2: Download complete 
Digest: sha256:28bd5fe8b56d1bd048e5babf5b10710ebe0bae67db86916198a6eec434943f8b
Status: Downloaded newer image for alpine:latest
e7a56772d35ab964551832a7d3fc6efaf486e6674b9857d27b4e6fb188e08860
                                                                                                                                                                                                                                           
┌──(kali㉿kali)-[~/Desktop/Docker]
└─$ sudo docker exec -it VulBuild sh           
/ # whoami
root
/ # 
```
