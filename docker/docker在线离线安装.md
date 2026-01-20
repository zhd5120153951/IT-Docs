# Ubuntu 22.04 上在线安装 Docker+NVIDIA GPU支持的步骤
1. 更新系统软件包:
```bash
sudo apt update && sudo apt upgrade -y
```
2. 安装依赖工具:
```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release software-properties-common -y
```
3. 添加 Docker 官方 GPG 密钥:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
4. 安装 Docker
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
docker version
```
5. 安装 NVIDIA Container Toolkit
```bash
添加 NVIDIA 仓库
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

安装工具包
sudo apt update
sudo apt install -y nvidia-container-toolkit

配置并重启 Docker
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

# Ubuntu 24.04 上在线安装 Docker+NVIDIA GPU支持的步骤

**区别于Ubuntu22.04**

1. 更新系统软件包:
```bash
sudo apt update && sudo apt upgrade -y
```
2. 安装依赖工具:
```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg2 lsb-release software-properties-common
```
3. 添加 Docker 官方 GPG 密钥:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
4. 安装 Docker
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
docker version
```
5. 安装 NVIDIA Container Toolkit
```bash
添加 NVIDIA 仓库
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

安装工具包
sudo apt update
sudo apt install -y nvidia-container-toolkit

配置并重启 Docker
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```