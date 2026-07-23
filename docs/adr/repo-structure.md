# ITV AI Agent Hub - OpenWebUI (OWUI) repo, structure, and updating strategy 

## Architecture decisions and recommendations

## 1. Purpose and scope

This document captures the key decisions for ITV Studios Agent Hub, specifically around its primary chosen interface platform, Open WebUI (OWUI). It follows on from a successful MVP programme also built using OWUI, and covers the approach for the production build: how we adopt and maintain OWUI, how we govern model access, and how our own AI agents are exposed to users.

Decisions here reflect the engineering team’s current thinking and should be revisited as the platform matures and OWUI itself evolves.

## 2. Licensing 

OWUI moved from a permissive BSD-3-Clause licence to a custom 'Open WebUI Licence' at version 0.6.6 around April 2025. This licence retains permissive use and modification rights, but adds a branding protection clause: 

> “deployments with more than 50 aggregate end users in any rolling 30-day period may not remove or alter the 'OWUI' name, logo, or UI marks unless they hold an Enterprise licence or specific written permission.”

### Decision

ITV’s plan is to purchase an OWUI Enterprise licence. This removes the branding restriction entirely and allows for full white-labelling. Enterprise licensing also offers:

- SLA-backed commercial support
- Access to Long-Term Support (LTS) release versions
- A supported path for full theming and custom branding without legal ambiguity

This decision removes licensing as a constraint on the technical approach to how we customise and brand the OWUI interface described in the rest of this document.

## 3. Repository forking and upstream sync strategy

The core OWUI product is a living codebase; an ongoing concern in active development and maintenance. This means we can benefit from OWUI’s engineering team producing new features, pushing bug fixes, and closing security vulnerabilities. Whilst this is an absolute win for the business, it creates a challenge for the Agent Hub Delivery Team in bringing these changes into our product from the upstream codebase. 

There are two approaches to take:

1. We pull a fork of the original OWUI project and track it as an upstream remote and merge in new tagged releases as they become available. 
2. We create a brand new repository based on the latest version of OWUI and manually merge in updates and releases as we diverge away from the original codebase.

### Decision

The recommendation – based on several discussions with businesses in similar positions, documentation and agentic-supported research – is that ITV Studios Agent Hub should is **built as a maintained fork** of `open-webui/open-webui`, rather than a from-scratch build, in order to benefit from OWUI's ongoing feature development, security fixes, and community ecosystem with minimal ongoing engineering cost.

In practice this means:

- Fork `open-webui/open-webui` into ITV's own GitHub organisation.
- Add the original repository as a git remote (upstream) and track tagged releases only — never the moving `main` or `dev` branches.
- Pull in new upstream releases via a short-lived `sync/vX.Y.Z` branch,
  - Apply changes via merging (**not rebasing**) into main.
  - Review changes via standard PR process.
  - Once deployed, tag with appropriate release version, e.g. `itv-vX.Y.Z`.
- Keep ITV-specific changes confined to configuration, branding assets, and deployment tooling — avoid modifying OWUI's own source files wherever possible, to keep merge conflicts rare and upstream syncs low-risk.

### Upstream open-source contributions

A path exists to contribute fixes back upstream to the main OWUI project where useful. Any contribution merged after v0.6.5 requires agreeing to OWUI's Contributor Licence Agreement (CLA), and the core team heavily reviews and reworks community PRs, so this is best reserved for genuinely general-purpose fixes rather than ITV-specific work.

## 4. Customisation approach

OWUI is layered in a way that supports the Agent Hub Delivery Team’s goal of staying easy to update and keep in sync with upstream releases. 

The proposed approach below favours the lowest-friction customisation layer available for each need:

| Layer | Use for |
|------ | --------|
|**CSS theming** <br> (custom.css, injected via our own Docker image layer) | Branding, colours, fonts, ITV visual identity — no core source changes required |
| **Functions, Tools, Filters** <br> (in-process Python plugins, managed via Admin Panel) | Lightweight behavioural changes: message filters, custom UI buttons, new model source wiring |
| **Pipelines** <br>(standalone microservice, OpenAI-API-compatible) | Heavier processing that shouldn't run inside the OWUI process itself |
| **Direct source modification** | Avoided wherever possible — reserved for cases with no other option, since this is where upstream merge conflicts arise |

It’s recommended that we minimise any direct customisation of OWUI’s existing codebase to reduce treacherous version updates. Modifications should be created as separate modules or Svelte components and brought into their relevant areas in as discreet a manner as possible. For example, adding custom buttons to the chat window or specialist HTML responses.

