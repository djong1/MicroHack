# Walkthrough Challenge 09 - Model, review, and deploy with Radius Canvas

[< Previous Challenge](../../challenges/challenge-08.md) - **[Home](../../Readme.md)** - [Next Challenge >](../../challenges/challenge-10.md)

Duration: 75-120 minutes

## Coach notes

Radius Canvas entered public preview on 3 September 2026. It is a canvas extension for the
GitHub Copilot app, shipped inside a plugin named `radius`, with source in
[`radius-project/ai-extensions`](https://github.com/radius-project/ai-extensions).

The teaching value of this challenge is the contrast, not the tool. Challenges 03-08 built
Adaptive Apps from the platform side: the team authored contracts, implemented them with
recipes, and drove `rad deploy` against persistent control planes. Canvas approaches the
same application from the developer side and automates the parts the team just did by
hand. Participants should leave able to say exactly which of those parts it automates and
which it does not.

Three misconceptions to head off early:

1. **Canvas is not a second way to reach the Challenge 05 environment.** It creates its own
   ephemeral Radius control plane inside a GitHub Actions runner for every run and keeps
   state in a private GHCR package. It never talks to `env-azure-prod`.
2. **Canvas does not make the application portable.** Resource types, recipes, and
   environments do that, and they are Challenge 03 and Challenge 04 work. Canvas changes
   who authors and reviews the model, and it gives Azure a fast on-ramp.
3. **The preview is Azure-only.** The published guidance is explicit: it supports
   containerized applications deployed to Azure, with AWS "coming soon". There is no K3s,
   Azure Local, or Arc target. Do not let a team burn an hour trying to point it at
   `k3s-azure-vm`; establishing that boundary *is* Task 6.

> [!NOTE]
> This is a preview capability under active development. Menu labels, generated file names,
> recipe pack contents, and view details will move. Coach the participants to record what
> they observe rather than to match a screenshot in this document. The behaviours this
> walkthrough relies on, listed in "Stable claims" below, are the ones the challenge is
> actually graded against.

### Stable claims this walkthrough depends on

| Claim | Where it comes from |
| --- | --- |
| Install from the Copilot app's plugin catalog, then restart the session | Plugin and docs guide |
| Modeling requires a Dockerfile and writes `.radius/app.bicep` | `radius-app-bicep` skill |
| The graph is `rad app graph` output rendered in the canvas | `radius-app-graph` skill |
| Four views: Modeled, Planned, Deployed, Diff | Extension README |
| Planned resolves a recipe per resource from a provider recipe pack | Core recipe resolver |
| Deploy commits and dispatches a GitHub Actions workflow | Docs guide and `radius-deploy` skill |
| The control plane is an ephemeral k3d cluster inside the run | `radius-deploy` skill |
| Trust is OIDC; the environment holds Actions variables, not secrets | `radius-environment` skill |
| Preview is Azure-only and applications need a Dockerfile | Docs guide preview banner |

### Environment hygiene

This challenge runs on the **host**, not in the devcontainer, because the canvas only runs
in the GitHub Copilot app. Confirm before the session that participants have `az`, `gh`,
and `kubectl` on the host and that `kubectl` reaches `aks-adaptive-apps`.

Have every team fork the trading application source and work on a branch. Nothing in this
challenge should be pushed to `microsoft/adaptive-apps`.

## Fixed target map

| Path | Radius control plane | Cluster | Namespace | Model |
| --- | --- | --- | --- | --- |
| Challenges 05-08, Azure | Persistent `ws-azure-prod` / `env-azure-prod` | `aks-adaptive-apps` | Challenge 05 application namespace | `iac/app.bicep` |
| Challenges 05-08, Local | Persistent `ws-local-prod` / `env-local-prod` | `k3s-azure-vm` | Challenge 05 application namespace | `iac/app.bicep` |
| Challenge 09, Canvas | Ephemeral `radius-cp` k3d cluster in the workflow run | `aks-adaptive-apps` | `trading-canvas` | `.radius/app.bicep` |

## Stage 1: Install the plugin and model the application

### Install

In the GitHub Copilot app, open **Customize** in the side menu, select **Plugins**, search
for `radius`, and install it. Restart the Copilot session so the skills and the canvas load.
The plugin's three-dot menu handles update and uninstall.

Verify the host prerequisites first:

```bash
az login
gh auth login --scopes read:packages,write:packages,workflow
kubectl config use-context aks-adaptive-apps
kubectl cluster-info
```

The Radius CLI is deliberately absent from this list. The extension manages its own binary
at `~/.radius/ai-extensions/bin/rad` (`%USERPROFILE%\.radius\ai-extensions\bin\rad.exe` on
Windows) and does not resolve `rad` from `PATH` or from the `~/.rad/bin` installation used
in earlier challenges. Point this out: it is why a broken `rad` in the devcontainer is not
the explanation for a Canvas problem.

### Model

Start a session with the fork selected as a GitHub repository, then ask:

```text
Show the application graph
```

The modeling skill analyses the repository and publishes three things:

| Artifact | Purpose |
| --- | --- |
| `.radius/app.bicep` | The generated application definition |
| `.radius/app.origin.json` | Origin record: which source produced the model, and a hash of the compiled bytes |
| `.radius/bicepconfig.json` | Resolves the Radius Bicep extensions the definition uses |

Two behaviours are worth surfacing:

- Modeling runs into a staging directory and is promoted atomically. A run that cannot
  produce a compiling model refuses to publish rather than leaving a half-written one.
- The canvas reads the origin record before rendering. Without it, the only question the
  canvas could answer is whether `app.bicep` exists, so a model whose source has moved on
  would render as though it were current. If the model was edited by hand, the skill asks
  before overwriting.

### Expected Modeled view

For the trading application, expect roughly:

- One application resource named after the repository.
- A container image resource per Dockerfile under `src/`, each pointing at its Dockerfile.
- A container per service: frontend, backend, and the AI agent.
- A PostgreSQL database resource, detected from the backend's database client
  initialization and the compose file.
- Connections from the backend to its backing services.

Have teams click **View source code** on each node. A container's source reference points
at the process entrypoint, not the Dockerfile; a container image's points at the Dockerfile.

Two absences are the most useful part of the exercise, and both are correct behaviour:

- **No route.** The modeler adds ingress only on explicit evidence. A Dockerfile `EXPOSE`,
  a health endpoint, a published compose port, or a component named "gateway" are
  deliberately not sufficient. Challenge 05 also declared no ingress and used
  `rad resource expose`, so the two models agree here for different reasons.
- **No workload identity.** There is no predefined type for it. See Stage 2.

## Stage 2: Reconcile the two models

This is the analytical core of the challenge. The generated model is authored against a
fixed allow-list of predefined `Radius.*` types. The Challenge 03 catalog exists precisely
because that allow-list does not cover everything a real platform needs.

### Model mapping

| Component | Canvas generated type | Challenge 05-08 type | Same intent? |
| --- | --- | --- | --- |
| Application | `Radius.Core/applications` | `Applications.Core/applications` | Yes; the newer namespace |
| Frontend, backend, AI agent | `Radius.Compute/containers` | `Applications.Core/containers` | Yes |
| Container builds | `Radius.Compute/containerImages` built from each Dockerfile | None; Challenge 05 consumes published `ghcr.io/microsoft/adaptive-apps` images | **No.** Canvas builds from source; Challenge 05 deploys an immutable published tag |
| Trading database | `Radius.Data/postgreSqlDatabases` | `Radius.Resources/postgreSqlDatabases` | Same intent, different owner of the contract |
| Message broker | No predefined MQTT type exists. The allow-list offers `Radius.Messaging/kafka` and `Radius.Messaging/rabbitMQ`; an unmatched need becomes a generated custom type under `Radius.Resources` | `Radius.Resources/mqttBrokers` | **No predefined equivalent** |
| Workload identity | No predefined type | `Radius.Resources/workloadIdentities` | **No predefined equivalent** |
| AI model | `Radius.AI/models` if the source shows a model endpoint | `Radius.Resources/aiModels` (Challenge 08) | Similar intent, different contract |
| Secrets | `Radius.Security/secrets` | Recipe-created Kubernetes Secret bound by `secretRef` | Different mechanism |
| Ingress | `Radius.Compute/routes`, only with explicit evidence | None; `rad resource expose` | Both decline to declare it |

### Model answers

**Which Challenge 03 contracts have no predefined equivalent?**
`mqttBrokers` and `workloadIdentities`. This is the headline finding. The trading
application's MQTT broker and its Entra workload identity are exactly the capabilities the
firm's platform team had to define itself, and they are exactly the two the generated model
cannot express with a predefined type. When the modeler has an unmatched need it generates
a custom type under the `Radius.Resources` namespace, which is the same namespace the team
used in Challenge 03 — arriving there from the other direction.

**Which capabilities are missing, and why?**

| Missing | Reason |
| --- | --- |
| Ingress / route | Deliberate conservatism; no explicit external-client evidence |
| Workload identity | No predefined type, and identity is not visible as a dependency in source |
| OIDC sign-in configuration (Challenge 06) | Configuration, not structure; it is parameters on a container, not a modeled resource |
| Service-mesh authorization (Challenge 07) | A platform policy concern that does not appear in application source at all |
| MQTT authentication method | A property of the broker recipe, which is why Challenge 05 read it from the resource rather than hard-coding it |

**What does the source reference buy?**
Every non-application resource carries a reference to the file and line where the resource
was detected, and the canvas turns it into a clickable link. It makes a reviewer able to
challenge the model. It is metadata only and does not affect deployment.

**Contract versus observation.**
`iac/app.bicep` is written against contracts the platform team controls, so it is a
statement about what the application is allowed to ask for. `.radius/app.bicep` is a
statement about what the source appears to need. The first survives a move to a platform
the modeler has never seen, because the contract is the thing the new platform implements.
The second has to be re-derived, and can only use types someone has already predefined.
This distinction is the answer to Task 6 and is worth landing hard.

## Stage 3: Environment, OIDC trust, and the plan

### Create the credential profile and environment

A credential profile is a reusable set of Azure tenant and subscription details used to
authenticate and to configure the GitHub-to-Azure OIDC trust. Create the profile, select
Azure as the provider, enter the tenant and subscription, verify, and save.

Then create the environment: name it, pick the GitHub account and the saved profile,
accept or adjust the pre-populated Entra app registration name, and select the lab
resource group, the `aks-adaptive-apps` cluster, and a namespace. **Enter `trading-canvas`.**
Do not select the Challenge 05 namespace.

### Account for what was created

```bash
# The GitHub environment and its variables. Expect variables, and no cloud secrets.
gh api /repos/<owner>/<repo>/environments --jq '.environments[].name'
gh variable list --env <environment-name> --repo <owner>/<repo>

# The generated workflows now in your repository.
gh workflow list --repo <owner>/<repo>

# The private state package.
gh api /user/packages?package_type=container --jq '.[].name'
```

Expected variables include the state-backend settings (an OCI backend, a per-environment
GHCR registry path, and an archive name) and the Azure targeting values: client ID, tenant
ID, subscription ID, resource group, and AKS cluster name. There is also an optional route
exposure setting that defaults to private.

Make the point explicitly: these are Actions **variables**, not secrets. The verification
workflow reads only variables. Nothing here is a credential, because the credential is
minted per run by OIDC.

Inspect the federated credential:

```bash
az ad app list --display-name "<entra-app-name>" --query "[].{id:id,appId:appId}" -o table
az ad app federated-credential list --id <object-id> \
  --query "[].{name:name,subject:subject,audience:audiences[0]}" -o table
```

The subject is exactly `repo:<owner>/<repo>:environment:<environment-name>` and the
audience is `api://AzureADTokenExchange`. Ask the team what happens if a fork, a different
branch, or a different environment name tries to use it. The answer — nothing, the token
does not match the subject — is the reason this is safer than a stored secret.

### Review the plan

Open the Planned view. Confirm the application, branch, and environment selectors, then
select each resource and record its resolved recipe and planned output resources.

For Azure, the plan resolves recipes from the `azure-avm` recipe pack published by
[`radius-project/resource-types-contrib`](https://github.com/radius-project/resource-types-contrib).
Kubernetes workload nodes are annotated with the managed service backing them, so a
deployment on this environment reads as backed by AKS.

**Model answers:**

- The database resolves to an Azure managed PostgreSQL offering through the `azure-avm`
  pack, built on Azure Verified Modules. Teams should recognise the shape from Challenge 04,
  where they used an Azure Verified Module themselves.
- Compared with `iac/recipes/postgres-azure-flex.bicep`: functionally similar, but the team
  authored, published to their own registry, and registered that recipe against
  `env-azure-prod`. Here the recipe pack is fetched from an upstream repository and applied
  by the environment created by the extension. Ownership moved; the mechanism did not.
- The decision point is the environment. In Challenge 04 it was `rad recipe register`
  against `env-azure-prod`. Here it is the recipe pack the workflow deploys alongside a
  `Radius.Core/environments` resource. Both are "the environment decides how a requirement
  is met".
- Reviewing a plan provisions nothing. Verify with a resource listing before and after:

```bash
az resource list --resource-group <lab-rg> --query "length(@)"
```

The count must be unchanged after reviewing the plan.

## Stage 4: Deploy and compare with Challenge 05

Select **Deploy Application** in the Planned view. The canvas commits the workflow files
and dispatches the run, then opens the Deployments view with live status. Node badges show
progress, success, and failure; deployment-state styling overrides diff colouring while a
run is in flight.

Have participants read the committed workflows before discussing them:

| Workflow | Role |
| --- | --- |
| `verify-azure.yml` | "Radius - Verify Credentials"; OIDC login, subscription check, cluster credentials, cluster info |
| `run-rad-commands.yml` | Dispatcher; detects the provider from the environment variables |
| `run-rad-commands-azure.yml` | The Azure deployment implementation |
| `delete-application.yml`, `delete-environment.yml` | Teardown, used in cleanup |

### What the run actually does

Reconstruct these steps from the run log:

1. Commit or update the dispatcher and provider workflows.
2. Authenticate to Azure with OIDC; no stored credential is used.
3. Fetch a kubeconfig for the target AKS cluster.
4. Install k3d and create an ephemeral `radius-cp` cluster; install the `rad` CLI and
   Terraform on the runner.
5. Install Radius into that ephemeral cluster, pointed at the target cluster.
6. Project GitHub OIDC tokens into the control-plane pods and register the Azure credential
   with workload identity.
7. Log in to GHCR and restore the previous run's control-plane state.
8. Deploy a `Radius.Core/environments` resource and the `azure-avm` recipe pack.
9. Install or validate the Gateway API only if the application declares routes.
10. Run `rad deploy` on `.radius/app.bicep`, then persist state back to GHCR. On failure,
    logs are uploaded as an artifact. The k3d cluster is always deleted.

Step 10 is the punchline: it is the same `rad deploy` from Challenge 05, in a different
place, against a control plane that did not exist a minute earlier and will not exist a
minute later.

### Reach the application

```text
Access my deployed application
```

Copilot sets up port forwarding and returns a URL. Note that the managed gateway defaults
to a cluster-internal service, so nothing is exposed publicly by default.

### Comparison table

| Question | Challenge 05 | Challenge 09 |
| --- | --- | --- |
| Where does the Radius control plane run? | Installed in the AKS and K3s clusters | An ephemeral k3d cluster on a GitHub Actions runner |
| How long does it live? | For the workshop | For one workflow run |
| Where does Radius state live between deployments? | In the control plane's database in the cluster | In a private GHCR package per environment |
| Who holds the credential that reaches Azure? | A registered Azure credential on the control plane | Nobody; minted per run through OIDC federation |
| Where is the recipe set defined and registered? | Authored by the team, published to their registry, registered on `env-azure-prod` | An upstream recipe pack deployed by the workflow |
| What is the unit of promotion? | A `rad deploy` invocation from a workstation | A commit on a branch plus a workflow dispatch |

### Prove isolation

```bash
kubectl get all -n trading-canvas
kubectl get all -n <challenge-05-namespace>

rad workspace switch ws-azure-prod
rad group switch rg-trading
rad app list
```

The Challenge 05 control plane must still list `adaptive-apps` and must not list the Canvas
application. Two Radius control planes are now driving the same cluster from different
namespaces, which is a good moment to ask what would happen if both owned the same
namespace. They would fight; that is why the challenge fixes a separate namespace.

## Stage 5: The Diff view as a review artifact

Have each team make one deliberate architectural change on a branch. Adding a cache client
to the backend works well: it should appear as an added backing service plus an added
connection, with everything else unchanged.

Re-model on the branch, then open the diff with the default branch as base and the branch
as head. Resources are tagged `added`, `removed`, `modified`, or `unchanged`, coloured
green, red, yellow, and grey.

Branch semantics matter and are commonly misunderstood:

- If the selected branch is the branch checked out in the workspace, writing
  `.radius/app.bicep` to the working tree is enough. No commit or push is needed to preview.
- If the selected branch is a different branch, the model must be committed and pushed to
  that branch before it can render there. The deploy workflow likewise checks the branch
  out from GitHub, so a local-only fix is not deployed.

Generate the Markdown summary and post it on a pull request.

### Model answer: what the diff cannot catch

The diff compares application models, so it catches structural change: added or removed
components, added or removed backing services, changed resource types, and changed
connections. It shows as `unchanged` anything that does not alter the model, including:

| Change | Diff result | Real risk |
| --- | --- | --- |
| A changed environment variable value | Unchanged or a property-level modification only | High; Challenge 06 turned on OIDC entirely through parameters |
| A changed image tag | Unchanged structure | High; different code, identical shape |
| A changed SQL query or business rule | Unchanged | High |
| A removed readiness probe | Unchanged structure | High; Challenge 05 relied on that probe to prove database access |
| Widened network exposure at the environment level | Unchanged | High; exposure is an environment variable, not part of the app model |

Correct conclusion: it is an excellent review aid and a poor gate. It tells a reviewer
where to look, which is exactly the problem the blog post describes for large
agent-generated changes. It does not tell them the change is safe, and a team that treats
a grey graph as approval has replaced review with reassurance.

## Stage 6: Model assessment of the portability claim

Push teams to produce something a platform lead could act on.

### What Canvas made easier

- No hand-authoring of a contract, a recipe, or an OCI publish step to reach an Azure
  managed database. Challenge 03 plus Challenge 04 collapsed into an upstream recipe pack.
- No control-plane installation or lifecycle to own for this path.
- No long-lived cloud credential anywhere in the pipeline.
- A shared visual definition that both a developer and an agent read the same way, and a
  branch-level architectural diff that did not previously exist.

### What it does not do today

| Limitation | Consequence for this MicroHack |
| --- | --- |
| Preview supports containerized applications deployed to Azure; AWS is stated as coming soon | `env-local-prod` on K3s is not reachable. The whole Local half of Challenges 01-08 is out of scope |
| Requires a Dockerfile | Fine here; a blocker for the brownfield work in Challenge 10 |
| Redeployments are incremental and do not prune removed resources | Removing a resource from the model does not remove it from Azure |
| Managed gateway supports HTTP and TLS routes only | The MQTT broker's non-HTTP traffic would need a user-managed gateway |
| One application per repository | The trading repository is fine; a monorepo of independent apps is not |
| Predefined type allow-list | No MQTT broker type and no workload identity type, the two custom contracts this application actually needs |
| Deployment history is per session, and canvas status polling stops before long runs finish | Read the workflow run, not the panel, for the truth |

### Where portability actually comes from

Resource types define the contract, recipes implement it per platform, and environments
choose the implementation. That is Challenge 03 and Challenge 04, and it is unchanged by
Canvas. Canvas changes who authors and reviews the model, and gives Azure an on-ramp. It
does not add a new portability mechanism, and using it does not remove the need for the
catalog — the two contracts it could not express are the proof.

### What would be needed for a K3s, Azure Local, or Arc target

- Provider support in the extension for a non-Azure, non-AWS compute platform, including
  credential and trust handling for a cluster that may be private or disconnected.
- A recipe pack for that platform. The team already has the substance of one: the
  Challenge 04 K3s recipe set.
- A path to a control plane that does not assume a GitHub-hosted runner can reach the
  cluster. The Challenge 01 topology puts the K3s API behind a localhost Bastion tunnel on
  purpose, which no hosted runner can traverse.

The middle item already exists. The first and third do not, and the third is an
architectural question rather than a missing feature.

### Recommendation

Give Canvas to application teams for Azure-targeted work and for pull-request review on
every branch, including branches that will ship elsewhere, because the diff is valuable
regardless of target. Keep the catalog and the recipe sets with the platform team, and
keep the Challenge 05 path as the mechanism of record for any application that must also
run at a disconnected site. Tell a team asking to use Canvas for such an application that
they can model and review with it today, and must deploy the non-Azure target through the
platform path until provider support exists.

## Troubleshooting

| Symptom | Cause and action |
| --- | --- |
| The canvas panel does not open | Ask Copilot to fix the Radius Canvas, or reload extensions and restart the app. Confirm the session was restarted after installing the plugin |
| Modeling reports no Dockerfile | Only real Dockerfiles count; a `.devcontainer` image builds the development environment, not the application |
| `gh` is not recognized | Install the GitHub CLI on the **host**, restart the terminal, and run `gh auth login`. The devcontainer's CLI does not help the Copilot app |
| Missing `read:packages` or `write:packages` | `gh auth refresh -s read:packages -s write:packages`. The extension ignores an ambient `GH_TOKEN` for package creation |
| Credential verification times out | If the default branch is protected, environment setup opened a pull request with the workflow files. Merge it, then retry |
| `Branch not pushed yet`, or the Radius extension is not recognised | Commit and push the entire `.radius` directory, including its `bicepconfig.json`, to the branch being deployed |
| An image build fails with `exec format error` | Review target-platform handling in the Dockerfile and the build platforms |
| Deployment reported as timed out after a few minutes | The canvas stopped polling; the run continues. Open the workflow run |
| A removed resource still exists after redeployment | Expected. Redeployments are incremental. Delete it deliberately |
| The graph looks stale | Refresh to rebuild from the selected branch's current definition. Check the origin record if the source has moved on |

Classify failures before reacting. Infrastructure, credential, and cluster failures are
fixed in the environment. Schema and modeling failures are fixed in the application
definition. Never fix a modeling failure by removing a required dependency, a secret
binding, or a connection just to get a clean compile.

## Clean up

Order matters. The environment deletion flow refuses to start while an application is still
deployed, and the Azure federated credential must still exist when the Radius environment
is deleted.

1. Delete the deployment from the Deployments view and let the deletion workflow finish.
2. Delete the environment. This deletes the Radius environment, the per-environment Azure
   federated credential, the GitHub environment, and the GHCR state package.
3. Confirm the Challenge 05 application and the AKS cluster are intact.

```bash
kubectl get all -n trading-canvas
kubectl get all -n <challenge-05-namespace>
az resource list --resource-group <lab-rg> -o table
```

> [!WARNING]
> Deleting a deployment can permanently remove infrastructure and data owned by the
> application. It does not delete the AKS cluster or the lab resource group, and the Entra
> app registration is intentionally left in place.

Only Azure-backed environments can be deleted through this flow today. If a team's
environment is left behind, record its remaining cost and scope rather than deleting Azure
resources by hand and losing the audit trail.

## Evidence checklist

- `.radius/app.bicep`, the origin record, and the Modeled view inventory with source
  references verified.
- The completed model mapping table, including the two contracts with no predefined
  equivalent.
- GitHub environment variables listing, showing no stored cloud credential.
- The federated credential subject and audience, with values redacted as agreed.
- Planned-view recipe resolution per resource, and the unchanged Azure resource count
  before and after planning.
- The workflow run, the ten reconstructed steps, and the Challenge 05 comparison table.
- Namespace isolation output from both namespaces, and `rad app list` on `ws-azure-prod`.
- The Diff view Markdown summary on a pull request, plus the list of changes it cannot
  catch.
- The written portability assessment.
