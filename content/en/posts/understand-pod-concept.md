+++
title = "The smallest schedulable unit in Kubernetes — Pod"
date = 2019-12-01T21:24:59+09:00
description = ""
toc = true
tags = [
    "container",
    "kubernetes",
]
categories = [
    "kubernetes",
    "container",
]
image = "fast-lane.jpg"
+++

# Overview
As I have mentioned in the previous blog,  lots of technologies the Kubernetes relies on are created with the help of the operating system. For example, the container is the specific process that corporates with the CGroup and Linux Namespace on OS. However, there is another essential OS concept used in Kubernetes: process group. It means one process group includes several processes. Those kinds of processes have a super affinity relationship. The `process group` is implemented as the `pod` in the Kubernetes.
Besides, the CGroup and Linux Namespace can supply the service for a process group initially.

# Why we need the Pod (containers group) in Kubernetes
You can imagine the scenario of scheduling multiple processes that belong to the same group.
```
// Group foo
// There is a special requirement of scheduling processes:
// A and B have to be scheduled into the same node.
Process A -- 10CPU
Process B -- 10CPU
Process C -- 5 CPU

// Node
Node A -- 15 CPU
Node B -- 20 CPU
```

If the `Process A` and `Process C` were scheduled into the `Node A`, firstly,  the scheduler doesn’t know how to schedule the `Process B`. Because the CPU resource almost is consumed by `Process A` and `Process C`  and the scheduling rule requires the `Process A` and `Process B` must run on the same node. That is a significant problem. Our scheduler seems to pick up a lousy scheduling strategy.

The root cause of the situation as above is the scheduler schedules processes belong to the same group separately. There is a super affinity relationship between `Process A` and `Process B`. We should handle them as a whole. So if you use the `container` instead of the `Process`, the same problem also happens in the Kubernetes world.

Util now, I think we can create a new scheduling strategy to solve this problem. It is unnecessary to create a new special content in kubernetes. If you also has the same confusion with me, let’s go back to the `Process` in the operating system.

When I perform the `pstree -g` command in my development VM, I captured the part of the result as below:
```
├-nvim(163523)─┬─gopls(163556)─┬─{gopls}(163556)
    │          ├─{gopls}(163556)
    │          ├─{gopls}(163556)
    │          ├─{gopls}(163556)
    │          ├─{gopls}(163556)
    │          └─{gopls}(163556)
    ├─node(163533)─┬─gopls(163533)─┬─{gopls}(163533)
    │     │        ├─{gopls}(163533)
    │     │        ├─{gopls}(163533)
    │     │        └─{gopls}(163533)
    │     ├─{node}(163533)
    │     ├─{node}(163533)
    │     └─{node}(163533)
    └─{nvim}(163523)
```

You can detect that there are so many processes corporate together to support the `nvim` work well.  And the process also needs multiple threads to corporate. The different processes (threads) play in different roles. But their target is the same:  finish all tasks when this process was performed in OS.

Although the docker community advises that developers should make the `single container pattern` as a development standard during containerizing applications. We cannot implement a production-level service within one binary file. It must include many containers, and they need to corporate together to serve customers.  So, how to handle the affinity relationship among multiple containers and obey the container development pattern at the same time? The answer is: `Pod`

`Pod` is a good proposal; it solves two problems as above: scheduling strategy and handling the affinity relationship.  `Pod` is a logic concept in Kubernetes.

# How to implement the `Pod` in Kubernetes ?
## For scheduling strategy

For the scheduling strategy of `Pod`,  the Kubernetes define a pod template that includes many fields to organize multiple containers. Every container can specify its own resource requests and limits for computing resources. That means the requirement of computing resources of the `Pod` is the sum of requirements of containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"

