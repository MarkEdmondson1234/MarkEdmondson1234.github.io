---
layout: post
title: Launch RStudio Server in the Google Cloud with two lines of R
comments: true
---

I've written previously about how to get [RStudio Server](https://www.rstudio.com/products/rstudio/download-server/) running on Google Compute Engine: the [first in July 2014](http://markedmondson.me/run-r-rstudio-and-opencpu-on-google-compute-engine-free-vm-image) gave you a snapshot to download then customise, the second in [April 2016](http://code.markedmondson.me/setting-up-scheduled-R-scripts-for-an-analytics-team/) launched via a Docker container.

Things move on, and I now recommend using the process below that uses the RStudio template in the new on CRAN [`googleComputeEngineR`](https://cran.r-project.org/web/packages/googleComputeEngineR/) package.  Not only does it abstract away a lot of the dev-ops set up, but it also gives you more flexibility by taking advantage of `Dockerfiles`.

## Launching an Rstudio Server

This example is taken from the [example workflows](https://cloudyr.github.io/googleComputeEngineR/articles/example-workflows.html#custom-team-rstudio-server) that are on the [`googleComputeEngineR` website](https://cloudyr.github.io/googleComputeEngineR), which includes other examples for Shiny, OpenCPU and R-clusters.

You do need to do a bit of [initial setup](https://cloudyr.github.io/googleComputeEngineR/articles/installation-and-authentication.html) to setup your Google project and download the authentication file, but after that you just need to issue these two commands:

```r
library(googleComputeEngineR)
vm <- gce_vm(template = "rstudio",
             name = "my-rstudio",
             username = "mark", password = "mark1234",
             predefined_type = "n1-highmem-2")
```

And thats it.  Wait a bit, it will output an IP address for you to log in with.

![rstudio-googleComputeEngineR-launch]({{ site.baseurl }}/images/rstudio-launch-example.png)

![rstudio login]({{ site.baseurl }}/images/rstudio-login.png)

You can now carry on by logging in and installing packages as you would on RStudio Desktop, then use [`gce_vm_stop(vm)`](https://cloudyr.github.io/googleComputeEngineR/reference/gce_vm_stop.html) and [`gce_vm_start(vm)`](https://cloudyr.github.io/googleComputeEngineR/reference/gce_vm_start.html) to stop and start your instance, or if say you are on a Chromebook and cannot run R locally, use the Google Cloud Web UI to start and stop it. 

## Further customisation

You customise further by creating a custom image that launches a fresh RStudio Server instance with your own packages and files installed.  This takes advantage of some Google Cloud benefits such as the [Container Registry](https://cloud.google.com/container-registry/) which lets you save private Docker containers. 

With that, you can save your custom RStudio server to its own custom image, that can be used to launch anew in another instance as needed:

```r
## push your rstudio image to container registry
gce_push_registry(vm, "my-rstudio", container_name = "my-rstudio")

## launch another rstudio instance with your settings
vm2 <- gce_vm(template = "rstudio",
              name = "my-rstudio-2",
              username = "mark", password = "mark1234",
              predefined_type = "n1-highmem-2",
              dynamic_image = gce_tag_container("my-rstudio"))
```

If you want to go further still, use [`Dockerfiles`](https://docs.docker.com/engine/reference/builder/) to customise the underlying linux libraries and CRAN/github packages to install in a more replicable manner - a good way to keep track in Github exactly how your server is configured.

A `Dockerfile` example is shown below - construct this locally:

```
FROM rocker/hadleyverse
MAINTAINER Mark Edmondson (r@sunholo.com)

# install cron and R package dependencies
RUN apt-get update && apt-get install -y \
    cron \
    nano \
    ## clean up
    && apt-get clean \ 
    && rm -rf /var/lib/apt/lists/ \ 
    && rm -rf /tmp/downloaded_packages/ /tmp/*.rds
    
## Install packages from CRAN
RUN install2.r --error \ 
    -r 'http://cran.rstudio.com' \
    googleAuthR shinyFiles googleCloudStorage bigQueryR gmailR googleAnalyticsR \
    ## install Github packages
    && Rscript -e "devtools::install_github(c('bnosac/cronR'))" \
    ## clean up
    && rm -rf /tmp/downloaded_packages/ /tmp/*.rds \
```

Then build it on your VM via [`docker_build`](https://cloudyr.github.io/googleComputeEngineR/reference/docker_build.html):

```r
docker_build(vm, 
             dockerfile = "file/location/dockerfile", 
             new_image = "my-custom-image")
```

You can then save this up to the Container Registry and launch as before:

```r
gce_push_registry(vm, "my-custom-image", image_name = "my-custom-image"
vm3 <- gce_vm(template = "rstudio",
              name = "my-rstudio-3",
              username = "mark", password = "mark1234",
              predefined_type = "n1-highmem-2",
              dynamic_image = gce_tag_container("my-custom-image"))
```
