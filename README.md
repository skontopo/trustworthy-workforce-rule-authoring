# Trustworthy Workforce Rule Authoring

A reproducible Proof of Concept (PoC) for governed natural-language rule authoring, semantic validation, and human-in-the-loop workforce support using Claude Desktop, GraphDB, ESCO, SHACL, SPARQL, and PROV-O.

## 1. What this repository demonstrates

The starting point is a simple workforce-support scenario. Alex is a junior industrial machinery mechanic. A task requires four competences, but only three are verified in Alex's profile. This scenario was demonstrated in [previous work](https://github.com/skontopo/trustworthy-workforce-ai-poc).

The new part of this PoC happens *before* Alex is assessed. Maria, the domain expert and supervisor, writes a rule in ordinary language:

> If a junior industrial machinery mechanic does not have verified competence to use testing equipment, the preliminary inspection of abnormal vibration in a conveyor drive requires human review and may only be approved with supervision.

Claude Desktop then helps map the words in the rule to existing graph concepts and proposes a machine-readable rule. GraphDB validates the candidate rule. Maria reviews it and decides whether it may become active. Only an approved rule is used in Alex's later assessment.

The full flow is:

```text
Natural-language rule
    -> concept alignment
    -> candidate RDF rule
    -> SHACL validation
    -> SPARQL validation
    -> human approval
    -> active rule
    -> Alex competence assessment
    -> Claude explanation
    -> human decision
    -> PROV-O audit trail
```

The important boundary is simple: Claude may interpret and explain, but it does not approve rules and it does not authorize Alex to perform the task.

## 2. Technologies used

- **Claude Desktop** - natural-language interface and rule formalization.
- **GraphDB 11.4** - RDF store and MCP server.
- **GraphDB MCP gateway** - connects local GraphDB to Claude Desktop through a local stdio MCP process.
- **ESCO** - occupation and skill identifiers.
- **SHACL** - structural and simple safety validation of candidate rules.
- **SPARQL** - reference checks, conflict checks, competence comparison, and rule matching.
- **PROV-O** - provenance for rule creation, validation, approval, assessment, and human decision.

This repository includes only the ESCO occupation and four ESCO skills needed by the example, so the PoC can be reproduced without downloading the full ESCO dataset. You may replace the included subset with the full ESCO dataset later.

## 3. Prerequisites

You need:

1. GraphDB 11.4 running locally.
2. Claude Desktop.
3. Node.js and npm for the GraphDB MCP gateway.

The GraphDB gateway package requires Node.js 18 or newer. A current Node.js LTS release is a good choice.

Check your installation:

```bash
node --version
npm --version
```

This guide assumes GraphDB is running at:

```text
http://localhost:7200
```

and uses a repository named:

```text
wfs-rules-poc
```

## 4. Create the GraphDB repository

Open GraphDB Workbench and go to:

```text
Setup -> Repositories -> Create new repository
```

Create a GraphDB repository with:

```text
Repository ID: wfs-rules-poc
Ruleset: empty
Enable SHACL validation: checked
```

> [!TIP]
> SHACL must be enabled when the repository is created. GraphDB does not let you switch an existing normal repository into a SHACL repository later.

Connect to the new repository.

## 5. Named graphs used by the PoC

The repository uses these named graphs:

```text
https://w3id.org/wfs/poc/graph/core
https://w3id.org/wfs/poc/graph/esco
https://w3id.org/wfs/poc/graph/submissions
https://w3id.org/wfs/poc/graph/candidates
https://w3id.org/wfs/poc/graph/active-rules
https://w3id.org/wfs/poc/graph/provenance
```

Their roles are:

| Graph          | Purpose                                                             |
|----------------|---------------------------------------------------------------------|
| `core`         | Alex, Maria, the task, the base vocabulary, and the rule vocabulary |
| `esco`         | ESCO occupation and skill resources                                 |
| `submissions`  | Original natural-language rule submissions                          |
| `candidates`   | Formal rules proposed by Claude but not yet approved                |
| `active-rules` | Human-approved rules that may affect assessments                    |
| `provenance`   | Validation, approval, assessment, and decision history              |

Keeping candidate rules separate from active rules is one of the main governance controls in the PoC.

## 6. Load the starting data

Use **Import -> User data** in GraphDB Workbench.

### 6.1 Load the original PoC data

Import:

