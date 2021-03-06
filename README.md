# Rocket - App Container runtime

_Release early, release often: Rocket is currently a prototype and we are seeking your feedback via pull requests_

Rocket is a cli for running App Containers. The goal of rocket is to be composable, secure, and fast. 

[Read more about Rocket in the launch announcement](https://coreos.com/blog/rocket). 

![Rocket Logo](logos/rocket-horizontal-color.png)

## Trying out Rocket

`rkt` is currently supported on amd64 Linux. We recommend booting up a fresh virtual machine to test out rocket. 

To install the `rkt` binary:

```
curl -L https://github.com/coreos/rocket/releases/download/v0.1.0/rocket-v0.1.0.tar.gz -o rocket-v0.1.0.tar.gz
tar xzvf rocket-v0.1.0.tar.gz
cd rocket-v0.1.0
./rkt help
```

Keep in mind while running through the examples that rkt needs to be run as root for most operations.

## Rocket basics

### Downloading an App Container Image (ACI)

Rocket uses content addressable storage (CAS) for storing an ACI on disk. In this example, the image is downloaded and added to the CAS. 

```
$ rkt fetch https://github.com/coreos/etcd/releases/download/v0.5.0-alpha.4/etcd-v0.5.0-alpha.4-linux-amd64.aci
sha256-701c24b2d275f0e291b807a464ae2390bcd8d7c5b4f2d7e47e6fd917cd5e5588
```

These files are now written to disk:

```
$ find /var/lib/rkt/cas/blob/
/var/lib/rkt/cas/blob/
/var/lib/rkt/cas/blob/sha256
/var/lib/rkt/cas/blob/sha256/70
/var/lib/rkt/cas/blob/sha256/70/sha256-701c24b2d275f0e291b807a464ae2390bcd8d7c5b4f2d7e47e6fd917cd5e5588
```

Per the App Container [spec](app-container/SPEC.md#image-archives) the sha256 is of the tarball, which is reproducable with other tools:

```
$ wget https://github.com/coreos/etcd/releases/download/v0.5.0-alpha.4/etcd-v0.5.0-alpha.4-linux-amd64.aci
...
$ gzip -dc etcd-v0.5.0-alpha.4-linux-amd64.aci
$ sha256sum etcd-v0.5.0-alpha.4-linux-amd64.tar
701c24b2d275f0e291b807a464ae2390bcd8d7c5b4f2d7e47e6fd917cd5e5588  etcd-v0.5.0-alpha.4-linux-amd64.tar
```

### Launching an ACI

To run an ACI, you can either use the sha256 hash, or the URL which you downloaded it from:

```
$ rkt run https://github.com/coreos/etcd/releases/download/v0.5.0-alpha.4/etcd-v0.5.0-alpha.4-linux-amd64.aci
Press ^] three times to kill container.
```

rkt will do the appropriate etag checking on the URL to make sure it has the most up to date version of the image.

Or, you can explicitly choose an image to run based on the sha256:

```
$ rkt run sha256-701c24b2d275f0e291b807a464ae2390bcd8d7c5b4f2d7e47e6fd917cd5e5588
Press ^] three times to kill container.
```

These commands are interchangeable.


## App Container basics

[App Container](app-container) is a [specification](app-container/SPEC.md) of an image format, runtime, and discovery protocol for running a container. We anticipate app container to be adopted by other runtimes outside of Rocket itself. Read more about it [here](app-container).

To validate the `rkt` with the App Container [validation ACIs](app-container/README.md) run:

```
$ rkt run -volume database:/tmp \
	https://github.com/coreos/rocket/releases/download/v0.1.0/ace-validator-main.aci \
	https://github.com/coreos/rocket/releases/download/v0.1.0/ace-validator-sidekick.aci
```

## Rocket internals

Rocket is designed to be modular and pluggable by default. To do this we have a concept of "stages" of execution of the container. 

Execution with Rocket is divided into a number of distinct stages. The
motivation for this is to separate the concerns of initial filesystem setup,
execution environment, and finally the execution of the apps themselves.

### Stage 0

The first step of the process, stage 0, is the actual `rkt` binary itself. This
binary is in charge of doing a number of initial preparatory tasks:
- Generating a Container UUID
- Generating a Container Runtime Manifest
- Creating a filesystem for the container
- Setting up stage 1 and stage 2 directories in the filesystem
- Copying the stage1 binary into the container filesystem
- Fetching the specified ACIs
- Unpacking the ACIs and copying each app into the stage2 directories

Given a run command such as:

```
rkt run --volume bind:/opt/tenant1/database \
	sha256-8a30f14877cd8065939e3912542a17d1a5fd9b4c \
	sha256-abcd29837d89389s9d0898ds908ds890df890908 
```

a container manifest compliant with the ACE spec will be generated, and the
filesystem created by stage0 should be:

```
/container
/stage1
/stage1/init
/stage1/opt
/stage1/opt/stage2/sha256-8a30f14877cd8065939e3912542a17d1a5fd9b4c
/stage1/opt/stage2/sha256-abcd29837d89389s9d0898ds908ds890df890908
```

where:
- `container` is the container manifest file
- `stage1` is a copy of the stage1 filesystem that is safe for read/write
- `stage1/init` is the actual stage1 binary to be executed
- `stage1/opt/stage2` are copies of the RAFs

At this point the stage0 execs `/stage1/init` with the current working
directory set to the root of the new filesystem.

### Stage 1

The next stage is a binary that the user trusts to set up cgroups, execute
processes, and other operations as root. This stage has the responsibility to
take the execution group filesystem that was created by stage 0 and create the
necessary cgroups, namespaces and mounts to launch the execution group:

- Generate systemd unit files from the Application and Container Manifests
  (containing, respectively, the exec specifications of each container and the
  ordering given by the user)
- Set up any external volumes (undefined at this point)
- nspawn attaching to the bridge and launch the execution group systemd
- Launch the root systemd
- Have the root systemd

This process is slightly different for the qemu-kvm stage1 but a similar
workflow starting at `exec()`'ing kvm instead of an nspawn.

### Stage 2

The final stage is executing the actual application. The responsibilities of
the stage2 include:

- Launch the init process described in the Application Manifest
