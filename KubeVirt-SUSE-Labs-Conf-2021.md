# KubeVirt @ SUSE: Status update

## Introduction

Even with rapid adoption of containerization technologies there is still a high demand for using virtual machines. Nowadays Kubernetes has become the de-facto standard when it comes to orchestration of containerized workloads. However originally it does not have the capability to operate with VMs. KubeVirt is an add-on for Kubernetes that aims to fill that gap. It integrates into the Kubernetes ecosystem and allows running and managing virtual machines at scale.

In this talk we will cover the current status of KubeVirt development at SUSE. We will highlight the major aspects of maintaining the downstream packages and container images and how they fit with the build infrastructure. We will then talk about the upstream contributions and ongoing activities. And we will also show the results of our recent research about running CPU, memory and disk IO workloads in KubeVirt VMs and plain KVM VMs, and from where the differences in terms of performance come from.


## What is KubeVirt?

[KubeVirt][1] is a virtualization management add-on for Kubernetes. It allows running virtual machines alongside the containerized workloads in a seamless and unified way on a single platform. Kubevirt extends Kubernetes by introducing new custom resource (CR) types: VirtualMachine (VM) and VirtualMachineInstance (VMI). Those are used to model various settings and options supported by KubeVirt and the underlying virtualization stack. The definition of a VM looks like bellow:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-cirros
  name: vm-cirros
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-cirros
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
        resources:
          requests:
            memory: 128Mi
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/cirros-container-disk-demo:devel
        name: containerdisk
      - cloudInitNoCloud:
          userData: |
            #!/bin/sh

            echo 'printed from cloud-init userdata'
        name: cloudinitdisk
```

The core KubeVirt [components][2] (pods) are:
- virt-api – API server; the entry point for all virtualization-related requests
- virt-controller – monitors CRs (i.e. VMs, VMIs, etc.) and manages associated pods
- virt-handler – privileged pod (DaemonSet) that runs on each node in the cluster
- virt-launcher – hosts the main VMI process (Libvirt daemon + Qemu)
- virt-operator – manages KubeVirt deployment

![KubeVirt components](images/kubevirt-components.png)

KubeVirt runs VMs in pods (one per VM) thus allowing them to be managed by Kubernetes in the same fashion as standard container applications.


## Downstream: KubeVirt @ SUSE

KubeVirt is written in Go language and it uses Bazel for building the binaries and container images. When it comes to downstream maintenance and integration with the Open Build Service (OBS), Bazel, unfortunately, does not turn out to be very “friendly”. The upstream build is based on Fedora packages and it requires several gigabytes of dependencies to be downloaded and stored in the repository. That makes it pretty hard and resource consuming to build the project in the “upstream-way”. In order to mitigate that problem the downstream build has been based on the native Go approach. As a result it has been split into two steps.

First, the main KubeVirt package is built in OBS as a standard spec-file project. It provides the RPMs with all the needed artifacts. Some of the RPMs are just “intermediate” and used only for building the “virt containers” later on.

Then at the second step the container images are built out of Dockerfiles. The “intermediate” RPMs with the binaries from the first step are “consumed” and installed in the final images:

![Kubevirt packages](images/kubevirt-packages.png)

That way it becomes easy to customize and build KubeVirt containers on top of openSUSE or SLE base images. E.g. the resulting Dockerfile (simplified a little bit though) may look like the following:

```Dockerfile
...
# virt-launcher container image
# KUBEVIRTFROM defined in prjconf, e.g.
#  BuildFlags: dockerarg:KUBEVIRTFROM=opensuse/tumbleweed
ARG KUBEVIRTFROM
FROM $KUBEVIRTFROM
...
RUN zypper -n install \
    ...
    kubevirt-container-disk \
    kubevirt-virt-launcher \
    ...
