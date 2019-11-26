# Controlling agent updates

The agent is considered an extension of the service.
As such, we have an auto-update mechanism to ensure that new features can rely on the agent support they require.
Conceptually, it's no different from binaries on the app tier or JavaScript served to the web frontend.

For instance, tasks express a minimum agent version required.
If a selected agent doesn't meet that requirement, the agent is first asked to update itself.
Other ways an agent can be updated include using a feature which demands an evergreen agent (YAML pipelines, for instance) or clicking the "upgrade all agents" button in a pool.

For some highly-regulated customers, this kind of evergreen infrastructure is at odds with their security/compliance needs.
- For instance, Customer "K" runs the agent inside a highly change-controlled datacenter.
Every alteration must be approved by a high-ranking official in their organization after manual source inspection.
Today, they diff between the release tags of their current version and the version being installed, looking for potential security issues or malicious changes.
- Customer "N" isn't as concerned about the security aspect but is in a regulated space.
They must satisfy their auditors that they're aware of all changes taking place inside their datacenter.

Such customers today are doing things like ACLing the agent's installation directory so it can't update itself or recompiling the agent without update support.
This leaves us in a situation where no one is happy:
we can't service the product correctly, and the customer has to maintain custom configuration / infrastructure to meet their compliance needs.

## Solution

_See below for other solutions considered._

We will add a mode to the project-level pool which puts it into "no update" mode.
In this mode, jobs will use the minimum agent version as a demand.
If no agent can satisfy the demand, the job will fail instead of forcing an agent update.

The error message must clearly indicate what feature or task demanded agent update.
It must also indicate that the customer opted into this behavior.
This way, the customer can decide whether to update their agents or edit the pipeline to remove the offending feature.
For example:
> This pool doesn't allow agent updates.
> An administrator can alter this setting on the project pool page.
> Task `MyTask` version 1.2.3 requires agent version 2.175.1, which could not be satisfied.

Manual agent updates (i.e. clicking the button in the web) should still send update requests as normal.

Additionally, the pool UI will be updated to show the agent's version.
Any agents not fully up to date (as compared to the version on the AT) should be visually marked.
(Not being 100% up to date isn't necessarily a problem, so it should info-level, not warning-level.)

## Potential problems & solutions

**If a small number of agents have been updated, then all jobs will funnel to that agent instead of spreading out across the pool.**
This behavior must be well-documented, both in user-facing material and in troubleshooting guides for the team.
We will inevitably see instances where this setting is on and causing a pool of agents to sit idle while a single agent in the pool is backed up.

**Control over update timing remains with the server.**
That may not be enough assurance for some customers that the bits won't slide out from under them.
Those customers may still recompile the agent or alter its permissions.
The situation is still improved, however, since the failure state will be predictable and observable (failed jobs) instead of nondeterministic and hard to diagnose (agents get sent an update message and simply go offline forever).

## Other solutions considered

### Lag time between ship & require
We considered altering our release process so that we ship the agent well in advance of requiring it.
This would give time to get the agent bits approved/whitelisted as the customer needs.
Of course if we have a critical bug, we'd have an exception process.

This approach constrains how and when we ship features.
For instance, multicheckout required agent changes and already took two sprints to fully deploy.
Adding another sprint or more of lag time would have meant taking basically an entire quarter to ship the feature.

It also doesn't truly address the customer concern.
In this model, the server still requires an update on its whim.

### Long-term support branch
We also considered a "long-term support branch" (LTSB) notion, tagging certain versions of the agent.
If an agent is tagged "LTSB", we either wouldn't auto-upgrade it at all, or we'd only autoupdate it to the next LTSB branch.
As above, we'd give a long bake time between releasing an LTS build and requiring it.

This approach would require a lot more discipline and tracking around shipping the agent.
We're not resourced for that level of planning, and the existing upgrade mechanism isn't really amenable to it anyhow.
If we chose the "wouldn't auto-upgrade LTSB agents at all" path, we've complicated the solution we chose without materially improving the customer experience.
