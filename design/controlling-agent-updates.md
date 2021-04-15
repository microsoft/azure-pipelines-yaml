# Controlling agent updates

The agent is essentially an extension of the service.
As such, we have an auto-update mechanism to ensure that new features can rely on the agent support they require.
At a very high level, it's no different from binaries on the app tier or JavaScript served to the web frontend.

For instance, tasks express a minimum agent version required.
If a selected agent doesn't meet that requirement, the agent is first asked to update itself.
Other ways an agent can be required to update include:
- using a feature which requires an updated agent (YAML pipelines, for instance)
- clicking the "upgrade all agents" button in a pool

Of course, there _are_ differences:
for one thing, deployed agents aren't under our control.
Practically, this hasn't mattered to Azure DevOps as a service provider, since a customer can "only hurt themselves" if they do something wrong or malicious with their agents.

For some highly-regulated customers, this kind of evergreen infrastructure is at odds with their security/compliance needs.
- For instance, Customer "K" runs the agent inside a highly change-controlled datacenter.
Every alteration must be approved by a high-ranking official in their organization after manual source inspection.
Today, they diff between the release tags of their current version and the version being installed, looking for potential security issues or malicious changes.
- Customer "N" is concerned about the security aspect due to being in a regulated space.
They must satisfy their auditors that they're aware of all changes taking place inside their datacenter.
Their auditors, in turn, are concerned that a malicious insider could alter the pipelines infrastructure in a malicious way (i.e. run a job on the agent that alters the agent or its host).

Such customers today are doing things like ACLing the agent's installation directory so it can't update itself or recompiling the agent without update support.
We either knock agents offline (the agent says it'll take the upgrade, then never comes back because it can't complete) or builds start queueing up (the agent says it'll take the update, but when it comes back, it's still running the old version).
This leaves us in a situation where no one is happy:
we can't service the product correctly, and the customer has to maintain custom configuration / infrastructure to meet their compliance needs.

## Solution

_See below for other solutions considered._

We will add a mode to the organization-level pool which puts it into "no update" mode.
In this mode, Azure Pipelines will inject the minimum agent version as a normal demand.
(Today, it's like a demand, but if no agent with the right version can be found, the job will get assigned to an outdated agent and that agent asked to update.)
If no agent can satisfy the demand, the job will fail instead of forcing an agent update.

The error message must clearly indicate what feature or task demanded agent update.
It must also indicate that the customer's org opted into this behavior.
This way, the customer can decide whether to update their agents or edit the pipeline to remove the offending feature.
For example:
> This pool doesn't allow agent updates.
> Task `MyTask` version 1.2.3 requires agent version 2.175.1, and no agents with that version are registered.
> An administrator can alter this setting on the organization pool page.

The administrator who set this policy can optionally include a reason, which we'll include in the error string.
> This pool doesn't allow agent updates.
> Task `MyTask` version 1.2.3 requires agent version 2.175.1, and no agents with that version are registered.
> (customer's words, limited to 400 characters)

Manual agent updates (i.e. clicking the button in the web) should still send update requests as normal.

Additionally, the pool UI will be updated to show the agent's version.
Any agents not fully up to date (as compared to the version on the AT) should be visually marked.
(Not being 100% up to date isn't necessarily a problem, so it should info-level, not warning-level.)

Another minor point: we'll stop auto-sliding YAML builds every sprint.
Instead, when we ship a feature that requires a new agent version, that feature will have to request the newer version.

## Potential problems & solutions

**If a small number of agents have been updated, then all jobs will funnel to that agent instead of spreading out across the pool.**
This can happen if an admin manually updates a few agents or clicks the single-agent "upgrade" button in the web.
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

### Agent-side config
We considered making this an agent-side flag.
The agent would report to the server that it's unwilling to accept update messages.

This breaks the update button on the web UI.
In fact, it totally blocks our ability to ever service those agents.
While the truly paranoid like it that way (as evidenced by the lengths they go to in blocking updates), having this be server-controlled keeps more scenarios working as designed.
(For instance, the on-prem feature allowing the AT to offer up a newer agent build.)
