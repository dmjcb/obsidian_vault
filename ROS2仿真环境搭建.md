## 一、安装基础包

```sh
sudo apt update

sudo apt install -y ubuntu-drivers-common linux-headers-$(uname -r)
```

## 二、安装NVIDIA驱动

```sh
sudo ubuntu-drivers install
```

## 三、安装docker

```sh
sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin
```

## 四、NVIDIA Container Toolkit

```sh
sudo apt-get update

sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg2
    
    
curl -fsSL \
https://nvidia.github.io/libnvidia-container/gpgkey \
| sudo gpg --dearmor \
-o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L \
https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
| sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
| sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```


```sh
sudo apt update

sudo apt install -y nvidia-container-toolkit
```

把 NVIDIA Runtime 接入 Docker

```sh
sudo nvidia-ctk runtime configure --runtime=docker
```

```sh
sudo systemctl restart docker
```