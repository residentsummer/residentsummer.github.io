---
layout: post
title: "Testing out IPVS mode with minikube"
categories: k8s minikube
---

Starting from kubernetes version 1.9 there is a new promising IPVS mode in
kube-proxy. One of its advantages is the possibility to pick a
[load-balancing method][ipvs_modes]: RR, least connected, source/destination
hashing, shortest delay plus some variations of those. What I was interested in
are the LC methods, as they're somewhat better than iptables' RR in terms of
handling inbound traffic under load.

## Trying it out

So, to use IPVS one should either [pass][kubeproxy_reference] `--proxy-mode ipvs`
to kube-proxy, or set the equivalent option in the config file. As it turned out,
minikube doesn't provide a way to pass extra config to kube-proxy...

{% include cut.html %}

Digging deeper into how things are set up inside minikube VM (as of 0.28.2), I've
found out that kube-proxy is launched inside its own pod, with config file being
mounted from a ConfigMap. After k8s boot-up with the default settings, we'll just
edit relevant config (set `mode: "ipvs"` in config.conf):

```bash
kubectl edit -n kube-system configmap/kube-proxy
```

To apply new configuration, delete the old pod and k8s will create a new one,
as required by corresponding DaemonSet:

```bash
$ kc get -n kube-system pods
NAME                                    READY     STATUS    RESTARTS   AGE
...
kube-proxy-49psk                        1/1       Running   0          11h
...
$ kc delete -n kube-system po/kube-proxy-49psk
pod "kube-proxy-49psk" deleted
$ kc get -n kube-system pods
NAME                                    READY     STATUS    RESTARTS   AGE
...
kube-proxy-x7qgq                        1/1       Running   0          7m
...
```

Now let's fire up kube-proxy logs and see how this went:

```
{% raw %}$ kc logs -n kube-system po/kube-proxy-x7qgq
E0805 09:46:12.625751       1 ipset.go:156] Failed to make sure ip set: &{{KUBE-CLUSTER-IP hash:ip,port inet 1024 65536 0-65535 Kubernetes service cluster ip + port for masquerade purpose} map[] 0xc420562080} exist, error: error creating ipset KUBE-CLUSTER-IP, error: exit status 1
E0805 09:46:42.645604       1 ipset.go:156] Failed to make sure ip set: &{{KUBE-LOAD-BALANCER-FW hash:ip,port inet 1024 65536 0-65535 Kubernetes service load balancer ip + port for load balancer with sourceRange} map[] 0xc420562080} exist, error: error creating ipset KUBE-LOAD-BALANCER-FW, error: exit status 1
E0805 09:47:12.677159       1 ipset.go:156] Failed to make sure ip set: &{{KUBE-NODE-PORT-UDP bitmap:port inet 1024 65536 0-65535 Kubernetes nodeport UDP port for masquerade purpose} map[] 0xc420562080} exist, error: error creating ipset KUBE-NODE-PORT-UDP, error: exit status 1
E0805 09:47:42.748946       1 ipset.go:156] Failed to make sure ip set: &{{KUBE-NODE-PORT-LOCAL-TCP bitmap:port inet 1024 65536 0-65535 Kubernetes nodeport TCP port with externalTrafficPolicy=local} map[] 0xc420562080} exist, error: error creating ipset KUBE-NODE-PORT-LOCAL-TCP, error: exit status 1{% endraw %}
```

Well, that doesn't look like we're good to go with IPVS... The problem is that
the kernel, used in minikube, lacks a few ipset-related modules, providing hash
types, used by kube-proxy in IPVS mode. I'm not aware of the method to add extra
modules to minikube's kernel, other than building a custom image, so we need to
go deeper...

## Building custom minikube.iso

According to [documentation][minikube_iso_doc], this is as simple as clone & make:

```bash
$ git clone https://github.com/kubernetes/minikube
$ cd minikube
$ make buildroot-image
$ make out/minikube.iso
```

Note, that you'll need docker for this to succeed, as everything is being
done inside a container (who needs a shitload of build dependencies on
their laptop?). Performing this steps on MacOS produced a strange error:

```
logs are lost :(
```

