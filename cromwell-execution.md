# Cromwell Execution

## Cromwell Wrappers
One challenge with cromwell is that it is a java program. Which involves running java with the cromwell jar file and calling the appropriate function (e.g., `java -jar cromwell.jar run`). We created a `bin` directory with a series of scripts in the form of `cromwell-run`. These are simple shell scripts but standardize the call process. For instance, there is a `cromwell-validate` which in fact uses the `womtool.jar` file for the validate function. However, by standardizing the execution using wrappers the process is more consistent. The current list of commands:

- cromwell-init (currently unused, would setup the skeleton)
- cromwell-outputs (generate workflow outputs in json file)
- cromwell-update (update the released version of cromwell/womtool from github)
- cromwell-yq (yml to json converter)
- cromwell-inputs (generate commented yml file of workflow inputs to stdout)
- cromwell-run (run workflow)
- cromwell-validate (validate workflow)

## Managing execution
Given the wrappers to normalize startup, we can hook into these to perform various startup functions. This is something that has been started but more is needed. Specifically, when executing `cromwell-run` we can check for inclusion of a `workflow-options.json` file, a `inputs.json` file and other files. Note that these files in our setup are not named this way (and start as yml files). 

Therefore at the moment our process consists of
- identify yml files and convert them to json (based on make)
- optionally include these files into the execution command line.

Ideally we can also compile any cromwell configuration and include as well. As a to-do, we should make this process more concrete and explicit so that additional hooks could be added in easily.

## Localization
This topic shows up in a number of different areas, as it intersects with containers and reference data. Here, the point is that we have settled on symlink-based localization. Since the primary goal is to modify a project directory (vs. an execution directory), using symlinks can make the process work. 

There are some very valid concerns regarding symlinks in pipelines. One is that things become less reproducible, particularly when it comes to containers. One approach we have taken is that containers can mount specific host volumes that allow a symlink to be followed (together with the `docker.allow-soft-links: true` option). We have a reference volume runtime attribute and a docker volume runtime attribute that can be used to mount specific directories. We have found this useful to set in the `workflow-options.json` file as a default runtime attribute (the docker volume) so that every container has access to the directory. In this way, we do allow for following paths outside of the container but constrain the specific directories.

A good rule of thumb for the workflow-options.json docker_volumes variable: be as specific as possible. I try to use the project directory, or in the case of raw data originating outside of this directory, as specific as possible to the raw data and project data directories. You can include multiple mounts by separating by commas:
```
docker_volumes: "/foo/bar/directory/fastqs,/foo/baz/otherdir/project" 
```
instead of `/foo`.

Note that this is somewhat enforced (see containers for more details) by the `contain_all` runtime attribute (the docker container can access nothing except what is explicitly mounted).

## Project Root
This is an embryonic notion, but partially implemented. I created an empty file `.cromwell-proj-root` as a marker file (similar to what R has). The idea would that startup scripts could check for or find this file so to be sure they are starting from the appropriate directory. It is not fully implemented yet, unfortunately.

