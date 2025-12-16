# setup
```
sudo dnf install -y dnf-utils
```
```
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

```
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
```
sudo systemctl --now enable docker
```
- เพิ่ม user ใช้ งาน docker
```
sudo usermod -aG docker (user)
```

- เพิ่มเสร็จ logout -> login