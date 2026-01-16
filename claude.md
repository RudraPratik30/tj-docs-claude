# ToolJet Documentation Writing Instructions

You are writing user-facing documentation for ToolJet, a low-code platform for building internal tools. Use the codebase as reference to generate accurate, helpful documentation.

---

## General Principles

### Voice & Tone
- Write in second person ("you") addressing the user directly
- Be concise and scannable — users are here to solve problems, not read prose
- Use active voice: "Click the button" not "The button should be clicked"
- Friendly but professional—avoid being overly casual or corporate
- Assume competence but not familiarity with ToolJet

### Style Guide: [Refer](/docs/versioned_docs/version-3.16.0-LTS/contributing-guide/documentation-guidelines/style-guide.md) {develop branch}

### Formatting Standards
- Use title case for headings
- Keep paragraphs short (2-4 sentences max)
- Use code blocks with appropriate syntax highlighting for any code, queries, or configuration
- Use tables for property/parameter documentation
- Use admonitions sparingly for important warnings or tips:
  - `:::info` for helpful context
  - `:::warning` for potential issues
  - `:::danger` for breaking changes or data loss risks

### Structure
- Start with a short description of what the thing does
- Lead with the most common use case and how does the feature is helpful for the user
- Put prerequisites at the top if any exist
- End with related links or next steps where appropriate

---

## Category-Specific Instructions

---

### 1. Datasource / Plugin Documentation

**Purpose:** Help users connect external data sources and services to ToolJet.

**User Persona:** Builder / Developer, Admin / Team Lead / Manager, Database Adminstrator 

**Code locations to reference:**
- Plugin definition: `plugins/packages/<plugin-name>/`
- Operations/queries: Look for `operations.ts` or `index.ts`
- Types: `lib/types.ts` or similar
- Frontend manifest: `manifest.json` or plugin config files

**Required sections:**

```markdown

Short description of the datasource and overview of what users can do after connecting to this data sorce in bullet points.

## Connection

### Required credentials
Explain what credentials/config the user needs before connecting.

## Supported operations

### [Operation Name]

Describe what this operation does.

**Required Parameters**

- Parameter: description

**Optional Parameters**

- Parameter: description

<img className="screenshot-full img-full" src="</img/marketplace/plugins/<plugin-name>/<img-name>.png>" alt="<img-description>" />

{For Claude Code Reference: along with screenshot-full class, use one of this class - img-full, img-l, img-m, img-s, use appropriate padding or margin, syntax - style={{ marginBottom:'15px' }}}

<details id="tj-dropdown">
<summary>**Example Values**</summary>

    ```yaml

    ```

</details>

<details id="tj-dropdown">
<summary>**Response Example**</summary>

    ```json

    ```

</details>


<br/>
---

## Need Help?

- Reach out via our [Slack Community](https://join.slack.com/t/tooljet/shared_invite/zt-2rk4w42t0-ZV_KJcWU9VL1BBEjnSHLCA)
- Or email us at [support@tooljet.com](mailto:support@tooljet.com)
- Found a bug? Please report it via [GitHub Issues](https://github.com/ToolJet/ToolJet/issues)

```

**Guidelines:**
- Extract all operations from the code—don't miss any
- If operations are in the form of api end points then crete tables in the following format for each entity:
   | Method	| Endpoint	| Description |
- Document all parameters, including optional ones with defaults
- Include realistic examples, not placeholder values
- Note any ToolJet-specific behaviors or transformations
- If the datasource has different auth methods, document each

---

### 2. Component Documentation

**Purpose:** Help users understand and configure UI components in the app builder.

**User Persona:** Builder / Developer

**Code locations to reference:**
- Component definition: `frontend/src/Editor/Components/<ComponentName>/`
- Properties/config: Look for `properties.js`, `config.js`, or similar
- Exposed variables: Check component state and exposed data
- Events: Look for event handlers and callbacks
- Styles: Style configuration options

**Required sections:**

