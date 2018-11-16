# Azure Pipelines agent Node v10 support

## Overview

Currently, the Azure Pipelines agent supports running JavaScript code via the Node handler. This handler runs version 6 of the node runtime, released in April 2016.  As of November 2018, Node v6 is under "maintenance" status.

Because task developers may desire to use features from a newer Node API, and also consume libraries which may no longer support Node v6, the agent will benefit from supporting more up-to-date Node versions.

Node v10 is the latest version under LTS status.

## Architecture

The agent will support a new handler: Node10. Node10 will syntactically look the same as the existing Node handler, with the only exception that Node10 will end up invoking the Node runtime, version 10.

We'll add a minimum agent version demand should the task.json require the Node10 handler.

**task.json sample**

``` json
    "execution": {
        "Node10": {
            "target": "myscript.js",
            "argumentFormat": ""
        }
    },
```

Initially, the agent is going to package both v6 and v10 Node runtimes, and leave the current v6 workflow unaltered. A feature flag `AGENT_USE_NODE10` will be introduced to the Node handler to switch between the v6 and v10 backends.

The Node v10 switching mechanism will be implemented by the existing `NodeHandler.cs` class.

## Rolling out feature

Rolling out Node v10 support will be executed in three phases:

1. New versions of the agents (with node v10) will be deployed to Ring 0. Then, we'll monitor pipeline runs from the build canary, with the feature flag off, and look out for any regressions. This phase is basically equivalent to what we already do for every new build release.

```
"Node10" -> NodeHandler.cs   -> Node v10 runtime
    
    
    
            (Feature flag off)
"Node"   -> NodeHandler.cs   -> Node v6 runtime
```

2. Once we're sure there are no pipeline run regressions, we turn the feature flag on to switch "Node" to the v10 handler in the build canary, and again, look out for any regressions. Simultaneously, we continue deploying the agent to later scale units. Initially with the feature flag off, then turned on.

```
"Node10" -> NodeHandler.cs   -> Node v10 runtime
                                 ^
                                /
                               /
            (Feature flag on) /
"Node"   -> NodeHandler.cs ---  Node v6 runtime
```

3. Once we've turned the feature flag everywhere on hosted, we'll still keep the v6 and v10 runtimes, and the feature flag OFF for the next on-prem release. This is because we don't have control over the tasks that are written against those on-prem environments. After the on-prem release, we'll remove the Node v6 runtime from the agent, in addition to the feature flag.

```
"Node10" -> NodeHandler.cs   -> Node v10 runtime
                                 ^
                                /
                               /
            (No feature flag) /
"Node"   -> NodeHandler.cs ---
```
