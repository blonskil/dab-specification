
# [DRAFT] Device Automation Bus (DAB)

A common framework to automate non-Android living room devices for test purposes

## Version
This document describes the protocol version **1.0**

## Limitations
* 3rd party applications are not directly integrated with the protocol and there is no direct interaction with the applications. This limitation is imposed only on the current version of the protocol. Deeper integration with the 3rd party applications is planned in the upcoming releases of the protocol.

## About the Specification

The messages are in the JSON format.

## MQTT

(from Wikipedia) MQTT is an ISO standard publish-subscribe-based messaging protocol. It consists of clients communicating with a server, often called a "broker". A client may be either a publisher of information or a subscriber. Each client can connect to the broker.

**DAB** is a protocol built on top of MQTT.

## Actors

### Broker
An MQTT broker installed on the device under test.

### Platform
A component on the device under test that has access to the device's resources

### 3rd Party Applications
Applications that can be installed or are pre-installed on the device under test

## Message Types

### Request / Response
This is a traditional request/response model built on top of the MQTT protocol.

The client publishes a message to a specific topic and includes a <request id>.
The request is handled and the handler returns the response on the defined topic. The response topic is always constructed as follows:

```$xslt
'_response' + <original topic> + <request id>
```

In order to send a message to a request topic, the client needs to append the request_id to the topic.

If the client wants to send a message to the request topic `x/y/z`, it needs to generate a `<request_id>`. The client then needs to publish a message to the MQTT topic `x/y/z/<request_id>`.

The response will be published to `_response/x/y/z/<request_id>`

We recommend that UUID (version4 - random) is used for the request IDs.

In order to receive the response, the client needs to subscribe to the response topic before publishing the request. 
For example:

```
mqtt.subscribe --topic '_response/dab/appLifecycle/list/## c0e1fb9d-03f9-4188-8350-5ba4acdea9f0'
mqtt.publish --topic 'dab/appLifecycle/list/## c0e1fb9d-03f9-4188-8350-5ba4acdea9f0' --payload '{}'
```

## Unsolicited Messages
This is an unsolicited message that can be sent to communicate temporal messages like memory usage. In most cases, the client will subscribe to a topic where these messages are being sent. The client may or may not have to send a request for these messages to be enabled. Subscribing to the topic does not give the client a history or even the last published message - they will only get future messages.

## Retained Messages
Retained messages are a variation of unsolicited messages. They are used to communicate things like state. The difference is that the last message to the topic is "retained". When a client subscribed to a topic, they get the last message so they can sync up with the current state.

## Protocol Compliance

In order for a device to be considered compliant with the protocol all operations need to be implemented. In some cases where the device limitations prevent from an operation from being implemented, the device still needs to handle the request and return the *not implemented* response.

## Application Lifecycle

The Application Lifecycle module is responsible for the lifecycle of the installed applications. There can only be one instance of an application running in the system and the instance is referenced by the application identifier. The protocol equips the devices with the telemetry capabilities that can be enabled during the application launch.

TODO need to describe the application lifecycle

### Application Identifier

Application identifier consist of a sequence of characters. The lower case   letters "a"--"z", digits, and hyphen ("-") are allowed. For resiliency, upper case letters are equivalent to lower case (e.g., allow "netflix" is the same as "Netflix"). The minimum number of characters is 1 and the maximum is 64.

### Listing all the Applications

Lists all the installed applications on the device.

#### Request topic

`dab/appLifecycle/list`

#### Request format

```typescript
interface ListRequest {
}
```

#### Response format

```typescript
interface ListResponse {
	status: number;
	error?: string;
	apps?: string[];
}
```

Parameter | Description
--- | ---
status | status code
error | *optional* description of the error
apps | a list application identifiers, *required* when the operation is successful

#### Statuses
Status | Description
--- | ---
202 | OK, request accepted
400 | bad request


