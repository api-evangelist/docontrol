---
name: docontrol-scheduled-drive-remediation
description: Run an on-demand or scheduled Google Drive exposure remediation through the DoControl API, selecting assets with a JSON filter and tracking the job to completion.
api: docontrol:docontrol-graphql-api
operations:
  - refreshToken
  - graphql
graphql_operations:
  - startGoogleRemediationAssessment
  - googleRemediationAssessment
generated: '2026-09-06'
method: generated
source: https://docs.docontrol.io/docontrol-user-guide/workflows/define-workflow-settings/action-settings/utilities/docontrol-api-action/api-for-on-demand-remediation.md
---

# Remediate Google Drive exposure on demand

Use this to clean up exposed Drive assets in bulk — for example, removing external collaborators
from documents that have not been viewed in six months.

**This flow writes and it is destructive to sharing state.** Removed collaborators must be re-added
manually if you get the filter wrong, and DoControl publishes no undo and no restore window.

## Step 0 — authenticate

Follow `docontrol-authenticate`.

## Step 1 — decide the filter before you decide the action

Filters are JSON objects passed to `filterString` as an escaped string, each clause shaped
`{"<field>":{"single":{"operator":"<OP>","value":"<VALUE>"}}}`. `EQUALS` is the only operator
DoControl publishes an example for. Fields seen in published examples: `permissionEmail`,
`ownerEmail`. DoControl also names sharing status, last-viewed date and file type as filterable,
and says clauses combine.

Because there is no dry-run mode and no preview operation on this API, the safest rehearsal is to
build and inspect the same filter in the console's inventory view first.

## Step 2 — start the job (`startGoogleRemediationAssessment`)

```graphql
mutation StartGoogleRemediationAssessment {
  startGoogleRemediationAssessment(
    input: {
      autoApproveInput: {remediateInherited: true, workflowId: "<REMEDIATION_WORKFLOW_ID>"},
      remediationType: <REMEDIATION_TYPE>,
      filterString: "<ESCAPED_JSON_FILTER>"
    }
  ) {
    jobId
    remediationType
    updatedAt
    executionId
  }
}
```

`remediateInherited: true` also remediates inherited permissions. Set it to `false` if you only
mean to touch permissions granted directly on the matched assets.

## Step 3 — track it (`googleRemediationAssessment`)

Poll `googleRemediationAssessment` and match your `jobId`. Done when `jobTypeStatus` is
`REMEDIATION_WORKFLOW_DONE`.

## Reversal

There is no reverse operation. The compensating path DoControl documents is the **Add collaborator**
remediation action, which re-grants access. Record the collaborator set you are about to remove
before you remove it — the API will not give it back to you afterwards.
