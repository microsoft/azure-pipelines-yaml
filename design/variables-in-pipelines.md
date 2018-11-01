# Moving variables forward

To bring Build and Release into one "unified pipelines" model, we need to evolve the variables we expose.
The Build.* and Release.* variables no longer make sense.
We also have a unique opportunity to clean up other aspects of the variables we expose.
Though we can make some "breaking changes", we have to give customers, first-party tasks, and Marketplace tasks a clear and easy path forward.

## Problems
* Build.* and Release.* variables aren't meaningful in unified pipelines.
* There are many dozens of variables, some which are similar but not quite the same.
We're wary of adding too many more, since we've run into technical limitations before (size of environment block on some systems).
Having similar-but-different variables increases the burden on customers and task authors to understand the system.
* Some tasks operate differently based on whether they're "running in build" vs "running in RM".
Soon, that distintion won't exist.

## Solution

1. Introduce a new Pipelines.* namespace for variables.
2. Determine which Build.* and Release.* variables need to appear in the Pipelines namespace.
3. Audit in-box tasks for dependencies on being in one or the other environment.
4. Create a new "compatibility mode" flag which injects all the old variables + all the new variables.
5. Define a migration strategy for builds and tasks to tell us when they no longer need compatibility mode.
