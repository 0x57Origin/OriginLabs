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
# Alpine Commands 
```
apk update
apk add openssh openrc
```
Now Alpine handles server service a tad bit different than Debian. So we will need openssh which is the SERVER & CLIENT. openrc is Alpine's service manager. 
---
# I'm going to add the host keys and accounts so I can make it act like a real server
```
ssh-keygen -A
adduser -D randomUserName
echo 'randomUserName:RandomPassword123!' | chpasswd
echo 'root:Root123!' | chpasswd
```
Host keys? They are unique cryptographic keys that the SSH server uses to identify itself and secure the initial connection to clients.

ssh-keygen -A will create 4 pair host keys (One for each major encryption) -> RSA, ECDSA, ED25519, and DSA  and also it will not break your working keys or mess with keys that already exists on the server / system. If your server is missing an RSA key but already has an ED25519 key, it will only create the missing RSA key and leave the ED25519 key completely alone.

adduser -D randomUserName - > -D stand for do not assign a password, it will create the account and right away lock it so no-one can log in.

echo 'randomUserName:randomUserPassword' | chpasswd -> chpasswd stands for Change Password. 

echo 'root:123' | chpasswd -> Chaining the root users password.

<img width="590" height="162" alt="image" src="https://github.com/user-attachments/assets/ca8e8421-4e86-4745-81f3-34e7fefe530e" />

---
## Now we get to break the server!!
```
Turn root login on and start the SSH server:
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
/usr/sbin/sshd

Explanation:
sed - stream editor. Alpine used sed to find and replace something in a file.
-i - in place. Edit and save it right away without printing it on the screen.
s - substitue.
^#*PermitRootLogin.* - What to find.
PermitRootLogin yes -  What to replace it with.

Breakdown of regex:
The / is just a divider. It separates the three parts of the substitute command:
s / find / replace /

^ = start of line
#* =  zero or more # characters (the line might be commented out as #PermitRootLogin, or not).
PermitRootLogin = The actual text
.* = anything else at the end of the line (like no, or prohibit-password)

```










