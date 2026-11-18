---
type: docs
title: "Workflow API reference"
linkTitle: "Workflow API"
description: "Detailed documentation on the workflow API"
weight: 1600
---

Dapr provides users with the ability to interact with workflows through its built-in workflow engine, which is implemented using Dapr Actors.

To interact with the workflow engine, see [how to author]{{% ref howto-author-workflow.md %}} and [manage workflows]({{% ref howto-manage-workflow.md %}}).


# HTTP API reference (Deprecated)

Below is the reference for the Dapr Workflow API.
This API is deprecated and will eventually be removed.

## Start workflow request

Start a workflow instance with the given name. Both query parameters are optional and independent of each other; see the table below.

```
POST http://localhost:<daprPort>/v1.0/workflows/<workflowComponentName>/<workflowName>/start?instanceID=<instanceID>&appID=<appID>
```

Note that workflow instance IDs can only contain alphanumeric characters, underscores, and dashes.

{{% alert title="Important" color="warning" %}}
It is strongly recommended to give every execution a new instance ID rather than reusing an old one. See [Implications of instance ID reuse]({{% ref "workflow-features-concepts.md#implications-of-instance-id-reuse" %}}). An instance ID can only be reused once the existing workflow with that ID, and every child workflow it created (checked recursively), has reached a terminal state (`COMPLETED`, `FAILED`, or `TERMINATED`). Otherwise the request fails with HTTP `500` and error code `ERR_START_WORKFLOW`, and the error message names the workflow that is not yet terminal. SDK workflow clients get the gRPC `AlreadyExists` status code instead, so they can detect the conflict without parsing the message. To free up an instance ID unconditionally, first purge the workflow using the [purge API]({{% ref "workflow_api.md#purge-workflow-request" %}}). See [workflow identity]({{% ref "workflow-features-concepts.md#workflow-identity" %}}) for more details.
{{% /alert %}}

### URL parameters

Parameter | Description
--------- | -----------
`workflowComponentName` | Use `dapr` for Dapr Workflows
`workflowName` | Identify the workflow type
`instanceID` | (Optional) Unique value created for each run of a specific workflow
`appID` | (Optional) App ID of the application that should host the workflow instance. Defaults to the local app. The target app must be in the same namespace and its [workflow access policy]({{% ref workflow-access-policy %}}) governs whether the call is permitted.

### Request content

Any request content will be passed to the workflow as input. The Dapr API passes the content as-is without attempting to interpret it.

### HTTP response codes

Code | Description
---- | -----------
`202`  | Accepted
`400`  | Request was malformed
`500`  | Request formatted correctly, error in dapr code. This includes error code `ERR_START_WORKFLOW` when a workflow with the given instance ID already exists and is not yet reusable (it, or one of its child workflows, is not in a terminal state)

### Response content

The API call will provide a response similar to this:

```json
{
    "instanceID": "12345678"
}
```

## Terminate workflow request

Terminate a running workflow instance with the given name and instance ID.

```
POST http://localhost:<daprPort>/v1.0/workflows/<workflowComponentName>/<instanceId>/terminate
```

{{% alert title="Note" color="primary" %}}
 Terminating a workflow terminates all of the child workflows created by the workflow instance.

Terminating a workflow has no effect on any in-flight activity executions that were started by the terminated instance. 

{{% /alert %}}

### URL parameters

Parameter | Description
--------- | -----------
`workflowComponentName` | Use `dapr` for Dapr Workflows
`instanceId` | Unique value created for each run of a specific workflow
`appID` | (Optional) App ID of the application hosting the workflow instance. Defaults to the local app. The target app must be in the same namespace and its [workflow access policy]({{% ref workflow-access-policy %}}) governs whether the call is permitted.

### HTTP response codes

Code | Description
---- | -----------
`202`  | Accepted
`400`  | Request was malformed
`500`  | Request formatted correctly, error in dapr code

### Response content

This API does not return any content.

## Raise Event request

For workflow components that support subscribing to external events, such as the Dapr Workflow engine, you can use the following "raise event" API to deliver a named event to a specific workflow instance.

```
POST http://localhost:<daprPort>/v1.0/workflows/<workflowComponentName>/<instanceID>/raiseEvent/<eventName>
```

{{% alert title="Note" color="primary" %}}
 The exact mechanism for subscribing to an event depends on the workflow component that you're using. Dapr Workflow has one way of subscribing to external events but other workflow components might have different ways.

{{% /alert %}}

### URL parameters

Parameter | Description
--------- | -----------
`workflowComponentName` | Use `dapr` for Dapr Workflows
`instanceId` | Unique value created for each run of a specific workflow
`appID` | (Optional) App ID of the application hosting the workflow instance. Defaults to the local app. The target app must be in the same namespace and its [workflow access policy]({{% ref workflow-access-policy %}}) governs whether the call is permitted.
`eventName` | The name of the event to raise

### HTTP response codes

Code | Description
---- | -----------
`202`  | Accepted
`400`  | Request was malformed
`500`  | Request formatted correctly, error in dapr code or underlying component

### Response content

None.

## Raise Error request

For workflow components that support subscribing to external events, such as the Dapr Workflow engine, use the raise error API to deliver a failure to a specific workflow instance waiting for the named event. Unlike raising an event, raising an error completes the external-event wait with an exception that the workflow can handle through its normal exception-handling logic.

```
POST http://localhost:<daprPort>/v1.0/workflows/<workflowComponentName>/<instanceID>/raiseError/<eventName>
```

### URL parameters

