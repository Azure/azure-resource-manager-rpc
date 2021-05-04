# Addendum
## Table of Contents ##

- [Instrumentation and Tracing across services](#instrumentation-and-tracing-across-services) <br/>
- [ETags for Resources](#etags-for-resources) <br/>
- [Regional Endpoints](#regional-endpoints) <br/>
- [Nested Resources](#nested-resources) <br/>
- [Resource Group Deletes](#resource-group-deletes) <br/>
- [Asynchronous Operations](#asynchronous-operations) <br/>
- [Creating or Updating Resources Asynchronously](#creating-or-updating-resources-asynchronously)
- [Delete Resource Asynchronously](#delete-resource-asynchronously)
- [Call Action POST Asynchronously](#call-action-post-asynchronously)
- [ProvisioningState property](#provisioningstate-property)
- [202 Accepted and Location Headers](#202-accepted-and-location-headers)
- [Operation Resource format](#operation-resource-format)
- [Retrying REST Calls](#retrying-rest-calls)
- [Designing Resources](#designing-resources)
- [Enumerating SKUs for an existing resource](#enumerating-skus-for-an-existing-resource)
- [Enumerating SKUs for a new resource](#enumerating-skus-for-a-new-resource)
- [Correlating resources created on behalf of customer](#correlating-resources-created-on-behalf-of-customer)


## Instrumentation and Tracing across services ## 

Resource providers should associate the correlation Id and client request Id with all of their management operations.

This will enable end-to-end troubleshooting from the portal, through CSM and to the RP, via four key headers:

1. x-ms-client-request-id: associated with a logical action in the client; e.g. loading the portal monitoring part may have a single client request id for many calls to ARM (e.g. load VMs + load metric definitions + load metrics)
2. x-ms-correlation-request-id: associated with a logical action in ARM; e.g. deleting a resource group or running a template
3. x-ms-request-id: returned by the RP per request; typically maps to the RP's ActivityId
4. x-ms-routing-request-id: maps to ARM's activity id (which is not otherwise exposed). This maps to implementation boundaries in ARM / the platform –RPs should not use this as their ActivityId.

## ETags for Resources ##

ETags may be returned for individual resources, and then sent via If-Match / If-None-Match headers for concurrency control. The resource provider is responsible for storing and validating ETags – the frontdoor will never inspect these values.

The behavior for ETags can be seen below:

| PUT | Resource does not exist | Resource exists |
| --- | --- | --- |
| If-Match = "" / absent | 201 Created | 200 OK |
| If-Match = "\*" | 412* Precondition Failed | 200 OK |
| If-Match = "xyz" | 412** Precondition Failed | 200 OK / 412 Precondition Failed |
| If-None-Match = "\*" | 201 Created | 412**** Precondition Failed |


| PATCH | Resource does not exist | Resource exists |
| --- | --- | --- |
| If-Match = "" / absent | 404 Not Found | 200 OK |
| If-Match = "\*" | 404** Not Found | 200 OK |
| If-Match = "xyz" | 404** Not Found | 200 OK / 412 Precondition Failed |

| DELETE | Resource does not exist | Resource exists |
| --- | --- | --- |
| If-Match = "" / absent | 204 No Content | 200 OK |
| If-Match = "\*" | 204*** No Content | 200 OK |
| If-Match = "xyz" | 204*** No Content | 200 OK / 412 Precondition Failed |

## Regional Endpoints ##

The Resource Provider should partition services across multiple regions in order to allow for continued service if there is a regional outage. It is recommended that the RP offer services in all Azure supported regions so that users may collocate interdependent resources within a resource group.

The RP should register all the regions that they support, and should provide a different regional endpoint for each region. This regional endpoint and its backing state/data should be isolated from the other regions in order to isolate failure within Azure. ARM will proxy the request to the appropriate region based on the resource&#39;s location.

The resource provider should register regional DNS names for each regional endpoint. The naming of this region should follow the below convention:

1. {region}.{provider namespace}.azure.com
2. {region}.{subservice name}.{provider namespace}.azure.com

In a concrete form:

1. westus.cache.api.azure.com, eastus.cache.api.azure.com
2. westus.redis.cache.api.azure.com

## Nested Resources ##

The RPC supports registering "nested" resource types for an RP with this format:

 _{RPNS}/{TYPE1}/{NAME1}/{TYPE2}/{NAME2}._

The resource provider URIs are still expected to follow REST guidelines (e.g.: azure.com/subscriptions/{SubId}/resourceGroup/ {resourceGroup\_name}/providers/{namespace}/{type1}/{type1\_name}/{type2}/{type2\_name}

Please see example URLs for nested resource types below:

| Http Method | URI |
| --- | --- |
| PUT | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}/{nestedResourceName}?api-version={api-version} |
| GET | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}/{nestedResourceName}?api-version={api-version} |
| GET | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}?api-version={api-version} |
| PATCH | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}/{nestedResourceName}?api-version={api-version} |
| DELETE | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}/{nestedResourceName}?api-version={api-version} |

## Resource Group Deletes ##

If the resource group containing resources is deleted, the RP will receive a DELETE call on each individual tracked resource in that group. In some cases, the resources will have interdependencies and therefore can only be deleted in a certain order that ARM does not understand.

To address this, ARM can simply ask each resource to delete in arbitrary order. If the resource refuses the deletion, then ARM will revisit it after trying to delete the rest of the group&#39;s resources. It will continue to revisit resources until all tracked resources are deleted, or there is no further delete progress (for example if a delete fails because of an ACL).

As a result, it is essential that the RPs follow proper REST guidelines when returning 2xx for a successful delete, and 4xx for a delete that is invalid (until a related resource is deleted first).

## Asynchronous Operations ##

Some REST operations can take a long time to complete. Although REST is not supposed to be stateful, some operations are made asynchronous while waiting for the state machine to create the resources, and will reply before the operation on resources are completed. For such operations, the following guidance applies.

**Please note:**

    * Location header should be absolute URI (partial URI is not supported)
    * Retry-After should be (integer), will not support http-date
    * Azure-AsyncOperation header should be absolute URI (partial URI is not supported)

## Creating or Updating Resources Asynchronously ##

### Creating/Updating using PUT ###

The API flow for PUT should be to:

1. Respond to the initial PUT request with a 202 Accepted.
2. The response headers **MUST** include a Location header that points to a URL where the ongoing operation can be monitored.
3. **Optional:** The response headers may include an Azure-AsyncOperation header pointing to an Operation resource (as described below).
4. Future GETs on the resource that was created should continue to return a 200 Status Code and provisioningState field that is \*non-terminal\* as long as the provisioning is in progress.
5. After the provisioning completes, the provisioningState field should transition to one of the terminal states (as described below).
6. The provisioningState field should be returned on all future GETs, even after it is complete, until some other operation (e.g. a DELETE or UPDATE) causes it to transition to a non-terminal state.

The PUT operation should return 200 OK if the current state already matches the state specified in the content of PUT request, AND there is no ongoing operation that would change that state. This reduces deployment times and improves odds of success. The PUT operation may (but is not required to) return 200 OK for cases when the operation can be finished in a timely fashion, such as when the user is updating tags or other informational properties. Whenever PUT operation returns 200 OK for a final state, the response **MUST NOT** include the Location header, and neither the Azure-AsyncOperation header. The only exception for this is compatibility with old clients that expect that header (see the important note below).

The RFC7231 dictates that PUT must be idempotent and the client might be retrying from a network or client-side error. In order to increase odds of success, the server should avoid starting the operation again if it already started. This can be performed by comparing the content of PUT request with the goal state of the ongoing operation. If it matches, the server should not start a new asynchronous operation. Instead it may immediately return 202 Accepted, with the same or compatible values in the Location header and in the optional Azure-AsyncOperation header of the initial PUT request. It is valid to give same polling URIs to different clients, provided they requested the same operation.

The server must never have more than one ongoing PUT operation for a given resource. If there is an ongoing operation with a goal state that does not match the one specified in the content of PUT request, the server should immediately return a 409 Conflict without doing any change. Similarly, if the server receives PUT request while there is an ongoing DELETE operation, the server should return 409 Conflict immediately.

**IMPOTANT:** Previous versions of this document stated that PUT could return 200 OK with a non-final provisioning state and the Azure-AsyncOperation header. It is therefore recommended that whenever a client sees a 200 OK, it should check for the presence of Azure-AsyncOperation header. If the header is present, the client must assume that the operation did not finish and start polling the URI from that header.

### Updating using PATCH ###

The API flow for PATCH on an existing resource should be to:

1. Respond to the initial PATCH request with a 202 Accepted.
2. The response headers **MUST** include a Location header that points to a URL where the ongoing operation can be monitored.
3. **Optional:** The response headers may include an Azure-AsyncOperation header pointing to an Operation resource (as described below).
4. If a provisioningState field is used for the resource, it **MUST** transition to a non-terminal state like &quot;Updating&quot;.
5. If the PATCH completes successfully, the URL that was returned in the Location header **MUST** now return what would have been a successful response if the API completed (e.g. a response body / header / status code).

The PATCH operation may also return 200 OK or 409 Conflict in certain cases. See guidance of PUT operation for details.

Different than PUT, the server may allow multiple ongoing PATCH operations, provided they are changing different properties of the resource. In other words, multiple ongoing operations are allowed as long as they do not cause conflicts between each other. Note that this requires the server to compare goal states of different PATCH requests and ensure no interference between requests. It is typically simpler to just return 409 Conflict whenever there is another ongoing operation.

While RFC7231 does not require PATCH to be idempotent, PATCH operations typically use same payload of PUT. The main difference is that in PATCH, the client doesn't want to change absent properties, while in PUT, absent properties should be reset to server default value. With this usage, PATCH can be idempotent and it is recommended that the server does this way.

## Delete Resource Asynchronously ##

The API flow should be to:

1. Respond to the initial DELETE request with a 202 Accepted.
2. The response headers **MUST** include a Location header that points to a URL where the ongoing operation can be monitored.
3. **Optional:** The response headers may include an Azure-AsyncOperation header pointing to an Operation resource (as described below).
4. If a provisioningState field is used for the resource, it **MUST** transition to a non-terminal state like &quot;Deleting&quot;.
5. If the DELETE completes successfully, the URL that was returned in the Location header **MUST** now return a 200 OK or 204 NoContent to indicate success and the resource **MUST** disappear.

The DELETE operation should return 404 if the object does not exist, AND there is no ongoing operation that would change that state. The DELETE operation may (but is not required to) return 200 OK or 204 No Content for cases when the operation can be finished in a timely fashion, such as when the user is deleting a resource for which there is no cleanup necessary.

The RFC7231 dictates that DELETE must be idempotent and the client might be retrying from a network or client-side error. In order to increase odds of success, the server should avoid starting the operation again if it already started. This can be performed by checking if there is already an ongoing DELETE operation. If yes, the server should not start a new asynchronous operation. Instead it may immediately return 202 Accepted, with the same or compatible values in the Location header and in the optional Azure-AsyncOperation header of the initial DELETE request. It is valid to give same polling URIs to different clients, provided they requested the same operation.

If the server receives a DELETE request while there is an ongoing PUT or PATCH operation, the server should return 409 Conflict immediately.

## Call Action POST Asynchronously ##

The API flow for POST {resourceUrl}/{action} should be:

1. Respond to the initial POST request with a 202 Accepted.
2. The response headers **MUST** include a Location header that points to a URL where the ongoing operation can be monitored.
3. **Optional:** The response headers may include an Azure-AsyncOperation header pointing to an Operation resource (as described below).
4. If the POST completes successfully, the URL that was returned in the Location header **MUST** now return what would have been a successful response if the API completed (e.g. a response body / header / status code). It is acceptable for this to be a 200/204 NoContent if the action does not require a response (e.g. restarting a VM).

The POST operation may (but is not required to) return 200 OK if the server can finish in a timely fashion.

There is no need for POST to be idempotent, but the server may avoid starting the operation again if it already started, to increase odds of success. This can be performed by comparing the content of POST request with the goal state of the ongoing operation. If it matches, the server may immediately return 202 Accepted, with the same or compatible values in the Location header and in the optional Azure-AsyncOperation header of the initial PUT request. It is valid to give same polling URIs to different clients, provided they requested the same operation.

The POST operation should return 409 Conflict if there is an ongoing operation with a different goal state. POST should also return 409 Conflict if there is an ongoing DELETE operation.

## ProvisioningState property ##

The provisioningState field has three terminal states: **Succeeded** , **Failed** and **Canceled**. If the resource returns no provisioningState, it is assumed to be **Succeeded**.

Each individual RP is able to define their own transitioning / ephemeral states that are set before the resource reaches these terminal states (e.g. "PreparingVMDisk", "MountingDrives", "SelectingHosts" etc.).

The basic functionality of the template orchestration engine and other clients is to:

1. PUT the resource to be created; and, then,
2. Poll at the resourceUri until the provisioningState enters a terminal state

before assuming that the resource creation has completed. In the case of the template orchestration engine, it would not create dependent resources until the resource creation is completed.

An example of the response body can be seen below:

**Response Body**

    {
      "location": "North US",
      "tags": {
        "key1": "value 1",
        "key2": "value 2"
      },

      "properties": {
        "provisioningState": "Succeeded",
        "comment": "Resource defined structure"
      },
    }

If the user does a PUT with provisioningState (e.g. after doing a GET), the resource provider should treat the field as a normal readonly property.

If the user provides a provisioningState matching the value \*on the server\*, the server should ignore it. If the user provides a provisioningState that does \*not\* match the value on the server, the resource provider should reject the call as a 400 BadReqeust.

## 202 Accepted and Location Headers ##

The asynchronous operation APIs frequently use the 202 Accepted status code and Location headers to indicate progress. The flow is described below:

1. Return HTTP code 202 (Accepted) with a Location header and, optionally, a Retry-After header. The time interval in the Retry-After header can only be specified in seconds, with a minimum of 10 seconds and a maximum of 10 minutes.
2. Clients invoke the URI specified in the Location header using the GET verb.
3. Clients should wait for the Retry-After interval, if it was specified, or the default of 60 seconds if it was not, before polling again.
4. If the operation is not complete, return 202 (Accepted) again with Location header and optionally an updated Retry-After header.
5. If the operation is complete, return the **exact same response** that would have been returned had the operation been executed synchronously.
6. If the 202 was returned in response to resource creation, the resource should \*not\* be visible until the async operation completes (i.e. GETs on the resource should return 404 and collection requests should not include the resource). If the resource should be visible, the provisioningState pattern should be used.

The Uri for the Location header can be either underneath the original request:

https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/{Original-URL-Fo-Request}/operationresults/{operationId}?api-version={api-version}

or underneath a subscription level operations URL:

https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/providers/{namespace}/locations/westus/operationresults/{operationId}?api-version={api-version}

The _endpoint_ can be determined by the hostname of the URI in the _referer_ header.

After this maximum time, clients will give up and treat the operation as timed out. Please note that the location header needs to be the full absolute URI.

| Clients | Resource Provider |
| --- | --- |
| DELETE /…/resourcegroups/rg1 |   |
|   | 202 AcceptedLocation: /…/resourcegroups/rg1/operationresults/id1 Retry-After: 60 |
| Waits 60 seconds |   |
| GET /…/resourcegroups/rg1/ operationresults/id1  |   |
|   | HTTP/1.1 202 AcceptedLocation: /…/resourcegroups/rg1/operationresults/id1 Retry-After: 10 |
| Waits 10 seconds |   |
| GET /…/resourcegroups/rg1/ operationresults/id1 |     |
|   | 204 NOCONTENT |

## Operation Resource format ##
The operation resource format returned by the Azure-AsyncOperation header is as follows-

    {
      "id": "/subscriptions/id/locations/westus/operationsStatus/sameguid",
      "name": "sameguid",
      "status" : "RP defined values | Succeeded | Failed | Canceled",
      "startTime": "<DateLiteral per ISO8601>",
      "endTime": "<DateLiteral per ISO8601>",
      "percentComplete": <double between 0 and 100>
      "properties": {
        /\* The resource provider can choose the values here, but it should only be
            returned on a successful operation (status being "Succeeded"). \*/
      },
      "error" : {
        /\* RP must return the *code* and *messages* fields. Please use the schema for the "ErrorResponse" Type from the Common Types definition in the Azure Rest API Specifications [repository](https://github.com/Azure/azure-rest-api-specs/blob/master/specification/common-types/resource-management/v2/types.json) \*/
        "code": "BadArgument",
        "message": "The provided database &#39;foo&#39; has an invalid username."
      }
    }

**The operation resource should be used for Azure-AsyncOperation header**. Location header should have different behavior described above.

The status values are similar to provisioningState – the transient / intermediate values can be service specific; however, the terminal values can be one of (1) Succeeded, (2) Failed or (3) Canceled. All other values imply that the operation is still running / being applied.

When reading the operation resource entity, the status code should always be 200. The response payload indicates if the async operation is still running or has completed (successfully or with a failure).

If the response status code is a 4xx or 5xx value, it indicates a failure in reading the _status_ but \*not\* in the underlying operation.

| Element name | Description |
| --- | --- |
| id | **Optional.** (but recommended). It should match what is used to GET the operation result. |
| name | **Optional.** (but recommended). It must match the last segment of the &quot;id&quot; field, and will typically be a GUID / system generated value. |
| status | **Required.** … |
| properties | **Optional** Like the properites envelope for resources, the resource provider can choose the values here. However, it should only be set if the status is Succeeded. |
| error | **Required if status == failed or status == canceled.** This is the OData v4 error format, used by the RPC and will go into the v2.2 Azure REST API guidelines.  The full set of optional properties (e.g. inner errors / details) can be found in the &quot;Error Response&quot; section. |
| error.code | **Required if status == failed or status == canceled.**  If status == failed, provide an invariant error code used for error troubleshooting, aggregation, and analysis. |
| error.message | **Required if status == failed.**   **Localized.**  If status == failed, provide an actionable error message indicating what error occurred, and what the user can do to address the issue. |
| startTime | **Optional.**  DateLiteral using ISO8601, per 1API guidelines. |
| endTime | **Optional.**  DateLiteral using ISO8601, per 1API guidelines. |
| percentComplete | **Optional.**  Double between 0 and 100. |

## Retrying REST Calls ##

The recommended pattern for REST clients is to retry calls that:

1. Return 408 indicating a request timeout;
2. Return a 5xx status code indicating a server failure (e.g. 500 InternalServerError; 503 Service Unavailable);
3. Experience a socket exception (e.g. DNS resolution failures; network drop; no listening socket).
4. Any other response where a Retry-After header is present.

The retry interval, strategy (e.g. exponential/linear) and max timeout should be defined by the _server_ and not the client. The server indicates how long it expects the client to wait before retrying the call by including a (Retry-After header)[https://tools.ietf.org/html/rfc7231#section-7.1.3] in the response. That allows various server workloads to advertise different retry policies.

## Designing Resources ##

The Azure Resource Provider contract and template language have special semantics around &quot;registered&quot; Azure resources – we call these **tracked resources**.

If these objects are addressable by URIs but not "registered", we call these **proxy only** resources to differentiate them.

**Background REST philosophy**

1. Azure API guideline compliant and OData compliant where it makes sense.
2. APIs must be callable from any client implementation, language or framework.
3. Making things simple and regular for machine processing often makes things easier for humans to understand, even when it increases verbosity.
4. When we choose between understandable URLs and understandable templates, we should make the templates human understandable and the URLs machine consumable.
5. Everything that is possible from a template should be possible through imperative APIs, but not necessarily as easily and may require special client code (orchestration is an obvious example).

**Objects should be Tracked resources if:**

1. They are billable entities, including freemium models
2. Need to be orchestrated by the template execution engine
3. Related to above, they have dependencies on other resources

**Objects should be top-level resources if:**

1. They should be resources as per above
2. Their lifetime is independent of other resources (only dependent on the containing resource group)
3. If the objects have an independent lifetime but are not important to the model, they should only be top-level resources if no other design can be made to work.

**Objects should be nested resources if:**

1. They meet the guidelines for Azure resources above (billable, orchestration supported, taggable)
2. Their lifetime is dependent on a containing resource

**Objects should be proxy only resources if:**

1. They don&#39;t need to be Azure resources for one of the reasons above
2. if the objects/properties are required for the parent resource to be created, then they should \*not\* be Azure resources
3. It is important that they be simple to create in context of a containing resource.
4. The lifecycle of the resources are managed outside ARM (e.g. agent based discovery).
5. They are very common and would incur a performance overhead on the system (for example by cluttering the indices and GET resources calls)

**Comparison of Tracked Resources and proxy only Resources**

| ​ **Feature** | ​ **Tracked Resources** | ​ **Proxy only Resources** |
| --- | --- | --- |
| ​Understood by template processor | ​Yes | ​ Yes |
| Can be top-level resource | Yes | Only if global/single region |
| Defined JSON format to payload | ​Yes -- must comply with general Azure resource format (id, type, name, tags, properties, location etc.) | ​Yes -- must comply with general Azure resource format (id, type, name, properties, etc.) but no tags or location |
| Defined URL format | ​Yes -- must use providers/{namespace}/{typename} URL | ​Yes -- must use providers/{namespace}/{typename} URL |
| ​Tagging and Indexing supported | ​Yes | ​No |
| ​Billing supported     | ​Yes | ​No |
| ​Have their own URLs | ​Yes | ​Yes |
| ​​Can be PUT (created) with the parent in a single call | ​No - $batch/deep insert not available | ​​Yes, see deep insert recommendation |
| ​Supports RBAC | ​Yes -- based on URL     | ​ Yes -- based on URL |
| ​Actions are advertised and supported by RBAC experience (/reboot for example) | ​Yes | ​Optional |
| ​Supports targeted imperative GET | ​Yes | ​ Yes |
| ​Supports targeted imperative PUT, PATCH | ​Yes | ​ Yes |
| ​Supports targeted DELETE | ​Yes | ​Yes |
| ​DELETEd when parent is | ​Yes, governed by deletionPolicy in Manfiest | ​RP must implement cascading delete and/or validation |
| Independent lifetime from parent | ​DELETEd when parent is, but otherwise independent | ​ RP must implement cascading delete and/or validation, but otherwise independent |
| ​Supported by orchestration service         | ​Yes | ​ Yes |
| ​Can take dependencies (DependsOn, [reference] on other resources     | ​Yes     | ​Yes |
| ​Can be the target of dependencies from other resources | ​Yes | ​Yes |
| ​In template can take values from variables | ​Yes | ​Yes |
| ​In template can take values from other resources | ​Yes | ​Yes |
| ​Can be independently included in external templates | ​Yes | ​Yes |
| ​Can be versioned independently from parent     | ​Yes | ​Yes |

## Enumerating SKUs for an existing resource ##

In many cases, it is desirable for users to resize an existing resource.  However, only a certain subset of sizes available from a given resource provider may be applicable to a given resource – e.g. an A1 machine may be updatable to other A series VMs, but may not be convertible to D series.  This API exposes the valid SKUs for an existing resource.

### Request ###

| Method | Request URI |
| --- | --- |
| GET | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{resourceProviderNamespace}/{resourceType}/{resourceName}/skus?api-version={api-version} |

**Arguments**

| Argument | Description |
| --- | --- |
| subscriptionId | The subscriptionId for the Azure user. |
| resourceGroupName | The resource group name uniquely identifies the resource group within the user subscriptionId. |
| resourceType | The type of the resource – the resource providers declare the resource types they support at the time of registering with Azure. |
| resourceName | The name of the resource. |
| api-version | Specifies the version of the protocol used to make this request. Format must match YYYY-MM-DD[-preview|-alpha|-beta|-rc|-privatepreview]. |

**Request Headers**

Only headers common to all requests.

**Request Body**

Empty

### Response ###

The response includes an HTTP status code, a set of response headers, and a response body.

**Status Code**

The resource provider should return 200 (OK) to indicate that the operation completed successfully.

If the resource does not exist, 404 (NotFound) should be returned. For other errors (e.g. internal errors) use the appropriate HTTP error code.  If no skus are valid, return 200 + an empty value array.

**Response Headers**

Headers common to all responses.

**Response Body**

    {
      "value": [
        {
          "resourceType": "type for sku"
          "sku": {
            "name": "sku name",
            "tier": "sku tier"
          },

          "capacity": {
            "minimum": min,
            "maximum": max,
            "default": default,
            "scaleType": "automatic|manual|none"
          }
        },

        {
          "resourceType": "type for sku"
          "sku": {
            "name": "sku name",
            "tier": "sku tier"
          },

          "capacity": {
            "minimum": min,
            "maximum": max,
            "default": default,
            "scaleType": "automatic|manual|none"
          }
        }
      ]
    }

For a detailed explanation of each field in the response body, please refer to the request body description in the PUT resource section. The only GET specific properties are "name", "type" and "id"

| Field | Description |
| --- | --- |
| resourceType | Required, string. The resource type that this object applies to. For example, for a database it'd be: Microsoft.SQL/servers/databases.   |
| sku | Required, object. The exact set of keys that define this sku. This matches the fields on the respective resource. |
| sku.name | Required, string. The name of the SKU. This is typically a letter + number code, such as A0 or P3 |
| sku.tier | Optional, string. The tier of this particular SKU. Typically one of: <ul> <li>Free</li> <li>Basic</li> <li>Standard</li> <li>Premium</li></ul>|
| capacity | Optional, object. If the SKU supports scale out/in then the capacity object must be included. If scale out/in is not possible for the resource this may be omitted. |
| capacity.scaleType | Required, enum. One of: <ul><li>None – meaning the capacity is not adjustable in any way (e.g. it is fixed)</li> <li>Manual – the user must manually scale out/in</li> <li>Automatic – the user is permitted to scale this SKU in and out</li></ul>|
| capacity.minimum | Required if scaleType != none, integerThe lowest permitted capacity for this resource. Typically 0 or 1. |
| capacity.maximum | Required if scaleType != none, integerThe highest permitted capacity for this resource. This may vary per SKU, or it may be the same. |
| capacity.default | Required if scaleType != none, integerThe default capacity.  Generally, if the user omits setting the capacity, the RP would use this value. |

## Enumerating SKUs for a new resource ##

For new resources, SKUs are enumerated via ARM's metadata store.  This allows users to enumerate SKUs based on location, api-version, feature flags, and offerId filters.

## Correlating resources created on behalf of customer ##
It is required that Resource Providers (RP) tag the resources when making service-to-service (S2S) calls using their internal subscription. When a customer call to provision a resource comes to a RP and that RP needs to call another RP using an internal subscription to complete the request, it must tag the resource(s) with the customer’s original fullyqualified resourceId. This tag will not be visible to the customers as it will reside in the internal subscription only. This is applicable for both PUT andPATCH.

The tag should have the following format –
Tag Key – resourceCreatedFor
TagValue – {fullyqualified resourceId of the resourcerequested by the customer. Ex-/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{resourceProviderNamespace}/{resourceType}/{resourcename}}

Example – When a customer requests an HDInsight cluster with
7 workers, HDI RP will call CRP to provision 7 VMs. It is now required that HDI
RP will add the tags to the request for creating the 7 VMs.

