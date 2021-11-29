<!-- markdownlint-disable MD013 -->
# Welcome

Great that you found your way here!
In this guide, you will find the prerequisites for the Kubernetes course.
It is important that you have completed them before the course so any troubles can be found and resolved on beforehand.

Do not feel intimidated if you do not understand exactly what each command here does.
The steps in this guide are just to make sure everything is setup properly so you can follow along during the course! :)

## Install WSL2

If you are running Windows, you should begin by installing WSL2.
Officially we only support Linux and Windows with WSL2.
Other setups can also work fine, e.g. running a VM with Linux or using PowerShell instead, but you are on your own with any problems that may arise.

The official installation instructions for WSL2 are found here:
<https://learn.microsoft.com/en-us/windows/wsl/install>

## Install dependencies

### Install Docker

1. Install Docker as described in the official documentation: <https://docs.docker.com/engine/install/ubuntu/>
2. Start the Docker service.

> ℹ️ For WSL2, see the FAQ at the end of this document on how to start the Docker service!

### Install Podman

For Ubunut 20.10 or later, just follow the official guide: <https://podman.io/getting-started/installation#ubuntu>

> ℹ️ For Ubuntu 20.04, follow [this guide for Podman](https://www.atlantic.net/dedicated-server-hosting/how-to-install-and-use-podman-on-ubuntu-20-04/) instead! See FAQ at the end of this document!

### Install Helm

Helm: <https://helm.sh/docs/intro/install/>

### Install `kubectl`

`kubectl`: <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/>

### Install `k3d`

You will be using `k3d` to setup a local Kubernetes cluster on your machine.
`k3d` is a lightweight wrapper around the minimal Kubernetes distribution called `k3s`.

Installation instructions are found here:
<https://k3d.io/#installation>

## Setup a Kubernetes cluster

```console
k3d cluster create --api-port 127.0.0.1:6443 --image rancher/k3s:v1.25.4-k3s1 primer --registry-create myregistry:127.0.0.1:1234

# If `KUBECONFIG` is set in your environment, you will also need to run this:
k3d kubeconfig merge primer
export KUBECONFIG="~/.k3d/kubeconfig-primer.yaml:${KUBECONFIG}"
```

## Check Kubernetes access

Assuming the previous steps were successful, you should now be able to connect to the Kubernetes cluster using `kubectl`:

```console
$ kubectl config use-context k3d-primer
Switched to context "k3d-primer".
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:43731
CoreDNS is running at https://127.0.0.1:43731/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:43731/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Also check that the Kubernetes version of your client matches the one on the server:

```console
$ kubectl version --short
Flag --short has been deprecated, and will be removed in the future. The --short output will become the default.
Client Version: v1.25.3
Kustomize Version: v4.5.7
Server Version: v1.25.3+k3s1
```

## Check container registry access

Now we will verify that you have access to the local container registry.

> 💡 Since we use a local registry with no valid TLS certificate, we must run `podman push` and `pull` commands with the `--tls-verify=false` flag.

```console
$ podman pull hello-world
Trying to pull docker.io/library/hello-world:latest...
Getting image source signatures
Copying blob 2db29710123e skipped: already exists
Copying config feb5d9fea6 done
Writing manifest to image destination
Storing signatures
feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412

$ # Make sure that `1234` match the port number used when creating the cluster and registry earlier.
$ podman tag hello-world "localhost:1234/hello-world:1"
$ podman push --tls-verify=false "localhost:1234/hello-world:1"
Getting image source signatures
Copying blob e07ee1baac5f done
Copying config feb5d9fea6 done
Writing manifest to image destination
Storing signatures
$ podman pull --tls-verify=false "localhost:1234/hello-world:1"
Trying to pull localhost:1234/hello-world:1...
Getting image source signatures
Copying blob 44f2b372045f skipped: already exists
Copying config feb5d9fea6 done
Writing manifest to image destination
Storing signatures
feb5d9fea6a5e9606aa995e879d862b825965ba48de054caab5ef356dc6b3412
```

## FAQ

### 1. Problems installing Podman on Ubuntu 20.04

By default, WSL2 comes with Ubuntu 20.04.
In releases before Ubuntu 20.10, Podman does not exist in the apt repositories, so we need some extra steps to install it properly.
The following steps are copied from
[Atlantic.net](https://www.atlantic.net/dedicated-server-hosting/how-to-install-and-use-podman-on-ubuntu-20-04/)
and have been confirmed to work on Ubuntu 20.04.

```shell
sudo apt-get update -y
sudo apt-get install curl wget gnupg2 -y
source /etc/os-release
sudo sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /' > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/xUbuntu_${VERSION_ID}/Release.key -O- | sudo apt-key add -
sudo apt-get update -qq -y
sudo apt-get -qq --yes install podman
```

Now verify the installation:

```console
$ podman --version
podman version 3.4.2
```

### 2. WARN[0000] "/" is not a shared mount, this could cause issues or missing mounts with rootless containers

If you get a warning saying `WARN[0000] "/" is not a shared mount, this could cause issues or missing mounts with rootless containers`, you can fix that by running `mount --make-rshared /`.
To do that automatically on every boot of WSL2, put the following in your `/etc/wsl.conf` file:

```ini
[boot]
command="mount --make-rshared /"
```

This only works for Windows 11.
For Windows 10 you must run it manually.

### 3. DNS problems on WSL2

Microsoft have implemented a strategy for DNS resolving in WSL2 which does not play well with containers.
It creates a custom `/etc/resolv.conf`, which is pointing to the Windows host machine's DNS server.
That is reachable from WSL2, but not from containers within WSL2.
And since both Docker and Podman will mount the (Linux) host's `resolv.conf` into each container, that breaks things.

In order to fix this, you should deactivate the automatic `resolv.conf` generation by WSL2 and manually create a good `resolv.conf`.

1. Ensure that the following is set in `/etc/wsl.conf`:

    ```ini
    [network]
    generateResolvConf = false
    ```

2. Restart WSL2 by running `wsl --shutdown` in PowerShell.

3. Edit `/etc/resolv.conf` and make sure there's some good Internet DNS servers there, e.g.:

    ```resolv
    # Cloudflare's DNS service 1.1.1.1:
    nameserver 1.1.1.1
    nameserver 1.0.0.1
    ```

### 4. Docker service is not running

The Docker service will not start automatically, since WSL2 normally runs without systemd.
To start it, you should run `sudo service docker start`.

You can do this automatically by putting the following in your `~/.bashrc`:

```shell
# Check that the docker client is installed
if command -v docker &> /dev/null; then
    # Check if docker service exists
    if service --status-all |& grep -qE ' docker$'; then
        # Start the docker service unless it's already running
        if ! service docker status > /dev/null ; then
            echo "Service docker not running!"
            echo "service docker start ..."
            # sudo service docker start
        fi
    else
        echo "* service docker is missing"
    fi
fi
```

### 5. `podman ps` fails with error

If `podman ps` fails with the following error,

```console
$ podman ps
ERRO[0000] error joining network namespace for container 4b81fc8eb08c41058718d6bf114db58eb13d79aa9fda8567da15ca4a62bdda81: error retrieving network namespace at /tmp/podman-run-1000/netns/cni-cd3363a5-fe49-d84a-d8e7-ac1c1586cb02: unknown FS magic on "/tmp/podman-run-1000/netns/cni-cd3363a5-fe49-d84a-d8e7-ac1c1586cb02": ef53
```

then you probably have the "shared mount" problem described above.
