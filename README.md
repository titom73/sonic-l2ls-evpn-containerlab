<img width="468" height="25" alt="image" src="https://github.com/user-attachments/assets/c2044462-e951-42bd-a4f0-5f256752f035" /><img width="468" height="25" alt="image" src="https://github.com/user-attachments/assets/0451d326-d207-4117-b0a2-e885d58ce517" /># sonic-l2ls-evpn-containerlab

The purpose of this repository is showing how to create a SONiC image to be used in a container lab setup and the configuration required at both the SONiC JSON and FRR levels to deploy a simple EVPN layer 2 service, where one leaf runs SONiC and the spine and the other leaf run SR Linux, as per the diagram:

![pic1](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/img_and_drawio/sonic-l2ls-evpn-containerlab.png)

# Download the SONiC image

1 - Go to the pipelines list: https://sonic-build.azurewebsites.net/ui/sonic/pipelines and scroll all the way down where "vs" platform is listed

2 - Choose a build (in this lab 202411 was used for no particular reason) and click on build history
  
3 - Pick one where "Result = succeeded" and click on Artifacts 

4 - Click on sonic-builimage.vs

5 - Download the target/sonic-vs.img.gz

![pic1](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/img_and_drawio/sonic-img-download.png)


# Build the SONiC container lab image

```bash
% ls | grep sonic
docker-sonic-vs.gz
```

Here there are 2 options:

1 - (not used in this repository) Simply load the docker-sonic-vs into docker, the drawback is that the end result is not going to be very similar to a SONiC router where processes like shmp, swss, bgp etc are running in separate containers

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

The advantage of using the vrnetlab tool is that the result truly mimics the SONiC architecture, where we can see different containers for different processes

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

To create or update the lab
```bash
containerlab deploy --reconfigure
```

After the lab is created it is expected that the SONiC router will take some time to load, use docker ps command to see when the status becomes healthy

```bash
$ docker ps | grep leaf1
42048a32ab14   vrnetlab/sonic_sonic-vs:202411       "/launch.py --userna…"   10 minutes ago   Up 10 minutes (healthy)   22/tcp, 443/tcp, 5000/tcp, 8080/tcp                      leaf1
```

To destroy the lab
```bash
containerlab destroy --cleanup
```

## SONiC configuration files

There are two key components

1 - JSON file (located at /etc/sonic/config_db.json)
```bash
#Write the configuration
sudo config save -y

#Load the configuration:
sudo config load /etc/sonic/config_db.json -y

#Reload (after loading a new config restart SONiC services):
sudo config reload -y 
```

The SONiC configuration cab be done by manipulating the JSON (and then reloading) or simply using the SONiC CLI adn then doing a config save.

**Important note 1**: The https://github.com/sonic-net/SONiC/wiki/Configuration provides some very good insights regarding the JSON file structure, however, it is not a complete schema definition to that JSON file (the information is scattered across several different websites)

**Important note 2**: Many of the BGP configuration options can be configured in the JSON file directly, however, some configuration options (e.g. import/export policies) require FRR configuration which requires editing the configuration file for FRR (next point)

2 - FRR configuration regarding protocols such as BGP, can be acceses using vtysh in the SONiC host
```bash
admin@sonic:~$ vtysh

Hello, this is FRRouting (version 10.0.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.
sonic#     
```

In this repository there is some BGP configuration in the JSON and some on the FRR config, some parts of the JSON file:

```bash
    "BGP_DEVICE_GLOBAL": {
        "STATE": {
            "idf_isolation_state": "unisolated",
            "tsa_enabled": "false",
            "wcmp_enabled": "false"
        }
    },
    "BGP_GLOBALS": {
        "default": {
            "default_ipv4_unicast": "true",
            "local_asn": "101",
            "router_id": "10.0.1.1"
        }
    },
    "DEVICE_METADATA": {
        "localhost": {
            "bgp_asn": "101",
            "buffer_model": "traditional",
            "default_bgp_status": "up",
            "default_pfcwd_status": "disable",
            "hostname": "sonic",
            "hwsku": "Force10-S6000",
            "mac": "22:d1:c7:63:8f:4a",
            "platform": "x86_64-kvm_x86_64-r0",
            "timezone": "UTC",
            "type": "LeafRouter"
        }
    },
```

