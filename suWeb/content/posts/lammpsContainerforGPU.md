---
title: "Build a lammps container and run it anywhere"
date: 2022-06-16T22:39:22+05:30
# weight: 1
# aliases: ["/first"]
tags: ["HPC", "LAMMPS", "Container"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Build a docker container with the packages you need and use it on a multitude of device ranging from a personal computer to a supercomputer with multiple GPU's"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
cover:
    image: "https://i.ibb.co/K0HVPBd/paper-mod-profilemode.png" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    responsiveImages: false
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/aximthered/website/suWeb/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

## Introduction

What is a **Container**?
> Linux Containers is an operating-system-level virtualization method for running multiple isolated Linux systems on a control host using a single Linux kernel.

For us this means that LAMMPS and its dependencies are neatly packaged inside an isolated space where only the bare minimum of the reqirements to run LAMMPS are present. Some major advantages of containerized applications are that we won't have to worry about clash of dependency and ease of sharing our software. 

There are two main platforms for containers; Singularity and Docker. Singularity is used mainly is HPC systems for multiple users while docker is used by users with root access.

In this post I will explain how to modified a NVIDIA NGC LAMMPS docker container-recipy to build a custom version of LAMMPS container with extra packages that utilizes KOKKOS and multi-platform support. 

## Prerequisites

1. WSL2 with docker windows or docker on linux system
2. Docker-Hub account
3. Google colab (optional)
2. Internet connection

## Getting the Recipy and Modifying it

what is **[NVIDIA NGC](https://catalog.ngc.nvidia.com/)**?
> Deploy performance-optimized AI/HPC software containers, pre-trained AI models, and Jupyter Notebooks that accelerate AI developments and HPC workloads on any GPU-powered on-prem, cloud, and edge systems.

The NVIDIA NGC LAMMPS container comes prepackaged with USER-REAXC, KSPACE, MOLECULE, REPLICA, RIGID, MISC, MANYBODY, ASPHERE, KOKKOS, OPENMP, MPI, REAXFF, DPD-BASIC, and ML-SNAP packages. To add any extra packages and make your own version of the docker container, first acquire the recipy file used to build the LAMMPS NGC container by following the below steps.

The LAMMPS containers can be found here -> **[link](https://catalog.ngc.nvidia.com/orgs/hpc/containers/lammps)**. The container build with the specific LAMMPS versions can be found in the Tags tab. Use the three dots to copy the pull tag and paste it in terminal. 

Next explore the docker image using

```bash
docker run --rm -it --entrypoint=/bin/bash docker:nvcr.io/hpc/lammps:patch_4May2022
# or
singularity build --sandbox lammpsElectrode.simg docker:nvcr.io/hpc/lammps:patch_4May2022
```
next `cd /usr/src` inside the container, where you will find the recipy.py and Dockerfile files. Copy the contents of the recipy.py file into a new file on your system. The recipy.py is the script that is used to generate the Dockerfile and Singularity definition file using HPCCM. 

Next we need to edit the recipy file. The edits depend on the extra packages you want to add. In my case I needed to install the ELECTRODE package which depends also on the KSPACE package, BLAS, LAPACK and libgfortran. 

Inside the recipy.py file make the following edits after line 65 to install libgfortran, BLAS and LAPACK (may vary depending on the package you need):
```Bash
Stage0 += apt_get(ospackages=["bzip2", "libhwloc-dev",
                  "openssh-client", "perl", "tar", "flex"])
Stage0 += apt_get(ospackages=["libgfortran5", "libblas-dev", "liblapack-dev"])
Stage0 += multi_ofed(
    inbox=False, mlnx_versions=["5.1-2.5.8.0", "5.2-2.2.0.0", "5.3-1.0.0.1"], prefix="/usr/local/ofed", symlink=False
)
```
Next make the changes to the generic_cmake function parameters. There are five generic_cmake functions in the recipy file, each for five GPU architectures ranging from sm60 to sm86. Make the following changes to each of the generic_cmake functions.  
Add the command `"-DPKG_ELECTRODE=yes",` to enable ELECTRODE package and turn off any that you don't require.  
Add  `-lblas -llapack` to the -DCMAKE_CXX_FLAGS_RELEASE in all the five generic_cmake functions, to link the BLAS and LAPACK library, as follows:
`"-DCMAKE_CXX_FLAGS_RELEASE='-march=haswell -mtune=haswell -O3 -pipe -DNDEBUG -lblas -llapack'",` 

and remove the line 502 (I got an error here, you can try without removing by making sure the Dockerfile and recipy.py files are present while building later):
```Bash
Stage1 += environment(variables={"UCX_MEMTYPE_CACHE": "n"})
# Stage1 += copy(_from="", src=["./Dockerfile", "./recipe.py"], dest="/usr/src/")
Stage1 += workdir(directory="/host_pwd")
```
Now we are ready to build our Dockerfile. 

To build the Dockerfile we use the HPC container maker software (HPCCM). Its easy to install on a google colab notebook (or on your own system).
```Bash
!pip install hpccm
# Upload the recipy.py file to the filesystem on colab
!hpccm --recipe /content/recipe.py --format docker > Dockerfile
```
This is the Dockerfile containing instructions on how to build our LAMMPS container.

Note for HPC users: The reason why I dont use singularity definition file using `!hpccm --recipe /content/recipe.py --format singularity --singularity-version=3.8.3 > lammps4may.def` is that; 1. It is not usually possible to build singularity images on HPC systems without root access and 2. The remote build option of singularity is limited by 1 hour on sylab. So the alternative is to build the docker image and upload it to Docker-hub. Now the docker image pulled to the HPC system and converted to singularity image easily.

## Build the Docker container and Push it onto DockerHub

Place the Dockerfile and recipy.py in a new folder and use the following command to build the image:

```
docker build -t Docker-Hub_Username/lammps:a_tag_name - < Dockerfile
```

Next login to the Docker-Hub 
```
docker login
```
and push the image to Docker-Hub:
```
docker push Docker-Hub_Username/lammps:a_tag_name
```

## Pull it to your prefered system

We can pull and run the image on any system using the following commands
For Docker:
```
docker pull Docker-Hub_Username/lammps:a_tag_name

docker run --rm --gpus all --ipc=host -v $PWD:/host_pwd -w /host_pwd Docker-Hub_Username/lammps:a_tag_name ./run_lammps.sh
```
For Singularity:
```
singularity pull docker:Docker-Hub_Username/lammps:a_tag_name

singularity run --nv -B $PWD:/host_pwd --pwd /host_pwd docker:Docker-Hub_Username/lammps:a_tag_name ./run_lammps.sh
```
## Final Notes