#### Sample response
```json
{
	"status": 200,
	"apps": [
		"netflix",
		"prime-video",
		"YouTube",
	]
}
```

### Launching an application

Launches an application.

#### Request topic

`dab/appLifecycle/launch`

#### Request format

```typescript
type Visibility = 'background' | 'foreground'

interface LaunchApplicationRequest {
	app: string;
	parameters?: string;
	mode?: Visibility;
	telemetry?: TelemetryRequest;
}
```

Parameter | Default|  Description
--- | --- | ---
app | - | application id
parameters |  | *optional*, parameters to pass to the application
mode | foreground | *optional*, supported values: background, foreground
telemetry | - | *optional*; see Telemetry section for the details. enables telemetry with the application

#### Response format

```typescript
interface LaunchApplicationResponse {
	status: number;
	error?: string;
}
```

Parameter | Description
--- | ---
status | status code
error | *optional* description of the error

#### Statuses
Status | Description
--- | ---
200 | OK, successful request
400 | bad request
403 | insufficient privileges to launch the application
404 | application not found
500 | error launching the application

#### Sample request

```json
{
	"app": "prime-video"
}
```

#### Sample response

Success:

```json
{
	"status": 200
}
```

Error:

```json
{
	"status": 500,
	"error": "internal application error"
}
```

### Killing the application

#### Request topic

`dab/appLifecycle/kill`

#### Request format

```typescript
interface KillApplicationRequest {
	app: string;
}
```

Parameter | Default|  Description
--- | --- | ---
app | - | application identifier

#### Response format

```typescript
interface KillApplicationResponse {
	status: number;
	error?: string;
}
```

#### Statuses

Status | Description
--- | ---
200 | OK, successful request
400 | bad request
403 | insufficient privileges to kill the application
501 | action not implemented


#### Example

```
mqtt: subscribe --topic _response/dab/appLifecycle/kill/x
mqtt: publish --topic dab/appLifecycle/kill/x
{
	"app": "media-application-1"
}

mqtt: < received _resopnse/dab/appLifecycle/kill/x
{
	"status": 200
}
```

#### Sample request

request:
```json
{
	"app": "prime-video"
}
```

### Setting the visibility

#### Request topic

`dab/appLifecycle/visibility`

#### Request format

```typescript
type Visibility = 'background' | 'foreground'

interface SetVisibilityRequest {
	app: string;
	mode: Visibility;
}
```

Parameter | Description
--- |  ---
app |  application identifier
mode | visibility mode


#### Response format

```typescript
interface SetVisibilityResponse {
	status: number,
	error?: string;
}
```

#### Statuses

Status | Description
--- | ---
200 | OK, successful request
400 | bad request
501 | action not implemented


## System

### Device Information

#### Notification Topic

`dab/device`

#### Message Format

```typescript
type NetworkConnectivityMode = 'ethernet' | 'wifi' | 'bluetooth';

interface DeviceInformation {
	make: string;
	model: string;
	serialNumber: string;
	year: number;
	chipset: string;
	firmware: string;
	/** ethernet, wifi */
	networkConnectivityMode: NetworkConnectivityMode[];
	macAddress: string;
}
```

Parameter | Description
--- | ---
make | -
model | -
serialNumber | -
year | -
chipset | -
firmware | -
features | -
networkConnectivityMode | -

### Restarting the device

(and leave DAB enabled)

#### Request topic

`dab/system/restart`

#### Request format

```typescript
interface RestartRequest {
}
```

#### Response format
```typescript
interface RestartResponse extends Response {
	status: number;
	error?: string;
}
```
#### Statuses
Status | Description
--- | ---
202 | OK, request accepted
403 | insufficient privileges to launch the application
501 | action not implemented

#### Sample response

Success:
```json
{
	"status": 200
}
```

Error:
```json
{
	"status": 403,
	"error": "insufficient privileges"
}
```

### Input

#### Key press