Given our own AI agents will live in a separate Python codebase/repository, the strong preference is to keep that separation intact rather than pulling agent logic into OWUI's Pipelines framework (see section 6).

## 5. Repository structure and branching

At a high level, the core Agent Hub would be comprised of these three repositories, matching the natural seams in the system:

```bash
studios-agent-hub/          # ← the main codebase, the OWUI fork
studio-agent-hub-agents/    # ← FastAPI + Bedrock agents (e.g. Sales Agent)
```
***note**: repository names are placeholders at this stage and this outline don’t account for separate ‘infra’ style repositories to support deployment and CPV3 provisioning*

The instance of LiteLLM and its proxy configuration is handled in a separate infra repository.

### Example agent-hub structure

```bash
agent-hub/
├── branding/                 # custom.css, logo and icon assets
├── deploy/                   # (if cpv3 provisioning lives here)
│   ├── helm/values-cpv3.yaml
│   └── terraform/       
├── docs/adr/ 
├── Dockerfile                # FROM ghcr.io/open-webui/open-webui:vX.Y.Z + COPY branding/
└── .github/workflows/        # build/push image, upstream diff checks, etc.
```

The original (vendored) OWUI source itself is not shown above because it’s not copied into the repository at all — it's pulled in via the fork relationship (git history) and consumed as a base image in the Dockerfile.

Our Agent Hub specific additions sit alongside this as configuration, assets, and deployment tooling, never as edits mixed into OWUI's own source tree. This is what keeps upstream syncs (section 3) low-conflict.

### Example branching model

| Example branch | Used for |
| -------------- | -------- |
|`main` | Core production/release branch; what's actually deployed |
| upstream (remote) | `open-webui/open-webui`, tracked, never pushed to |
|`sync/vX.Y.Z` | short-lived branch per upstream release pulled in |
| `feature/AH-123`, `AH456-some-name` | standard Jira-ticket branches (existing team convention) |

## 6. Model and agent governance via LiteLLM

Agent Hub does not connect directly to approved models, or to our own agents (e.g. Sales Agent). All model traffic is routed through a LiteLLM proxy, which acts as the single governance point for ‘what's exposed and to whom’.

### Decision and rationale

- Foundation models (e.g. Bedrock, OpenAI) will be registered in LiteLLM as model deployments, so they present identically to Agent Hub as OpenAI-API-compatible ‘models’.
- Each FastAPI agent (e.g. Sales Agent) wraps its logic behind its own `/v1/chat/completions-compatible` endpoint which can then be connected to OWUI directly via the Pipelines framework. This enables us to support rich HTML or complex UI components called by and from agents or tools by dealing with OWUI's internal HTML renderer.
- This approach helps to maintain a restricted governance path: 
  - model allow-listing, 
  - per-team/per-key access control,
  - budgets, 
  - rate limiting, 
  - and usage/cost tracking and metrics
- This also helps to keep the OWUI fork closer to stock, since very minimal custom pipe or connector code needs to live inside it. 

Sensitive data handling (e.g. the Sales Agent's access to Rights and financial data) happens entirely inside the agent's own API service, ahead of the call to Bedrock. OWUI and LiteLLM have no awareness of the underlying data sources — they only see a model that produces relevant answers.

### Open item

Model-level visibility (e.g. restricting the Sales Agent to specific teams) could be enforced at two independent layers: 

1. OWUI's group-based model visibility controls.
2. LiteLLM's own key/team-based access controls. 

The specific mapping of ITV's user groups to both layers is not yet finalised and will involve some sort of Okta grouping.

## 7. Security and operational considerations

- Functions, Tools, and Pipes execute arbitrary Python with the same file and network permissions as the OWUI process. **Community-sourced plugins should not be imported without code review**, particularly in a walled cpv3 environment.
- Outbound calls OWUI makes by default (community marketplace, update checks, telemetry) should be audited and disabled where not needed.
- Configuration that can persist to the database via the Admin UI (e.g. OAuth settings) should be pinned to environment variables (`ENABLE_OAUTH_PERSISTENT_CONFIG=false` and equivalents) to keep configuration reproducible and version-controlled rather than living as manual UI changes.

## 8. Open questions

1. Preferred cadence for pulling in upstream OWUI releases (e.g. monthly review vs. on-demand).
2. Model-level visibility (see ‘Open item’ in section 7 above).
3. Naming conventions, including repo name, branching, and so on.
