# JupyterHub-Kubernetes
Resources on how to set up Jupyterhub on a Kubernetes cluster

## Zero To Jupyterhub
This site contains a guide on how to set up Kubernetes on the popular platforms such as Amazon, Google Cloud, Azure etc as well as on how to configure a Jupyterhub instance.

Site: https://zero-to-jupyterhub.readthedocs.io/en/latest/  
Github: https://github.com/jupyterhub/zero-to-jupyterhub-k8s  
Gitter: https://gitter.im/jupyterhub/jupyterhub

Most Basic Jupyterhub Setup:
Assuming that your Kubernetes cluster is running, a Jupyterhub instance can be deployed using a single YAML configuration file.
The Zero To Jupyterhub guide will provide instructions on how to create the basic YAML file but the most basic configuration will look something like this:
>config.yaml

    proxy:
      secretToken: "<RANDOM_HEX>"

The RANDOM_HEX value can be generated using the command:
`openssl rand -hex 32`

Please follow the guide on Zero To Jupyterhub on how to create the most basic instance.  
https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub.html

## Authentication
Kubernetes and Jupyterhub provide many ways to authenticate users using methods such as LDAP, OpenID Connect, OAuth2 etc. The full list can be found here:  
https://zero-to-jupyterhub.readthedocs.io/en/latest/authentication.html

### No Authentication
If you have tried the most basic Jupyterhub Setup using the guide at the beginning, you will have noticed that there is no set username or password. This means that you can enter any values for the username and\
 password and a container will be spawned with a Notebook for you. This method is useful for testing the notebooks and trying custom notebooks.

### OpenID Connect
OpenID Connect is an identity layer on top of the OAuth 2.0 protocol.
Keycloak can be authenticated against to provide access to the cluster.

### LDAP and Active Directory
JupyterHub supports LDAP and Active Directory authentication.

## Making Custom Notebooks
Jupyterhub will allow us to use any Docker container which contains the Jupyter Notebook packages. We can therefore create our own Docker container recipes (Dockerfiles) which we can then build and push to Dockerhub or a private Docker registry.

### Dockerhub
Dockerhub is a large library for container images which can be accessed freely. Some users have created their own Jupyter Notebook container images either from scratch (with their own preferred packages also installed) or they have extended the official Jupyter Notebook images by adding packages to another Dockerfile.

#### Using a Container Image from Dockerhub
To use a container image that is already on Dockerhub, we can either add it to the `config.yaml` file which will download it only when it is needed and cache it or we can download it ourselves and have it cached for us. If we allow it to be downloaded only when it is needed, the first user will have to wait longer for their container/Notebook to start. 

To have it download only when needed, the following needs to be added to the `config.yaml`

    singleuser:
      image:
        # Get the latest image tag at:
        # https://hub.docker.com/r/jupyter/datascience-notebook/tags/
        # Inspect the Dockerfile at:
        # https://github.com/jupyter/docker-stacks/tree/master/datascience-notebook/Dockerfile
         name: jupyter/datascience-notebook
         tag: 177037d09156
This will download the official Data Science Notebook with a specific version(tag). Every user will then be given the Data Science Notebook when they spawn a Notebook.

To pre-download and cache the container image for future use, each note in the cluster will need to run the following command:

`docker pull jupyter/datascience-notebook`

Many container images on Dockerhub also have a link to Github where we can look at the Dockerfile to see what packages are installed. We can then either extend their Dockerfile or copy it and remove some packages.

#### Uploading Custom Image to Dockerhub or Private Repository

If you have a custom image that you want to use for your Notebooks, they must be built uploaded to either a private repository or Dockerhub before they can be used.  

Assuming that you already have a `Dockerfile` for the image, the image can be built and pushed to Dockerhub with the following commands:  
`docker build -f /path/to/Dockerfile -t <tag> .`

To push the new image to Dockerhub, use the command:
`docker push yourhubusername/yourimage`