```text
data/01-base-poc.ttl
```

Target graph:

```text
https://w3id.org/wfs/poc/graph/core
```

This contains Alex, Maria, Alex's verified competences, the inspection task, and the base PoC vocabulary.

> [!IMPORTANT]
> Unlike the earlier PoC, the task does **not** contain a hard-coded `CoDecision` classification or a hard-coded human-review flag. The approved rule will now supply this logic.

### 6.2 Load the minimal ESCO subset

Import:

```text
data/02-esco-subset.ttl
```

Target graph:

```text
https://w3id.org/wfs/poc/graph/esco
```

The file contains the ESCO occupation `industrial machinery mechanic` and the four ESCO skills used by the task.

### 6.3 Load the rule-authoring vocabulary

Import:

```text
data/03-rule-vocabulary.ttl
```

Target graph:

```text
https://w3id.org/wfs/poc/graph/core
```

This adds concepts such as `FormalRule`, `RuleSubmission`, `RuleValidation`, `Junior`, `Approved`, and the properties needed to describe a formal rule.

### 6.4 Load the SHACL shapes

Import:

```text
data/04-rule-shapes.ttl
```

Use this special SHACL shapes graph as the target graph:

```text
http://rdf4j.org/schema/rdf4j#SHACLShapeGraph
```

The shapes check that a candidate rule contains the required fields and enforce a few simple safety conditions. For example, a rule triggered by a missing competence cannot recommend independent approval.

## 7. Check the GraphDB setup

Open the SPARQL editor and run:

```text
queries/00-check-setup.rq
```

You should see the `core` and `esco` named graphs populated with triples.

> [!IMPORTANT]
> Do not expect the SHACL shapes graph to appear in this query. GraphDB stores SHACL shapes separately from normal data graphs.

## 8. Connect GraphDB to Claude Desktop through MCP

GraphDB 11.4 exposes an MCP server on the same port as GraphDB, at:

```text
http://localhost:7200/mcp
```

GraphDB's MCP SPARQL tool is read-only for this PoC: it can execute `SELECT`, `CONSTRUCT`, and `DESCRIBE`. Therefore, Claude is used to **read** and reason over GraphDB, while SPARQL updates are run manually in Workbench. This manual write boundary is intentional.

### 8.1 Install the GraphDB MCP gateway

The GraphDB-maintained gateway converts the local stdio connection used by Claude Desktop into requests to the GraphDB MCP server.

Install it globally:

```bash
npm install -g graphdb-mcphub-gateway
```

Check that the executable is available:

macOS/Linux:

```bash
which mcphub-gateway
```

Windows:

```powershell
where mcphub-gateway
```

If the command is not found, close and reopen your terminal after installing Node.js/npm, then try again.

### 8.2 Configure Claude Desktop

In Claude Desktop, open:

```text
Settings -> Developer -> Edit Config
```

The normal config file locations are:

macOS:

```text
~/Library/Application Support/Claude/claude_desktop_config.json
```

Windows:

```text
%APPDATA%\Claude\claude_desktop_config.json
```

A ready-to-copy example is in:

```text
config/claude_desktop_config.example.json
```

Its content is:

```json
{
  "mcpServers": {
    "graphdb": {
      "command": "mcphub-gateway",
      "env": {
        "MCPHUB_SERVER_URL": "http://localhost:7200/mcp"
      }
    }
  }
}
```

If you already have other MCP servers configured, merge the `graphdb` entry into the existing `mcpServers` object instead of replacing the whole file.

If Claude cannot find `mcphub-gateway`, replace the command with the absolute path returned by `which mcphub-gateway` or `where mcphub-gateway`.

Quit and restart Claude Desktop after saving the file.

### 8.3 Verify the connection

Open a new Claude conversation. Open the connector/tool menu and verify that `graphdb` appears and exposes GraphDB tools.

A simple first test is:

```text
Use GraphDB to list the available repositories.
```

You should see:

```text
wfs-rules-poc
```

Then try:

```text
Using the wfs-rules-poc repository, find the label of Alex's occupation.
```

Claude should retrieve `industrial machinery mechanic` from the graph.

## 9. Start a clean Claude conversation

Open:

```text
prompts/claude-prompts.md
```

Paste **Prompt 0 - Working rules for Claude** into a new Claude Desktop conversation.

This prompt tells Claude to:

