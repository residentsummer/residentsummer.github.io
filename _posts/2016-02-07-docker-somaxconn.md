---
layout: post
title: "Adjusting privileged network options for docker containers"
categories: docker
---

*&lt;docker has changed the devops landscape, blah-blah-blah...&gt;*

Now, let's get to the practical aspects of running containers in production.

## The challenge

It's a popular opinion that default value of `net.core.somaxconn` is too low for
heavily loaded web-servers (not going to benchmark it, just take it as an
example). No probs, let's set it to a higher value:

```bash
sysctl -w net.core.somaxconn=1024
```

Well, it's not going to work.

{% include cut.html %}

* Neither from inside the container. Isolation is one of the purposes of putting
  your stuff in docker. Hence, containerized code are denied the right to modify
  host system parameters. In our particular case, `sysctl` won't work because of
  the `/proc` being mounted read-only.
* Nor from the docker host. In most cases, containers are started with its own
  network namespaces and host settings won't apply to them.

## Options

Googling all over the internet ([this][so_answer] answer on stackoverflow is
particularly helpful) yields a few solutions:

* Use host network stack (`--net=host`) for containers in question. This
  actually works for some scenarios.
* Mount host `/proc` inside a container and configure network stack from within.
  This option obviously breaks the isolation.
* <del>Run container in privileged mode.</del>

While the first option may be desirable sometimes (e.g. to skip a layer of
a virtual network), it doesn't cover all cases.

## Solution

Why does using `--net=host` work? Obviously, because host network can be easily
configured with all values we need. Why does the second option work? Because
with the writable `/proc/sys` we can adjust the required params from within a
container.

And here is the trick - *host network namespace is not the only one we can
reuse!* Docker provides a few choices for the container's network configuration.
With one of them being `--net=container:<id>` (use the network stack of
container &lt;id&gt;), we have all we need to tune network namespace as required.

Let's try to make "practical" implementation:

* Create a container that will hold the network namespace:

  ```bash
  netns_owner=$(docker run -d busybox /bin/tail -f /dev/null)
  ```

  or even better

  ```bash
  netns_owner=$(docker run -d kubernetes/pause)
  ```

* Configure the required parameters

  ```bash
  docker run --net=container:${netns_owner} --privileged --rm \
    busybox sysctl -w net.core.somaxconn=1024
  ```

* Run application container attached to that namespace:

  ```bash
  docker run --net=container:${netns_owner} your_image
  ```

## Some "details" to bear in mind

* Most of network-related container configuration (publishing ports,
  establishing links) should now be done on creation of network namespace
  owner. E.g.:

  ```bash
  netns_owner=$(docker run -d -p 80:80 kubernetes/pause)
  ```

* Network container should be running for namespace to stay alive (until
  [12035][docker_netns_pinning] is done). Don't forget to remove it together
  with the app container.
* It may ruin scaling in docker-compose.


  [so_answer]: http://stackoverflow.com/questions/26177059/refresh-net-core-somaxcomm-or-any-sysctl-property-for-docker-containers/26197875#26197875
  [docker_netns_pinning]: https://github.com/docker/docker/issues/12035