```markdown
# [Component Name]

Short description of the component's purpose and use cases.

## Example Usage

Provide a practical example showing common usage when a entrprise level organization may use this component. Give a proper scenario and how this component can solve this issue, don't add fillers, make it to the point, short and concise.

## Properties

Go through the frontend for what properties are there before the events section, if there are multiple sections then create separate h3 for each section.

and make a table in the following format for each property

| Property | Description | Expected Value |

### Events

| Event | Description |
|-------|-------------|
| On click | Triggered when... |

## Component Specific Actions (CSA)

Following actions of component can be controlled using the component specific actions(CSA):

| <div style={{ width:"100px"}}> Action </div> | <div style={{ width:"135px"}}> Description </div> | <div style={{width: "200px"}}> How To Access </div>|
| :------------ | :---------- | :------------ |


## Exposed Variables

| Variable | Description | How To Access |
|:--------|:-----------|:------------|

## Validation {if any}

| <div style={{ width:"100px"}}> Validation Option </div> | <div style={{ width:"200px"}}> Description </div> | <div style={{width: "200px"}}> Expected Value </div>|
|:---------------|:-------------------------------------------------|:-----------------------------|

 
## Additional Actions {Check what actions are available}

| <div style={{ width:"100px"}}> Action </div> | <div style={{ width:"150px"}}> Description </div> | <div style={{ width:"250px"}}> Configuration Options </div>|
|:------------------|:------------|:------------------------------|
| Loading state      | Enables a loading spinner, often used with `isLoading` to indicate progress.    | Enable/disable the toggle button or dynamically configure the value by clicking on **fx** and entering a logical expression. |
| Visibility         | Controls component visibility.                                                  | Enable/disable the toggle button or dynamically configure the value by clicking on **fx** and entering a logical expression. |
| Disable            | Enables or disables the component.                                              | Enable/disable the toggle button or dynamically configure the value by clicking on **fx** and entering a logical expression. |
| Tooltip            | Provides additional information on hover. Set a string value for display.                                 | String (e.g., `Enter your name here.` ).                       |


## Devices

|<div style={{ width:"100px"}}> Property </div> | <div style={{ width:"150px"}}> Description </div> | <div style={{ width:"250px"}}> Expected Value </div>|
|:---------- |:----------- |:----------|
| Show on desktop | Makes the component visible in desktop view. | You can set it with the toggle button or dynamically configure the value by clicking on **fx** and entering a logical expression. |
| Show on mobile | Makes the component visible in mobile view. | You can set it with the toggle button or dynamically configure the value by clicking on **fx** and entering a logical expression. |

## Styles

Check if there are multiple sections in the styles tab then create a h3 for each section and create a table in following format for each section

| <div style={{ width:"100px"}}> Property </div> | <div style={{ width:"150px"}}> Description </div> | <div style={{ width:"250px"}}> Configuration Options </div>|
|:---------------|:------------|:---------------|

<br/>
---

## Need Help?

- Reach out via our [Slack Community](https://join.slack.com/t/tooljet/shared_invite/zt-2rk4w42t0-ZV_KJcWU9VL1BBEjnSHLCA)
- Or email us at [support@tooljet.com](mailto:support@tooljet.com)
- Found a bug? Please report it via [GitHub Issues](https://github.com/ToolJet/ToolJet/issues)

```

**Guidelines:**
- Extract ALL properties from the code—completeness matters here
- Group properties logically (Data, Events, Styles, General)
- Document every exposed variable users can reference
- Document all component actions
- Show the component's data binding syntax
- Include validation rules if applicable

---

### 3. Feature Documentation

**Purpose:** Explain platform capabilities that span multiple areas (versioning, import/export, environments, etc.)

**Persona:** Admins, C-Level management, Team evaluation the platform for their use case, developer / builder

**Code locations to reference:**
- Varies by feature—identify relevant backend services and frontend components
- API endpoints in `server/src/`
- Frontend implementation in `frontend/src/`

**Required sections:**

```markdown
# [Feature Name]

Short description of what this feature enables. 2-3 sentences expanding on what the feature does and why users would use it.

## Prerequisites

- What's needed before using this feature (plan tier, permissions, setup)

## How it works

Explain the mental model. What happens under the hood? What should users understand conceptually?

## Using [feature]

### [Primary workflow]

Step-by-step instructions for the main use case.

### [Secondary workflow]

Additional workflows if applicable.

## Configuration options

If there are settings or options, document them.

| Option | Description | Default |
|--------|-------------|---------|
| option | What it does | value |

## Limitations

Be upfront about what the feature doesn't do or known constraints.

## Related

- Links to related features or how-tos
```

**Guidelines:**
- Focus on user goals, not implementation details
- Be clear about any limitations or edge cases
- Note if feature requires specific plan/license
- Include practical examples

---
<!-- 
### 4. How-to Guides

**Purpose:** Task-oriented guides that help users accomplish specific goals.

**Required sections:**

```markdown
# How to [accomplish specific task]

One-sentence description of what the user will achieve.

## Prerequisites

- What the user needs before starting

## Steps

### 1. [First major step]

Explain the step with enough detail to complete it.

### 2. [Second major step]

Continue with clear, actionable instructions.

## Verification

How the user can confirm they've succeeded.

## Next steps

- What users commonly do after this
- Links to related how-tos
```

**Guidelines:**
- Title should complete the sentence "How to..."
- Keep focused—one task per guide
- Number the steps
- Include expected outcomes at each major step
- Don't over-explain concepts—link to concept docs instead

