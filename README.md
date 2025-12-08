# sonic-l2ls-evpn-containerlab

The purpose of this repo is showing how to create a SONiC image to be used in a container lab setup and the configuration required at both the SONiC and FRR levels to deploy a EVPN layer 2 service, where one leaf runs SONiC and the spine and the other leaf run SR Linux, as per the diagram:

![pic1](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/img_and_drawio/sonic-l2ls-evpn-containerlab.png)

# Donwload the SONiC image

1 - Go to the pipelines list: https://sonic-build.azurewebsites.net/ui/sonic/pipelines and scroll all the way down where "vs" platform is listed

2 - Choose a build (in this lab 202411 was used for no particular reason) and click on build history
  
3 - Pick one where "Result = succeeded" and click on Artifacts 

4 - Click on sonic-builimage.vs

5 - Donwload the target/sonic-vs.img.gz

![pic1](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/img_and_drawio/sonic-img-download.png)


# Build the SONiC container lab image

```bash
% ls | grep sonic
docker-sonic-vs.gz
```

Here there are 2 options:

1 - (not used in ths repository) Simply load the docker-sonic-vs into docker, the drawback is that the end result is not going to be similar to a SONiC router where processes like shmp, swss, bgp etc are running in separate containers 

```bash
$ docker load < docker-sonic-vs
$ docker image ls
REPOSITORY                             TAG       IMAGE ID       CREATED         SIZE
docker-sonic-vs                        latest    2d9c647a53df   4 hours ago     797MB 
```

Using the above in the clab.yml file the "kind" field to be used is "sonic-vs" (kind: sonic-vs)

2 - Use the vrnetlab tool (https://containerlab.dev/manual/vrnetlab/) to create a SONiC image that truly mimics the SONiC architecture in terms of different processes running in different containers

After installing vrnetlab place the uncompress file in the vrnetlab directory, rename it to `sonic-vs-[version].qcow2` and run `make`.

```bash
$ mv sonic-vs.img sonic-vs-202411.qcow2
$ make
$ docker images | grep vrnetlab
vrnetlab/sonic_sonic-vs                202411    0abf9ef806c8   About a minute ago   6.42GB
```

Using the above in the clab.yml file the "kind" field to be used is "sonic-vm" (kind: sonic-vm)

```bash
    leaf1:
      kind: sonic-vm
      image: vrnetlab/sonic_sonic-vs:202411
```

The advantage of using the vrnetlab tool is that the result truly mimics the SONiC architecture, where we can see differend containers for different processes

```bash
admin@sonic:~$ docker ps
CONTAINER ID   IMAGE                             COMMAND                  CREATED              STATUS              PORTS     NAMES
6c0b1e92be5c   docker-router-advertiser:latest   "/usr/bin/docker-ini…"   46 seconds ago       Up 41 seconds                 radv
476887704c68   docker-gbsyncd-vs:latest          "/usr/local/bin/supe…"   47 seconds ago       Up 42 seconds                 gbsyncd
b0671ba89f97   docker-eventd:latest              "/usr/local/bin/supe…"   48 seconds ago       Up 43 seconds                 eventd
8249518149b4   docker-fpm-frr:latest             "/usr/bin/docker_ini…"   48 seconds ago       Up 45 seconds                 bgp
61e5be1a45a9   docker-syncd-vs:latest            "/usr/local/bin/supe…"   49 seconds ago       Up 46 seconds                 syncd
f2287fc3db9c   docker-teamd:latest               "/usr/local/bin/supe…"   49 seconds ago       Up 47 seconds                 teamd
a459756a8bf2   docker-orchagent:latest           "/usr/bin/docker-ini…"   53 seconds ago       Up 50 seconds                 swss
43f5388f05d5   docker-database:latest            "/usr/local/bin/dock…"   About a minute ago   Up About a minute             database
```

## Deploying the lab

The lab is deployed with the [containerlab](https://containerlab.dev) project, where [`evpn_sonic_l2ls.clab.yml`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/evpn_sonic_l2ls.clab.yml) file declaratively describes the lab topology.

Create/update the lab
```bash
containerlab deploy --reconfigure
```

After the lab is created it is expected that the SONiC router willl take some time to load, it should be monitored using the docker ps command
```bash
$ docker ps | grep leaf1
42048a32ab14   vrnetlab/sonic_sonic-vs:202411       "/launch.py --userna…"   10 minutes ago   Up 10 minutes (healthy)   22/tcp, 443/tcp, 5000/tcp, 8080/tcp                      leaf1
```

Destroy the lab
```bash
containerlab destroy --cleanup
```






