# sonic-l2ls-evpn-containerlab

The purpose of this repo is showing how to create a SONiC image to be used in a container lab setup and the configuration required at both the SONiC and FRR levels to deploy a EVPN layer 2 service, where one leaf runs SONiC and the spine and the other leaf run SR Linux, as per the diagram:

![pic1](https://github.com/missoso/sonic-l2ls-evpn-containerlab/blob/main/img_and_drawio/sonic-l2ls-evpn-containerlab.png)

# Donwload the SONiC image

1. Go to the pipelines list: https://sonic-build.azurewebsites.net/ui/sonic/pipelines

2. Scroll all the way to the bottom where "vs" platform is listed

3. Pick a branch name that you want to use (e.g. 202405) and click on the "Build History".

4. On the build history page choose the latest build that has succeeded (check the Result column) and click on the "Artifacts" link

5. In the new window, you will see a list with a single artifact, click on it

6. Scroll down until you see target/docker-sonic-vs.gz name and download it

# Build the SONiC container lab image

```bash
% ls | grep sonic
docker-sonic-vs.gz
```

Here there are 2 options