- use GraphDB rather than invent identifiers;
- verify all ESCO and local IRIs;
- preserve Maria's original wording;
- produce only a candidate rule;
- never approve the rule itself.

## 10. Maria submits the natural-language rule

In GraphDB Workbench, run:

```text
queries/01-create-submission.rq
```

This creates `ex:rule-submission-001` in the `submissions` graph and records Maria as its author.

The submitted rule is:

```text
If a junior industrial machinery mechanic does not have verified
competence to use testing equipment, the preliminary inspection of
abnormal vibration in a conveyor drive requires human review and may
only be approved with supervision.
```

At this point the rule is only a draft. It has no effect on Alex.

## 11. Ask Claude to resolve the concepts

Paste **Prompt 1 - Resolve concepts** from:

```text
prompts/claude-prompts.md
```

Claude should query GraphDB and map the phrases in the rule to existing IRIs.

The important mappings are:

| Phrase                        | Expected IRI                                                                 |
|-------------------------------|------------------------------------------------------------------------------|
| junior                        | `https://w3id.org/wfs/poc#Junior`                                            |
| industrial machinery mechanic | `http://data.europa.eu/esco/occupation/269c47e7-9017-4aa6-bce8-49e89a696a64` |
| inspection task               | `https://w3id.org/wfs/poc/resource/conveyor-vibration-inspection`            |
| use testing equipment         | `http://data.europa.eu/esco/skill/74374d09-7a54-4a05-ae78-d5940423de7c`      |
| co-decision                   | `https://w3id.org/wfs/poc#CoDecision`                                        |
| approve with supervision      | `https://w3id.org/wfs/poc#ApproveWithSupervision`                            |

If Claude cannot map a term uniquely, it should stop and ask for clarification rather than inventing an IRI.

## 12. Ask Claude to create the formal candidate rule

Paste **Prompt 2 - Formalize the rule**.

Claude should return Turtle describing a candidate `FormalRule`.

The central part should look like this:

```turtle
ex:rule-conveyor-junior-testing-gap-v1
    a wfs:FormalRule ;
    wfs:appliesToTask ex:conveyor-vibration-inspection ;
    wfs:appliesToOperatorLevel wfs:Junior ;
    wfs:triggerMissingCompetence
        <http://data.europa.eu/esco/skill/74374d09-7a54-4a05-ae78-d5940423de7c> ;
    wfs:setsDecisionAuthority wfs:CoDecision ;
    wfs:setsHumanReview true ;
    wfs:recommendedOutcome wfs:ApproveWithSupervision ;
    wfs:ruleVersion 1 ;
    wfs:ruleStatus wfs:Candidate .
```

For exact reproduction, the expected candidate is included as:

```text
data/05-candidate-rule-example.ttl
```

This file is deliberately not loaded during initial setup.

## 13. Put the candidate in the staging graph

In GraphDB Workbench, go to Import.

Either:

1. paste Claude's Turtle output, or
2. use `data/05-candidate-rule-example.ttl`.

Import it into:

```text
https://w3id.org/wfs/poc/graph/candidates
```

The candidate is still not active.

## 14. SHACL validation runs automatically

Because the repository was created with SHACL enabled, GraphDB checks the imported candidate when the transaction is committed.

The rule should pass.

Examples of candidates that should fail include:

- a rule with no `setsHumanReview` value;
- a missing-competence rule that recommends `ApproveIndependently`;
- an `OperatorExclusive` rule that recommends approval.

If SHACL rejects a candidate, fix the candidate and import it again. Do not publish it.

## 15. Run the reference validation

SHACL checks structure, but the PoC also checks whether references make sense in this graph.

Run:

```text
queries/02-reference-validation.rq
```

A valid candidate returns:

```text
0 rows
```

The query checks that:

- the task exists;
- the occupation exists in the ESCO graph;
- the skill exists in the ESCO graph;
- the operator level exists;
- the trigger skill is actually required by the task.

## 16. Run the conflict validation

Run:

```text
queries/03-conflict-validation.rq
```

For the first rule, the expected result is:

```text
0 rows
```

The query looks for an already approved rule with the same task, occupation, operator level, and missing competence.

If it finds the same conclusion, it reports `DUPLICATE`.

If it finds a different authority or outcome, it reports `CONFLICT`.

A conflict is not resolved automatically. Maria must decide what to do.

## 17. Run the explicit safety validation

Run:

