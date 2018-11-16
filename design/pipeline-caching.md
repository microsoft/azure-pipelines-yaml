# Pipeline Caching

** Status: Work in Progress **

We have observed the need for Azure Pipelines to implement a caching mechanism that allows the outputs of steps within the pipeline to be skipped if a suitable cached version of the output already exists.

Provided the cost of determining a cache hit and acquiring the contents of the cache is cheaper than the cost of producing the output again from scratch then the pipeline's performance will be increased.

Caching can introduce many complexities into build pipelines and our goal is to strike the balance between "correct" and "good enough".

## Proposal

TODO: Work in progress.