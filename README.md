# JupyterHub-Kubernetes
Resources on how to set up Jupyterhub on a Kubernetes cluster

## Zero To Jupyterhub
This site contains a guide on how to set up Kubernetes on the popular platforms such as Amazon, Google Cloud, Azure etc as well as on how to configure a Jupyterhub instance.

The configuration tutorial will still work if Kubernetes has been set up on another provider or on baremetal.

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
If you have tried the most basic Jupyterhub Setup using the guide at the beginning, you will have noticed that there is no set username or password. This means that you can enter any values for the username and password and a container will be spawned with a Notebook for you. This method is useful for testing the notebooks and trying custom notebooks.

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

To pre-download and cache the container image for future use, each node in the cluster will need to run the following command:

`docker pull jupyter/datascience-notebook`

Many container images on Dockerhub also have a link to Github where we can look at the Dockerfile to see what packages are installed. We can then either extend their Dockerfile or copy it and remove some packages.

#### Uploading Custom Image to Dockerhub or Private Repository

If you have a custom image that you want to use for your Notebooks, they must be built uploaded to either a private repository or Dockerhub before they can be used.  

Assuming that you already have a `Dockerfile` for the image, the image can be built and pushed to Dockerhub with the following commands:  
`docker build -f /path/to/Dockerfile -t <tag> .`

To push the new image to Dockerhub, use the command:
`docker push yourhubusername/yourimage`

Be sure to create your own Dockerhub account before you try to push to it.

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

This will make sure the user is not root and install the silx and pyFAI packages after the DataScience packages. 

We can install additional software using `pip`, `conda` as well as `apt-get install` etc. To install something or run a command as root, make sure to change the USER to root before the command.

To add environment variables such as a proxy, we can do:

`Dockerfile`  

    FROM jupyter/datascience-notebook
    USER $NB_UID
    ENV http_proxy=http://proxy.example.com:3128


## Adding Multiple Notebooks to Spawner
As shown earlier, the user can have a default Notebook which will be spawned as soon as they authenticate on JupyterHub. This Notebook will be the same for everyone with the same packages and resources e.g CPU, RAM etc.

There is also the option to allow the user to choose which kind of Notebook they want when they authenticate. It could be from a Notebook with the Data Science packages installed or it could contain the Tensorflow packages. We are also able to allocate the number of CPUs and RAM given to each type of Notebook which will suit many different purposes.

This option can be set in the `config.yaml` like so:

    hub:
      extraConfig: |
        c.KubeSpawner.profile_list = [
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


## Mounting Additional Directories to Notebook
One of the goals of using the Notebooks within Calipsoplus was to allow users to access files in their home directory at each institute. This requires each institute to create volumes and mount them to a container.

This will depend on the setup of each institute but in this example NFS will be used.
 
### NFS
Network File System (NFS) is a distributed file system protocol allowing a user on a client computer to access files over a computer network much like local storage is accessed.

Many popular Kubernetes providers such as Amazon and Google support NFS but it is more difficult to set up ourselves. For this reason, we are able to access NFS by mounting the NFS file systems onto each Kubernetes cluster node and allow the container to access them from the host. This can be done like so:

    hub:
      extraConfig: |
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
                'name': 'homedir',
                'mountPath': '/home/joyvan/home'
            }
        ]

In the c.KubeSpawner.volumes list, the `{username}` will be replaced with the username that was authenticated. Therefore, if all of the users' home directories are in organised such as  
`/users/username1`  
`/users/username2`  
this will be very easy to mount to each container.

#### UID and GID
One of the requirements of NFS is that each directory can only be accessed if a user's UID and GID have permission to enter that directory. Again this can be set in the `config.yaml` so that a user's Notebook will be run with the user's UID and GID.  

By default, every Notebook is run as root but we can change this to the user's providing we have a way to get this information.

To set the UID and GID for a Notebook, add the following to the `config.yaml`:  
    
    hub:
      extraConfig: |
          c.KubeSpawner.uid = <value>
          c.KubeSpawner.gid = <value>


### User Home Directory

#### Every User is Not on the Same Path
If every user is on the same path  e.g /users/<username>, the example above will allow you to mount the home directory.  

In the case that there are multiple paths, it is also possible to create a method which will retrieve the user's home directory path which can then be mounted in the notebook. For this to work, we need to extend the KubeSpawner like this:

    hub:
      extraConfig: |

        from kubespawner import KubeSpawner
        from tornado import gen
        import yaml

        class CustomKubeSpawner(KubeSpawner):
            def get_username(self):
                return self.user.name

            def get_home_dir_path(self, username):
                # code to get the path as a string e.g REST API
                return user_path
             
            @gen.coroutine
            def start(self):
                path = self.get_home_dir_path(self.get_username())
                self.volumes =[
                {
                   'name': 'homedir',
                   'hostPath': {
                        'path': str(path)
                    }
                }
                ]

                self.volume_mounts = [
                {
                    'name': 'homedir',
                    'mountPath': '/home/jovyan/home'
                }
                ]
                return (yield super().start())

        c.JupyterHub.spawner_class = CustomKubeSpawner


