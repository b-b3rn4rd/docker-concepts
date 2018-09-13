Docker concepts beyond the basics (WIP)
==================

Motivation
---------------

*The following page is in working progress and being updated with additional information* 

I've been using Docker and ECS for almost two years with the main focus on learning
ECS and ECR features. However, I always felt uncomfortable with not fully understanding
the Docker concepts above the basics, so I decided to take a step back and make a research that will give answers to the following questions:

* What *actually* Docker is and it's architecture?
* How Docker images, layers and containers working together?
* What is `--privileged` mode and rootless containers ?
* Docker networking and DNS (TBA)

### What actually Docker is and it's architecture?

Essentially Docker is a platform that plumbs individual small libraries. Each library is responsible for performing necessary tasks in the most reliable and simple way possible.
Since, the version 1.11 Docker has the following architecture: 
![docker-layers](http://img.scoop.it/kZ7lPEDwkIFnoq4nqnhtRbnTzqrqzN7Y9aBZTaXoQ8Q=)

##### Docker Engine
Docker Engine is a client-server application with REST API that specifies interfaces for interacting with the daemon, and a command line interface (CLI) client that talks to the daemon using `docker <command>` commands.

Daemon manages things like volumes, networking, scaling and authentication, it interacts with containerd to run containers using containerd's gRPC interface [7]
![docker-engine](https://i1.wp.com/blog.docker.com/wp-content/uploads/chart-g.png?w=820&ssl=1)
##### Containerd
Containerd component - is a daemon for Linux or Windows with gRPC interface, it is responsible for managing the container lifecycle, it performs
such tasks as image push & push, container execution and supervision. 

The main concept in Containerd's architecture is a bundle. A bundle contains container's configuration metadata and rootfs. Containerd
provides API to extract/pack bundles from the disk and it's execution.

Containerd is using RunC to run container, however, it's compatible with any container runner that is built according to the OCI specification. [10]

![containerd](https://containerd.io/img/chart-a.png)

Containerd comes with a client library built on top of the api for testing and debugging purposes. The following code
gives a good example how containerd operates to pull an image and start the container [12]:
```golang
package main

import (
	"context"
	"fmt"
	"log"
	"syscall"
	"time"

	"github.com/containerd/containerd"
	"github.com/containerd/containerd/cio"
	"github.com/containerd/containerd/oci"
	"github.com/containerd/containerd/namespaces"
)

func main() {
	if err := redisExample(); err != nil {
		log.Fatal(err)
	}
}

func redisExample() error {
	// create a new client connected to the default socket path for containerd
	client, err := containerd.New("/run/containerd/containerd.sock")
	if err != nil {
		return err
	}
	defer client.Close()

	// create a new context with an "example" namespace
	ctx := namespaces.WithNamespace(context.Background(), "example")

	// pull the redis image from DockerHub
	image, err := client.Pull(ctx, "docker.io/library/redis:alpine", containerd.WithPullUnpack)
	if err != nil {
		return err
	}

	// create a container
	container, err := client.NewContainer(
		ctx,
		"redis-server",
		containerd.WithImage(image),
		containerd.WithNewSnapshot("redis-server-snapshot", image),
		containerd.WithNewSpec(oci.WithImageConfig(image)),
	)
	if err != nil {
		return err
	}
	defer container.Delete(ctx, containerd.WithSnapshotCleanup)

	// create a task from the container
	task, err := container.NewTask(ctx, cio.NewCreator(cio.WithStdio))
	if err != nil {
		return err
	}
	defer task.Delete(ctx)

	// make sure we wait before calling start
	exitStatusC, err := task.Wait(ctx)
	if err != nil {
		fmt.Println(err)
	}

	// call start on the task to execute the redis server
	if err := task.Start(ctx); err != nil {
		return err
	}

	// sleep for a lil bit to see the logs
	time.Sleep(3 * time.Second)

	// kill the process and get the exit status
	if err := task.Kill(ctx, syscall.SIGTERM); err != nil {
		return err
	}

	// wait for the process to fully exit and print out the exit status

	status := <-exitStatusC
	code, _, err := status.Result()
	if err != nil {
		return err
	}
	fmt.Printf("redis-server exited with status: %d\n", code)

	return nil
}
```

*It's worth mentioning that networking is out of the scope for containerd and must be handled and provided to containerd via higher level systems.*

##### RunC
Runc is responsible for creating and running containers according to the OCI specification. It creates containers with namespaces, cgroups, capabilities, and filesystem access controls.
To create a container RunC requires a bundle which consists of rootfs and a `config.json` configuration file for setting up a running container.
In the Docker architecture a bundle is created by containerd. [6]
```bash
# create the top most bundle directory
mkdir /mycontainer
cd /mycontainer

# create the rootfs directory
mkdir rootfs

# export busybox via Docker into the rootfs directory
docker export $(docker create busybox) | tar -C rootfs -xvf -

#  generate a spec in the format of a config.json file inside your bundle
runc spec

# run as root
runc run mycontainerid
```


### How Docker images, layers and containers working together ?

A Docker image consists of layers where each layer represents an instruction from image’s Dockerfile. Each layer in the image is read-only and contains a set of differences from the layer before it.

When you create a new container, you add a new writable layer on top of the underlying layers. This layer is often called the “container layer”. All changes made to the running container, such as writing new files, modifying existing files, and deleting files, are written to this writable container layer. 

When an existing file in a container is modified, the storage driver performs a *copy-on-write* (CoW) operation. 

The CoW operation for the current *overlay2* driver performs the following steps:
1) Search for the file to update through layers starting from the container layer
2) Perform `copy_up` operation to copy found file to the container layer
3) Changes made to a file at the container layer are invisible to the image layers

