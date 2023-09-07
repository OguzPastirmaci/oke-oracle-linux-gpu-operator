## Building the CUDA images with Oracle Linux 8

### Deploy an Oracle Linux 8 instance and install Docker

Deploy an Oracle Linux 8 instance and install Docker. You can use the default UEK kernel. You don't need a GPU instance, a regular E3/E4 will work. Increase the boot volume to 150+ GB and have 4+ cores.

Tested with [Oracle-Linux-8.8-2023.08.16-0](https://docs.oracle.com/en-us/iaas/images/image/7afc0d76-6d2d-4060-ba3b-34fb8c0080a4/).

```
# Install Docker
sudo dnf install -y dnf-utils zip unzip
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

sudo dnf remove -y runc
sudo dnf install -y docker-ce --nobest

sudo systemctl enable docker.service
sudo systemctl start docker.service

sudo usermod -aG docker $USER
```

### Clone the CUDA container image repository

```
sudo dnf install -y git

git clone https://gitlab.com/nvidia/container-images/cuda.git
```

### Use the Rocky Linux 8 Dockerfiles as base
We will use the Rocky Linux 8 Dockerfiles as our base and make changes.


```
CUDA_VERSION=11.7.1

cp -R cuda/dist/$CUDA_VERSION/rockylinux8 cuda/dist/$CUDA_VERSION/oraclelinux8

sed -i 's/rockylinux/oraclelinux/g' cuda/dist/$CUDA_VERSION/oraclelinux8/base/Dockerfile
sed -i 's/rockylinux/oraclelinux/g' cuda/dist/$CUDA_VERSION/oraclelinux8/devel/Dockerfile
sed -i 's/rockylinux/oraclelinux/g' cuda/dist/$CUDA_VERSION/oraclelinux8/runtime/Dockerfile
```

### Build the CUDA container images
The below command will build 3 images (base, devel, runtime). The builds will take some time to finish.

```
cd cuda
./build.sh -d --image-name <my-remote-container-registry>/cuda --cuda-version $CUDA_VERSION --os oraclelinux --os-version 8 --arch x86_64 --push
```
