# API Reference

If you're unfamiliar with mbed Device Connector please read our short introduction to [general concepts](index.md).

The mbed Device Connector Web API offers functions that manage:

* [Versions](#versions)
* [Endpoints](#endpoints)
* [Resources](#resources)
* [Notifications](#notifications)
* [Subscriptions](#subscriptions)
* [Traffic limits](#traffic-limits)

__A note about asynchronous functions__

A number of functions in the mbed Device Connector API are asynchronous, because it's not guaranteed that an action (such as writing to a device) will happen straight away, as the device might be in deep sleep. These APIs are marked with '(async)' in the API reference. For information about handling asynchronous functions, see: [Asynchronous requests](index.md#asynchronous-requests).

## Versions

### Detecting the versions of REST API

To check the version of the mbed Device Connector API:

	GET /rest-versions

The most recent version of the API is the last element in the result list.

**Response**

Response|Description
---|---
200|Successful response with a list of versions supported by the server.

Acceptable content-types:

- application/json

**Example**

    GET /rest-versions
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json

    [
        "v1",
        "v2"
    ]


#### Calling the latest version of the REST API

To call the latest version of the API, you can either use the existing URI without any changes, or you can prefix the latest version of the above list to your URI:

**Example**

    /some-url
    /v2/some-url


#### Calling an old version of the REST API

To call an old version of the API, you must prefix the desired version of the above list to your URI:

**Example**

    /v1/some-url

## Endpoints

Endpoints are physical devices running [mbed Client](https://www.mbed.com/en/development/software/mbed-client/). For more information about mbed Device Connector’s data model, see the [REST API introduction](index.md#the-mbed-device-connector-data-model).

### Listing all endpoints

	GET /endpoints


**Query string parameters**

Name|Type|Description
---|---|---
type|`string`|Filter endpoints by endpoint-type.

**Response**

Response|Description
---|---
200|Successful response with a list of endpoints.

Acceptable content-types:

- application/json
- application/link-format

*Endpoint list JSON structure*

```json
[{
  "name": "{endpoint name, set by the client in M2MInterfaceFactory::create_interface}",
  "type": "{endpoint type, set by the client in M2MInterfaceFactory::create_interface}",
  "status": "{endpoint status: always ACTIVE, but reserved for future use}",
  "q": "{queue mode: true|false (default: false)}"
}]
```

**Example**

    GET /endpoints
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json

    [
      { "name":"node-001", "type":"Light", "status":"ACTIVE"},
      { "name":"node-003", "type":"QueueModeNode", "status":"ACTIVE", "q": true}
    ]

#### Queue mode

When an endpoint is in queue mode, messages sent to the endpoint will not wake up the physical device. Rather, the messages are queued and delivered when the device wakes up and connects to mbed Device Connector itself. Queue mode can also be used when the device is behind a NAT and cannot be reached directly by mbed Device Connector.

### Listing an endpoint's resources

	GET /endpoint/{endpoint-name}

The list of resources are cached by mbed Device Connector, and this call does not wake up the device.

**Note:** `endpoint-name` needs to be an exact match of the name of the endpoint. It is not possible to use wildcards here.

**Response**

Response|Description
---|---
200|Successful response with a list of resources.
404|Endpoint not found.

Acceptable content-types:

- application/json
- application/link-format

*Endpoint resources JSON structure*

```json
[{
  "uri": "{URL for this resource}",
  "rt": "{resource type}",
  "obs": "{observable, whether you can subscribe to changes for this resource: true|false}",
  "type": "{content type}"
}]
```

**Example**

    GET /endpoints/node-001
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json

    [
      { "uri":"/actuator/0/speed", "rt":"5851", "obs":"true", "type":"text/plain"},
      { "uri":"/led/1/state", "rt":"5850", "obs":"false"}
    ]

*You are encouraged to use [LWM2M](http://technical.openmobilealliance.org/Technical/technical-information/omna/lightweight-m2m-lwm2m-object-registry) resource types.*

## Resources

All resource APIs are [asynchronous](index.md#asynchronous-requests). Be aware that these APIs will only respond if the device is turned on and connected to mbed Device Connector. You can also receive [notifications](#notifications) when a resource changes.

### Non-confirmable requests

All resource APIs have the parameter `noResp`. If a request is made with `noResp=true`, mbed Device Connector makes a CoAP non-confirmable requests to the device. This means that these requests are not guaranteed to arrive at the device, nor do you get an `async-response-id` back.

If calls with this parameter enabled succeed, they return with the status code `204 No Content`. If the underlying protocol does not support non-confirmable requests, or if the endpoint is registered in queue mode, the response is status code `409 Conflict`.

### Reading from a resource ([async](index.md#asynchronous-requests))

    GET /endpoint/{endpoint-name}/{resource-path}

**Query string parameters**

|Name|Type|Description|
|---|---|---|
|cacheOnly|`boolean`|If true, the response will come only from cache. Optional. Default: `false`.|
|noResp|`boolean`|See [Non-confirmable requests](#non-confirmable-requests)|

**Response**

|Response|Description|
|---|---|
|202|Accepted. Returns an asynchronous response ID.|
|205|No cache available for resource.|
|404|Requested endpoint's resource is not found.|
|409|Conflict. Endpoint is in queue mode and synchronous request can not be made. If noResp=true, the request is not supported.|
|410|Gone. Endpoint not found.|
|412|Request payload was not valid.|
|413|Precondition failed.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|

Acceptable content types:

- application/json

*Resource JSON structure*

```
[{
  "id": "{async response ID}",
  "status": "{HTTP status code, for example 200 for OK}",
  "payload": "{requested data, base64 encoded}",
  "ct": "{content type}",
  "max-age": {how long this value will be valid, in seconds}
}]
```

**Example**

    GET /endpoints/node-001/actuator/0/speed
    Accept: application/json

    HTTP/1.1 200 OK
    Content-Type: application/json

    {"async-response-id": "1073741829#521f9d17-c5d7-4769..."}

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "1073741829#521f9d17-c5d7-4769...",
    "status": 200,
    "payload": "MTI=",
    "ct": "text/plain",
    "max-age": 0
  }]
}
```

### Writing to a resource ([async](index.md#asynchronous-requests))

This API can be used to write new values to existing resources, or to create new resources on the device. The `resource-path` does not have to exist - it can be created by the call.

    PUT /endpoint/{endpoint-name}/{resource-path}

**Query string parameters**

|Name|Type|Description|
|---|---|---|
|noResp|`boolean`|See [Non-confirmable requests](#non-confirmable-requests)|


**Response**

|Response|Description|
|---|---|
|202|Accepted. Returns an asynchronous response ID.|
|409|Conflict. Endpoint is in queue mode and synchronous request can not be made. If noResp=true, the request is not supported.|
|410|Gone. Endpoint not found.|
|412|Request payload was not valid.|
|413|Precondition failed.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|

Acceptable content types:

- application/json

*Resource JSON structure*

```
[{
  "id": "{async response ID}",
  "status": "{HTTP status code, for example 200 for OK}",
  "payload": "",
  "ct": "{content type}",
  "max-age": {how long this value will be valid, in seconds}
}]
```

**Example**

    PUT /endpoints/node-001/actuator/0/speed
    Accept: application/json
    Content-Type: text/plain

    95

    HTTP/1.1 200 OK
    Content-Type: application/json

    {"async-response-id": "1073741827#521f9d17-c5d7-4769..."}

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "1073741827#521f9d17-c5d7-4769...",
    "status": 200,
    "payload": "",
    "ct": "text/plain",
    "max-age": 60
  }]
}
```

**Handling the update in mbed Client**

The [mbed Client overview](https://docs.mbed.com/docs/mbed-client-guide/en/latest/Introduction/#the-write-operation) contains information on how to process updates on the device side.

### Executing a function on a resource ([async](index.md#asynchronous-requests))

    POST /endpoint/{endpoint-name}/{resource-path}

**Query string parameters**

|Name|Type|Description|
|---|---|---|
|noResp|`boolean`|See [Non-confirmable requests](#non-confirmable-requests)|

The body of the request  is passed in as a `char*` to the function in mbed Client.

**Response**

|Response|Description|
|---|---|
|202|Accepted. Returns an asynchronous response ID.|
|404|Requested endpoint's resource is not found.|
|409|Conflict. Endpoint is in queue mode and synchronous request cannot be made. If noResp=true, the request is not supported.|
|410|Gone. Endpoint not found.|
|412|Request payload was not valid.|
|413|Precondition failed.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling ongoing another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|


Acceptable content types:

- application/json

*Resource JSON structure*

```
[{
  "id": "{async response ID}",
  "status": "{HTTP status code, for example 200 for OK}",
  "payload": "",
  "ct": "{content type}",
  "max-age": {how long this value will be valid, in seconds}
}]
```

**Example**

    POST /endpoints/node-001/actuator/0/drive
    Accept: application/json
    Content-Type: text/plain

    Data that will be passed to the function on the device

    HTTP/1.1 202 OK
    Content-Type: application/json

    {"async-response-id": "1073741827#521f9d17-c5d7-4769..."}

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "1073741827#521f9d17-c5d7-4769...",
    "status": 200,
    "payload": "",
    "ct": "text/plain",
    "max-age": 60
  }]
}
```

**Handling the execute operation in mbed Client**

The [mbed Client overview](https://docs.mbed.com/docs/mbed-client-guide/en/latest/Introduction/#the-execute-operation) contains information on how to handle the execute operation on the device side.

### Deleting a resource ([async](index.md#asynchronous-requests))

A request to delete a resource must be handled by both mbed Client and mbed Device Connector. The resource is not deleted from mbed Device Connector until the delete is handled by mbed Client.


    DELETE /endpoint/{endpoint-name}/{resource-path}

|Name|Type|Description|
|---|---|---|
|noResp|`boolean`|See [Non-confirmable requests](#non-confirmable-requests)|


**Response**

|Response|Description|
|---|---|
|202|Accepted. Returns an asynchronous response ID.|
|404|Requested endpoint's resource is not found.|
|409|Conflict. Endpoint is in queue mode and synchronous request can not be made. If noResp=true, the request is not supported.|
|410|Gone. Endpoint not found.|
|412|Request payload was not valid.|
|413|Precondition failed.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling ongoing another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|


Acceptable content types:

- application/json

*Resource JSON structure*

```
[{
  "id": "{async response ID}",
  "status": "{HTTP status code, for example 200 for OK}",
  "payload": "",
  "ct": "{content type}",
  "max-age": {how long this value will be valid, in seconds}
}]
```

**Example**

    DELETE /endpoints/node-001/logs/0/debug
    Accept: application/json
    Content-Type: text/plain

    HTTP/1.1 202 OK
    Content-Type: application/json

    {"async-response-id": "1073741827#521f9d17-c5d7-4769..."}

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "1073741827#521f9d17-c5d7-4769...",
    "status": 200,
    "payload": "",
    "ct": "text/plain",
    "max-age": 60
  }]
}
```

**Handling the delete operation in mbed Client**

The [mbed Client overview](https://docs.mbed.com/docs/mbed-client-guide/en/latest/Introduction/) contains information on how to handle the delete operation on the device side.

## Notifications

Resources can be marked as observable, which allows your application to subscribe to updates from these resources, meaning the server will notify  the  application when a resource changes. An application can either [subscribe to an individual resource](#subscriptions), or use automatic subscriptions  based on matching endpoints to [pre-subscription data](#automatically-subscribe-to-resources).

Whether or not a resource is observable is determined by [mbed Client](https://docs.mbed.com/docs/mbed-client-guide/en/latest/Introduction/#the-observe-feature), not by mbed Device Connector.

### Receiving notifications

There are two ways of receiving notifications:

* [Register a notification callback](#registering-a-notification-callback).
* [Use long polling](#long-polling).

We'll review the notification data structure before looking at the registration and notification functions.

#### Notification data structure

A notification message contains the following server event types:

|Notification type|Description|
|---|---|
|notifications|Contains resource notifications. **Note that the payload is base64-encoded.**|
|registrations|List of new endpoints that have registered with mbed Device Connector (with resources).|
|reg-updates|List of endpoints that have updated registration.|
|de-registrations|List of endpoints that were removed in a controlled manner.|
|registrations-expired|List of endpoints that were removed because the registration has expired.|
|async-responses|Responses to asynchronous proxy request. **Note that the payload is base64-encoded.**|

The notification message is delivered in JSON format:

    {
       "notifications":[
         {
            "ep":"{endpoint-name}",
            "path":"{uri-path}",
            "ct":"{content-type}",
            "payload":"{base64-encoded-payload}",
            "timestamp":"{timestamp}"
            "max-age":"{max-age}"
         }
       ],

       "registrations":[
         {
           "ep": "{endpoint-name}",
           "ept": "{endpoint-type}",
           "q": "{queue-mode, default: false}",
           "resources": [ {
             "path": "{uri-path}",
             "if": "{interface-description}",
             "rf": "{resource-type}"
             "ct": "{content-type}",
             "obs": "{is-observable (true|false) }"
           } ]
         }
       ],

       "reg-updates":[
         {
           "ep": "{endpoint-name}"
           "ept": "{endpoint-type}"
           "q": "{queue-mode, default: false}",
           "resources": [ {
             "path": "{uri-path}",
             "if": "{interface-description}",
             "rf": "{resource-type}"
             "ct": "{content-type}",
             "obs": "{is-observable (true|false) }"
           } ]
         }
       ],

       "de-registrations":[
         {
           "{endpoint-name}",
           "{endpoint-name2}"
         }
       ],

       "registrations-expired":[
         {
           "{endpoint-name}",
           "{endpoint-name2}"
         }
       ],

       "async-responses":[
         {
            "id": "{async-response-id}",
            "status":  {http-status-code},
            "error":  {error-message},
            "ct":  "{content-type}",
            "max-age": {max-age},
            "payload":  "{base64-encoded-payload}"
         }
       ]
    }

### Registering a notification callback

#### Registering a callback URL

Register a URL to which the server should deliver notifications.

    PUT /notification/callback

**Note:** Only one URL can be active. If you register a new URL when another is already active, the old URL is replaced by the new.

**Body**

The body of the request needs to be a JSON object with the following format:

|Name|Type|Description|
|---|---|---|
|url|`string`|The URL to which notifications need to be sent. We recommend that you serve this URL over HTTPS.|
|headers|`object`|Headers (key/value) that need to be sent with the request. Optional.|

**Note:** The total length of the URL and header values must be no more than 400 characters.

The values in the `headers` field are sent as HTTP headers with every request that mbed Device Connector makes to your URL. You could use this to verify that a request originated from mbed Device Connector.

**Response**

|Response|Description |
|---|---|
|204|Successfully subscribed.|
|400|Given URL is not accessible.|

When you register a URL, mbed Device Connector sends a PUT request to that URL immediately.

The server responds with `400 Bad Request` if:
* The URL you passed in is not reachable.
* The total length of the URL and header values is more than 400 characters.

**Tip:** The response body includes information on why the request failed.

**Example:**

    PUT /notification/callback HTTP/1.1
    Content-Type: application/json
    Content-Length: 87

    {"url" : "http://example.com/notification?x=123", "headers" : {"Authorization" : "auth", "test-header" : "test"}}

    HTTP/1.1 204 No Content

#### Checking the notification callback

Returns the notification callback URL and the headers that have been set in [Registering a notification callback](#registering-a-notification-callback).

    GET /notification/callback

**Response**

|Response|Description|
|---|---|
|200|URL found.|
|404|No callback URL is registered.|

**Example**

    GET /notification/callback HTTP/1.1
    Content-Type: application/json
    Content-Length: 87

    {"url" : "example.com", "headers" : {"Authorization" : "auth", "test-header" : "test"}}

#### Deleting the notification callback

To delete the Callback URL and remove all subscriptions:

	DELETE /notification/callback

**Response**

|Response|Description |
|---|---|
|204|Successfully removed.|
|404|No callback URL is registered.|

### Long polling

As an alternative to the notification callback, you can use HTTP long-poll requests to receive notifications. You open a request from the client to mbed Device Connector, and the request will be kept open until either an event notification is delivered to the client or the request times out (after 30 seconds). In both cases, the client should open a new polling request after the previous one closes.

	GET /notification/pull

**Note:** It is mandatory to create a persistent connection, by sending the `Connection: keep-alive` header, to avoid excessive TLS handshakes.

**Response**

|Response|Description |
|---|---|
|200|OK.|
|204|No new notifications.|

## Subscriptions

After you set up [notifications](#notifications), you can subscribe to either individual resources or pre-subscribe to resources

### Subscribing to an individual resource ([async](index.md#asynchronous-requests))

This function subscribes to an individual resource.

	PUT /subscriptions/{endpoint-name}/{resource-path}

**Response**

|Response|Description|
|---|---|
|200|Successfully subscribed.|
|202|Accepted. Asynchronous response ID.|
|404|Endpoint or its resource not found.|
|412|Cannot make a subscription for a non-observable resource.|
|413|Cannot make a subscription due to failed precondition.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|

**Example:**

    PUT /subscriptions/node-001/piezo/0/heart-rate

    HTTP/1.1 202 Accepted
    Content-Type: application/json

    {"async-response-id": "5734979#node-001@test.domain.com/path1"}

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "5734979#node-001@test.domain.com/path1",
    "status": 200,
    "payload": "",
    "ct": "text/plain",
    "max-age": 60
  }]
}
```

### Removing a subscription

	DELETE /subscriptions/{endpoint-name}/{resource-path}

**Response**

|Response|Description |
|---|---|
|204|Successfully removed subscription.|
|404|Endpoint or endpoint's resource not found.|

### Removing all subscriptions

	DELETE /subscriptions

**Response**

|Response|Description|
|---|---|
|204|Successfully removed subscriptions.|

### Checking resource subscription status

	GET /subscriptions/{endpoint-name}/{resource-path}

**Response**

|Response|Description|
|---|---|
|200|Resource is subscribed.|
|404|Resource is not subscribed.|

### Reading endpoint's subscriptions

	GET /subscriptions/{endpoint-name}

**Response**

|Response|Description|
|---|---|
|200|List of subscribed resources.|
|404|Endpoint not found or there are no subscriptions for that endpoint.|

Acceptable content-types:

- text/uri-list

**Example:**

    GET /subscriptions/node-001

    HTTP/1.1 200 OK
    Content-Type: text/uri-list

    /subscriptions/node-001/dev/temp
    /subscriptions/node-001/dev/power


### Removing endpoint's subscriptions

	DELETE /subscriptions/{endpoint-name}

**Response**

|Response|Description|
|---|---|
|204|Successfully removed.|
|404|Endpoint not found.|

### Automatically subscribe to resources

Besides manually subscribing to resources, you can also automatically subscribe to resources based on their endpoint name, endpoint type, or listed resources (a pre-subscription). mbed Device Connector will automatically send a subscription request when it encounters an endpoint or resource that matches these definitions.

    PUT /subscriptions

**Body**

The body has to be a JSON array that contains objects that may have the following fields:

|Name|Type|Description|
|---|---|---|
|endpoint-name|`string`|The endpoint name (optionally having an `*` character at the end).|
|endpoint-type|`string`|The endpoint type.|
|resource-path|`Array`|A list of resources to subscribe to. Allows wildcards to subscribe to multiple resources at once.|

If you send an empty array, the pre-subscription data will be removed.

**Note:** Changing the pre-subscription data removes *all* the existing subscriptions, this includes individual subscriptions.

**Response**

|Response|Description|
|---|---|
|200|Successfully set pre-subscription data.|

**Example**

    PUT /subscriptions
    Content-Type: application/json

    [
       {
         endpoint-name: "node-001",
         resource-path: ["/dev"]
       },
       {
         endpoint-type: "Light",
         resource-path: ["/sen/*"]
       },
       {
         endpoint-name: "node*"
       },
       {
         endpoint-type: "Sensor"
       },
       {
         resource-path: ["/dev/temp","/dev/hum"]
       }
    ]

    -->
    HTTP/1.1 200 OK


### Getting automatic subscriptions

	GET /subscriptions

The server returns with the same JSON structure as described above. If there are no pre-subscribed resources, it returns with an empty array.

## Traffic limits

The number of transactions and registered endpoints for each mbed user is limited per 24 hours. The counter for the number of transactions includes both incoming (endpoint registrations, registration updates and removals, notifications) and outgoing (proxy and subscription requests).

To read the current value of the limit counter:

	GET /limits

**Response**

|Response|Description|
|---|---|
|200|OK|

**Example**

    GET /limits

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
        "transaction-quota": 10000,
        "transaction-count": 7845,
        "endpoint-quota": 100,
        "endpoint-count": 50
    }

