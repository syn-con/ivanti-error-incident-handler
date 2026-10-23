# Ivanti Error Incident Handler

Example n8n error workflow demonstrating how failed n8n workflow executions can be logged automatically as incidents in Ivanti Neurons for ITSM using the `@synergyconsulting/n8n-nodes-ivanti-neurons-for-itsm` community node.

## Description

### Quick overview

This repository contains a complete example of an n8n error-handling workflow that reports workflow failures to Ivanti Neurons for ITSM.

The example includes:

* an n8n error workflow that receives failed executions through the Error Trigger
* detection of an existing open incident for the failed workflow
* creation of a new incident when no related incident exists
* automatic reopening of a related incident that is already Resolved
* Journal notes added to the related incident for every repeated failure
* detailed error information in HTML format, including links to the failed execution and workflow
* decoding of the Ivanti API error response from the n8n error buffer

The workflow creates one incident per monitored workflow. Repeated failures are added to the existing incident instead of creating duplicates.

> **Note:** This repository is an example implementation intended for demonstration and integration reference purposes. Review the field mappings, statuses and customer assignment before using it in a production environment.

## How it works

1. A monitored workflow fails, and n8n starts this workflow because it is set as that workflow's **Error workflow**.
2. The **Error Trigger** receives the workflow name and ID, the node that failed, the error message, the API response and the execution link.
3. The workflow waits 30 seconds before checking Ivanti.
4. The workflow searches Ivanti for an incident that is not Closed and has the failed workflow's ID in `XSC_N8N_WF_ID`.
5. If no related incident exists, a new incident is created with status `Logged`. The full error details go into Symptom and the workflow ID into `XSC_N8N_WF_ID`.
6. If a related incident exists and is Resolved, its status is set back to Active.
7. For every related incident that is not Closed, a Journal note with the details of the new failure is added.

## Incident handling

| Situation | Result |
| --- | --- |
| No related incident exists | New incident is created with status `Logged` |
| Related incident exists and is open | Journal note is added |
| Related incident exists and is Resolved | Status is set to `Active` and a Journal note is added |
| Related incident exists but is Closed | Ignored. A new incident is created |

## Repository contents

```text
.
├── README.md
├── ivanti_error_incident_handler_workflow.json
└── workflow.png
```

### `ivanti_error_incident_handler_workflow.json`

Example n8n workflow responsible for:

* receiving failed executions through the Error Trigger
* searching for a related Ivanti incident by workflow ID
* creating a new incident with full error details
* reopening Resolved incidents
* adding Journal notes for repeated failures

### `workflow.png`

Screenshot of the workflow on the n8n canvas.

## Setup

### 1. Install n8n

Use a self-hosted n8n instance that supports community nodes.

### 2. Install the Ivanti Neurons for ITSM community node

Install:

```text
@synergyconsulting/n8n-nodes-ivanti-neurons-for-itsm
```

The workflow depends on this node for communicating with Ivanti Neurons for ITSM.

### 3. Configure Ivanti credentials

Create an Ivanti Neurons for ITSM API credential in n8n and configure:

| Field | Purpose |
| --- | --- |
| Tenant URL | Ivanti Neurons for ITSM tenant for the target environment |
| API Key | REST API key used to authenticate Ivanti requests |

Assign the credential to each Ivanti node in the workflow.

### 4. Prepare Ivanti

Create a custom text field on the **Incident** business object:

```text
XSC_N8N_WF_ID
```

The workflow uses this field to link incidents to the n8n workflow that failed.

### 5. Import the workflow

Import `ivanti_error_incident_handler_workflow.json` into n8n.

After importing:

* configure the Ivanti credentials on all Ivanti nodes
* in **Create New Incident**, replace the **ProfileLink** value with the RecId of the employee who should be the customer of these incidents
* review the Subject, Symptom and status values for your Ivanti configuration
* save the workflow

The workflow does not need to be activated. n8n runs error workflows automatically.

### 6. Connect monitored workflows

In every workflow that should be monitored:

* open **Settings → Error workflow**
* select **Ivanti Error Incident Handler**
* make sure **Save failed production executions** is enabled, so the **Open execution** link works

Do not set an Error workflow on **Ivanti Error Incident Handler** itself. If you did, a failure in this workflow could start a loop.

### 7. Configure Ivanti (optional)

If a quick action or business rule asks for notes when a resolution is backed out, it can fill them in automatically for incidents created by this workflow:

```text
$(if nvl(XSC_N8N_WF_ID, "") != "" then "The n8n workflow failed again after this incident was resolved. Status set back to Active automatically. See the latest Journal note for error details." else Prompt("Enter Notes for the backed out resolution", ""))
```

`Prompt()` only works when a person makes the change in the Ivanti UI, not when the change comes through the API.

## Data formats

### Error Trigger input

n8n sends the failed execution to the Error Trigger in the following general structure:

