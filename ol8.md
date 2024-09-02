## Building the CUDA and GPU driver container images with Oracle Linux 8

### Deploy an Oracle Linux 8.8 instance and install Docker

Deploy an Oracle Linux 8.8 instance and install Docker. You can use the default UEK kernel. You don't need a GPU instance, a regular E3/E4 will work. Increase the boot volume to 150+ GB and have 4+ cores.

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

### Use the Rocky Linux 8 Dockerfile as base
We will use the Rocky Linux 8 Dockerfile as our base and make changes.


```
CUDA_VERSION=

cp -R cuda/dist/$CUDA_VERSION/rockylinux8/base cuda/dist/$CUDA_VERSION/oraclelinux8
cp cuda/NGC-DL-CONTAINER-LICENSE cuda/dist/$CUDA_VERSION/oraclelinux8

sed -i 's/rockylinux/oraclelinux/g' cuda/dist/$CUDA_VERSION/oraclelinux8/Dockerfile

```

### Build the CUDA container images
The below command will build 3 images (base, devel, runtime). The builds will take some time to finish.

```
cd cuda/dist/$CUDA_VERSION/oraclelinux8
docker build . -t <your repository>/cuda:$CUDA_VERSION-base-oraclelinux8 --build-arg TARGETARCH=amd64
```

Example:

```
cd cuda/dist/$CUDA_VERSION/oraclelinux8
docker build . -t oguzpastirmaci/cuda:$CUDA_VERSION-base-oraclelinux8 --build-arg TARGETARCH=amd64
```

### Push the image to your registry
Push the image you built in your previous step to your registry with `docker push <your repository>/cuda:12.0.1-base-oraclelinux8`. 

Example:

```
docker push oguzpastirmaci/cuda:12.0.1-base-oraclelinux8
```

### Building the GPU driver image
Once you built and pushed the CUDA images to your repo, download the files in the [ol8 directory](./files/ol8) to a directory in your image builder machine.

IMPORTANT: Make sure that you change the first line in the Dockerfile to match your repository and CUDA version.

`FROM <your repository>/cuda:<CUDA version you built>-base-oraclelinux8`

```
mkdir -p ~/driver && cd ~/driver

curl -s https://raw.githubusercontent.com/OguzPastirmaci/oke-oracle-linux-gpu-operator/main/files/ol8/Dockerfile -o Dockerfile

curl -s https://raw.githubusercontent.com/OguzPastirmaci/oke-oracle-linux-gpu-operator/main/files/ol8/nvidia-driver -o nvidia-driver

curl -s https://raw.githubusercontent.com/OguzPastirmaci/oke-oracle-linux-gpu-operator/main/files/ol8/empty -o empty

sudo chmod +x nvidia-driver
```

Then build the GPU driver image with the below command and push it to your registry:

```
DRIVER_VERSION=
CUDA_VERSION=

docker build . -t <your repository>/driver:$DRIVER_VERSION-ol8.8 --build-arg DRIVER_VERSION=$DRIVER_VERSION --build-arg CUDA_VERSION=$CUDA_VERSION --build-arg TARGETARCH=amd64

docker push <your repository>/driver:$DRIVER_VERSION-ol8.8
```

Example:

```
DRIVER_VERSION=535.54.03
CUDA_VERSION=12.0.1

docker build . -t oguzpastirmaci/driver:$DRIVER_VERSION-ol8.8 --build-arg DRIVER_VERSION=$DRIVER_VERSION --build-arg CUDA_VERSION=$CUDA_VERSION --build-arg TARGETARCH=amd64

docker push oguzpastirmaci/driver:$DRIVER_VERSION-ol8.8
```

Finally, when the GPU driver image is available in your registry, you can deploy it with the GPU Operator

```
helm install --wait \
  -n gpu-operator --create-namespace \
  gpu-operator nvidia/gpu-operator \
  --version v24.1.1 \
  --set operator.defaultRuntime=crio \
  --set driver.repository=<The repository that you pushed your image> \
  --set driver.version=<The driver version in your pushed image. Only the version, don't add ol8.8 at the end> \
  --set toolkit.version=v1.14.5-ubi8
```

Example:

```
helm install --wait \
  -n gpu-operator --create-namespace \
  gpu-operator nvidia/gpu-operator \
  --version v24.6.1 \
  --set operator.defaultRuntime=crio \
  --set driver.repository=oguzpastirmaci \
  --set driver.version=535.54.03 \
  --set toolkit.version=v1.14.5-ubi8
```



