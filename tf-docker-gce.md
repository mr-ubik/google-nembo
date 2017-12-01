# TensorFlow + GPU + Keras + Jupyter + Docker + Google Cloud = <3 

# TOC

# CAVEAT DATA *

TensorFlow is all the rage right now, it is the hottest deep learning framework on the market and there doesn't seem to be any signs of it slowing down its growth. Deploying the CPU version is quite straightforward, the GPU version instead is quite the annoying brat.

In order to minimize the chances of everything going wrong it is recommended you keep in mind the following requirements.

## Linux Distribution

Most of the resources out there use Ubuntu as the core Linux distribution
both for the Docker host and the Docker image. Usually, for reasons we will explore in the next point the recommended version is the 16.04.



## Cuda Drivers

Tensorflow supports all recent CUDA drivers' (not to be confounded with `cuda-toolkit`) versions.

## cuda-toolkit

The current version of TensorFlow (1.4.0 as of 30/11/2017) when installed using the precompiled binaries requires `cuda-toolkit 8.0`, Nvidia website lists only Ubuntu 16.04 and 16.10 as supported Ubuntu versions. Overall the 16.04 is the best of the two due to better stability and support since it is a LTS version. Note also that P100 GPUs require the 9.0 version thus making it useful only if we are comfortable about building Tensorflow from source.


# Configuring the host

## Creating the machine on Google Cloud Compute Engine

The following command will instantiate a
- 4 cores
- 26 Gb Ram
- OS:Ubuntu 16.04 
- 100Gb SSD for boot disk and most importantly access to a Nvidia K80
	
<!-- --service-account "698553121915-compute@developer.gserviceaccount.com" \ -->
```bash
#!/bin/bash
gcloud beta compute --project "{PROJECT}" instances create "dl-lab-gpu" \
	--zone "europe-west1-b" --machine-type "n1-highmem-4" \
	--network "default" --no-restart-on-failure \
	--maintenance-policy "TERMINATE" \
	--scopes "https://www.googleapis.com/auth/cloud-platform" \
	--accelerator type=nvidia-tesla-k80,count=1 \
	--min-cpu-platform "Automatic" --image "ubuntu-1604-xenial-v20171121a" \
	--image-project "ubuntu-os-cloud" --boot-disk-size "100" \
	--boot-disk-type "pd-ssd" --boot-disk-device-name "instance-1"
```

## Installing the prerequisites package

```bash
apt-get update -y && apt-get upgrade -y
apt-get install \
	build-essential \
    curl \
    wget \
    git \
    gcc \
    htop \
    libcupti-dev \
    apt-transport-https \
    ca-certificates \
    software-properties-common
```

## Install drivers

```bash
#!/bin/bash
curl -O http://www.nvidia.com/content/DriverDownload-March2009/confirmation.php?url=/tesla/384.66/nvidia-driver-local-repo-ubuntu1604-384.66_1.0-1_ppc64el.deb&lang=us&type=Tesla
```

## Install cuda-toolkit v8.0  

Version 9.0 as said it is not supported with the binary TensorFlow (which is the one we are going to use).
We are going to download the network version of the deb [from this page](https://developer.nvidia.com/cuda-80-ga2-download-archive).


```bash
#!/bin/bash
echo "Checking for CUDA and installing."
# Check for CUDA and try to install.
if ! dpkg-query -W cuda; then
  # The 16.04 installer works with 16.10.
  curl -O https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_8.0.61-1_amd64.deb
  dpkg -i cuda-repo-ubuntu1604_8.0.61-1_amd64.deb
  apt-get update -y
  apt-get install cuda -y
fi
```

We can test the installation with `nvidia-smi`.

## Install cuDNN
[This is the cuDNN page on the Nvidia website.](https://developer.nvidia.com/cudnn)

```bash
#!/bin/bash
echo "Checking for cuDNN and installing."
# Check for CUDA and try to install.
if ! dpkg-query -W cudnn; then
  # The 16.04 installer works with 16.10.
  curl -O https://developer.nvidia.com/compute/machine-learning/cudnn/secure/v7.0.4/prod/8.0_20171031/Ubuntu16_04-x64/libcudnn7_7.0.4.31-1+cuda8.0_amd64
  dpkg -i libcudnn7_7.0.4.31-1+cuda8.0_amd64.deb
  apt-get update
  apt-get install cudnn
fi
```

## Install nvidia-docker


### Install docker

```bash
#!/bin/bash
echo "Adding Docker’s official GPG key"
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - 
ehco "Set up the Docker stable repository."
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   
sudo apt-get -y update
sudo apt-get -y install docker-ce
```

### Install nvidia-docker

```bash
#!/bin/bash
# If you have nvidia-docker 1.0 installed: we need to remove it and all existing GPU containers
docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
sudo apt-get purge -y nvidia-docker

# Add the package repositories
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/ubuntu16.04/amd64/nvidia-docker.list | \
sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update

# Install nvidia-docker2 and reload the Docker daemon configuration
sudo apt-get install -y nvidia-docker2
sudo pkill -SIGHUP dockerd

# Test nvidia-smi with the latest official CUDA image
docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
```

### Add user to docker groups 

Always having to type `sudo` before `docker` or `nvidia-docker` is a pain. We can easily bypass this need by adding users to the docker group.

```bash
#!/bin/bash
sudo groupadd docker
sudo gpasswd -a $USER docker
newgrp docker
```