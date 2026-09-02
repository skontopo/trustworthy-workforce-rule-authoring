# Claude Desktop prompts

Use these prompts in a new Claude Desktop conversation after GraphDB MCP is connected.

## Prompt 0 - Working rules for Claude

```text
You are assisting with governed rule authoring for the workforce-support proof of concept.

Your role is limited to:

1. Reading GraphDB through MCP.
2. Finding existing IRIs for terms used by the expert.
3. Mapping natural-language terms to existing ontology, ESCO, and PoC resources.
4. Producing candidate Turtle that follows the supplied rule vocabulary.
5. Explaining validation results.

Rules you must follow:

- Never invent an ESCO, task, competence, occupation, or controlled-value IRI.
- Verify every IRI by querying GraphDB.
- Use the ESCO named graph:
  https://w3id.org/wfs/poc/graph/esco
- Use the PoC named graph:
  https://w3id.org/wfs/poc/graph/core
- Use only the FormalRule properties supplied in this conversation.
- Do not generate SPARQL INSERT, DELETE, or UPDATE commands unless I explicitly request them.
- Do not write directly to GraphDB.
- Do not approve or reject rules.
- If a term is ambiguous, stop and ask for clarification.
- Preserve the expert's original wording.
- Return a mapping table before returning candidate Turtle.
- A candidate rule is not active until a human approves it.
```

## Prompt 1 - Resolve concepts

Run `queries/01-create-submission.rq` first in GraphDB Workbench. Then ask Claude:

```text
Use GraphDB MCP to resolve the concepts in ex:rule-submission-001.

Find and verify the IRIs for:

- junior operator level;
- industrial machinery mechanic;
- preliminary inspection of abnormal vibration in a conveyor drive;
- use testing equipment;
- co-decision;
- approve with supervision.

Return a table with:

1. phrase from the rule;
2. selected IRI;
3. graph label;
4. named graph;
5. reason for the mapping.

Do not produce the formal rule yet.
Do not invent an IRI.
Stop if any term cannot be resolved uniquely.
```

Expected mappings:

- Junior: `https://w3id.org/wfs/poc#Junior`
- industrial machinery mechanic: `http://data.europa.eu/esco/occupation/269c47e7-9017-4aa6-bce8-49e89a696a64`
- task: `https://w3id.org/wfs/poc/resource/conveyor-vibration-inspection`
- use testing equipment: `http://data.europa.eu/esco/skill/74374d09-7a54-4a05-ae78-d5940423de7c`
- co-decision: `https://w3id.org/wfs/poc#CoDecision`
- approve with supervision: `https://w3id.org/wfs/poc#ApproveWithSupervision`

## Prompt 2 - Formalize the rule

```text
Using only the verified IRIs, formalize ex:rule-submission-001.

Create a FormalRule with IRI:

https://w3id.org/wfs/poc/resource/rule-conveyor-junior-testing-gap-v1

Use exactly these properties:

- wfs:originalRuleText
- wfs:appliesToTask
- wfs:appliesToOccupation
- wfs:appliesToOperatorLevel
- wfs:triggerMissingCompetence
- wfs:setsDecisionAuthority
- wfs:setsHumanReview
- wfs:recommendedOutcome
- wfs:ruleVersion
- wfs:ruleStatus
- prov:wasDerivedFrom
- prov:wasAttributedTo
- prov:wasGeneratedBy

Set the rule status to wfs:Candidate.

Also create a RuleFormalization activity named:

https://w3id.org/wfs/poc/resource/formalization-001

Maria is the original author.
Claude Desktop performs the formalization.

Return one valid Turtle block.
Do not insert it into GraphDB.
Do not add facts that were not stated or verified.
```

The expected output is also provided in `data/05-candidate-rule-example.ttl` so that the demo can still be reproduced if Claude formats its output differently.

## Prompt 3 - Prepare human review

Run the validation queries first. If all checks pass, ask Claude:

```text
Prepare the candidate rule for human review.

Retrieve and present:

1. the original natural-language rule;
2. the selected ontology and ESCO mappings;
3. the formal candidate rule;
4. the validation result;
5. the effect the rule would have on Alex's case.

Do not approve the rule.

End by asking Maria to choose exactly one of:

- APPROVE
- REVISE
- REJECT
```

For this running example, choose `APPROVE`, then run `queries/06-publish-rule.rq` manually in GraphDB Workbench.

## Prompt 4 - Explain Alex's assessment

After the rule is active, ask Claude:

```text
Use GraphDB MCP to assess Alex for the preliminary conveyor-vibration inspection.

Use the competence information in the graph and the approved rules in the active-rules graph.

Produce a short assessment containing:

1. how many required competences are verified;
2. the missing competence;
3. the identifier and original wording of the applicable approved rule;
4. the decision-authority level;
5. the rule's recommended outcome;
6. a clear statement that this is an AI recommendation, not authorization.

Do not infer facts outside the query results.
Do not provide repair, diagnostic, shutdown, restart, or machine-operation instructions.

If no approved rule is found, state that the case must be escalated and do not recommend an outcome.
```

If Claude does not reliably generate the intended SPARQL, run `queries/08-competence-assessment.rq` and `queries/09-applicable-rule.rq` in Workbench and paste the results into Claude. For a stronger GraphDB MCP setup, the same queries can later be registered as user-generated MCP prompts in GraphDB.
