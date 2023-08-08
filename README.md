# Running the GPU driver in a container with Oracle Linux 7 with Nvidia GPU Operator

Kubernetes provides access to special hardware resources such as NVIDIA GPUs, NICs, RoCE/Infiniband adapters and other devices through the device plugin framework. However, configuring and managing nodes with these hardware resources requires configuration of multiple software components such as drivers, container runtimes or other libraries which are difficult and prone to errors. The NVIDIA GPU Operator uses the operator framework within Kubernetes to automate the management of all NVIDIA software components needed to provision GPU. These components include the NVIDIA drivers (to enable CUDA), Kubernetes device plugin for GPUs, the NVIDIA Container Toolkit, automatic node labelling using GFD, DCGM based monitoring and others.

Nvidia GPU operator is deployed with a Helm chart that allows you to attach pods to the right GPUs and handle them as resources. You can use the GPU Operator with pre-installed GPU drivers on the host or deploy the GPU drivers on the cluster using a container. This allows you to switch Cuda and GPU driver versions quickly by redeploying the GPU Operator driver container with a different version (as long as the combination is supported by Nvidia).

You will need:

- A node with Docker installed
- A Docker repository to push your image to (OCIR, Docker Hub, etc.)


### Clone the Nvidia driver repository

```
git clone https://gitlab.com/nvidia/container-images/driver.git

cd driver/centos7
```

### Save the following content as ol7_latest.repo in driver/centos7 directory

```
[ol7_latest]
name=Oracle Linux $releasever Latest ($basearch)
baseurl=https://yum.oracle.com/repo/OracleLinux/OL7/latest/$basearch/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
gpgcheck=1
enabled=1
```

### Create the OL7 RPM key in driver/centos7 directory
```
curl -s https://yum.oracle.com/RPM-GPG-KEY-oracle-ol7 -o RPM-GPG-KEY-oracle
```

So the final content of driver/centos7 directory will be

```
-rw-rw-r-- 1 opc opc  3106 Aug  4 18:11 Dockerfile
-rw-rw-r-- 1 opc opc     0 Aug  4 17:28 empty
-rwxrwxr-x 1 opc opc  1839 Aug  4 17:28 install.sh
-rwxrwxr-x 1 opc opc 21100 Aug  4 17:28 nvidia-driver
-rw-rw-r-- 1 opc opc   203 Aug  8 20:37 ol7_latest.repo
-rw-rw-r-- 1 opc opc   210 Aug  4 17:28 README.md
-rw-rw-r-- 1 opc opc  1011 Aug  8 20:37 RPM-GPG-KEY-oracle
```

### Edit the Dockerfile in driver/centos7 directory
Open the Dockerfile and add the following lines after `ADD install.sh /tmp/`

```
ADD ol7_latest.repo /etc/yum.repos.d/
ADD RPM-GPG-KEY-oracle /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
```

### Build and push the custom image
Make sure you choose a compatible GPU driver and CUDA pair. More info [here.](https://docs.nvidia.com/deploy/cuda-compatibility/)


```
DRIVER_VERSION=
CUDA_VERSION=

docker build . -t <yourrepository name>/driver:$DRIVER_VERSION-ol7.9 --build-arg DRIVER_VERSION=$DRIVER_VERSION --build-arg CUDA_VERSION=$CUDA_VERSION --build-arg TARGETARCH=amd64
```

Example:

```
DRIVER_VERSION=510.85.02
CUDA_VERSION=11.7.1

docker build . -t oguzpastirmaci/driver:$DRIVER_VERSION-ol7.9 --build-arg DRIVER_VERSION=$DRIVER_VERSION --build-arg CUDA_VERSION=$CUDA_VERSION --build-arg TARGETARCH=amd64
```

Then push the image to your image repository.

Example:

```
docker push oguzpastirmaci/driver:510.85.02-ol7.9
```


## Testing your image

The instructions are here for non-RDMA workloads only.

Deploy an OKE cluster for testing your image. You can use the template [here.](https://github.com/OguzPastirmaci/misc/blob/master/oke/terraform/non-rdma.tf)

Once your cluster is up and running, follow the steps below.

### Get the latest Helm 3 version
```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Add Helm repos for Network Operator and GPU Operator
```sh
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```


### Deploy GPU Operator with the image you built
In your OKE cluster, run the following command to deploy the GPU Operator with the image you built. 

```
helm install --wait \
  -n gpu-operator --create-namespace \
  gpu-operator nvidia/gpu-operator \
  --version v23.3.2 \
  --set operator.defaultRuntime=crio \
  --set driver.repository=<The repository that you pushed your image> \
  --set driver.version=<The driver version in your pushed image. Only the version, don't add ol7.9 at the end> \
  --set toolkit.version=v1.13.5-centos7
```

Example:

```
helm install --wait \
  -n gpu-operator --create-namespace \
  gpu-operator nvidia/gpu-operator \
  --version v23.3.2 \
  --set operator.defaultRuntime=crio \
  --set driver.repository=oguzpastirmaci \
  --set driver.version=510.85.02 \
  --set toolkit.version=v1.13.5-centos7
```