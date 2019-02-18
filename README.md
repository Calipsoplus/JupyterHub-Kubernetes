# JupyterHub-Kubernetes
Resources on how to set up Jupyterhub on a Kubernetes cluster

## Zero To Jupyterhub
This site contains a guide on how to set up Kubernetes on the popular platforms such as Amazon, Google Cloud, Azure etc	as well	as on how to configure a Jupyterhub instance.

https://zero-to-jupyterhub.readthedocs.io/en/latest/
Github: https://github.com/jupyterhub/zero-to-jupyterhub-k8s
A Gitter is also provided: https://gitter.im/jupyterhub/jupyterhub

Most Basic Jupyterhub Setup:
Assuming that your Kubernetes cluster is running, a Jupyterhub instance can be deployed using a single YAML configuration file.
The Zero To Jupyterhub guide will provide instructions on how to create the basic YAML file but the most basic configuration will look something like this:
`config.yaml`
`proxy:
   secretToken: "<RANDOM_HEX>"
`
The RANDOM_HEX value can be generated using the command: `openssl rand -hex 32`

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
### Dockerhub
### Github
### Docker Stacks
#### Extending Docker Stacks (Code sample)

## Adding multiple Notebooks to Spawner
Code showing profiles

## Mounting additional directories to Notebook
Need to mount the data to each cluster node.
### NFS
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