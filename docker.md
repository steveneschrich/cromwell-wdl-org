# Containers for Cromwell

Cromwell supports using containers for the computations specified. This is a fantastic way in which application configuration can be bundled up, particularly in the case of dreaded dependencies. To get a piece of software running can involve additional libraries, additional applications and the configuration of paths, etc. Using containers reduces these problems, as the container (image) is preconfigured per the application specs.

Nothing is never quite so simple, as developing containers is a topic in it's own right. And then integrating containers into workflows, particularly workflows on HPC, adds more complications. 

## Container Strategy
One of the first things we addressed when it comes to building/using containers is the way in which to systematically do it. When running on HPC, docker is a non-starter. Fortunately, singularity/apptainer can fill this gap with relatively little loss of functionality. The community appears to have settled on the idea of building out docker containers and then converting them via apptainer:

- Use podman/docker to build docker containers
- Push docker containers to dockerhub repo
- Use apptainer to pull/convert docker containers for cromwell

An article about security related to this approach can be seen here (https://github.com/dirkpetersen/hpc-containers).

## Reusing Containers
Many docker-based containers already exist! For many (if not most) purposes, there is a container for your software of interest. There is a community (see https://biocontainers.pro/) for a great place to start with this. Note that biocontainers developed a number of containers, however they also take advantage of bioconda (https://bioconda.github.io/) for many packages. In either case, the website (https://biocontainers.pro/) has a searchable option that will provide docker URL's for various versions of containers. Do be sure to find (and use) the specific version of interest to you.

Given an existing container at a specific repo, it is very simply to use it in apptainer:
```sh
apptainer shell docker://alpine:latest
```

Within cromwell, a runtime attribute specifies the container to use.
```
docker: "alpine:latest"
```


## Note on Docker-based Caveat in Cromwell
Cromwell supports the use of a `docker` tag in the runtime attributes of a task/workflow. In the application configuration for cromwell, you can override this to use apptainer instead (see the cromwell-config repo for details). 

However cromwell has implemented rerunning (caching) support for workflows. In these cases, a container needs to have the same sha256 in order to reproduce the workflow. Built into cromwell is a feature that queries the docker hub in cases where a tag like 'tool:1.2.3' is specified to get the sha256 that corresponds to the specific version (since multiple images can be uploaded with this tag). It is hardcoded to execute "docker image" and "docker pull" (if the image is not local). In a HPC environment, you may see a number of error messages about not being able to execute docker - this is the reason behind the errors. It does not hurt the current workflow, but would cause additional problems for job caching. 
 
A potential solution to the docker/apptainer problem is to have a docker-compatible command line for apptainer. In particular the "docker image" command. We could prepend this "docker" to the path of the executing cromwell and then translate apptainer information into docker output format for the sake of cromwell. I would imagine this caching behavior would eventually be exposed to configuration.
 
A further complication in this is that apptainer/singularity images (sif) do not seem to contain this sha256. Since the docker image is pulled and converted to sif, there seems no obvious way to connect the docker image and the sif. However, apptainer does do something similar since it uses a cache when possible. More understanding of the [cache](https://apptainer.org/docs/user/main/build_env.html) is needed.

Note that it may be stored in ~/.local/share/containers/cache/blob-info-cache-v1.boltdb. This repo (https://github.com/etcd-io/bbolt) seems to have the necessary code to access/extract from db. So possible we can get the sha256 from this db and go.

https://apptainer.org/user-docs/3.1/singularity_and_docker.html#running-action-commands-on-public-images-from-docker-hub
https://cromwell.readthedocs.io/en/stable/tutorials/Containers/


