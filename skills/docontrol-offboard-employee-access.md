---
name: docontrol-offboard-employee-access
description: Revoke a terminated employee's access to company Google Drive assets shared with their personal email, using DoControl's HRIS lookup and Google remediation mutation.
api: docontrol:docontrol-graphql-api
operations:
  - refreshToken
  - graphql
graphql_operations:
  - hrisUsers
  - startGoogleRemediationAssessment
  - googleRemediationAssessment
generated: '2026-09-06'
method: generated
source: https://docs.docontrol.io/docontrol-user-guide/workflows/define-workflow-settings/action-settings/utilities/docontrol-api-action/api-for-offboarding-employees.md
---

# Offboard a terminated employee's asset access

DoControl publishes this flow end to end. It resolves the employee's *personal* email from the
connected HRIS, then removes that email as a collaborator from company Google Drive assets.

**This flow writes, and the write is only partly reversible.** Removed collaborators can be
re-added with the Add collaborator remediation, but DoControl documents no undo, and no window
inside which one is guaranteed. Confirm the target before you run it.

## Step 0 — authenticate

Follow `docontrol-authenticate`. You need a bearer token and an admin-privileged key.

## Step 1 — resolve the personal email (`hrisUsers`)

```graphql
query MyQuery {
  hrisUsers(
    input: {
      filters: {primaryEmail: {single: {operator: EQUALS, value: "<ORG_EMAIL>"}}},
      serviceId: "<HRIS_SERVICE_ID>"
    }
  ) {
    nodes {
      personalEmail
      primaryEmail
      employmentEndDate
      employmentStatus
    }
  }
}
```

`serviceId` names the connected HRIS — DoControl's published example uses `sap-successfactors`.
Check `employmentStatus` and `employmentEndDate` before acting; do not remediate on the strength
of a ticket alone.

Save `personalEmail`. It is the join key for the next step.

## Step 2 — start the remediation (`startGoogleRemediationAssessment`)

```graphql
mutation StartGoogleRemediationAssessment {
  startGoogleRemediationAssessment(
    input: {
      autoApproveInput: {remediateInherited: true, workflowId: "<REMEDIATION_WORKFLOW_ID>"},
      remediationType: GOOGLE_REMOVE_COLLABORATORS,
      filterString: "[{\"permissionEmail\":{\"single\":{\"operator\":\"EQUALS\",\"value\":\"<PERSONAL_EMAIL>\"}}},{\"ownerEmail\":{\"single\":{\"operator\":\"EQUALS\",\"value\":\"<ORG_EMAIL>\"}}}]"
    }
  ) {
    jobId
    remediationType
    updatedAt
    executionId
  }
}
```

`filterString` is a JSON array passed as an escaped string. Keep **both** clauses: `permissionEmail`
narrows to assets shared with the personal address, `ownerEmail` narrows to assets the employee
owned. Dropping one widens the blast radius.

Documented `remediationType` values:

| Intent | Value |
| --- | --- |
| Remove external collaborators | `GOOGLE_REMOVE_COLLABORATORS` |
| Remove internal collaborators | `GOOGLE_REMOVE_INTERNAL_COLLABORATORS` |
| Remove all collaborators | `GOOGLE_REMOVE_ANY_COLLABORATOR` |
| Remove public sharing | `GOOGLE_REMOVE_PUBLIC_SHARING` |
| Remove org-wide sharing | `GOOGLE_REMOVE_ORG_WIDE_SHARING` |
| Change owner | `GOOGLE_CHANGE_OWNER` |
| Remove permissions outside the domain | `GOOGLE_REMOVE_COLLABORATOR_FROM_ORG` |
| Remove specific collaborators | `GOOGLE_REMOVE_SPECIFIC_COLLABORATOR_FROM_ASSET` |

Keep the returned `jobId`. There is no other way to find this job again.

## Step 3 — poll for completion (`googleRemediationAssessment`)

```graphql
query MyQuery {
  googleRemediationAssessment {
    nodes {
      jobId
      jobTypeStatus
      executionId
    }
  }
}
```

This query takes no filter, so match on the `jobId` you saved. The job is finished when
`jobTypeStatus` is `REMEDIATION_WORKFLOW_DONE`. No other status value is documented, so treat
anything else as still running and keep polling; do not treat an unknown value as failure.

## Constraints that will bite you

- One query or mutation per request. No batching.
- 5MB maximum payload, and **pagination is not supported** — a filter that matches too much has no
  documented way to be split.
- 30-second timeout with no response.
- No idempotency key. If step 2 times out, you cannot tell whether the job started. Poll step 3
  before resending.