--- 

### 5. Concepts Documentation

**Purpose:** Explain foundational mental models and abstractions in ToolJet.

**Required sections:**

```markdown
# [Concept Name]

One-sentence definition.

## What is [concept]?

2-3 paragraph explanation of the concept, what problem it solves, and how it fits into ToolJet.

## How [concept] works

Explain the mechanics at a conceptual level. Use diagrams if helpful.

## Key characteristics

- Bullet points of important properties or behaviors

## Example

Concrete example showing the concept in action.

## Related concepts

- Links to related concepts
```

**Guidelines:**
- Don't document how to USE something here—that's for how-tos
- Focus on understanding, not procedures
- Use analogies if they help
- Keep it high-level but accurate

--- 

### 6. Workflows Documentation

**Purpose:** Document the workflow automation system.

**Code locations to reference:**
- Workflow engine: `server/src/workflows/` or similar
- Nodes/steps: Individual workflow step implementations
- Triggers: Trigger configurations

**Required sections:**

```markdown
# [Workflow Node/Feature]

One-sentence description.

## Overview

What this node/feature does in the context of workflow automation.

## Configuration

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| field | type | description | Yes/No |

## Inputs

What data this node accepts.

## Outputs

What data this node produces for downstream nodes.

## Example

Show a practical configuration example.

## Error handling

How errors are surfaced and can be handled.
```

---

### 7. Security & Access Control Documentation

**Purpose:** Document authentication, authorization, and security features.

**Code locations to reference:**
- Auth modules: `server/src/auth/`, SSO implementations
- RBAC: Permission definitions, role configurations
- Middleware: Security-related middleware

**Required sections:**

```markdown
# [Security Feature]

One-sentence description.

## Overview

What this feature secures and why it matters.

## Prerequisites

- Required plan tier
- Required permissions to configure
- External requirements (IdP setup, etc.)

## Configuration

Step-by-step setup with all options documented.

## How it works

Explain the security model—what gets protected and how.

## Best practices

Security recommendations.

## Troubleshooting

Common issues, especially auth failures.
```

**Guidelines:**
- Be precise about security behaviors—vagueness causes security issues
- Document all configuration options
- Include troubleshooting for auth failures
- Note any audit logging

---

### 8. Self-hosting / Deployment Documentation

**Purpose:** Help users deploy and operate ToolJet.

**Code locations to reference:**
- Docker configurations: `docker-compose*.yml`
- Environment variables: `.env.example`, config modules
- Kubernetes manifests if available
- Server configuration: `server/src/config/`

**Required sections:**

```markdown
# [Deployment Method/Topic]

One-sentence description.

## Prerequisites

- System requirements
- Required software/tools

## Installation

Step-by-step instructions.

## Configuration

### Environment variables

| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| VAR_NAME | What it does | Yes/No | value |

## Verification

How to confirm successful deployment.

## Upgrading

How to upgrade to new versions.

## Troubleshooting

Common deployment issues and solutions.
```

**Guidelines:**
- Document ALL environment variables from the code
- Include realistic resource requirements
- Note breaking changes in upgrades
- Provide rollback procedures where relevant

---

### 9. Troubleshooting Documentation

**Purpose:** Help users diagnose and fix common issues.

**Required sections:**

```markdown
# Troubleshooting [area]

## [Error message or symptom]

### Cause

Why this happens.

### Solution

How to fix it, step by step.

--- 

## [Next error]

Continue pattern...
```

**Guidelines:**
- Use exact error messages as headings when possible (helps search)
- Provide specific solutions, not vague suggestions
- Include how to gather diagnostic information
- Link to relevant docs for deeper context

--- -->

## Code-to-Documentation Checklist

When generating documentation from code, verify:

- [ ] All properties/parameters documented
- [ ] All events/callbacks documented
- [ ] All exposed variables documented
- [ ] Default values are accurate
- [ ] Types are correct
- [ ] Required vs optional is clear
- [ ] Examples are realistic and functional
- [ ] No placeholder text remains
- [ ] Links to related docs included

---

## Terminology

Use consistent terminology:

| Use | Don't use |
|-----|-----------|
| Datasource | Data source, data-source |
| Workspace | Organization (unless specifically about org settings) |
| Query | Request, call (in datasource context) |
| Component | Widget, element |
| Application | App  |
| RunJS | JavaScript, JS code (when referring to the action) |
| RunPy | Python, Py code (when referring to the action) |

---

## When Uncertain

- If code behavior is ambiguous, note it and flag for review
- If a feature seems incomplete or buggy, document current behavior and note limitations
- If you can't determine something from code alone, add a `TODO: verify` comment