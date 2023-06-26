# Reference Data

Reference data refers to files/directories that genomics workflows often need to refer to the genome, transcriptome, or annotation of molecular products. These files are typically large but are somewhat fixed for a particular run of a pipeline. The question is how should these files be handled within cromwell?

## The Problem
Suppose we have a reference file (fasta file) for the human genome that is needed by an aligner. One can access this file in a number of different ways:

- Create file within container when building container
- Mount the file from a local filesystem when running the container
- Refer to the file on local filesystem when running the container
- Download file when needed from within the container

There are pros and cons associated with each approach. Some of the dimensions to consider would include
- performance: downloading a very large file each time it's needed can be costly
- relocatable: when running in a new environment, referring to /share/referencefile.fasta is not portable and requires people to find, create, and store this file
- reusable: creating images with specific reference files limits the applicability of the container to a very limited range of uses


## Comments

### Creating the file within built container
I first rejected this notion and it leads to very customized containers. For instance, snap-aligner-with-human-38:2.0.1. In reality, it's not so obvious that it is a bad choice. In particular, when using containers if the last layer is the reference data then most code layers will be shared among all versions. It does however lead to very large container sizes, which could be problematic.

### Data versioning
One thought was that tools like [dvc](dvc.org) could be used as the "docker" equivalent for data. The WDL could include explicit (or perhaps via variables) data file names that can be retrieved by a cromwell backend if not locally cached. This data can then be mounted into the container on execution. The backend configuration could include how to access (via dvc) the reference file while the WDL (or WDL variables) could be used to specify the file and version. 

### Docker volumes
I have tried researching docker volumes. This seems like it could work out, however it appears that docker volumes are not "first class" citizens in the same way that containers are. You cannot push these to repos or pull them from repos. You can export/import but this is definitely not the same semantics. Docker has documentation [here](https://docs.docker.com/storage/).

### Singularity Binds
Similarly, singularity (apptainer) has binds that can be from squashfs filesystems or the like. They document it [https://apptainer.org/docs/user/main/bind_paths_and_mounts.html](https://apptainer.org/docs/user/main/bind_paths_and_mounts.html).

This [paper](https://arxiv.org/pdf/2002.06129.pdf) describes the type of architecture that makes sense if I could figure out how to distribute, download, version the data.

### Cromwell filesystems
Finally, cromwell has [documentation](https://cromwell.readthedocs.io/en/stable/filesystems/Filesystems/) about filesystems that are supported in the software.
