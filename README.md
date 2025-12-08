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

Here there are 2 options