```json
{
  "execution": {
    "id": "10712",
    "url": "https://n8n.example.com/workflow/HR7PoPpwozwDtyG5/executions/10712",
    "error": {
      "message": "Bad request - please check your parameters",
      "description": "Request failed with status code 400",
      "timestamp": 1792749669812,
      "context": {
        "data": { "type": "Buffer", "data": [123, 13, 10, "..."] }
      },
      "node": {
        "name": "Read an attachment1",
        "type": "@synergyconsulting/n8n-nodes-ivanti-neurons-for-itsm.ivantiNeuronsForItsm",
        "parameters": { "resource": "attachment", "operation": "read" }
      }
    },
    "lastNodeExecuted": "Read an attachment1",
    "mode": "webhook",
    "executionContext": {
      "triggerNode": { "name": "Ivanti Neurons for ITSM Connector Trigger" }
    }
  },
  "workflow": {
    "id": "HR7PoPpwozwDtyG5",
    "name": "IncGetAttachError"
  }
}
```

If the trigger node of the monitored workflow fails, n8n sends a `trigger` object instead of `execution`. The workflow handles both cases.

### Incident output

**Subject:**

```text
[N8N]<workflow name> Workflow failed
```

**Symptom and Journal note:**

```text
N8N workflow execution failed.

Workflow:      IncGetAttachError
Workflow ID:   HR7PoPpwozwDtyG5
Execution ID:  10712
Links:         Open execution | Open workflow
Trigger:       Ivanti Neurons for ITSM Connector Trigger (webhook)
Time:          2026-10-22 13:41:09

Failed node:   Read an attachment1
Node type:     @synergyconsulting/n8n-nodes-ivanti-neurons-for-itsm.ivantiNeuronsForItsm
Action:        attachment / read

Error:         Bad request - please check your parameters
Details:       Request failed with status code 400
API response:  ISM_4000 – Invalid Request Payload – Attachment with 6F80EC02F27E4CC49531D1877DE5DB5B not found
```

The content is stored as HTML, so it is formatted when viewed in Ivanti.

## Requirements

* A self-hosted n8n instance
* The `@synergyconsulting/n8n-nodes-ivanti-neurons-for-itsm` community node installed in n8n
* Valid Ivanti Neurons for ITSM REST API credentials
* Permission to search, create and update Incidents and to create Journal notes
* The custom field `XSC_N8N_WF_ID` on the Incident business object
* An Employee record to use as the customer of the incidents

## Customization

The example can be adapted in several ways:

* change the Subject and Symptom format
* change the initial status of new incidents
* change the wait time before searching for a related incident
* add fields such as Category, Service, OwnerTeam, Impact, Urgency or Source
* use a different field than `XSC_N8N_WF_ID` to link incidents to workflows
* change which statuses count as open or reopenable
* send a notification (email, Teams, Slack) in addition to the incident
* reuse the same pattern for other ITSM systems

## Example integration flow

```text
     Monitored n8n workflow
               |
               | execution fails
               v
         Error Trigger
               |
               v
         Wait (30 s)
               |
               v
     Find Related Incident
  (Status != Closed AND XSC_N8N_WF_ID = workflow ID)
               |
       +-------+---------------------+
       |                             |
  not found                        found
       |                             |
       v                             v
 Create New Incident            Is Resolved?
                                     |
                          +----------+----------+
                          |                     |
                         yes                    no
                          |                     |
                          v                     |
                  Set Status Active             |
                          |                     |
                          +----------+----------+
                                     |
                                     v
                                Insert Note
```

## Known limitations

* **No source record data.** The Error Trigger does not include the data of the Ivanti record that the failed workflow was processing. The incident describes the failure, not the original record.
* **Trigger failures.** If the trigger node of a monitored workflow fails, n8n sends no execution data. Some fields then show `-` and the links do not work.
* **Duplicate notes.** **Find Related Incident** returns all matches. If two incidents that are not Closed share the same `XSC_N8N_WF_ID`, a Journal note is added to both.
* **Status names.** The **Is Resolved** check compares against the exact status name `Resolved` and is case-sensitive.

## Security notes

This repository intentionally contains an example implementation rather than a production-ready application.

Before adapting it for production:

* store Ivanti API keys only in n8n credentials, not in the workflow JSON
* remove pinned test data from the Error Trigger before exporting or sharing the workflow
* review what error details are written to Ivanti, because error messages may contain record data or internal URLs
* use an Ivanti API user with only the permissions the workflow needs
* use HTTPS for both n8n and Ivanti
* follow your organization's security and data-handling requirements

## Additional information

This workflow is only one example of how n8n errors can be handled. The same Error Trigger pattern can be used to report failures to any system that n8n can reach.

The workflow JSON is intended to demonstrate the integration pattern and provide a starting point for building organization-specific error-handling automations.
