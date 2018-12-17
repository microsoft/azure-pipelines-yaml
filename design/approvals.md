# Approvals

As we design approvals for the unified pipelines, here are the scenarios to consider.

## Protected environment

The main reason for approval is to gate the deployment to an environment so that a human can verify whatever he/she needs to verify. In Azure Pipelines, a person responsible for managing deployments to an environment may seek an approval every time a deployment is about to be initiated to that environment. When you run a pipeline, the approver would want the run to pause before entering the stage where the environment is being deployed to. The approver can then review the information pertaining to the run, and then approve/reject the run to proceed into the stage.

## Other protected resources

Besides an environment, there are many other forms of resources and  secrets in Azure Pipelines that need protection and possible approval prior to their use in a stage. Examples include service connections, variable groups, agent pools, and secure files. For example:

- A stage in a pipeline uses a script to deploy to a PaaS website. The credentials to the PaaS website are stored in a variable group. The owner of the website requires a pause and approval prior to the deployment.
- The packaging stage in a pipeline uses a special agent pool to sign the binaries. Each agent in that pool has the signing certificate installed on it. Prior to the execution of that stage, a build manager requires approval to be granted to verify that no malicious code gets access to that agent certificate.

## Information to help with approval

Whenever an approval is required, the person may grant the approval or reject it. The following information helps the person make this decision:

- What is the quality of the artifacts produced in this run? Have all tests passed?
- Has the code been reviewed by another person?
- Who else has reviewed and approved this run prior to me (either on this stage or on an earlier stage)?
- Once I approve this, which stages does this run go into? Where will it stop? Will it require additional approvals down the line?
- What are the code changes (commits) that are part of this run?
- What are the work items that are being completed as part of this run? Are they completed fully, or are they partially done?
- Are there new work items (bugs) that have been filed on this run?

## Multiple protected resources

A single stage may deploy to multiple environments or use multiple protected resources. All of those resources might require an approval by the same person.

## Multiple approvals

A single resource used in a stage may require approvals for at least two persons to satisfy compliance requirements of an organization.

## Sequence of approvals

Each stage in a pipeline may require an approval. However, for convenience, an approval may only be asked for once, and that approval might be valid for the duration of that entire run.

## Who can approve

Compliance needs may require that the same person initiating the change (code change, starting a run, etc) not approve the run.

## Retry failed run

Users of Azure Pipelines are able to restart a failed run half way through. In such cases, the run resumes from where it failed and continues to the rest of the stages as it passes. At the time of retrying the failed run, a user might be able to change the configuration inputs (for e.g., change the variable value in a variable group). As a result, it may be necessary to seek approval again for the failed stage even though it might have been granted earlier. Also, in this case, an approver must have a clear indication that he/she is approving a retry of a failed run.

## Redeploy a previous version (rollback)

Users of Azure Pipelines are able to go back to an older run that has passed in all stages, and restart that in order to override a failed deployment that happened after it. Once again, if a stage in this run requires an approval, then that approval must be granted again. The approver must have a clear indication that he/she is approving a rollback.

## Multiple runs waiting for an approval

Consider a pipeline that has a build stage followed by a deployment stage that requires an approval. As builds complete, they queue up for approval. It might happen that multiple runs queue up for approval, and the behavior of such runs waiting for approval depends on what the users want:

- If this pipeline models a configuration change in production, then each run is independent and non-cumulative, and hence each approval must be granted independently for that change to be deployed.
- If this pipeline models a code change from the same branch, then each runs encompasses the changes from the previous runs, and hence it is ok for all the prior runs to be cancelled once the latest run is approved.

## Release branch deployment model

A number of customers deploy to their environments from the release branches instead of from their master branch (e.g., mseng). In this case, you may be deploying from two different release branches simultaneously into the pipeline. For e.g., releases/m140 may go through build, continue to ring 0, and then stop at ring 1. Another run originating from releases/m139 may go through build, skip rings 0 and 1, and then directly proceed to rings 2, 3, and 4. To accommodate such scenarios, one may make specific changes such as the following at the time of creating a run:

- Skip a few stages
- Terminate the run at a certain stage without going all the way through

In the event that such changes are made to the run, an approver must have clear visibility of the graph so that he can review and approve that run.

## Resource authorization control

A resource may not require an approval per-se, but it may not be authorized for use in all pipelines. In such cases, a run that uses an unauthorized resource may be allowed to:

- Fail
- Pause at the boundary of the stage, so that the resource owner can review, approve, or reject the run

In other words, resource authorization policies may be considered to be a special case of approvals.
