# Ideas

Some general thoughts on possible next steps with this type of computing.

## Notebooks
Is it possible that, despite very long-running jobs, we can use computational notebooks? That is, weaving code and text together into a narrative document of an analysis? We are used to jupyter, etc which is a very iterative process of developing a pipeline of sorts. What would a notebook look like here? Each step would need to be cached (or estimated, or placeholder) so that the evolution of a document could occur. When done, we could run the entire process (which could take days), but was there a way to have created the notebook on the go first? One approach would be to subset the data (or samples) in such a way that approximate results would be generated so that the notebook could be constructed. Or simply an effective way of leaving off during the construction of a notebook and re-entering after a long-running job completes.