### Experimental Data
Still in progress - Same method for home directories should work.  

If the experiment data is on NFS, we are also able to mount the entire NFS volume. Each user will be able to navigate through the directories but they will only see the files and directories that they have access to using their UID and GID. This will also take more time to load as the number and size of the files will be quite large.  

If the user only wants a specific set of data, we should be able to mount this in a similar way to the home directory.

## Sample Configuration
    cull: # Containers last 1 hour
      timeout: 3600
      every: 3600 # Does check every hour if containers need shutdown from inactivity
    hub:
      extraConfig: |
        import time
        import requests
        import json
        import asyncio
        import os

        from kubespawner import KubeSpawner
        from tornado import gen
        import yaml
 
       class CustomKubeSpawner(KubeSpawner):
            def get_username(self):
                return self.user.name

            def get_home_dir_path(self, username):
                # code to get the path as a string e.g REST API
                return user_path

            @gen.coroutine
            def start(self):
                path = self.get_home_dir_path(self.get_username())
                self.volumes =[
                {
                   'name': 'homedir',
                   'hostPath': {
                        'path': str(path)
                    }
                }
                ]

                self.volume_mounts = [
                {
                    'name': 'homedir',
                    'mountPath': '/home/jovyan/home'
                },
                return (yield super().start())

        c.JupyterHub.spawner_class = CustomKubeSpawner
        
        def get_uid(Spawner):
            # code to get the user's uid
            username = Spawner.get_username()
            return uid

        def get_gid(Spawner):
            # code to get the user's gid
            username = Spawner.get_username()
            return gid
    
        c.CustomKubeSpawner.uid = get_uid # callable 
        c.CustomKubeSpawner.gid = get_gid # callable
        c.CustomKubeSpawner.supplemental_gids = [100] # Needed to access some files as Joyvan 1000:1000 is the default

        c.Spawner.args = ['--NotebookApp.allow_origin=*'] # Sometimes needed if CORS is a problem
        c.CustomKubeSpawner.cmd = ["/usr/local/bin/start-singleuser.sh"]

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
      extraEnv:
        OAUTH2_AUTHORIZE_URL: ..../auth
        OAUTH2_TOKEN_URL: .../token
        USERNAME: {username}
    auth:
      type: custom
      custom:
        className: oauthenticator.generic.GenericOAuthenticator
        config: # See OpenID Connect in the Zero to Jupyterhub tutorial
    singleuser:
      storage:
         type: none
    proxy:
      secretToken: "<RANDOM_HEX>"

## Current problems
### Terminal: I have no name@jupyter-username
To mount the NFS files, we had to change the 
UID and GID to the same values as in the institute's file system. Unfortunately, this UID is unlikely to exist in the `/etc/passwd`  
When the user opens the terminal in the Notebook, this will result in `I have no name@jupyter-username`

For now, this seems to be relatively harmless as any files modified will still be modified with the user's UID.   
This could be simply cosmetic but more testing is required.  

If any packages require the user's username, this will not work as the user is not in `/etc/passwd`.  
The temporary solution for this is to set certain environment variables. See below:  

    import getpass
    print getpass.getuser()

The documentation for getpass is as follow:
>getpass.getuser() 
>Return the login name of the user. Availability: Unix, Windows.
>This function checks the environmentvariables LOGNAME, USER, LNAME and USERNAME, in order, and returns the value of the first one which is set to a non-empty string. If none are set, the login name from the password database is returned on systems which support the pwd module, otherwise, an exception is raised.

#### Possible solution 1:  
The Dockerfile allows us to change the default username by adding a username during the startup script.  
Unfortunately, we need to be root to do this. We cannot be root as this will not work as the UID and GID will be different and therefore we cannot access the NFS data.

#### Possible solution 2:  
Making /etc/passwd writable by the user.   
We can add a small script at the beginning of the container which will add the user's UID to `/etc/passwd` and correct this without the user noticing.   
The user could change the `/etc/passwd` values thus giving them any UID they want and could allow them to access files they should not have permission to e.g other users' experimental data.

#### Possible solution 3:
Giving users sudo privileges.  
See Possible solution 2.

### Packages Missing That Should Be Installed During Image Build
Some of the software packages that are installed in the Data Science container image are not accessible. E.g trying to do:  
    
    import seaborn
This will fail as the package is not there.

If this container image is run using Docker, the packages are there.  
If it is run in Kubernetes, the package is not there but it can be installed in the terminal.  
This is likely something to do with the UID and GID permissions but more testing is needed. 


