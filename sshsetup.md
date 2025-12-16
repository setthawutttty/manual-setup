# ssh
```
sudo dnf install -y openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo systemctl status sshd
```

```
sudo firewall-cmd --add-service=ssh --permanent
sudo firewall-cmd --reload
```

เช็ค ip 
- ip a

