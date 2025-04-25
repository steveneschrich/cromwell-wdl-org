# Cromwell/WDL Organization

Cromwell/WDL is a set of tools for managing genomics-related workflows, which are a series of complex bioinformatics tools operating on large sets of data. It is developed by the Broad Institute and is available on gitub (https://github.com/broadinstitute/cromwell). The Workflow Description Language (WDL) itself has become a standard for multiple engines (including cromwell) to implement (https://github.com/openwdl/wdl/). 

As a tool, cromwell (and WDL) is powerful for codifying the specific tasks that are required in a complex, multi-step process that is endemic to bioinformatics. Ideally, these tools provide the opportunity for reproducibility, consistency and automation. 

Like all powerful tools, however, there are challenges. In the case of cromwell and WDL (independently), there are a number of use cases that have to be met. As a result, the tool can be highly multi-purpose but also difficult to scale up in complexity. I have seen this coding in general; some languages are very amenable to scaling up particularly by encouraging/enforcing various standards even in the more simplistic development tasks. But think of a complicated C development process with automake, etc. To go from `helloworld.c` to a massively complex application is not a linear (or obvious) process. That is my impression in utilizing cromwell/wdl as a tool in this space. It provides a lot of power but also needs conventions for making development and use easier/simpler. 

A notable aspect of the openwdl effort is the statement of some values (https://docs.openwdl.org/overview.html#values). While I share many of these values, the implementation remains an evolving thing. I fully agree with avoiding flexibility at all costs. Despite many frustrations, having a limited number of options for expressing a particular intent can at least simplify the idioms that are used (and read) when developing and using workflows.

This repository is my attempt at various strategies and mental models associated with building, running and maintaining cromwell/WDL pipelines. There are some excellent sites that have worked on this problem, most notably biowdl (https://biowdl.github.io/). My opinion is that a healthy degree of convention would allow for broader use, development and adoption. 


One of (I hope) my contributions to this process is a categorization of all of the different topics that could use some consideration of conventions.

- Cromwell Configuration
  - Config files
  - Logging
  - Server mode vs. database
- Cromwell Execution
  - Wrappers
  - Startup Management
- Containers
- [Reference data](reference_data.md)
- 
- WDL Code Organization
- WDL Coding
  - Optional Execution
  - Task Development
  - Structs
  - Serialization
  - Parameter passing
  - Sample Sheets
- [Philosophy](philosophy.md)
  - Separating Bioinformatics and Engineering
  - Isolation vs. project-directory execution
  - Piecewise vs all-in-one (research vs. production)
- Project Directory
  - Structure
  - Symlinking




- https://broadinstitute.github.io/warp/docs/About_WARP/VersionAndReleasePipelines
- https://gatk.broadinstitute.org/hc/en-us/articles/360035890811


- [Backends](backends.md)
- [Docker](docker.md)
