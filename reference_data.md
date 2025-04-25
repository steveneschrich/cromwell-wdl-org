# Reference Data

Reference data refers to files/directories that genomics workflows often need to refer to the genome, transcriptome, or annotation of molecular products. These files are typically large but are somewhat fixed for a particular run of a pipeline. The question is how should these files be handled within cromwell?

Here we are only considering the use of containerized cromwell solutions. The problem may be easier without containers (or not), however there are benefits to be had by using containers (not least application install/configuration). Importantly, our container solution looks like the following (see the container docs for more details):

- Use docker/podman to build containers
- Use apptainer to run containers

The why behind this involves security and permissions. Dirk Petersen wrote an article on this (https://github.com/dirkpetersen/hpc-containers).

## The Problem
Suppose we have a reference file (fasta file) for the human genome that is needed by an aligner. One can access this file in a number of different ways:

- Create file within the container when building container
- Download file when needed from within the container
- Refer to the file on local filesystem when running the container
- Mount the file from a local filesystem when running the container


There are pros and cons associated with each approach. Some of the dimensions to consider would include
- **performance**: downloading a very large file each time it's needed can be costly
- **relocatable**: when running in a new environment, referring to `/share/referencefile.fasta` is not portable and requires people to find, create, and store this file
- **reusable**: creating images containing specific reference files limits the applicability of the container to a very limited range of uses.
- **scalable**: running on a toy example should not be too painful (setup, etc) but the same solution should ideally scale to hundreds/thousands of samples or pipelines.

## Options

### Create the file within built container
I first rejected this notion because it leads to very customized containers. For instance, snap-aligner-with-human-38:2.0.1. In reality, it's not so obvious that it is a bad choice. In particular, when using containers if the last layer is the reference data then most code layers will be shared among all versions. It does however lead to very large container sizes, which could be problematic, as well as very specific containers to select. Therefore it fails in the reusable criteria.

### Download file when needed
Obviously this is simple - include a URL (perhaps within WDL) from which the reference file is downloaded. This would happen as part of the workflow. Great idea, although it means continually fetching a multi-gigabyte file (for instance) across 1000 sample pairs running in a WDL scatter. Not a performant option. And in the case of some sites, this could either involve significant costs (egress fees) or getting blocked (ncbi). Like other solutions here, it is simple but ultimately not a scalable choice. Therefore it fails on the performance criteria.

### Refer to the file on the filesystem
This approach is perhaps the easiest to conceptualize. Namely the user downloads the right reference files somewhere and then refers to these files from within WDL. When the file referred to is used a `File` variable, it will get localized into the WDL pipeine appropriately. The first choice is usually to create a hard link, so there is very little cost associated with this approach. We have done this at MCC with reference files. An interesting issue we noted is that there are filesystem limits on the number of hard links to a file. Believe it or not, we have reached these limits. Thus this approach fails with respect to relocatable and scalable criteria. Soft links avoids these problems, but does create others.

### Mount the file in the container
As noted above, we are focused on using cromwell with containers. A side benefit of this approach is that when starting up a container you can specify a file (or directory or filesystem) to mount within the container. This is an interesting solution as it allows us to have a fixed path within the container (e.g., /reference) no matter where the file actually lives. And the reference file can be an arbitrary version. Thus we can download the reference file somewhere in the filesystem to avoid performance issues, but the WDL pipeline would refer to the files in the same directory structure (relocatable). Further, it is reusable since the mounted files can be different underneath (by using a different volume). And finally, the solution appears to be scalable since the files would/should be mounted read-only. Note also that there is no localization performed using this approach since the container management handles this job. Which means that less data is moved around and the number of hard links cannot be exceeded.

An interesting negative to this solution recently came up for me. While mounting a /ref in the container is straightforward, it is not clear how runtime attribute size calculations would work. If the size of a reference file is used to scale requested RAM (for instance), then you could not know this answer prior to container execution (which provides reference to the file). A minor issue, but a real one (for large reference genomes).

### Note on File Localization
Within cromwell, there are options (configurable through the backend configuration file) to specify which localization approaches are used (and in what order). There are hard links, soft links and copying files. I did find some documentation which (obviously, perhaps) suggests that tasks/workflows using docker keywords **exclude** the soft-link option. This makes sense since the working directory of the job is mounted inside the container, which means that symlinks cannot be followed. This means effectively that hard links are made regardless of the filesystem (by default). One thing we have observed is that you can have too many hard links, causing processes to fail. Of note, by mounting (binding) a file into the container you avoid the hard link question. 

**Update** Cromwell now supports the ability to allow symlinks when using docker containers. This is noted elsewhere in my writeup, but briefly if we limit the number of external filesystems we access (in reference cases or in project data file access cases) then we can simply mount the "project" directory and "reference" directory to the container. Symlinks would resolve to these directories within the container. This is a hybrid solution of direct-mount a volume (like a squashfs) and mount an existing directory. Localization (if needed) can still occur but we can control the scope of accessible files/directories from within the workflow.

## Exploring a Solution
Given the potential solution of mounting files/directories into a container for accessing reference files, we explore some of the potential options for implementation as well as considerations.

### Docker volumes
I researched docker volumes. This seems like it could work out, however it appears that docker volumes are not "first class" citizens in the same way that containers are. You cannot push these to repos or pull them from repos. You can export/import but this is definitely not the same semantics. Docker has documentation [here](https://docs.docker.com/storage/). Having some type of docker volume approach but perhaps with dedicated "data" dockerhubs seems like a better strategy. In particular, docker (and apptainer) have support for caching images thereby reducing the cost of continually refetching the same data. 

Ultimately, our WDL approach is to use apptainer for running containers which would require conversion of data out of docker volumes into other formats. Since there is no benefit to docker volumes (pushing/pulling), there is no real benefit here.

### Bind Mounts
Singularity (apptainer) and docker/podman can mount (bind) filesystems from files or from squashfs filesystems (volume in a file). They document it [https://apptainer.org/docs/user/main/bind_paths_and_mounts.html](https://apptainer.org/docs/user/main/bind_paths_and_mounts.html). 

The most straightforward option is to simply map an existing file into a specific file in the container (e.g., /etc/motd -> /etc/motd). This could be done in the case of a single file (/ref/human.fa -> /ref/human.fa). The extension of this approach is to simply mount directories (/my/reference/location -> /ref). Then the container/wdl runs via the /ref uri which can be hard-coded due to flexibility in mounting.

Moreover, you can also mount files that represent whole filesystems. The idea is that a file (e.g., human-ref-1.0.squashfs) represents a filesystem with reference files. One then mounts this file as filesystem (like a docker volume) in the container and off you go. This [paper](https://arxiv.org/pdf/2002.06129.pdf) describes the type of architecture that makes sense if I could figure out how to distribute, download, version the data.

The bind mount approach appears to be the most promising direction since you can either use a single file or multiple files bundled together into a single filesystem that is then mounted when starting the job. And importantly, these can be compressed (squashfs) and read-only (via mount flags) to avoid accidental overwrites, etc. The approach requires integration into cromwell (see below), but then provides for most of the needs for these types of files.

### Cromwell filesystems
Finally, cromwell has [documentation](https://cromwell.readthedocs.io/en/stable/filesystems/Filesystems/) about filesystems that are supported in the software. In the end, I don't think this is necessary although one could imagine something like a dvc-backed (see below) filesystem that is remote (via url) and managed for local cached access.


### Outstanding Issues
However, a few outstanding things must be considered:

- Distributing a file/directory/squashfs. In a shared filesystem, we can have a single location for the squashfs that everything mounts. And there are various repositories/archives for this type of information. But accurate naming and uris would be required, something that I have not really tackled. 
- Downloading the file/squashfs. The first time you want to use it in an environment, the user would have to manually download it. It would be nice to have some type of caching (e.g., refer to the URL and the system can decide if it needs to download, based on md5 or the like). This is somewhat like a "first-class" docker volume, meaning something that is versioned, checksummed, cacheable and pullable. I assume this will develop over time.
- Versioning. If the file/squashfs changes remotely, how would you know and update this? In the case of docker, checksums/versions can be used - how would we do this absent that type of infrastructure?

#### Data versioning
One thought was that tools like [dvc](dvc.org) could be used as the "docker" equivalent for data. The WDL could include explicit (or perhaps via variables) data file names that can be retrieved by a cromwell backend if not locally cached. This data can then be mounted into the container on execution. The backend configuration could include how to access (via dvc) the reference file while the WDL (or WDL variables) could be used to specify the file and version. But this would require significant development.


## Solution
In the end, creating a squashfs filesystem or a local reference directory consisting of the necessary reference files seems the best approach if an individual file is not sufficient. Some benefits:

- It allows the reference files to be external to the containers so that reference reuse is very doable.
- It keeps containers smaller
- It allows tasks to specify which version of reference to use at runtime.

Making a squashfs is simple:
```
mksquashfs data/snap-2.0.1 data/snap-hs37d5.squashfs
```

And having `data/snap-2.0.1` available directly is even simpler (after downloading the data and begging for more disk space).

I will detail how I integrate this filesystem into wdl below. But it is worth noting that while this solves my initial problem, there is one final aspect that remains unsolved. It would be great to have a caching approach (similar to docker) where the referenced squashfs could be queried remotely, compared against a local checksum, then downloaded if needed. Creating some kind of caching mechanism would greatly improve a number of steps. 

### WDL Solution
In order to accomodate mounting reference volumes in cromwell, I created a few conventions. I added a new `runtime` variable for tasks. Actually, I created two (for different purposes): `docker_volumes` and `reference_volume`. The first is designed as a general mount directory (probably should be renamed to project_volumes or something). The second is the purpose here.


```
reference_volume: "data/snap-hs37d5.squashfs:/ref:image-src=/"
```
or
```
reference_volume: "data/snap-hd37d5:/ref"
```

This variable can be added to a task such that `/ref` is accessible to the task as the directory holding reference files. If a specific named file (e.g., `human.fasta`) is required, we could include a hard-coded path for it:
```
File the_ref_file = "/ref/human.fasta"
```
In this case, whatever source directory is used it is always mapped to `/ref`.


**Note**: The biowdl project has a good solution for these types of configurations. These should be specified as a variable to the task (String) so that it can be overridden as needed. Then the runtime would be:
```
reference_volume: referenceVolume
```
and the task would have a parameter of type `String referenceVolume` which could be set for a specific call within wdl.

### Backend Cromwell Support

The reference_volume is only understood if the `application.conf` is customized to accomodate it. That, however, is rather simple. When configuring the call (we use apptainer), include the variable (reference_volume):

```
runtime-attributes = """
            String? docker
	        String? reference_volume
            """
```

and then tweak the submit script:

```
submit-docker = """
              apptainer exec --bind ${cwd}:${docker_cwd} ~{"--bind" reference_volume}
  docker://${docker} ${job_shell} ${script}
            """
```

This will allow us to create a reference volume infrastructure directly into cromwell.
