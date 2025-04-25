# Cromwell Configuration
Cromwell as an executable has a lot of flexibility in it's configuration. That is both very powerful and very difficult to manage. Included below are several areas in which we've tried to address these issues.

## Config Management
There is `application.conf` file which allows various settings to be configured for cromwell execution. We have tried to develop a fairly documented config file (actually a set of files) for this purpose. While not exactly true, one can consider there to be a "default" installation configuration that you could maintain in an application.conf file. 

Below are the highlights of the different configuration pathways. While we have not solved this problem, it does appear that certain settings must be set in specific ways (only in application.conf, etc). Ideally having a centralized settings file (perhaps yml-based) which can then be distributed to each specific location for execution may simplify the approach for the user.

### application.conf
After several iterations, we have settled on a "project" directory approach. We clone a structure into a new project which includes a `conf` directory. Here we have a `cromwell.conf` which is the general configuration for cromwell. Additionally, we have a `conf/backends` directory where there are a number of different backend definitions included. And a `db` directory for the configuration associated with different db options. The hope is that these files are "generally" static but can be tweaked by project as necessary. In particular, one could modify the `cromwell.conf` file to pick a specific db or backend. Finally, we also include a `include local.conf` option for the ability to override settings specifically. In summary,

- conf/
  - cromwell.conf (main config)
  - local.conf (local overrides)
  - backends/ (definitions of backends)
  - db/ (definitions of databases)


### java startup
A further complication to cromwell configuration is that it is a java application. As such, one specifies the config file (from above) using a java define. And there are several logging parameters that only appear to be set from java defines. We have a `bin/cromwell-run` script that wraps these settings and can be modified as needed for the project. A more complete implementation would be to have a `conf/env` type file that has these settings which would be incorporated into `bin/cromwell-run` dynamically. 

### Workflow options
Finally, there is a JSON file that can be incorporated on execution to control some workflow-related options. One can set the backend here. We use it to set the default `docker_volumes` directory that gets mounted for all containers executing tasks (e.g., in [reference_data](reference_data.md)). 

Of very specific note, I was not able to find a way to override the backend configuration on the number of concurrent jobs via the workflow options. Looking at the source code, the only way is to define this in a `local.conf` override. For example
```
backend.providers.slurm-apptainer.config.concurrent-job-limit=100
```

## Input management
One of the excellent strategies that biowdl adopted was the use of `dockerImages`. It is a map with a key (the container name) and a docker reference. That way, when buried deep within a task, one can refer to this map to get the relevant container to execute. By setting up a map in `dockerImages.yml`, we can control container execution throughout a cromwell execution. This is one (of several) example of input data/files that are provided to a workflow. The most obvious example is `inputs.json` which can be used to drive the workflow.

Although not completed resolved, we have implemented the following approach. Input files are in `yml` format and converted (automatically) to json for internal cromwell use. The reason is that yml is far easier to document with comments and less messy when it comes to human-readable format. It should be noted that json is not designed to be human-readable so this is not surprising.

At present, we create yml files in the `conf` directory as inputs for a workflow. To hook into the system (below), we enforce a `conf/filestem.yml` config file for use in the `filestem.wdl` workflow. The conf directory also holds the `dockerImages.yml` file. There is also a `Makefile` in the conf directory with a hook to `bin/cromwell-yq` which converts yml to json output. Thus when you execute `bin/cromwell-run` with a workflow, the Makefile is used to build the json files before starting. 

This is also a work in progress. We will likely migrate the inputs yml file to co-locate with the workflow file. And as mentioned earlier, consolidating all configuration into yml (with converters to JSON) would be a far simpler configuration option for users. But this has not yet been implemented.


