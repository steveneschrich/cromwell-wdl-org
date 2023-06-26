# Notes on using docker

Docker can be used to grab container images and execute them in docker. You can configure through the backend. However, there is a feature in cromwell that queries the docker hub in cases where a tag like 'tool:1.2.3' is specified to get the sha256 that corresponds to the specific version (since multiple images can be uploaded with this tag).

It is hardcoded to execute "docker image" and "docker pull" (if the image is not local). So in order to hook this for a non-docker hub source we would need to have a "docker" that is prepended to the path prior to executing cromwell. This does work, and it is how I figured out what was happening. A wrapper docker can then grab the necessary information and spit it back to cromwell.

A further complication in this is that apptainer/singularity images (sif) do not seem to contain this sha256. Since the docker image is pulled and converted to sif, there seems no obvious way to connect the docker image and the sif. However, apptainer does do something similar since it uses a cache when possible. More understanding of the [cache](https://apptainer.org/docs/user/main/build_env.html) is needed.


https://apptainer.org/user-docs/3.1/singularity_and_docker.html#running-action-commands-on-public-images-from-docker-hub
https://cromwell.readthedocs.io/en/stable/tutorials/Containers/