...
```

However such a “two-steps” approach introduces several challenges.

### Challenge #1: consistency

KubeVirt is usually deployed to the cluster using the manifests. Those are YAML files with settings and definitions. The manifests include the information about the container registry to pull the images from and the corresponding tags. This info gets generated during the build of the RPMs but it is also required in the Dockerfiles for proper tagging. I.e. the tags of the resulting container images must exactly match the tags generated in the manifests. Providing those manually for each container image every time a new release comes out is error-prone and time consuming.

![KubeVirt deployment](images/kubevirt-deploy.png)

A solution towards automation of the process has been based on OBS build services. It is heavily inspired by *docker_label_helper*, *kiwi_metainfo_helper* and *obs-service-replace_using_package_version*. An OBS service can be used as an equivalent of *BuildRequires* spec-file directive to pre-install dependencies into the build environment for container images. For the KubeVirt build the two additional services have been introduced:

**obs-service-replace_using_env** – this is an independent generic OBS service that reads the environment variables from a given file and performs the substitution in the sources (e.g. in the Dockerfile).

**obs-service-kubevirt_containers_meta** – another build service that gets **generated** during the build along with the manifests RPM. It provides the environment file specific for the KubeVirt use-case. The file is consumed by *obs-service-replace_using_env* during the build of container images and all the required metadata gets injected into the Dockerfile.

![KubeVirt patch dockerfile](images/kubevirt-patch-dockerfile.png)

That way the tags and registry information obtained during the build of the manifests RPM get automatically propagated to containers via the build service. The scheme ensures that the released images are properly tagged and eventually the manifests refer to the existing containers.

### Challenge #2: unique tags

The second problem comes from the fact that each KubeVirt release (and actually each build) needs to be identified by a unique tag. That is needed to ensure proper deployment with respect to the default image pull policy [IfNotPresent][3]. The upstream KubeVirt relies on SHA digests: those are calculated from the container images and then injected into the manifests. Unfortunately that is not an option for the downstream build. There is the “chicken-egg” problem: downstream container images depend on the manifests (more precisely: on the meta-data generated when building the manifests) while the SHA digests can be calculated only **after** building the images (and by that time the manifests should be already built to ensure consistent tagging). Apart from that there is currently no straightforward way in OBS to get the SHA digests and somehow provide them to an RPM package.

The solution here is to use the RPM version information provided by OBS during the build of the manifests. The tag can be composed as **%{version}-%{release}**. Assuming that the *%{release}* part gets incremented each time a build is triggered, this scheme ensures unique tags for every build. Using just *%{version}* is not enough since the downstream revisions also need to be considered in order to handle patch updates.

The described approach works well for the Tumbleweed rolling model where everything is automatically rebuilt once a dependency is updated. However it brings certain issues when it comes to fixed releases like the ones for SLE. The containers have strict dependency on the manifests and the generated tags. It is not possible to release only one container since every new update requires a unique tag in order to be properly deployed to a cluster. Therefore it is necessary to build and release all the components together.

### Containerized Data Importer

[Containerized Data Importer][4] (CDI) is another add-on for Kubernetes focused on persistent storage management. It is primarily used for building and importing Virtual Machine Disks for KubeVirt.

CDI introduces DataVolume CR which is an abstraction on top of standard Kubernetes [Persistent Volume Claim][5] (PVC). It is used to automate the creation of VM disks and to populate them with data from various sources: container registries, http servers, local images, etc.

KubeVirt and CDI can work independently but when used together they provide a better user experience. Thus CDI is also maintained downstream and released along with KubeVirt. The CDI project has a similar structure (i.e. it is written in Go language and it uses Bazel for building binaries and container images) and therefore the same “two-steps” approach is used.

### Current state

- Rolling release in Tumbleweed
  - container images available at `registry.opensuse.org/kubevirt/...`
- Tech preview in SLE15 SP2: KubeVirt 0.40.0 and CDI 1.30.0
  - container images available at `registry.suse.com/suse/sles/15.2/...`
- Tech preview in SLE15 SP3: KubeVirt 0.45.0 and CDI 1.37.1
  - container images available at `registry.suse.com/suse/sles/15.3/...`

[The Harvester project][6] adopts SUSE downstream KubeVirt images: the release 0.2.0 uses KubeVirt based on openSUSE Tumbleweed; SLE15 SP3 based images are planned to be used for 0.3.0 and GA.


## Upstream involvement

Areas of contribution:

- Bugfixes and improvements
  - SUSE builds are based on openSUSE and SLE container images and include a different version of the KVM stack: Qemu and Libvirt. While the upstream KubeVirt currently uses Fedora-based containers. That helps with identifying issues that are not reproducible in the upstream CI and improve the interoperability of the project.
- Improving the CI:
  - Helping the community with identifying and fixing flaky tests as well as extending the coverage and efficiency.
- Delivering new features:
  - Initial support of cgroups v2: several issues preventing KubeVirt from running virtual machines on the nodes with cgroups v2 were identified and fixed. Apart from that the pre-submit and periodic jobs on dedicated CI lanes were enabled to avoid regressions. Device handling is still in WIP state though...
  - Live-migration of VMs with hot-plugged volumes: the support of hot-plug volumes has been around for quite a while in KubeVirt. However it was not possible to perform live-migration of virtual machines with dynamically attached storage. Several discussions were initiated around that and a series of patches was introduced to lift that limitation.
  - [WIP] LaunchSecurity with AMD SEV: this is an on-going activity. The idea is to bring the support of confidential computing to KubeVirt. Starting by leveraging the SEV (Secure Encrypted Virtualization) extension available on the AMD’s EPYC CPUs.


## Research: the cost of containerizing VMs

Something that we decided to investigate is what are the differences when it comes to tuning virtualization workloads when using the traditional KVM stack and KubeVirt.
This was a joint project with the NARA Institute of Science and Technology ([NAIST][7]) in Japan and some preliminary results have been presented to [KVM Forum 2021][8].

### The experimental setup

We used a 32 physical CPUs server both as the KVM host and as the KubeVirt worker node, i.e., where the VMs were actually running, in both cases.
It had two NUMA nodes with 8 cores each and hyperthreading enabled.
In some more details:
- Intel(R) Xeon(R) Silver 4208 CPU @ 2.10GHz
  - CPU(s): 32
  - NUMA nodes (== sockets): 2
  - Threads per core: 2
  - Cores per socket: 8
- Family / Model / Stepping: 6 / 85 / 7
- MHz (min/max): 800 / 3200
- Cache:
  - L1 (i & d): 512 KB
  - L2: 16 MiB
  - L3: 22 MiB
- RAM: 32 GiB (16 GiB on each node)
- Disk:	rotational device (no SSD)

![Server](images/servertopo.png)

From a software point of view, the host was Ubuntu 20.04.2 LTS, with kernel 5.4.0.
We used the latest available KubeVirt release which, when we run the experiments, was 0.44.0 and included QEMU 5.2.0 and libvirt 7.0.0.
Kubernetes version was 1.21 and the container runtime was [docker][9].
For the KVM experiments, we built from sources and used the very same versions of those components.

We run our experiments inside a 1 vCPU VM, and then we repeated them inside a 4 vCPUs VM.
The VM had 8 Gigabytes of RAM (in both cases) and was running openSUSE Leap 15.2. We used [MMTests][10] as our benchmarking suite, as it can orchestrate running benchmarks inside (one or even multiple) VM(s).

We have run several benchmarks:
- [cyclictest][11]
  - thread wakeup time was 1 ms, they had FIFO priority, and hackbench was running in background
  - 2 configurations:
    - threads pinned to vCPUs
    - threads not pinned (unbound)
- [NASA Parallel Benchmark][12]
  - 2 threads (which is the half the number of vCPUs of our VM) in parallel, via [OpenMP][13]
  - various computational kernels:
    - bt, cg, ep, ft, is, sp, ua
- [STREAM][14]
  - 2 threads (which is the half the number of vCPUs of our VM) in parallel, via [OpenMP][13]
  - multiple memory operations:
    - copy, scale, add, triadd
- [hackbench][15]
  - with multiple processes, communicating via pipes (implemented by `perf bench sched pipe`)
  - 2 configurations:
    - processes groups (80 tasks), 4 thread groups (160 tasks)
- [kernbench][16]
  - with `vmlinux` as build target (with `defconfig`)
  - 4 configurations:
    - `make -j 1`, `make -j 2` and `make -j 4`
- [iozone][17]
  - for synchronous IO
  - multiple operations:
    - write, rewrite, read, reread, random red, random write, backward read
  - multiple configurations (with different file sizes):
    - 1GB, 2GB, 4GB

All benchmarks were, of course, running inside the VMs.
We have also considered different load conditions for the host.
In fact, We run all the benchmarks on an host which was only running our VM, and was otherwise idle.
Then we run them while imposing, at the same time, an extraneous 50% load on the host and also with 100% extraneous load.
For generating load on the host we used various [stress-ng][18] threads in a way that they can simulate having had other VMs running at the same time of our one.
Basically, in what we call the _idle host_ case, only the VM was running.
In the _host loaded_ case, `stress-ng` was used to generate a 1400% (out of a total of 3200%) artificial load on the host.
This can be considered representative of a situation where the host would be running 7 additional (to our one) VMs with 4 vCPUs, each one of which is 50% loaded.
Finally, in the _highly loaded_ case, `stress-ng` was generating approximately 2800% artificial load, which could be considered similar to having 7 additional VMs with 4 vCPUs, each of which is 100% loaded.
Considering that also our VM had (at most) 4 vCPUs, the highly loaded case was indicative of a fully loaded, but not oversubscribed, situation.

This paper includes a subset of all the results that we collected.

## References

[1]: https://github.com/kubevirt/kubevirt
[2]: https://github.com/kubevirt/kubevirt/blob/main/docs/README.md
[3]: https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy
[4]: https://github.com/kubevirt/containerized-data-importer
[5]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
[6]: https://github.com/harvester/harvester
[7]: http://www.naist.jp/en/
[8]: https://www.youtube.com/watch?v=x_czS9Iuo2o
[9]: https://www.docker.com/
[10]: https://github.com/gormanm/mmtests
[11]: https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start
[12]: https://www.nas.nasa.gov/software/npb.html
[13]: https://www.openmp.org/
[14]: https://www.cs.virginia.edu/stream/
[15]: https://man7.org/linux/man-pages/man1/perf-bench.1.html
[16]: http://ck.kolivas.org/apps/kernbench/kernbench-0.50/
[17]: https://www.iozone.org/
[18]: sttress-ng
