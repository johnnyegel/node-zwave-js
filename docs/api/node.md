# ZWaveNode

A Z-Wave node is a single device in a Z-Wave network. In the scope of this library, the `ZWaveNode` class provides means to control nodes and retrieve their information.

**Note:**
All methods except `interview` (which you should not use yourself) are only safe to use **after** the node has been interviewed.  
Most properties are only defined **after** the node has been interviewed. The exceptions are:

-   `id`
-   `status`
-   `interviewStage`
-   `keepAwake`

Since a node also represents the root endpoint of a device (see [`getEndpoint`](#getEndpoint) for a detailed explanation), the `ZWaveNode` class inherits from the [`Endpoint` class](api/endpoint.md). As a result, it also supports all methods and properties of that class.

## ZWaveNode methods

### `getValue`

```ts
getValue<T?>(valueId: ValueID): T | undefined
```

Retrieves a stored value from this node's value database. This method takes a single argument specifying which value to retrieve. See the [`ValueID` interface](api/valueid.md) for a detailed description of this argument's type.
If the type of the value is known in advance, you may pass an optional type argument to the method.

The method either returns the stored value if it was found, and `undefined` otherwise.

> [!NOTE]
> This does **not** communicate with the node to refresh the value.

### `getValueMetadata`

```ts
getValueMetadata(valueId: ValueID): ValueMetadata
```

Every value in Z-Wave has associated metadata that defines the range of allowed values etc. You can retrieve this metadata using `getValueMetadata`. Like `getValue` this takes a single argument of the type [`ValueID`](api/valueid.md#ValueID).

Metadata in `zwave-js` can be separated into a **static** and a **dynamic** part - whatever makes sense in the current context. Whenever metadata is exposed to applications, the dynamic part is merged on top of the static one. `getValueMetadata` is guaranteed to return at least some very basic metadata (in the form of a [`ValueMetadata`](/api/valueid.md#ValueMetadata) object), even if the value was not found.

> [!NOTE]
> Changes in metadata are communicated through the [`"metadata updated"`](#quotmetadata-updatedquot) event. Note that this **only** gets used when the dynamic part changes. This event is not emitted if a value **only** has static metadata.
>
> If applications plan to use metadata, they **must not** assume that metadata does not exist if there was no `"metadata updated"` event. Instead the `getValueMetadata` method **must** be used to retrieve the metadata initially.

### `getDefinedValueIDs`

```ts
getDefinedValueIDs(): TranslatedValueID[]
```

When building a user interface for a Z-Wave application, you might need to know all possible values in advance. This method returns an array of all ValueIDs that are available for this node.

### `setValue`

```ts
async setValue(valueId: ValueID, value: unknown, options?: SetValueAPIOptions): Promise<boolean>
```

Updates a value on the node. This method takes the following arguments:

-   `valueId: ValueID` - specifies which value to update
-   `value: unknown` - The new value to set
-   `options?: SetValueAPIOptions` - Optional options for the resulting commands

This method automatically figures out which commands to send to the node, so you don't have to use the specific commands yourself. The returned promise resolves to `true` after the value was successfully updated on the node. It resolves to `false` if any of the following conditions are met:

-   The `setValue` API is not implemented in the required Command Class
-   The required Command Class is not supported by the node/endpoint
-   The required Command Class is not implemented in this library yet
-   The API for the required Command Class is not implemented in this library yet

> [!ATTENTION] By default, the driver assumes to be talking to a single application. In this scenario a successful `setValue` call is enough for the application to know that the value was changed and update its own cache or UI. Therefore, the `"value updated"` event is not emitted after `setValue` unless the change was verified by the device.
>
> To get `"value updated"` events nonetheless, set the driver option `emitValueUpdateAfterSetValue` to `true`.

The `options` bag contains options that influence the resulting commands, for example a transition duration. Each implementation will choose the options that are relevant for it, so you can use the same options everywhere.

<!-- #import SetValueAPIOptions from "zwave-js" -->

```ts
type SetValueAPIOptions = Partial<ValueChangeOptions>;
```

### `pollValue`

```ts
async pollValue<T>(valueId: ValueID): Promise<T | undefined>
```

Polls a value from the node. The `valueId` parameter specifies which value to poll. You can define the expected return type by passing it as the type parameter `T`.

This method automatically figures out which commands to send to the node, so you don't have to use the specific commands yourself. The returned promise resolves to the current value reported by the node or `undefined` if there was no response. It will be **rejected** if any of the following conditions are met:

-   The `pollValue` API is not implemented in the required Command Class
-   The required Command Class is not implemented in this library yet
-   The API for the required Command Class is not implemented in this library yet

> [!WARNING]
> Polling can impose a heavy load on the network and should not be done too frequently.

> [!WARNING]
> Some value IDs share a command, so make sure not to blindly call this for every property. Otherwise you'll end up with multiple duplicate requests.

### `refreshValues`

```ts
refreshValues(): Promise<void>
```

Refreshes all non-static sensor and actuator values from this node. Although this method returns a `Promise`, it should generally **not** be `await`ed, since the update may take a long time.

> [!WARNING]  
> **DO NOT** use this method too frequently. Depending on the devices, this may generate a lot of traffic.

### `refreshCCValues`

```ts
refreshCCValues(cc: CommandClasses): Promise<void>
```

Refreshes all non-static values from the given CC (all endpoints). Although this method returns a `Promise`, it should generally **not** be `await`ed, since the update may take a long time.

> [!WARNING]  
> **DO NOT** use this method too frequently. Depending on the devices, this may generate a lot of traffic.

### `getEndpoint`

```ts
getEndpoint(index: 0): Endpoint;
getEndpoint(index: number): Endpoint | undefined;
```

In Z-Wave, a single node may provide different functionality under different end points, for example single sockets of a switchable plug strip. This method allows you to access a specific end point of the current node. It takes a single argument denoting the endpoint's index and returns the corresponding endpoint instance if one exists at that index. Calling `getEndpoint` with the index `0` always returns the node itself, which is the "root" endpoint of the device.

### `getEndpointCount`

```ts
getEndpointCount(): number
```

Returns the current endpoint count of this node.

### `getAllEndpoints`

```ts
getAllEndpoints(): Endpoint[]
```

This method returns an array of all endpoints on this node. At each index `i` the returned array contains the endpoint instance that would be returned by `getEndpoint(i)`.

### `isControllerNode`

```ts
isControllerNode(): boolean
```

This is a little utility function to check if this node is the controller.

### `hasSecurityClass`

```ts
hasSecurityClass(securityClass: SecurityClass): Maybe<boolean>
```

Returns whether the node was assigned the given security class - or `"unknown"` if that information isn't known yet.

### `getHighestSecurityClass`

```ts
getHighestSecurityClass(): SecurityClass | undefined
```

Returns the highest security class this node was granted or `undefined` if that information isn't known yet. This can be used to distinguish whether a node is communicating with S2, S0 or insecure.

### `refreshInfo`

```ts
refreshInfo(options?: RefreshInfoOptions): Promise<void>
```

Resets (almost) all information about this node and forces a fresh interview. The information about granted security classes is not reset by default, but can be reset by setting the `resetSecurityClasses` option to `true`.

<!-- #import RefreshInfoOptions from "zwave-js" -->

```ts
interface RefreshInfoOptions {
	/**
	 * Whether a re-interview should also reset the known security classes.
	 * Default: false
	 */
	resetSecurityClasses?: boolean;
}
```

> [!WARNING] After calling this method, the node will no longer be `ready`. Keep this in mind if you rely on the `ready` state in your application.

### `interviewCC`

```ts
interviewCC(cc: CommandClasses): Promise<void>
```

Rediscovers all capabilities of a single CC on this node and all endpoints. Although this method returns a `Promise`, it should generally **not** be `await`ed, since the update may take a long time.

> [!NOTE] This method should only be used when necessary, for example when CC capabilities were not discovered correctly. It can be considered to be a more targeted version of `refreshInfo`.

### `getFirmwareUpdateCapabilities`

```ts
getFirmwareUpdateCapabilities(): Promise<FirmwareUpdateCapabilities>
```

Retrieves the firmware update capabilities of a node to decide which options (e.g. firmware targets) to offer a user prior to the update.

<!-- #import FirmwareUpdateCapabilities from "zwave-js" -->

```ts
type FirmwareUpdateCapabilities =
	| {
			/** Indicates whether the node's firmware can be upgraded */
			readonly firmwareUpgradable: false;
	  }
	| {
			/** Indicates whether the node's firmware can be upgraded */
			readonly firmwareUpgradable: true;
			/** An array of firmware targets that can be upgraded */
			readonly firmwareTargets: readonly number[];
			/** Indicates whether the node continues to function normally during an upgrade */
			readonly continuesToFunction: Maybe<boolean>;
			/** Indicates whether the node supports delayed activation of the new firmware */
			readonly supportsActivation: Maybe<boolean>;
	  };
```

### `beginFirmwareUpdate`

```ts
beginFirmwareUpdate(data: Buffer, target?: number): Promise<void>
```

> [!WARNING] Use at your own risk! We don't take any responsibility if your devices don't work after an update.

Starts an OTA firmware update process for this node. This method takes two arguments:

-   `data` - A buffer containing the firmware image in a format supported by the device
-   `target` - _(optional)_ The firmware target (i.e. chip) to upgrade. `0` updates the Z-Wave chip, `>=1` updates others if they exist

The library includes helper methods (exported from `zwave-js/Utils`) to prepare the firmware update.

```ts
extractFirmware(rawData: Buffer, format: FirmwareFileFormat): Firmware
```

`rawData` is a buffer containing the original firmware update file, `format` describes which kind of file that is. The following formats are available:

-   `"aeotec"` - A Windows executable (`.exe` or `.ex_`) that contains Aeotec's upload tool
-   `"otz"` - A compressed firmware file in Intel HEX format
-   `"ota"` or `"hex"` - An uncompressed firmware file in Intel HEX format
-   `"hec"` - An encrypted Intel HEX firmware file
-   `"gecko"` - A binary gecko bootloader firmware file with `.gbl` extension

You can use the helper method `guessFirmwareFileFormat` to guess which firmware format a file has based on the file extension and contents.

```ts
guessFirmwareFileFormat(filename: string, rawData: Buffer): FirmwareFileFormat
```

-   `filename`: The name of the firmware file (including the extension)
-   `rawData`: A buffer containing the original firmware update file

If successful, `extractFirmware` returns an object of the following form, whose properties can be passed to `beginFirmwareUpdate`:

<!-- #import Firmware from "zwave-js" -->

```ts
interface Firmware {
	data: Buffer;
	firmwareTarget?: number;
}
```

If no firmware data can be extracted, the method will throw.

Example usage:

```ts
// Extract the firmware from a given firmware file
let actualFirmware: Firmware;
try {
	const format = guessFirmwareFileFormat(filename, rawData);
	actualFirmware = extractFirmware(rawData, format);
} catch (e) {
	// handle the error, then abort the update
}

// try the update
try {
	await this.driver.controller.nodes
		.get(nodeId)!
		.beginFirmwareUpdate(
			actualFirmware.data,
			actualFirmware.firmwareTarget,
		);
	console.log(`Node ${nodeId}: Firmware update started`);
} catch (e) {
	// handle error
}
```

### `abortFirmwareUpdate`

```ts
abortFirmwareUpdate(): Promise<void>
```

Aborts an active firmware update process.

### `ping`

```ts
ping(): Promise<boolean>
```

Pings the node and returns whether it responded or not.

### `testPowerlevel`

```ts
testPowerlevel(
	testNodeId: number,
	powerlevel: Powerlevel,
	testFrameCount: number,
	onProgress?: (acknowledged: number, total: number) => void,
): Promise<number>;
```

Instructs the node to send powerlevel test frames to the other node with ID `testNodeId` using the given powerlevel. Returns how many frames were acknowledged during the test.

Depending on the number of test frames and involved hops, this may take a while. You can use the optional `onProgress` callback to get regular updates on the test progress.

> [!ATTENTION] This will throw when the target node is a FLiRS node or a sleeping node that is not awake.

### `checkLifelineHealth`

```ts
checkLifelineHealth(
	rounds?: number,
	onProgress?: (round: number, totalRounds: number, lastRating: number) => void,
): Promise<HealthCheckSummary>
```

Checks the health of the connection between the controller and this node and returns the results. The test is done in multiple rounds (1...10, default: 5), which can be configured using the first parameter. To monitor the progress, the optional `onProgress` callback can be used.

> [!WARNING] This should **NOT** be done while there is a lot of traffic on the network because it will negatively impact the test results.

The returned object contains the measurements of each round as well as a final rating (which is the worst of all round ratings):

```ts
interface LifelineHealthCheckSummary {
	/** The check results of each round */
	results: LifelineHealthCheckResult[];
	/** The health rating expressed as a number from 0 (not working at all) to 10 (perfect connectivity). */
	rating: number;
}
```

where each result looks as follows:

<!-- #import LifelineHealthCheckResult from "zwave-js" -->

```ts
interface LifelineHealthCheckResult {
	/**
	 * How many times at least one new route was needed. Lower = better, ideally 0.
	 *
	 * Only available if the controller supports TX reports.
	 */
	routeChanges?: number;
	/**
	 * The maximum time it took to send a ping to the node. Lower = better, ideally 10 ms.
	 *
	 * Will use the time in TX reports if available, otherwise fall back to measuring the round trip time.
	 */
	latency: number;
	/** How many routing neighbors this node has. Higher = better, ideally > 2. */
	numNeighbors: number;
	/** How many pings were not ACKed by the node. Lower = better, ideally 0. */
	failedPingsNode: number;
	/**
	 * The minimum powerlevel where all pings from the node were ACKed by the controller. Higher = better, ideally 6dBm or more.
	 *
	 * Only available if the node supports Powerlevel CC
	 */
	minPowerlevel?: Powerlevel;
	/**
	 * If no powerlevel was found where the controller ACKed all pings from the node, this contains the number of pings that weren't ACKed. Lower = better, ideally 0.
	 *
	 * Only available if the node supports Powerlevel CC
	 */
	failedPingsController?: number;
	/**
	 * An estimation of the Signal-to-Noise Ratio Margin in dBm.
	 *
	 * Only available if the controller supports TX reports.
	 */
	snrMargin?: number;

	/** See {@link LifelineHealthCheckSummary.rating} */
	rating: number;
}
```

The health rating is computed similar to Silabs' PC Controller IMA tool where 10 means _perfect_ and 0 means _not working at all_. The following table shows which requirements must be fulfilled to achieve a given rating. If the min. powerlevel or SNR margin can not be measured, the condition is assumed to be fulfilled.

| Rating | Failed pings |   Latency | No. of neighbors | min. powerlevel | SNR margin |
| -----: | -----------: | --------: | ---------------: | --------------: | ---------: |
|  ✅ 10 |            0 |   ≤ 50 ms |              > 2 |        ≤ −6 dBm |   ≥ 17 dBm |
|   🟢 9 |            0 |  ≤ 100 ms |              > 2 |        ≤ −6 dBm |   ≥ 17 dBm |
|   🟢 8 |            0 |  ≤ 100 ms |              ≤ 2 |        ≤ −6 dBm |   ≥ 17 dBm |
|   🟢 7 |            0 |  ≤ 100 ms |              > 2 |               - |          - |
|   🟢 6 |            0 |  ≤ 100 ms |              ≤ 2 |               - |          - |
|        |              |           |                  |                 |            |
|   🟡 5 |            0 |  ≤ 250 ms |                - |               - |          - |
|   🟡 4 |            0 |  ≤ 500 ms |                - |               - |          - |
|        |              |           |                  |                 |            |
|   🔴 3 |            1 | ≤ 1000 ms |                - |               - |          - |
|   🔴 2 |          ≤ 2 | > 1000 ms |                - |               - |          - |
|   🔴 1 |          ≤ 9 |         - |                - |               - |          - |
|        |              |           |                  |                 |            |
|   ❌ 0 |           10 |         - |                - |               - |          - |

> [!NOTE] This test builds on some functionality that is not available for all controller or nodes. If the node does not support `Powerlevel CC` or the controller does not support TX reports, this check will be less reliable.

> [!NOTE] The test results are also printed to the driver logs. If you want to format the results in the same way in your application, you can use the `formatLifelineHealthCheckSummary` and/or `formatLifelineHealthCheckRound` methods which are exposed from `zwave-js/Utils`.

### `checkRouteHealth`

```ts
checkRouteHealth(
	targetNodeId: number,
	rounds?: number,
	onProgress?: (round: number, totalRounds: number, lastRating: number) => void,
): Promise<HealthCheckSummary>
```

Checks the health of connection between this node and the target node and returns the results. At least one of the nodes **must** support `Powerlevel CC`.

The test is done in multiple rounds (1...10, default: 5), which can be configured using the first parameter. To monitor the progress, the optional `onProgress` callback can be used.

> [!WARNING] This should **NOT** be done while there is a lot of traffic on the network because it will negatively impact the test results.

The returned object contains the measurements of each round as well as a final rating (which is the worst of all round ratings):

```ts
interface RouteHealthCheckSummary {
	/** The check results of each round */
	results: RouteHealthCheckResult[];
	/** The health rating expressed as a number from 0 (not working at all) to 10 (perfect connectivity). */
	rating: number;
}
```

where each result looks as follows:

<!-- #import RouteHealthCheckResult from "zwave-js" -->

```ts
interface RouteHealthCheckResult {
	/** How many routing neighbors this node has. Higher = better, ideally > 2. */
	numNeighbors: number;
	/**
	 * How many pings were not ACKed by the target node. Lower = better, ideally 0.
	 *
	 * Only available if the source node supports Powerlevel CC
	 */
	failedPingsToTarget?: number;
	/**
	 * How many pings were not ACKed by the source node. Lower = better, ideally 0.
	 *
	 * Only available if the target node supports Powerlevel CC
	 */
	failedPingsToSource?: number;
	/**
	 * The minimum powerlevel where all pings from the source node were ACKed by the target node. Higher = better, ideally 6dBm or more.
	 *
	 * Only available if the source node supports Powerlevel CC
	 */
	minPowerlevelSource?: Powerlevel;
	/**
	 * The minimum powerlevel where all pings from the target node were ACKed by the source node. Higher = better, ideally 6dBm or more.
	 *
	 * Only available if the source node supports Powerlevel CC
	 */
	minPowerlevelTarget?: Powerlevel;

	/** See {@link RouteHealthCheckSummary.rating} */
	rating: number;
}
```

The health rating expressed as a number from 0 (not working at all) to 10 (perfect connectivity). The following table shows which requirements must be fulfilled to achieve a given rating. If the min. powerlevel of one node can not be measured, the condition is assumed to be fulfilled.

| Rating | Failed pings | No. of neighbors | min. powerlevel |
| -----: | -----------: | ---------------: | --------------: |
|  ✅ 10 |            0 |              > 2 |        ≤ −6 dBm |
|   🟢 8 |            0 |              ≤ 2 |        ≤ −6 dBm |
|   🟢 7 |            0 |              > 2 |               - |
|   🟢 6 |            0 |              ≤ 2 |               - |
|        |              |                  |                 |
|   🔴 3 |            1 |                - |               - |
|   🔴 2 |            2 |                - |               - |
|   🔴 1 |          ≤ 9 |                - |               - |
|        |              |                  |                 |
|   ❌ 0 |           10 |                - |               - |

> [!NOTE] Due to missing insight into re-routing attempts between two nodes, some of the values for the lifeline health rating don't exist here. Furthermore, it is not guaranteed that a route between two nodes and lifeline with the same health rating have the same quality.

> [!NOTE] The test results are also printed to the driver logs. If you want to format the results in the same way in your application, you can use the `formatRouteHealthCheckSummary` and/or `formatRouteHealthCheckRound` methods which are exposed from `zwave-js/Utils`.

## ZWaveNode properties

### `id`

```ts
readonly id: number
```

Returns the ID this node has been assigned by the controller. This is a number between 1 and 232.

### `name`

```ts
name: string | undefined;
```

The user-defined name of this node. Uses the value reported by `Node Naming and Location CC` if it exists.

> [!NOTE]
> Setting this value only updates the name locally. To permanently change the name of the node, use the [`commandClasses` API](api/endpoint.md#commandClasses).

### `location`

```ts
location: string | undefined;
```

The user-defined location of this node. Uses the value reported by `Node Naming and Location CC` if it exists.

> [!NOTE]
> Setting this value only updates the location locally. To permanently change the location of the node, use the [`commandClasses` API](api/endpoint.md#commandClasses).

### `status`

```ts
readonly status: NodeStatus;
```

This property tracks the status a node in the network currently has (or is believed to have). Consumers of this library should treat the status as readonly. Valid values are defined in the `NodeStatus` enumeration:

-   `NodeStatus.Unknown (0)` - this is the default status of a node. A node is assigned this status before it is being interviewed (including manual re-interviews when calling `refreshInfo`).
-   `NodeStatus.Asleep (1)` - Nodes that support the `WakeUp` CC and failed to respond to a message are assumed asleep.
-   `NodeStatus.Awake (2)` - Sleeping nodes that recently sent a wake up notification are marked awake until they are sent back to sleep or fail to respond to a message.
-   `NodeStatus.Dead (3)` - Nodes that **don't** support the `WakeUp` CC are marked dead when they fail to respond. Examples are plugs that have been pulled out of their socket. Whenever a message is received from a presumably dead node, they are marked as unknown.
-   `NodeStatus.Alive (4)` - Nodes that **don't** support the `WakeUp` CC are considered alive while they do respond. Note that the status may switch from alive to awake/asleep once it is discovered that a node does support the `WakeUp` CC.

Changes of a node's status are broadcasted using the corresponding events - see below.

### `interviewStage`

```ts
readonly interviewStage: InterviewStage
```

This property tracks the current status of the node interview. It contains a value representing the **last completed step** of the interview. You shouldn't need to use this in your application. If you do, here are the possible values.

<!-- #import InterviewStage from "zwave-js" -->

```ts
enum InterviewStage {
	/** The interview process hasn't started for this node */
	None,
	/** The node's protocol information has been queried from the controller */
	ProtocolInfo,
	/** The node has been queried for supported and controlled command classes */
	NodeInfo,

	/**
	 * Information for all command classes has been queried.
	 * This includes static information that is requested once as well as dynamic
	 * information that is requested on every restart.
	 */
	CommandClasses,

	/**
	 * Device information for the node has been loaded from a config file.
	 * If defined, some of the reported information will be overwritten based on the
	 * config file contents.
	 */
	OverwriteConfig,

	/** The interview process has finished */
	Complete,
}
```

> [!WARNING]
> DO NOT rely on the numeric values of the enum if you're using it in your application.
> The ordinal values are likely to change in future updates. Instead, refer to the enum properties directly.

### `deviceClass`

```ts
readonly deviceClass: DeviceClass | undefined;
```

This property returns the node's **DeviceClass**, which provides further information about the kind of device this node is.

<!-- #import DeviceClass from "zwave-js" -->

```ts
interface DeviceClass {
	readonly basic: BasicDeviceClass;
	readonly generic: GenericDeviceClass;
	readonly specific: SpecificDeviceClass;
	readonly mandatorySupportedCCs: readonly CommandClasses[];
	readonly mandatoryControlledCCs: readonly CommandClasses[];
}
```

<!-- #import BasicDeviceClass from "@zwave-js/config" -->

```ts
interface BasicDeviceClass {
	key: number;
	label: string;
}
```

<!-- #import GenericDeviceClass from "@zwave-js/config" -->

```ts
interface GenericDeviceClass {
	readonly key: number;
	readonly label: string;
	readonly requiresSecurity?: boolean | undefined;
	readonly supportedCCs: readonly CommandClasses[];
	readonly controlledCCs: readonly CommandClasses[];
	readonly specific: ReadonlyMap<number, SpecificDeviceClass>;
}
```

<!-- #import SpecificDeviceClass from "@zwave-js/config" -->

```ts
interface SpecificDeviceClass {
	readonly key: number;
	readonly label: string;
	readonly zwavePlusDeviceType?: string | undefined;
	readonly requiresSecurity?: boolean | undefined;
	readonly supportedCCs: readonly CommandClasses[];
	readonly controlledCCs: readonly CommandClasses[];
}
```

### `zwavePlusVersion`

```ts
readonly zwavePlusVersion: number | undefined
```

Returns the version of the `Z-Wave+` Command Class this node supports. If the CC is supported, the value only exists **after** the node is ready.

### `zwavePlusNodeType`

```ts
readonly zwavePlusNodeType: ZWavePlusNodeType | undefined
```

If the `Z-Wave+` Command Class is supported, this returns the `Z-Wave+` node type this device has, which is one of the following values:

<!-- #import ZWavePlusNodeType from "zwave-js" -->

```ts
enum ZWavePlusNodeType {
	Node = 0x00, // ZWave+ Node
	IPGateway = 0x02, // ZWave+ for IP Gateway
}
```

### `ready`

```ts
readonly ready: boolean
```

Whether the node is ready to be used.

### `zwavePlusRoleType`

```ts
readonly zwavePlusRoleType: ZWavePlusRoleType | undefined
```

If the `Z-Wave+` Command Class is supported, this returns the `Z-Wave+` role type this device has, which is one of the following values:

<!-- #import ZWavePlusRoleType from "zwave-js" -->

```ts
enum ZWavePlusRoleType {
	CentralStaticController = 0x00,
	SubStaticController = 0x01,
	PortableController = 0x02,
	PortableReportingController = 0x03,
	PortableSlave = 0x04,
	AlwaysOnSlave = 0x05,
	SleepingReportingSlave = 0x06,
	SleepingListeningSlave = 0x07,
}
```

### `isListening`

```ts
readonly isListening: boolean | undefined;
```

Whether this node is always listening.

### `isFrequentListening`

```ts
readonly isFrequentListening: false | "250ms" | "1000ms" | undefined;
```

Indicates the wakeup interval if this node is a FLiRS node. `false` if it isn't.

### `canSleep`

```ts
readonly canSleep: boolean | undefined;
```

Whether this node can sleep. If this is the case it must be expected to be asleep most of the time.

### `isRouting`

```ts
readonly isRouting: boolean | undefined;
```

Whether the node supports routing/forwarding messages. If both this property and `isListening` are true, it can be assumed that the node **will** route messages.

### `supportedDataRates`

```ts
readonly supportedDataRates: readonly number[] | undefined;
```

A list of data rates (in bits per second) this node supports. Possible values are `9600`, `40000` and `100000`.

### `maxDataRate`

```ts
readonly maxDataRate: number
```

The maximum data rate this nodes supports.

### `supportsSecurity`

```ts
readonly supportsSecurity: boolean | undefined;
```

Whether this node supports secure communication (S0 or S2).

> [!WARNING] Nodes often report this incorrectly - do not blindly trust this value.

### `isSecure`

```ts
readonly isSecure: boolean | "unknown";
```

Whether this node is communicating securely with the controller, meaning that it was granted at least one security class. More specific information can be retrieved with the [`hasSecurityClass`](#hasSecurityClass) or the [`getHighestSecurityClass`](#getHighestSecurityClass) methods.

### `supportsBeaming`

```ts
readonly supportsBeaming: boolean | undefined;
```

Whether this node can issue wakeup beams to FLiRS nodes

### `protocolVersion`

```ts
readonly protocolVersion: ProtocolVersion | undefined
```

The Z-Wave protocol version this node implements.

<!-- #import ProtocolVersion from "zwave-js" -->

```ts
enum ProtocolVersion {
	"unknown" = 0,
	"2.0" = 1,
	"4.2x / 5.0x" = 2,
	"4.5x / 6.0x" = 3,
}
```

### `firmwareVersion`

```ts
readonly firmwareVersion: string
```

The version of this node's firmware.

### `manufacturerId`, `productId` and `productType`

```ts
readonly manufacturerId: number
readonly productId: number
readonly productType: number
```

These three properties together identify the actual device this node is.

### `deviceConfig`

```ts
readonly deviceConfig: DeviceConfig | undefined
```

Contains additional information about this node, loaded from a [config file](/development/config-files.md#device-configuration-files).

### `deviceDatabaseUrl`

```ts
readonly deviceDatabaseUrl: string | undefined
```

The URL to the device in the device database.

### `keepAwake`

```ts
keepAwake: boolean;
```

In order to save energy, battery powered devices should go back to sleep after they no longer need to communicate with the controller. This library honors this requirement by sending nodes back to sleep as soon as there are no more pending messages.
When configuring devices or during longer message exchanges, this behavior may be annoying. You can set the `keepAwake` property of a node to `true` to avoid sending the node back to sleep immediately.

## ZWaveNode events

The `Node` class inherits from the Node.js [EventEmitter](https://nodejs.org/api/events.html#events_class_eventemitter) and thus also supports its methods like `on`, `removeListener`, etc. The available events are avaiable:

### `"wake up"` / `"sleep"`

A sleeping node has woken up or gone back to sleep. The node is passed as the single argument to the callback:

```ts
(node: ZWaveNode) => void
```

### `"dead"` / `"alive"`

A non-sleeping node has stopped responding or just started responding again. The node is passed as the single argument to the callback:

```ts
(node: ZWaveNode) => void
```

### `"interview started"`

The initial interview or reinterview process for this node has started. The node is passed as the single argument to the callback:

```ts
(node: ZWaveNode) => void
```

### `"interview stage completed"`

A state of the interview process for this node was completed. Only the name of the stage is provided and should not be relied on as stage names are subject to change:

```ts
(node: ZWaveNode, stageName: string) => void
```

### `"interview completed"`

The initial interview or reinterview process for this node was completed. The node is passed as the single argument to the callback:

```ts
(node: ZWaveNode) => void
```

> [!TIP]
> Because sleeping nodes may have wake up times of minutes up to several hours, it may take a very long time until this event is emitted. It might be desirable to wake up nodes manually to speed up the initial interview process.

### `"interview failed"`

The interview process for this node or one of the interview attempts has failed. The second argument includes more detailed information, including a formatted error message that explains why and a flag to indicate whether this was the final attempt.

```ts
(node: ZWaveNode, args: NodeInterviewFailedEventArgs) => void
```

where the third argument looks like this:

```ts
interface NodeInterviewFailedEventArgs {
	/* The same error message as the second argument */
	errorMessage: string;
	/* If true, there will be no further interview attempt*/
	isFinal: boolean;
	/*
	 * If applicable, these properties tell you how many interview attempts were made so far and how many are the maximum.
	 * Either both properties are present or both are absent.
	 */
	attempt?: number;
	maxAttempts?: number;
}
```

### `"ready"`

This is emitted when enough information about the node is known that it can safely be used. This includes protocol information and supported/controlled CCs.

The driver will also try to identify the node, its CC versions, endpoints, capabilities for each CC, etc. However, this **does not** mean that **all** this information is known to the driver due to potential timeouts, lost messages and/or unresponsive nodes during the interview. Therefore, the driver will do its best to work with what it has.

The node is passed as the single argument to the callback:

```ts
(node: ZWaveNode) => void
```

There are two situations when this event is emitted:

1. The interview of a node is completed for the first time ever.
2. The node that has previously been interviewed completely and it either responds or is asleep.

> [!NOTE]
> This event does not imply that the node is currently awake or will respond to requests.

### `"firmware update progress"`

```ts
(node: ZWaveNode, sentFragments: number, totalFragments: number) => void;
```

Firmware update progress has been made. The callback takes the node itself, the already sent fragments, and the total fragments to be sent:

### `"firmware update finished"`

```ts
(node: ZWaveNode, status: FirmwareUpdateStatus, waitTime?: number) => void;
```

The firmware update process is finished. The returned status indicates whether the update was successful and if it was, a wait time may be needed before the device is functional again.

### `"value added"` / `"value updated"` / `"value removed"`

A value belonging to this node was added, updated or removed. The callback takes the node itself and an argument detailing the change:

```ts
// value added:
(node: ZWaveNode, args: ZWaveNodeValueAddedArgs) => void;
// value updated:
(node: ZWaveNode, args: ZWaveNodeValueUpdatedArgs) => void;
// value removed:
(node: ZWaveNode, args: ZWaveNodeValueRemovedArgs) => void;
```

The event arguments have the shape of [`TranslatedValueID`](api/valueid.md) with two additional properties:

-   `prevValue` - The previous value (before the change). Only present in the `"updated"` and `"removed"` events.
-   `newValue` - The new value (after the change). Only present in the `"added"` and `"updated"` events.

### `"value notification"`

Some values (like `Central Scene` notifications) are stateless, meaning their value only has significance **the moment** it is received. These stateless values are not persisted in the value DB in order to avoid triggering automations when restoring the network from the cache.

To distinguish them from the statful values, the `"value notification"` event is used. The callback takes the node itself and an argument detailing the change:

```ts
(node: ZWaveNode, args: ZWaveNodeValueNotificationArgs) => void;
```

The event argument has the shape of [`TranslatedValueID`](api/valueid.md) with an additional `value` property containing the current value at the time of the event.

> [!NOTE]
> If these values are displayed in a UI somehow, it is advised to perform some kind of auto-invalidation, e.g. after a fixed short time interval.

### `"metadata updated"`

The dynamic metadata for one of this node's values was added or updated.
The callback takes the node itself and an argument detailing the change:

```ts
(node: ZWaveNode, args: ZWaveNodeMetadataUpdatedArgs) => void;
```

The event argument has the shape of [`TranslatedValueID`](api/valueid.md) with one additional property:

-   `metadata` - The new metadata or undefined (in case it was removed). See ValueMetadata for a detailed description of the argument.

### `"notification"`

This event serves a similar purpose as the `"value notification"`, but is used for more complex CC-specific notifications.
The base callback signature has the following shape:

```ts
type ZWaveNotificationCallback = (
	node: ZWaveNode,
	ccId: CommandClasses,
	args: Record<string, unknown>,
): void;
```

where

-   `node` is the current node instance
-   `ccId` is the identifier for the CC which raised this event
-   `args` is a CC-specific argument object

The CCs that use this event bring specialized versions of the callback and arguments.

#### `Entry Control CC`

uses the following signature

<!-- #import ZWaveNotificationCallbackParams_EntryControlCC from "zwave-js" -->

```ts
type ZWaveNotificationCallbackParams_EntryControlCC = [
	node: ZWaveNode,
	ccId: typeof CommandClasses["Entry Control"],
	args: ZWaveNotificationCallbackArgs_EntryControlCC,
];
```

where the argument object has the type

<!-- #import ZWaveNotificationCallbackArgs_EntryControlCC from "zwave-js" -->

```ts
interface ZWaveNotificationCallbackArgs_EntryControlCC {
	eventType: EntryControlEventTypes;
	dataType: EntryControlDataTypes;
	eventData?: Buffer | string;
}
```

#### `Notification CC`

uses the following signature

<!-- #import ZWaveNotificationCallbackParams_NotificationCC from "zwave-js" -->

```ts
type ZWaveNotificationCallbackParams_NotificationCC = [
	node: ZWaveNode,
	ccId: CommandClasses.Notification,
	args: ZWaveNotificationCallbackArgs_NotificationCC,
];
```

where the argument object has the type

<!-- #import ZWaveNotificationCallbackArgs_NotificationCC from "zwave-js" -->

```ts
interface ZWaveNotificationCallbackArgs_NotificationCC {
	/** The numeric identifier for the notification type */
	type: number;
	/** The human-readable label for the notification type */
	label: string;
	/** The numeric identifier for the notification event */
	event: number;
	/** The human-readable label for the notification event */
	eventLabel: string;
	/** Additional information related to the event */
	parameters?: NotificationCCReport["eventParameters"];
}
```

#### `Powerlevel CC`

The event is emitted when a node finishes its powerlevel test of another node and sends the test result.
It uses the following signature

<!-- #import ZWaveNotificationCallbackParams_PowerlevelCC from "zwave-js" -->

```ts
type ZWaveNotificationCallbackParams_PowerlevelCC = [
	node: ZWaveNode,
	ccId: CommandClasses.Powerlevel,
	args: ZWaveNotificationCallbackArgs_PowerlevelCC,
];
```

where the argument object has the type

<!-- #import ZWaveNotificationCallbackArgs_PowerlevelCC from "zwave-js" -->

```ts
interface ZWaveNotificationCallbackArgs_PowerlevelCC {
	testNodeId: number;
	status: PowerlevelTestStatus;
	acknowledgedFrames: number;
}
```

with

<!-- #import PowerlevelTestStatus from "zwave-js" -->

```ts
enum PowerlevelTestStatus {
	Failed = 0x00,
	Success = 0x01,
	"In Progress" = 0x02,
}
```

### `"statistics updated"`

This event is emitted regularly during and after communication with the node and gives some insight that would otherwise only be visible by looking at logs. The callback has the signature

```ts
(node: ZWaveNode, statistics: NodeStatistics) => void
```

where the statistics have the following shape:

<!-- #import NodeStatistics from "zwave-js" -->

```ts
interface NodeStatistics {
	/** No. of commands successfully sent to the node */
	commandsTX: number;
	/** No. of commands received from the node, including responses to the sent commands */
	commandsRX: number;
	/** No. of commands from the node that were dropped by the host */
	commandsDroppedRX: number;
	/** No. of outgoing commands that were dropped because they could not be sent */
	commandsDroppedTX: number;
	/** No. of Get-type commands where the node's response did not come in time */
	timeoutResponse: number;
}
```