Parameter | Description
--------- | -----------
`workflowComponentName` | Use `dapr` for Dapr Workflows
`instanceID` | Unique value created for each run of a specific workflow
`appID` | (Optional) App ID of the application hosting the workflow instance. Defaults to the local app. The target app must be in the same namespace and its [workflow access policy]({{% ref workflow-access-policy %}}) governs whether the call is permitted.
`eventName` | Name of the external event the workflow is waiting for

### Request content

The request body must be a JSON object describing the failure. The `errorType` field is required and must contain a non-empty string. The other fields are optional.

Field | Type | Description
----- | ---- | -----------
`errorType` | string | Required. The error type used when delivering the failure to the workflow.
`errorMessage` | string | Error message describing the failure.
`stackTrace` | string | Stack trace associated with the failure.
`innerFailure` | object | Nested failure details, using the same fields.
`isNonRetriable` | boolean | Indicates whether the failure should be treated as non-retriable.

```json
{
  "errorType": "PaymentFailed",
  "errorMessage": "The payment could not be processed",
  "stackTrace": "PaymentService.Process: payment provider returned a decline"
}
```

The workflow runtime delivers these failure details to the named external-event wait as an exception. The workflow can handle that exception through its normal exception-handling logic.

### HTTP response codes

Code | Description
---- | -----------
`202` | Accepted
`400` | Request was malformed or the failure details are invalid (including a missing or empty `errorType`)
`500` | Request formatted correctly, but an error occurred in Dapr or the underlying workflow component

### Response content

None.

## Pause workflow request

Pause a running workflow instance.

```
POST http://localhost:<daprPort>/v1.0/workflows/<workflowComponentName>/<instanceId>/pause
```

### URL parameters

Parameter | Description
--------- | -----------
`workflowComponentName` | Use `dapr` for Dapr Workflows
`instanceId` | Unique value created for each run of a specific workflow
`appID` | (Optional) App ID of the application hosting the workflow instance. Defaults to the local app. The target app must be in the same namespace and its [workflow access policy]({{% ref workflow-access-policy %}}) governs whether the call is permitted.

### HTTP response codes

Code | Description
---- | -----------
`202`  | Accepted
`400`  | Request was malformed
`500`  | Error in Dapr code or underlying component

### Response content

None.

## Resume workflow request

Resume a paused workflow instance.

```
POST http://localhost:<daprPort>/v1.0/workflows/<workflowComponentName>/<instanceId>/resume
```

### URL parameters

Parameter | Description
--------- | -----------
`workflowComponentName` | Use `dapr` for Dapr Workflows
`instanceId` | Unique value created for each run of a specific workflow
`appID` | (Optional) App ID of the application hosting the workflow instance. Defaults to the local app. The target app must be in the same namespace and its [workflow access policy]({{% ref workflow-access-policy %}}) governs whether the call is permitted.

### HTTP response codes

Code | Description
---- | -----------
`202`  | Accepted
`400`  | Request was malformed
`500`  | Error in Dapr code

### Response content

None.

## Purge workflow request

Purge the workflow state from your state store with the workflow's instance ID.

```
POST http://localhost:<daprPort>/v1.0/workflows/<workflowComponentName>/<instanceId>/purge
```

{{% alert title="Note" color="primary" %}}
Only `COMPLETED`, `FAILED`, or `TERMINATED` workflows can be purged.
{{% /alert %}}

### URL parameters

Parameter | Description
--------- | -----------
`workflowComponentName` | Use `dapr` for Dapr Workflows
`instanceId` | Unique value created for each run of a specific workflow
`appID` | (Optional) App ID of the application hosting the workflow instance. Defaults to the local app. The target app must be in the same namespace and its [workflow access policy]({{% ref workflow-access-policy %}}) governs whether the call is permitted.

### HTTP response codes

Code | Description
---- | -----------
`202`  | Accepted
`400`  | Request was malformed
`500`  | Error in Dapr code

### Response content

None.

## Get workflow request

Get information about a given workflow instance.

```
GET http://localhost:<daprPort>/v1.0/workflows/<workflowComponentName>/<instanceId>
```

### URL parameters

Parameter | Description
--------- | -----------
`workflowComponentName` | Use `dapr` for Dapr Workflows
`instanceId` | Unique value created for each run of a specific workflow
`appID` | (Optional) App ID of the application hosting the workflow instance. Defaults to the local app. The target app must be in the same namespace and its [workflow access policy]({{% ref workflow-access-policy %}}) governs whether the call is permitted.

### HTTP response codes

Code | Description
---- | -----------
`200`  | OK
`400`  | Request was malformed
`500`  | Error in Dapr code

### Response content

The API call will provide a JSON response similar to this:

```json
{
  "createdAt": "2023-01-12T21:31:13Z",
  "instanceID": "12345678",
  "lastUpdatedAt": "2023-01-12T21:31:13Z",
  "properties": {
    "property1": "value1",
    "property2": "value2",
  },
  "runtimeStatus": "RUNNING",
 }
```

Parameter | Description
--------- | -----------
`runtimeStatus` | The status of the workflow instance. Values include: `"RUNNING"`, `"COMPLETED"`, `"CONTINUED_AS_NEW"`, `"FAILED"`, `"CANCELED"`, `"TERMINATED"`, `"PENDING"`, `"SUSPENDED"`  

The `properties` map does not include `dapr.workflow.output` when `runtimeStatus` is `"FAILED"`. A failed workflow has no successful output. Instead, the `properties` map carries the failure in these keys:

Parameter | Description
--------- | -----------
`dapr.workflow.failure.error_type` | The error type for a workflow with `runtimeStatus` `"FAILED"`.
`dapr.workflow.failure.error_message` | The error message for a workflow with `runtimeStatus` `"FAILED"`.
`dapr.workflow.failure.stack_trace` | The stack trace for a workflow with `runtimeStatus` `"FAILED"`, when available.