```text
queries/04-safety-validation.rq
```

The expected result is:

```text
0 rows
```

This query provides an easy-to-read second safety check on top of SHACL.

## 18. Record successful validation

Only if:

- SHACL passes;
- reference validation returns zero rows;
- conflict validation returns zero rows; and
- safety validation returns zero rows,

run:

```text
queries/05-record-validation-passed.rq
```

This creates a `RuleValidation` activity and a `ValidationReport` in the provenance graph.

## 19. Ask Claude to prepare the rule for human review

Paste **Prompt 3 - Prepare human review**.

Claude should summarize:

- Maria's original text;
- the selected IRIs;
- the formal candidate;
- the validation result;
- the expected effect on Alex.

Claude must then ask Maria to choose:

```text
APPROVE
REVISE
REJECT
```

For this running example, choose:

```text
APPROVE
```

Claude's answer is not itself the approval record. The next Workbench step records the human approval.

## 20. Publish the human-approved rule

Run manually in GraphDB Workbench:

```text
queries/06-publish-rule.rq
```

This:

1. copies the candidate rule into the `active-rules` graph;
2. changes its active status to `Approved`;
3. records Maria's review and approval in the provenance graph.

The candidate graph remains as a record of what was proposed.

## 21. Verify that the rule is active

Run:

```text
queries/07-verify-active-rule.rq
```

You should see one rule with:

```text
status: Approved
human review: true
authority: CoDecision
outcome: ApproveWithSupervision
```

The rule can now affect Alex's assessment.

## 22. Run the competence comparison

For a deterministic check in Workbench, run:

```text
queries/08-competence-assessment.rq
```

Expected result:

```text
conduct routine machinery checks    verified
inspect industrial equipment        verified
troubleshoot                        verified
use testing equipment               missing
```

This is the same competence gap as in the earlier PoC.

## 23. Find the approved rule that applies to Alex

Run:

```text
queries/09-applicable-rule.rq
```

Expected result:

```text
rule: rule-conveyor-junior-testing-gap-v1
missing skill: use testing equipment
authority: CoDecision
human review: true
recommended outcome: Approve with supervision
```

> [!TIP]
> This is the key extension over the earlier PoC: the authority and recommended outcome now come from a human-approved rule rather than being hard-coded on the task.

## 24. Ask Claude to explain Alex's case

Paste **Prompt 4 - Explain Alex's assessment**.

Claude should use GraphDB MCP to retrieve the relevant competence and approved-rule information.

A suitable answer is:

> Alex has three of the four competences required for the task. His competence to use testing equipment is not verified. Approved rule `rule-conveyor-junior-testing-gap-v1` applies. The rule classifies the case as co-decision, requires human review, and recommends approval only with supervision. This is an AI-generated recommendation and does not authorize Alex to begin the task.

If Claude generates an unsuitable SPARQL query, use the deterministic queries in Steps 22 and 23 and provide their results to Claude. GraphDB also supports user-generated MCP prompts containing prevalidated SPARQL templates; these are a useful later improvement for a more stable demo.

## 25. Record the AI assessment and recommendation

After Claude produces the expected assessment, run:

```text
queries/10-record-assessment.rq
```

This records that the assessment used:

- Alex's profile;
- the task;
- the approved rule.

It also records Claude's recommendation.

## 26. Record Maria's final decision and Alex's acknowledgement

For this example, Maria chooses:

```text
Approve with supervision
```

Run:

```text
queries/11-record-human-decision.rq
```

This creates:

- Maria's human review activity;
- Maria's final decision;
- Alex's acknowledgement.

The final authorization therefore remains human.

## 27. Inspect the provenance chain

Run:

```text
queries/12-provenance-trace.rq
```

The resulting data links the main stages:

```text
Maria's natural-language submission
    -> Claude formalization
    -> GraphDB validation
    -> Maria rule approval
    -> approved rule
    -> Alex assessment
    -> Claude recommendation
    -> Maria decision
    -> Alex acknowledgement
```

> [!TIP]
> This is the main provenance extension over the earlier PoC. The system can now explain not only which rule was used, but also where that rule came from and who approved it.

## 28. Negative tests

These tests are useful because they show that validation is doing real work.

### Test A - Missing field

Remove this line from the candidate rule:

```turtle
wfs:setsHumanReview true ;
```

Try to import the candidate again. SHACL should reject it.

