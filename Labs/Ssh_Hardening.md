# I am currently locking down SSH (Secure Shell) on Linux box - Turn off root login & lock account after 3rd bad password, then prove that it worked!
---
## I am going to use Kali for this but I'm not really going to mess with my KALI settings so Docker it is.
---
## Docker Installation 
```
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker
```
Then in kali terminal put **systemctl status docker** and it will give you the systemmd service logs and you will see **Active: active (running)** in green coloring.