At first, these messages doesn't make sense - `make` is unable to move the file
it created a few commands ago. But if you consider previous similar command, it
clicks: Apple uses case-insensitive FS by default! Minus 5 points to those who
came up with the idea of using case-differing file extensions for build process,
though. To overcome this obstacle, I've adopted the trick from [one poor soul
trying to build a cross-compilation toolchain for RPi 3][hdutil_trick] - use a
case-sensitive disk image as buildroot's output directory:

```bash
$ rm -rf out
$ hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -volname buildroot_out -size 20g ./buildroot_out.sparseimage
$ hdiutil attach -mountpoint ./out ./buildroot_out.sparseimage
$ make out/minikube.iso
```

> You need to be careful here, though - depending on how your docker file sharing
> set up, you may need to ssh into the VM and ensure that correct volume is
> mounted at `out`. If it NFS, for example, you'll need to add a line in
> `/etc/exports`, restart nfsd, and explicitly mount that dir.

Once the build finishes, we need to tweak the Linux kernel configuration to
include ipset modules. Minikube's custom ISO documentation says it can be done
with `make linux-menuconfig`, but for MacOS it's not the case (menuconfig won't
start). Also, the kernel version, specified in the minikube's Makefile, doesn't
match one used to build an ISO - you'll need to correct this as well (kernel
source can be found in `out/buildroot/output/build/linux-4*`; in my case, it was
linux-4.15):

```diff
diff --git a/Makefile b/Makefile
index 50f113aab..c3905d62e 100755
--- a/Makefile
+++ b/Makefile
@@ -33,7 +33,7 @@ MINIKUBE_VERSION ?= $(ISO_VERSION)
 MINIKUBE_BUCKET ?= minikube/releases
 MINIKUBE_UPLOAD_LOCATION := gs://${MINIKUBE_BUCKET}

-KERNEL_VERSION ?= 4.16.14
+KERNEL_VERSION ?= 4.15

 GOOS ?= $(shell go env GOOS)
 GOARCH ?= $(shell go env GOARCH)
```

Finally, to configure the kernel we'll reuse buildroot's docker image:

```bash
$ docker run -it --rm -v $PWD:/minikube gcr.io/k8s-minikube/buildroot-image bash
root@559247e3fa03:/# apt-get update
root@559247e3fa03:/# apt-get install -y --no-install-recommends libncurses-dev
root@559247e3fa03:/# cd /minikube
root@559247e3fa03:/minikube# make linux-menuconfig
```

Go to Network &gt; Netfilter &gt; IP Sets, tick all hash types as modules,
save &amp; exit. Now build an ISO once more (it should be fast). To use the new
image, delete the old cluster and recreate it with our new `--iso-url`:

```bash
$ minikube stop && minikube delete
$ minikube start --iso-url=file://$PWD/out/minikube.iso
```

After cluster starts up, repeat the steps we've done to switch to IPVS in
kube-proxy and voila:

```
I0811 21:26:07.996804       1 feature_gate.go:230] feature gates: &{map[]}
I0811 21:26:08.064640       1 server_others.go:183] Using ipvs Proxier.
W0811 21:26:08.086817       1 proxier.go:349] clusterCIDR not specified, unable to distinguish between internal and external traffic
W0811 21:26:08.086847       1 proxier.go:355] IPVS scheduler not specified, use rr by default
I0811 21:26:08.087178       1 server_others.go:210] Tearing down inactive rules.
I0811 21:26:08.142232       1 server.go:448] Version: v1.11.0
I0811 21:26:08.154958       1 conntrack.go:98] Set sysctl 'net/netfilter/nf_conntrack_max' to 131072
I0811 21:26:08.155260       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I0811 21:26:08.155338       1 conntrack.go:98] Set sysctl 'net/netfilter/nf_conntrack_tcp_timeout_established' to 86400
I0811 21:26:08.155394       1 conntrack.go:98] Set sysctl 'net/netfilter/nf_conntrack_tcp_timeout_close_wait' to 3600
I0811 21:26:08.155634       1 config.go:102] Starting endpoints config controller
I0811 21:26:08.155661       1 controller_utils.go:1025] Waiting for caches to sync for endpoints config controller
I0811 21:26:08.155703       1 config.go:202] Starting service config controller
I0811 21:26:08.155709       1 controller_utils.go:1025] Waiting for caches to sync for service config controller
I0811 21:26:08.256254       1 controller_utils.go:1032] Caches are synced for service config controller
I0811 21:26:08.256369       1 controller_utils.go:1032] Caches are synced for endpoints config controller
```