### Github
#### Docker Stacks
Jupyter has released some container images with all of the Notebook software installed with various software packages. These can either be downloaded directly from Dockerhub or you can clone the repository from Github and either build/modify/extend them yourself.  

Link: https://github.com/jupyter/docker-stacks

Other users also have their own repositories with recipes which can also be modified or extended.  

#### Extending a Notebook
To extend an already built image, e.g the DataScience Notebook, we need to create our own Dockerfile.
To install the packages silx and pyFAI, we can create a Dockerfile that looks something like this:

`Dockerfile`  

    FROM jupyter/datascience-notebook
    USER $NB_UID
    RUN conda install pyfai silx -c conda-forge

This will make sure the user is note root and install the silx and pyFAI packages after the DataScience packages. 

To add environment variables such as a proxy, we can do:

`Dockerfile`  

    FROM jupyter/datascience-notebook
    USER $NB_UID
    ENV http_proxy=http://proxy.example.com:3128


## Adding multiple Notebooks to Spawner
As shown earlier, the user can have a default Notebook which will be spawned as soon as they authenticate on JupyterHub. This Notebook will be the same for everyone with the same packages and resources e.g CPU, RAM etc.

There is also the option to allow the user to choose which kind of Notebook they want when they authenticate. It could be from a Notebook with the Data Science packages installed or it could contain the Tensorflow packages. We are also able to allocate the number of CPUs and RAM given to each type of Notebook which will suit many different purposes.

This option can be set in the `config.yaml` like so:

    hub:
      extraConfig: |
        c.CustomKubeSpawner.profile_list = [
        {
            'display_name': 'Jupyter scipy - Official',
            'default': True,
            'kubespawner_override': {
                'image': 'jupyter/scipy-notebook:latest',
                'cpu_limit': 1,
                'mem_limit': '2G',
             }
        },
        {                                                                                                   
            'display_name': 'SILX Notebook - Python',                                                                                                                                       
            'kubespawner_override': {                                                                       
                'image': 'aicampbell/jupyter-notebook-silx:latest',                                         
                'cpu_limit': 2,                                                                             
                'mem_limit': '2G',                                                                          
            }                                                                                               
         }            
    ]

A checkbox will appear with the container images allowing the user to choose which type of Notebook they want which will then be spawned. It is also possible to add other resources such as GPU's to a profile but this involves enabling them on the Kubernetes cluster.


## Mounting additional directories to Notebook
One of the goals of using the Notebooks was to allow users to access files in their home directory at each institute. This requires each institute to create volumes and mount them to a container.

This will depend on the setup of each institute but in this example NFS will be used.
 
### NFS
Network File System (NFS) is a distributed file system protocol allowing a user on a client computer to access files over a computer network much like local storage is accessed.

Many popular Kubernetes providers such as Amazon and Google support NFS but it is more difficult to set up ourselves. For this reason, we are able to access NFS by mounting the NFS file systems onto each Kubernetes cluster node and allow the container to access them from the host. This can be done like so:

     c.KubeSpawner.volumes = [
        {
            'name': homedir,
            'hostPath': {
                'path': 'path/to/homes/{username}'
            }
        }
    ]
    c.KubeSpawner.volume_mounts = [
        {
            'name': 'homedir'),
            'mountPath': '/home/joyvan/home'
        }
]

In the c.KubeSpawner.volumes list, the `{username}` will be replaced with the username that was authenticated. Therefore, if all of the users' home directories are in organised such as  
`/users/username1`  
`/users/username2`  
this will be very easy to mount to each container.

#### UID and GID

### User Home Directory

#### Assuming Every User is on the Same Path
E.g /users/<username>

#### REST API
How to get the data from the REST API and mount using HostPath

### Experimental Data
Still in progress - Same method for home directories should work

## Sample Configuration

## Current problems
UID does not exist in the container (I have no name) - Can't run as sudo as can't access NFS