![layers](https://docs.docker.com/v17.09/engine/userguide/storagedriver/images/container-layers.jpg)

When the container is deleted, the writable layer is also deleted. The underlying image remains unchanged. [1]

The current Docker driver - `overlay2` stores layers in folders on a Linux host, it uses directories: 
* `lowerdir` to store image layers 
* `upperdir` to store container writable layer. 

The result of merging files in those two directories is stored in directory called `merged` which becomes the containers mount point.
When a file exists in both container and image layers, file container layer takes the precedence. [2]

![overlay](https://docs.docker.com/storage/storagedriver/images/overlay_constructs.jpg)



### What is `--privileged` mode and rootless containers ?

By default, Docker runs containers with a limited set of Linux [capabilities](https://linux.die.net/man/7/capabilities) and without access to host devices, 
 when the `--privileged` option is specified a container will run with [all available capabilities](https://github.com/moby/moby/blob/8e610b2b55bfd1bfa9436ab110d311f5e8a74dcb/daemon/exec_linux.go#L27) 
and access to all devices on the host. Privileged flag also disables [AppArmor](https://en.wikipedia.org/wiki/AppArmor) and [Seccomp](https://en.wikipedia.org/wiki/Seccomp) security profiles by setting them to `unconfined`

Device access is controlled by [cgroups](https://www.kernel.org/doc/Documentation/cgroup-v1/devices.txt) and gives container access
to specific block storage device or loop device or audio device.

Linux capabilities determine what a process/container is allowed to do, by default capabilities bounding set is inherited at fork from the thread's parent.



Starting from the version 1.10 Docker provides support for user namespaces [14]. User namespace is the mechanism for remapping UIDs inside a container.
User namespaces allow to re-map users to a less-privileged user on the Docker host while having root access in the container.
Since the mapped user has no privileges on the host machine itself it helps to prevent privilege-escalation attacks from within a container.

Simple configuration example, for more examples, see [following article](https://docs.docker.com/engine/security/userns-remap)

```bash
# run docker container
$ docker run -it -d ubuntu
  Unable to find image 'ubuntu:latest' locally
  latest: Pulling from library/ubuntu
  124c757242f8: Pull complete 
  9d866f8bde2a: Pull complete 
  fa3f2f277e67: Pull complete 
  398d32b153e8: Pull complete 
  afde35469481: Pull complete 
  Digest: sha256:de774a3145f7ca4f0bd144c7d4ffb2931e06634f11529653b23eba85aef8e378
  Status: Downloaded newer image for ubuntu:latest
  54bbb44dc7b1a151385b4537d861825b0bc590624521e0d7190204c7daf78ec8

# show container's process information, notice the process is running under root
$ docker top 54bbb44dc7b1a15
  UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
  root                4371                4354                0                   00:52               pts/0               00:00:00            /bin/bash

# enable user re-mapping, use default dockremap user
$ sudo vim /etc/docker/daemon.json
  {
    "userns-remap": "default"
  }
$ sudo service docker start  

# check that docker has created the default remapping user
$ id dockremap
  uid=112(dockremap) gid=116(dockremap) groups=116(dockremap)

$ grep dockremap /etc/subuid
  dockremap:231072:65536
  
# run docker container
$ docker run -it -d ubuntu
  Unable to find image 'ubuntu:latest' locally
  latest: Pulling from library/ubuntu
  124c757242f8: Pull complete 
  9d866f8bde2a: Pull complete 
  fa3f2f277e67: Pull complete 
  398d32b153e8: Pull complete 
  afde35469481: Pull complete 
  Digest: sha256:de774a3145f7ca4f0bd144c7d4ffb2931e06634f11529653b23eba85aef8e378
  Status: Downloaded newer image for ubuntu:latest
  1b3c562331df4d8a2389f0caaa1ebc4577289674a3634bc1254440cfc6fc9531
  
# check that docker container's process is running as remapped user
$ docker top 1b3c562331df4d8a2389f
  UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
  231072              5226                5209                0                   01:18               pts/0               00:00:00            /bin/bash
```



Links
-----------

[1] https://docs.docker.com/v17.09/engine/userguide/storagedriver/imagesandcontainers/
[2] https://docs.docker.com/storage/storagedriver/overlayfs-driver/#how-the-overlay-driver-works
[3] https://www.slideshare.net/jpetazzo/anatomy-of-a-container-namespaces-cgroups-some-filesystem-magic-linuxcon
[4] http://jancorg.github.io/blog/2015/01/03/libcontainer-overview/
[5] https://blog.docker.com/2015/06/runc/
[6] https://blog.docker.com/2016/04/docker-engine-1-11-runc/
[7] https://medium.com/tiffanyfay/docker-1-11-et-plus-engine-is-now-built-on-runc-and-containerd-a6d06d7e80ef
[8] https://medium.com/@jessgreb01/digging-into-docker-layers-c22f948ed612
[9] https://www.docker.com/resources/what-container
[10] https://blog.docker.com/2016/04/docker-containerd-integration/
[11] https://github.com/containerd/containerd/blob/master/docs/getting-started.md
[12] https://github.com/opencontainers/runc/tree/master/libcontainer
[13] https://github.com/moby/moby/blob/7129bebe0a93455668e8b320ca835f02220e120c/daemon/oci_linux.go#L87
[14] https://docs.docker.com/engine/security/userns-remap/
[15] https://success.docker.com/article/introduction-to-user-namespaces-in-docker-engine
[16] https://docs.docker.com/engine/security/userns-remap/#user-namespace-known-limitations