```

## For container design pattern
The container design pattern usually means multiple containers need to share some resources for the corporation so that they can serve to users well. So, the first problem we need to solve is creating an environment that can let multiple containers share network, volume, etc. A direct thought is implementing it in the docker level.  We can specify the volume or network parameters when the container is started.

```bash
docker run mysql --volume-from=web-server --network-from=web-server
```

If we implement it as above, that means the `web-server` container needs to be started earlier than `mysql`. It breaks the equal relationship among containers that belong to the same pod. So we need to create another container.  It should be the first to be started. We call it `infra container` . 

### Infra container

#### Principle

I draw the graph to help us understand the infra-container.

![](The%20smallest%20schedulable%20unit%20in%20Kubernetes%20%E2%80%94%20Pod%20(ep1)/729CEF17-3EB5-4DD9-BDA6-FD8D927E6A9E.png)

Containers within the pod usually share the volume and network resources. For the network resource, infra-container creates its own Linux Network namespace firstly. The MySQL and web-server containers are added into the same Network Namespace when they were created. That means they can use the `localhost` to communicate with each other. If you can understand Mount Namespace and infra-container. You must know that the volume is implemented in the pod level (infra-container level). That means we can mount any volume into the infra-container. If the MySQL and web-server can share the Mount Namespace with the infra-container, they can see the same volume directory.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  Mounts:
    - name: share-data
      hostPath:
        path: /data
  containers:
  - name: db
    image: mysql
    volumeMounts:
      - name: share-data
        mountPath: /data
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    volumeMounts:
      - name: share-data
        mountPath: /data
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"

```

The example as above implement a volume directory on the host machine and mount it into the frontend pod. The volume directory can be seen by wordpress container and mysql container. Any container writes some data to the `/data` that can be shared with another container.

#### Implementation

If you try to run the command `docker ps`, you must see some containers that use the same image called `gcr.io/google_containers/pause`

![](The%20smallest%20schedulable%20unit%20in%20Kubernetes%20%E2%80%94%20Pod%20(ep1)/C55377DA-80EC-4730-86D5-FA0E5EB4D043.png)

The `pause` container’s lifecycle is the same as `pod`. The google company uses `pause` container to implement the concept of `infra-container`. You also can develop yours. `pause` container usually is responsible for two things:

1. Creating a sharing environment for its children containers
		1. Linux Namespace(e.g. Linux Network Namespace)
		2. Volume
2. Enable the Linux PID Namespaces among containers group by the pod. So the `pause` container can be the super parent process to help us recycle orphan containers.

Actually, you can use `unshare` command to create a process in new Linux Namespaces.

```bash
sudo unshare --pid --uts --ipc --mount -f chroot rootfs /bin/sh
```

Because the child process usually inherits the Namespace from the parent process. If you use `unshare` command to create the new process, it creates its own Linux Namespace. It is too complicated to use original Linux commands to implement the `infra-container`. 

Apart from the sharing Namespace and volume, the pause container plays an important role: the super parent process. It will be responsible for reaping the zombie processes.

> zombie process is a kind of child process which has already finished all tasks, but the process’s item stayed in `process table` of the OS kernel. That means the parent process doesn’t clean it though call the `wait` function.  

Actually, there are two reasons that can cause the zombie process:

1. The parent process forgets to call the `wait` system call.
2. The parent process died before the child process, and the new parent process may don’t exist, or it doesn’t call the `wait`.

For the first reason, I think we cannot resolve it. It depends on the developers. For the second reason, we can implement the `reaping` function on infra-container. Once their original parent process can not clean any child process run in the same pod, the super-parent process in infra-container recycles it. We need to enable PID namespace sharing among containers in the same pod for implementing this function.

Here is the source code of `pause` container by Google:

```c
/*
Copyright 2016 The Kubernetes Authors.
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

static void sigdown(int signo) {
  psignal(signo, "Shutting down, got signal");
  exit(0);
}

static void sigreap(int signo) {
  while (waitpid(-1, NULL, WNOHANG) > 0);
}

int main() {
  if (getpid() != 1)
    /* Not an error because pause sees use outside of infra containers. */
    fprintf(stderr, "Warning: pause should be the first process\n");

  if (sigaction(SIGINT, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 1;
  if (sigaction(SIGTERM, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 2;
  if (sigaction(SIGCHLD, &(struct sigaction){.sa_handler = sigreap,
                                             .sa_flags = SA_NOCLDSTOP},
                NULL) < 0)
    return 3;

  for (;;)
    pause();
  fprintf(stderr, "Error: infinite loop terminated\n");
  return 42;
}
```

Through the example above, you can see that the pause will be the first(super parent) process of the Pod. If it catches the signal `SIGCHLD`, it calls `waited` to reap it. The most important is the pause process calls `pause` to wait signals during an infinity loop.