### Test B - Unsafe independent approval

Replace:

```turtle
wfs:recommendedOutcome wfs:ApproveWithSupervision ;
```

with:

```turtle
wfs:recommendedOutcome wfs:ApproveIndependently ;
```

SHACL should reject the rule because the rule is triggered by a missing competence.

### Test C - Unknown skill

Replace the trigger skill with:

```text
https://example.org/nonexistent-skill
```

The basic RDF structure may still be valid, but `queries/02-reference-validation.rq` should report:

```text
Unknown ESCO skill IRI
```

This demonstrates why both SHACL and graph-aware SPARQL checks are used.

### Test D - Conflicting rule

After version 1 has been approved, create another candidate with the same task, occupation, operator level, and trigger skill, but a different outcome such as `wfs:Reject`.

Run:

```text
queries/03-conflict-validation.rq
```

It should report:

```text
CONFLICT
```

Do not publish the new candidate until a human resolves the conflict.

## 29. Reset the demo

To rerun the rule-authoring flow from the beginning, run:

```text
queries/99-reset-demo.rq
```

This clears only:

- submissions;
- candidates;
- active rules;
- provenance.

It keeps the core data, ESCO subset, and SHACL shapes.

## 30. Why some writes are manual

The GraphDB MCP server exposes read/query tools to MCP clients. Its SPARQL query interface executes `SELECT`, `CONSTRUCT`, and `DESCRIBE` queries, not SPARQL Update operations.

For this reason, the PoC uses a clear split:

```text
Claude Desktop + MCP: read, map, formalize, explain
GraphDB Workbench: insert, approve, publish, record decisions
```

This is useful for the research scenario because rule publication and final professional decisions remain explicit human-controlled actions.

## 31. What is deliberately outside this PoC

This repository does not implement:

- a custom or local LLM;
- GraphRAG over a document corpus;
- vector retrieval;
- continual learning;
- automatic rule publication;
- automatic conflict resolution;
- production authentication and authorization;
- full rule lifecycle/version management beyond the example version number.

The goal is narrower: demonstrate a complete and auditable path from an expert-written natural-language rule to a validated, human-approved symbolic rule that can later affect a competence assessment.

## 32. Troubleshooting

### Claude does not show GraphDB tools

1. Confirm GraphDB is running at `http://localhost:7200`.
2. Confirm `mcphub-gateway` runs from your terminal.
3. Check that the JSON in `claude_desktop_config.json` is valid.
4. Fully quit and restart Claude Desktop.
5. If necessary, replace `mcphub-gateway` in the config with its absolute path.
6. Check Claude Desktop's MCP logs.

### `mcphub-gateway` is not found

Run:

```bash
npm install -g graphdb-mcphub-gateway
```

Then reopen the terminal. If Node.js is not installed, install a current LTS version first. The gateway package requires Node.js 18 or newer.

### Candidate import fails immediately

This usually means SHACL rejected the candidate. Read GraphDB's validation message and compare the candidate with `data/05-candidate-rule-example.ttl`.

### Reference validation returns rows

A row is a validation problem. Check that all expected files were imported into the correct named graphs and that Claude used only verified IRIs.

### Applicable-rule query returns no rows

Check all of the following:

- the rule is present in the `active-rules` graph;
- its status is `wfs:Approved`;
- Alex is `wfs:Junior`;
- Alex has the expected ESCO occupation;
- the missing skill is `use testing equipment`;
- that skill is required by the task.

## 33. Official documentation used for this guide

- GraphDB 11.4 - Using GraphDB LLM tools with external clients: https://graphdb.ontotext.com/documentation/11.4/using-graphdb-llm-tools-with-external-clients.html
- GraphDB 11.4 - SHACL validation: https://graphdb.ontotext.com/documentation/11.4/shacl-validation.html
- GraphDB MCP gateway: https://github.com/Ontotext-AD/graphdb-mcp-gateway
- Model Context Protocol - Connect to local MCP servers: https://modelcontextprotocol.io/docs/develop
- SHACL Recommendation: https://www.w3.org/TR/shacl/
- PROV-O Recommendation: https://www.w3.org/TR/prov-o/

## License

The original resources in this repository are released under the
[MIT License](LICENSE), unless otherwise stated.

Third-party resources, including ESCO concepts and identifiers, remain
subject to the terms and conditions of their respective providers.