In the folder [`configs`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/tree/main/configs) there is the [`JSON file`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/configs/leaf1-config.json) and [`FRR file`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/configs/leaf1-frr-bgp.cfg) used

## SONiC node configuration after boot 

After boot there are two steps required

1 - Replace the file /etc/sonic/config_db.json at the SONiC host (apparently it is not possible to pass the configuration file directly via the clab.yml definition hence the need for this step)

1.1 - Copy the file to the host

1.2 - Reload the host so that the "new" configuration in the JSON file becomes active 

The above steps are summarised in the shell script [`deploy_sonic_cfg.sh`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/deploy_sonic_cfg.sh)

2 - Load the desided [`FRR BGP configuration`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/configs/leaf1-frr-bgp.cfg) 

After step 1 there will be some BGP parameters already there (the ones that are part of the JSON), however, others need to be added directly at the FRR configuration level.

This can be achieved by running the script [`deploy_bgp_vtysh.sh`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/deploy_bgp_vtysh.sh)

## SONiC specifics regarding container lab topology file and interface naming

In the file that describes the topology [`evpn_sonic_l2ls.clab.yml`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/evpn_sonic_l2ls.clab.yml) the interfaces for a device of type sonic-vm or sonic-vs are named eth1 to ethN where in the SONiC router there are named Ethernet0, Ethernet4, Ethernet8 and so on. The matching rules used is that eth1 in the clab.yml file relates to the 1st Ethernet interface in the SONiC router, eth2 to the 2nd and so on ... so in the topology file we see eth1 and eth2 regarding leaf1 (the SONiC router):

```bash
  links:
    - endpoints: ["spine1:e1-1", "leaf1:eth1"] # eth1 maps to Ethernet0
    - endpoints: ["spine1:e1-2", "leaf2:e1-49"]
    - endpoints: ["client1:eth1", "leaf1:eth2"] # eth2 maps to Ethernet4
    - endpoints: ["client2:eth1", "leaf2:e1-1"]
```

While in the SONiC router itself:
```bash
admin@sonic:~$ show interfaces status 
  Interface            Lanes       Speed    MTU    FEC           Alias    Vlan    Oper    Admin    Type    Asym PFC
-----------  ---------------  ----------  -----  -----  --------------  ------  ------  -------  ------  ----------
  Ethernet0      25,26,27,28  4294967.3G   9100    N/A    fortyGigE0/0  routed      up       up     N/A         N/A
  Ethernet4      29,30,31,32  4294967.3G   9100    N/A    fortyGigE0/4   trunk      up       up     N/A         N/A
  Ethernet8      33,34,35,36         40G   9100    N/A    fortyGigE0/8  routed    down       up     N/A         N/A
 Ethernet12      37,38,39,40         40G   9100    N/A   fortyGigE0/12  routed    down       up     N/A         N/A
 Ethernet16      45,46,47,48         40G   9100    N/A   fortyGigE0/16  routed    down       up     N/A         N/A
 Ethernet20      41,42,43,44         40G   9100    N/A   fortyGigE0/20  routed    down       up     N/A         N/A
 Ethernet24          1,2,3,4         40G   9100    N/A   fortyGigE0/24  routed    down       up     N/A         N/A
```

## SONiC node VXLAN configuration

The configuration is a simple Layer 2 EVPN where access port Ethernet4 is an untagged port belonging to VLAN 100, which is part of VXLAN 100 (that uses VNI 100) and the VTEP endpoint is the loopback interface (10.0.1.1)


```bash
    "VLAN": {
        "Vlan100": {
            "vlanid": "100"
        }
    },
    "VLAN_MEMBER": {
        "Vlan100|Ethernet4": {
            "tagging_mode": "untagged"
        }
    },
    "VXLAN_TUNNEL": {
        "VXLAN100": {
            "src_ip": "10.0.1.1"
        }
    },
    "VXLAN_TUNNEL_MAP": {
        "VXLAN100|map_100_Vlan100": {
            "vlan": "Vlan100",
            "vni": "100"
        }
    }
```


## SONiC node BGP configuration

The FRR configuration is straighforward, 2 BGP peerings, one eBGP acting as an underlay and one iBGP as the overlay where the spine acts as an iBGP route reflector, the [`FRR BGP configuration`](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/configs/leaf1-frr-bgp.cfg) file is self explanatory