### Init  container

In addition to the `infra-container`, there is another special container called `Init container`. It will be performed before others which are defined in spec.containers section of the pod. That means we can utilize this mechanism to do some preparations in advance, such as copy necessary files, performing the setting script, e.g.

Besides, if you want to do some works before the container starts or stops, you can use the lifecycle of container: [Container Lifecycle Hooks - Kubernetes](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/).

### Examples of container design pattern
If you try to recall the solution of collecting the container logs, you can know that we use an another container design pattern `sidecar`.

![](The%20smallest%20schedulable%20unit%20in%20Kubernetes%20%E2%80%94%20Pod%20(ep1)/logging-with-sidecar-agent.png)

We deploy a logging-agent container within the pod as a sidecar container, and it shares the same volume called `logging` with app-container. In the default situation,  the container outputs logs into the stdout. We can use the logging SDK in our service, and it output the logs into a file that stay in the persistent volume shared by logging-agent. The logging-agent can forward logs into remote logging service, such as elastic-search. Although, we cannot see logs of the container through `kubectl logs`. We can find another approach to resolve it: [GitHub - phusion/baseimage-docker: A minimal Ubuntu base image modified for Docker-friendliness](https://github.com/phusion/baseimage-docker). There is a logging service that was included in the base image, syslog-ng. It can help us redirect logs from the file too stdout. Generally, I prefer to find or develop a convenient base image with a small size for my services.

## Implement the Pod by ourselves
Let’s create the fake `pod` with the help of `pause` container.
First of all, we can run a pause container:

```bash
docker run -d --name pause -p 8080:80 gcr.io/google_containers/pause-amd64:3.0
```

Then, we can run a blog application that shares resources with the `pause` container.

```bash
docker run -d --name ghost --net=container:pause --ipc=container:pause --pid=container:pause ghost
```

When we run the command `docker inspect <ghost_container_id>`

```bash
        "Config": {
            "Hostname": "52784ef438f9",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "2368/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "NODE_VERSION=10.17.0",
                "YARN_VERSION=1.19.1",
                "GOSU_VERSION=1.10",
                "NODE_ENV=production",
                "GHOST_CLI_VERSION=1.13.1",
                "GHOST_INSTALL=/var/lib/ghost",
                "GHOST_CONTENT=/var/lib/ghost/content",
                "GHOST_VERSION=3.0.3"
```

That means we can access the blog through the 2368 port. But we want to run another nginx container within the same Linux Network Namespace of ghost container. It is a more straightforward approach to help us detect the `Sharing Network Namespace`.

```bash
docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause nginx
```

The nginx config as below:

```nginx
error_log stderr;
events { worker_connections  1024; }
http {
    access_log /dev/stdout combined;
    server {
        listen 80 default_server;
        server_name example.com www.example.com;
        location / {
            proxy_pass http://127.0.0.1:2368;
        }
    }
}
```

Now we can access the `http://localhost:8080` to access the ghost blog.

![](The%20smallest%20schedulable%20unit%20in%20Kubernetes%20%E2%80%94%20Pod%20(ep1)/2D37EECB-3FE2-4329-B55A-F7B332771CE2.png)

There is a sharing topology among containers as above:

1. Pause <-> Ghost
2. Pause <—> Nginx
3. Nginx <—> Ghost

When you can see the home page of Ghost blog through accessing the port of Nginx server, those containers have shared the Linux Namespace.

![](The%20smallest%20schedulable%20unit%20in%20Kubernetes%20%E2%80%94%20Pod%20(ep1)/pause_container.png)

Managing the lifetime and relationship of containers is too complex, so the kubernetes create `Pod` to help us do that.

# Summarize
Finally, after you saw this blog, I want to tell you 3 things:

1. It is better to understand any new concept of Kubernetes in the operating system perspective. Because the Kubernetes relies on lots of fundamental technologies of the operating system. 
2. Even if the small and basic concept like `Pod`, there are many pieces of knowledge can be learned.
3. Don't learn too much about a particular concept, and knowledge has no end. Know what it is, why, and how it is enough. 

# Reference
- [The Almighty Pause Container - Ian Lewis](https://www.ianlewis.org/en/almighty-pause-container)
