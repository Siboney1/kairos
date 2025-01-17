+++
title = "Installation"
date = 2022-02-09T17:56:26+01:00
weight = 1
chapter = false
+++

## Introduction

The Kairos releases ships for user convenience a set of artifacts that can be used to install kairos on a node. 
However, Kairos Kubernetes Native components allows the creation of these artifacts inside Kubernetes from a set of input images. 

In the quickstart below we will use the artifacts generated by the kairos releases as we assume we don't have any prior Kubernetes cluster available.

Most of the operations are completely automated, so there is little interaction needed with the Kubernetes components.

Installation can be also completely driven by the Kubernetes Native components if you have already a Kubernetes cluster to manage installations, with zero touch configuration. See the [CRD quickstart (WIP)]().

The goal of this quickstart is to install kairos on a node and create a single-node Kubernetes cluster with [k3s](https://k3s.io).

To simplify, we will install kairos inside a VM, however, your mileage and configuration may vary based on your setup, see the documentation for a more exhaustive list of examples.


### Prerequisites

- A VM hypervisor that boots ISOs
- A Linux machine where to run the kairos-cli (optional, we will see)
- A configuration file (cloud-init)

### Download

Kairos can be used to turn any distro in an immutable system, however, for user convenience there are several artifacts published as part of the releases to get started.

You can find the latest releases in the [release page over Github](https://github.com/kairos-io/provider-kairos/releases). For instance, if we would like to pick the alpine based version, we would download the `kairos-alpine-v0.57.0-k3sv1.21.14+k3s1.iso` ISO file, where `v1.21.14+k3s1` in the name is the `k3s` version and `v0.57.0` is the kairos one.

{{% notice note %}}
The releases in the [kairos-io/kairos](https://github.com/kairos-io/kairos/releases) repository are the `kairos` core images that ship without `k3s` and `p2p` full-mesh functionalities, however further extensions can be installed dynamically in runtime by using the [kairos bundles]() mechanism.

The releases in [kairos-io/provider-kairos](https://github.com/kairos-io/provider-kairos/releases) instead ships `k3s` and `p2p` full-mesh support that needs to be explictly enabled. In follow-up releases there will be available also _k3s-only_ artifacts.

{{% /notice %}}

### Booting up

Download the ISO, and boot it up in to the hypervisor of your choice. If it's a baremetal, you can flash the image directly into an usb stick with dd:

```bash
dd if=/path/to/iso of=/path/to/dev bs=4MB
```

or just use [Etcher](https://www.balena.io/etcher/).

You should be greeted with a GRUB boot menu:
There are several entries, depending on how you plan to install kairos. Let's pick the first entry, or wait the default timeout.
The machine will boot, and eventually a QR code will be printed out of the screen:

![livecd](https://user-images.githubusercontent.com/2420543/189219806-29b4deed-b4a1-4704-b558-7a60ae31caf2.gif)

### Configuration

At this stage the machine is waiting for the configuration to continue futher with the installation process. Configuration can be either served via QR code, or manually via logging into the box and starting the installation process with a config file. The config file is a YAML file mixed with `cloud-init` syntax and the config of `kairos` itself.

In this example we will configure the node as a single-node kubernetes cluster, so we enable `k3s`, and we set a default password for the `kairos` user to later access to the box, alongside with some ssh keys:

```yaml
#node-config

stages:
   initramfs:
     - name: "User settings"
       hostname: "hostname.domain.tld"
       authorized_keys:
         kairos:
         - "ssh-rsa AAA..."
         - "github:mudler"
       users:
        kairos:
          passwd: "kairos"
       dns:
        path: /etc/resolv.conf
        nameservers:
        - 8.8.8.8

k3s:
  enabled: true
```

Save the config file as `config.yaml` as we will use it later in the process.

Note:
- The `stages.initramfs` block will configure the `kairos` user (default) with the `kairos` password. Note, the `kairos` user is already configured with sudo.
- `authorized_keys` can be used to add additional keys to the user in order to SSH into
- `hostname` sets the machine hostname
- `dns` sets the DNS for the machine
- `k3s.enabled=true` enables `k3s`. 

{{% notice note %}}

Several configuration can be added at this stage. [See the configuration reference](/reference/configuration) for further reading.

{{% /notice %}}

### Deployment

{{% notice note %}}

Below there are instruction using the `kairos-cli` which works only on Linux. 

To install by logging over SSH into the box see [Manual install](/installation/manual) or [Interactive install](/installation/interactive) for driving the installation manually from the console.

{{% /notice %}}

To trigger the installation process via QR code you need to use the `kairos-cli`, the cli is currently available only for Linux and Windows. It can be downloaded from the releases artifact:

```bash
curl -L https://github.com/kairos-io/provider-kairos/releases/download/v0.57.0/kairos-cli-v0.57.0-Linux-x86_64.tar.gz -o - | tar -xvzf - -C .
```

The CLI allows to register a node with a screenshot, an image, or a token. During pairing the configuration is sent over and the node will continue the installation process.

In a terminal in your desktop/workstation, run:

```
kairos register --reboot --device /dev/sda --config config.yaml
```

Note:
- By default the CLI will take automatically a screenshot to get the QR code. Make sure to get that fit into the screen. Alternatively an image path or a token can be supplied via args (e.g. `kairos register /img/path` or `kairos register <token>`)
- The `--reboot` flag will make the node reboot automatically after the installation is completed
- The `--device` flag will instruct to install kairos in the specified drive, replace `/dev/sda` with your drive. It will take over and overwrite any content, please be cautious.
- The `--config` flag is used to specify the config file to drive the installation

Wait now for few minutes for the config to propagate to the node, the installation should start then and automatically reboot afterward.

### Accessing the node

After boot the node will start and load into the system. When the console is available we should be already able to SSH.

To access to the host, login as `kairos`:

```bash
ssh kairos@IP
```

Note:
- `sudo` is configured for the `kairos` user

You should be greeted with a welcome message:

```
Welcome to kairos!

Refer to https://kairos.io for documentation.
kairos@kairos:~> 
```

At this point, it can take few moments to get the k3s server running, but eventually we should be able to inspect the service and see k3s running, for example with systemd-based flavors:

```
$ sudo systemctl status k3s
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: disabled)
    Drop-In: /etc/systemd/system/k3s.service.d
             └─override.conf
     Active: active (running) since Thu 2022-09-01 12:02:39 CEST; 4 days ago
       Docs: https://k3s.io
   Main PID: 1834 (k3s-server)
      Tasks: 220
```

The `k3s` kubeconfig file is available at `/etc/rancher/k3s/k3s.yaml`. Please refer to the [k3s](https://rancher.com/docs/k3s/latest/en/) documentation.

## See also

There are other ways to install kairos:

- [Automated installation](/installation/automated)
- [Manual login and installation](/installation/manual)
- [Create decentralized clusters](/installation/p2p)
- [Take over installation](/installation/takeover)
- [Raspberry PI](/installation/raspberry)
- [Netboot (TODO)]()
- [CAPI Lifecycle management (TODO)]()

## What's next?

- [Upgrade nodes with Kubernets](/upgrade/kubernetes)
- [Upgrade nodes manually](/upgrade/manual)
- [Immutable architeture](/architecture/immutable)