Now lets see which ipset modules are really needed (e.g. to submit a patch to
minikube). And also check out the IPVS configuration, that was done by the proxy:

```bash
$ kc exec -it kube-proxy-lxj2d ipset list | grep Type | sort -u
Type: bitmap:port
Type: hash:ip,port
Type: hash:ip,port,ip
Type: hash:ip,port,net
$ kc exec -it kube-proxy-lxj2d ipvsadm
OCI runtime exec failed: exec failed: container_linux.go:348: starting container process caused "exec: \"ipvsadm\": executable file not found in $PATH": unknown
command terminated with exit code 126
```

Oh, not again! No `ipvsadm` in kube-proxy image... Screw it, just install a
package via [toolbox][toolbox]:

```bash
$ minikube ssh
$ toolbox
[root@minikube ~]# dnf -y install ipvsadm
[root@minikube ~]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  minikube:https rr
  -> minikube:pcsync-https        Masq    1      0          0
TCP  minikube:domain rr
  -> 172.17.0.2:domain            Masq    1      0          0
  -> 172.17.0.3:domain            Masq    1      0          0
TCP  minikube:http rr
  -> 172.17.0.4:websm             Masq    1      0          0
TCP  localhost:ndmps rr
  -> 172.17.0.4:websm             Masq    1      0          0
TCP  minikube:ndmps rr
  -> 172.17.0.4:websm             Masq    1      0          0
TCP  minikube:ndmps rr
  -> 172.17.0.4:websm             Masq    1      0          0
UDP  minikube:domain rr
  -> 172.17.0.2:domain            Masq    1      0          0
  -> 172.17.0.3:domain            Masq    1      0          0
```

> As you would've guessed by now, this didn't worked on the first try :)
> `dnf` failed on fetching package indices due to low disk space.
> Somehow I've only got 1G on rootfs... Worked this around by mounting
> tmpfs at `/var/cache/dnf`.

I don't understand all of the output, but there is certainly a DNS service on TCP
and UDP. As you can see, IPVS scheduler for all the virtual services is `rr`.

## Trying it out, for real

First, I'd like to check the default scheduler. We'll spin up an http server
and bench it with `ab`.

```bash
$ kc run hello-server --image kennethreitz/httpbin --port 80 --replicas 2
$ kc expose deployment hello-server
$ # Wait till it's ready
$ kc get svc
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
hello-server   ClusterIP   10.104.236.6   <none>        80/TCP    37m
kubernetes     ClusterIP   10.96.0.1      <none>        443/TCP   3h
$ ab -c 10 -n 100 "http://10.104.236.6/ip"
```

Let's check the stats:

```bash
[root@minikube ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
--- snip ---
TCP  10.104.236.6:80 rr
  -> 172.17.0.5:80                Masq    1      0          50
  -> 172.17.0.6:80                Masq    1      0          50
```

As expected, requests have landed 50/50 on both pods. Now we'll change the
scheduler to `lc`. To do this, edit the configmap once again and delete
kube-proxy's pod. Then we need to simulate an uneven load. I came up with
this brittle scenario:

* scale the deployment to a single pod
* start a bunch of long requests
* scale it back to two pods
* run the test

```bash
$ kc scale deployment hello-server --replicas 1
$ # Wait till the second pod dies
$ (ab -c 10 -n 10 "http://10.104.236.6:80/delay/10" &) && \
      kc scale deployment hello-server --replicas 2 && \
      sleep 8 && \
      ab -c 10 -n 100 "http://10.104.236.6/ip"
[root@minikube ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
--- snip ---
TCP  10.104.236.6:80 lc
  -> 172.17.0.5:80                Masq    1      0          10
  -> 172.17.0.6:80                Masq    1      0          100
```

Yay, it worked! While the first pod was "busy" serving delayed requests,
the second handled a hundred of short ones.

SUCCESS 🎉🎉🎉

  [ipvs_modes]: https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-ipvs
  [kubeproxy_reference]: https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/
  [minikube_iso_doc]: https://github.com/kubernetes/minikube/blob/master/docs/contributors/minikube_iso.md#build-instructions
  [hdutil_trick]: https://www.jwz.org/blog/2016/11/buildroot/
  [toolbox]: https://github.com/coreos/toolbox
