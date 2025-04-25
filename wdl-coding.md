# WDL Coding

Some various thoughts on coding approaches when building wdl pipelines. One of my design principles is that the language and conventions should make it as painless as possible to develop a new task/workflow. If it does not, then the conventions should be improved.

## Sample Sheets
The biowdl project is a big inspiration for this approach. They defined by convention (and code) a csv/tsv-based file format for sample sheets. These describe samples, libraries and readgroups as input for pipelines. Much like the tidyverse, it is important to have standard data types that most routines can accept and produce, rather than all custom types.

We have adopted this convention of sample sheet whole-heartedly. As with biowdl, a challenge with wdl is that the native input type is actually json so that conversion of sample sheet to json is actually a step in the pipeline. Or possibly a pre-execution step (see elsewhere in the docs). 

An added benefit of sample tables is that the input to complex workflows is a sample table, not an array of File or other object. This provides much better visibility after-the-fact of what was run. 

## Standardized Samples
This feature took a while to work out. Briefly, files come into a pipeline with many different and original naming conventions. This can cause great grief when it comes to filename collisions (files in different directories having the same names) or interpreting output (what does S1abcd mean?). With the use of the biowdl approach to sample sheets, we implemented a task that will standard filenames using the nomenclature from the sample sheet.

Specifically, we use symlinks to create a flat file structure of the form: `sample_libZ_rgZ.R1/2.extension`. For example, `EXP1RNA2_lib1_rg1.R1.fastq.gz`. Note that we can take the extension of the file in the sample sheet and build the filestem, creating a standardized name. This task (in biowdl.wdl) will create symlinks in the project data directory (reads_standardized) based on input data (from data-raw/imports). Importantly, there is a flag that will follow the symlink to the original file when making these standardized links. For instance, assuming /data/x.fast.gz is symlinked to data-raw/imports/x.fastq.gz. When we run the standardize pipeline, the file data/sample_lib1_rg1.R1.fastq.gz will point to /data/x.fast.gz rather that data-raw/imports/x.fastq.gz. This lowers the number of symlinks for a human to traverse when tracing a file.


## Objects vs Atomic Types
This is an ongoing debate in general. WDL supports using structs, which is a very convenient collection of variables into a single entity. This is a great shorthand to pass around to various functions. And often, working through the data type actually develops the algorithms and workflow out more completely.

However, there are definitely downsides. I have developed very structure-specific code which, after some time, becomes obsolete when new data arrives. The genius of standardizing on table-based data processing avoids many of these problems. The question here is which path to choose? Objects can be directly serialized to files, so this limits some portability concerns. But having pipelines take objects means they are very limited in scope.

While developing some wdl, I evolved the following hierarchy:
- Type w1: tasks take very specific WDL-like code/inputs
- Type w2: workflows take a broader set of inputs but are generally atomic types
- Type w3: objects and higher-level processing workflows

Using this type of hierarchy, one could consider that w3 files (for which I used a `.w3` extension instead of `.wdl`) accept sample sheets and/or manage objects. Once things are sufficiently managed they can be passed off to more traditional workflows (`.wdl`) that manage File, String, etc which are ultimately invoking tasks. As an example, I have a set of fastq pairs: multiple readgroups per sample, for multiple samples. If I write a QC pipeline, then I can read in the sample sheet and scatter over samples/readgroups. Once I get to a readgroup (two File variables), I could call a w2 pipeline (accepting File read1, File read2) that executes a QC pipeline. The `w3` would handle splitting up a sample into readgroups, then calling another pipeline on the readgroups. It would also handle bundling things back up. 

In short, using this type of hierarchy with w3 as boxing/unboxing code for w2 wdl code, led to a natural separation of activities within the wdl code. Management of complex objects is handled in w3, whereas the actual pipeline against a file is handled in w2.

## Parameter Passing
This topic is one that continues to evolve for me. Consider any sufficiently complex task - let's say the STAR aligner. When you look at basic tutorials, the command is simple:
```
star input.1.fq input.2.fq
```
There are relatively few "real" parameters to the program. But there are *many* modifiers. The biowdl tasks solve this problem generally with an optional parameter, often with a default set. This way, a user does not need to specify the parameter unless they really want to and it generally behaves as expected. In particular, if there is no default value then the parameter is not included using the following syntax:
```
~{"--do-something " + varName}
```
This translates to only including the flag when `varName` is set. 

The above description should be expanded. However, one key finding is that there are relatively few "input" parameters and many "optional" parameters (I call them behavior parameters). My thinking is that "input" parameters should be exposed more clearly in the pipeline execution, while "behavior" parameters can be relegated to inputs to the workflow overall (e.g., `inputs.json`). In that file, you can have a complex path to a very in-the-weeds setting:
```
sampleQC.fastqQC.fastq-screen.widget: "blue"
```

This behavior parameter is then well-documented in the inputs file, whereas the file inputs to the task are explicitly passed in the task call:
```
call fastq-screen.fastq-screen {
    input: fastq = file1
}
```
This makes the wdl code simpler and documents behavior (I think) more clearly. Of note, I have been guilty of writing complex wdl in which a parameter is passed through many different workflows just to show up in a specific task. It seems way easier to document this in an inputs file as a behavior variable than track it all the way through execution.

## Task Development

Another innovation from biowdl is the notion that a task will generally wrap an executable. There are instances of custom (embedded) scripting for specific tasks, but it has been very powerful to have a wrapper around an executable. For instance, consider `star`. Rather than have tasks for the various ways you might use star, the idea is to simple have a `star.wdl` with parameters reflecting the various command line options of the program. In this way, you can call star however you want from your workflow without any more coding. This was (to me) a truly revolutionary way to think of tasks.

One modification to the above approach comes from the many commands which have subcommands. For instance, `samtools index` or `samtools view`. These are generally implemeted in `samtools.wdl` as `Index` and `View` separately. While there will be redundancy in code (particularly in parameter handling), it has been quite powerful.

## Task Library
This is fairly simple. I forked the biowdl repo and continue expanding tasks that are of use to me, with the intent to create a pull request at some point. However it is a flat folder of tasks which seems a reasonable approach.

## Workflow Library
This is an area that has not been worked out yet. There appear to be some workflows that are somewhat reusable. However, my thinking is that they should be managed in a way similar to java packages. Prefix them with a domain or the like (e.g., org.moffitt.eschrichlab/rna_alignment.wdl). I am very suspicious that there are truly reusable or generic workflows. There are absolutely resuable tasks (that is the point of biowdl), but workflows are another story. Therefore it may be best to consider workflows as a large universe of (hopefully) non-clashing namespaces that could be reused where possible.

## Optional Execution
This has not been implemented yet, however it is an important concept to develop. Briefly, we can create a "deluxe" version of (for instance) a QC pipeline. However, we don't always want to run everything. Rather than copy the pipeline, comment out specific sections, etc. it would be much easier to simply have indicators to run components (or not). My thoughts were to define a series of "behavior" flags to a workflow (that would be defined within an inputs.json file). These would enable/disable various features. 

BBSR does this for a number of tools and could be broadly implemented, provided optional inputs/outputs can be managed. Of note, this type of optional execution should be distinguished from call caching approaches since this feature should be a per-workflow option rather than an execution-specific operation.

## Call Caching
There are various ways for cromwell to perform automatic call caching. When the inputs and docker image does not change, simply reuse the results of the previous call. In my architecture of modifying a project directory state, caching with no knowledge may be less important. Since we have knowledge of the process there may be a simpler approach to maintain status of success which can then be used to skip recalculating previous results. This has not been implemented although BBSR has a system.