TODO: need to specify the list of keyCodes

##### Request topic

`dab/system/input/key-press`

##### Request format
```typescript
interface KeyPressRequest {
	keyCode: number;
	longPress?: boolean;
}
```

##### Response format
```typescript
interface KeyPressResponse {
	status: number;
	error?: string;
}
```

### Language

#### Setting the language

##### Request topic
`dab/system/language/set`

##### Request format
```typescript
interface SetLanguageRequest {
	language: string;
}
```

##### Response format

```typescript
interface SetLanguageResponse {
	status: number;
	error?: string;
}
```

##### Sample request

```json
{
	"language": "fr_FR"
}
```

#### Getting the currently set language

##### Request topic

`dab/system/language/get/`

##### Request format

```typescript
interface GetLanguageRequest {
}
```

##### Response format

```typescript
interface GetLanguageResponse {
	status: number;
	language?: string;
	error?: string;
}
```

##### Sample response

```json
{
	"status": 200,
	"language": "en_US"
}
```

#### Listing available languages

##### Request topic
`dab/system/language/list`

##### Request format

```typescript
interface GetAvailableLanguagesRequest {
}
```

##### Response format
```typescript
interface GetAvailableLanguagesResponse {
	status: number;
	languages?: string[];
	error?: string;
}
```

##### Sample response

```json
{
	"status": 200,
	"languages": [
		"en_GB",
		"en_US",
		"fr_FR"
	]	
}
```

## Telemetry

### Discovery

To discover the available metrics in the system, the client needs to subscribe to `dab/telemetry`

### Request format

```typescript
type Frequency = 1 | 5 | 30;

type Metric = "cpu" | "memory";

interface TelemetryRequest {
	frequency: Frequency;
	metrics: Metric[];
}
```

Parameter | Description
--- | ---
frequency | sampling frequency, expressed in seconds
metrics | a non-empty list of metrics to be published

### Metrics

| metric  | unit | description
|---|---|---|
| cpu | percentage | CPU usage
| memory | kilobytes | Memory usage


### Sample request

```json
{
	"metrics" : ["cpu", "memory"],
	"frequency": 5
}
```

### Telemetry delivery topics

Each application in the system will have a dedicated telemetry topic to which the metrics are published when telemetry is requested.

The format of the topic is:
`dab/telemetry/metrics/<app_id>`

where `<app_id>` is the application id as returned by the list.

#### Example

The application identified by 'media-player-1' in the system will have its metrics delivered to the topic `dab/telemetry/metrics/media-player-1`

### Message format

```typescript
interface Telemetry {
	metrics: Metric[],
	frequencies: Frequency[]
}
```

The metrics are delivered to the topic that was specified during the app launch.

```typescript
interface TelemetryMessage {
	timestamp: number;
	metric: Metric;
	value: number;
}
```

Parameter | Description
--- | ---
timestamp | Epoch time in milliseconds
metric | metric name
value | value of the metric

### Sample messages

```json
{
	"timestamp": 1568815387403,
	"metric": "cpu",
	"value": 33.5
}
```

```json
{
	"timestamp": 1568815390414,
	"metric": "memory",
	"value": 44040192
}
```

## Versioning of the protocol

A device implementing the protocol and its subsequent versions must publish a retained message to the versioning topic to make sure that the client can support the implemented version.

It is possible for a device to implement more than one version of the protocol.

### Version

A protocol version consists of the major version and the minor version delimited by a full stop character `.`
Major and minor versions are non-negative integers.

#### Examples

Valid versions:
`1.0`, `2.3`, `0.48`

Invalid versions
`1.a`,  `x.5`

### Notification topic

`dab/version`

### Notification format

```typescript
interface Version {  
    versions: string[]  
}
```

Parameter | Description
--- | ---
versions | a list of versions of the protocol implemented on the device

## Security / enabling DAB
* DAB is disabled by default
* in order to use DAB, the user needs to

