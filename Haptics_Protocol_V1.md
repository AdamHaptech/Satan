# Haptics Protocol V1

```toc
max_depth: 4
title: Haptics Protocol - Table of Contents
style: number
```

## Revision History

| Version      | Author                   | Notes         |
| ------------ | ------------------------ | ------------- |
| 2022-07-17-1 | Adam Jackson, Striker VR | Initial draft | 

## TODO
- For each Request specify a timeout period such that ITPP can conclude that the message was not received
- Enumerate operating modes
	- Functional test (for testing prior to leaving manufacturing...test full blaster operation maybe with human in the loop)
	- Built in test (internal diagnostics)
	- Game mode
	- Charging mode (maybe the same as game mode?)
	- Calibration? (is there any calibration needed?)
- Enumerate profiles
	- Either enumerate static list of possible profiles 
	- -OR-
	- Add messages for programming custom profiles and messages for dynamically enumerating installed profiles
	- -OR-
	- Have a static list of installed profiles but allow ITPP to query them dynamically (almost the same as previous option but doesn't require programming new profiles dynamically)

## Packet
All communications with the haptics hardware are encapsulated in the following packet format.

### Packet Format
All fields are in *little endian* byte ordering. Use of network byte order, which is big endian, is not used because the processors involved in this system are little endian. Shuffling bytes around to honor network byte order would be a useless academic exercise.

| Field name | Length (bytes) | Description                                                                                                                    |
| ---------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Padding    | 3              | Padding to allow for easy alignment on 32 bit systems. These bytes should be zero for protocol version 1.                      |
| Version    | 1              | Protocol version. At this time 1 is the only valid value.                                                                      |
| Length     | 4              | An unsigned integer the represents the *remaining* length of the packet. This length does not include the 1 byte version field | 
| Tag        | 4              | A value that identifies a packet used for matching responses to requests.                                                      |
| Type       | 4              | Packet type: an enumeration of event (0), request (1), response (2)                                                            |
| Opcode     | 4              | Identifies payload format                                                                                                      |
| Payload    | variable       | Format is dependent upon Opcode and Type field, length varies                                                                  |
| CRC        | 4              | 32 bit CRC check (@todo specify algorithm or possibly remove from protocol)                                                    |


#### Padding Field
This is unused space for protocol version 1. These 3 unused bytes should be zero. They are present in the protocol to simplify alignment on 32 bit systems.

#### Version Field
At this time 1 is the only valid protocol version. Therefore, this byte must contain the value 1 to be considered a valid packet.

#### Length Field
The length specifies the length of the remainder of the packet in bytes. This includes the length of the Type field, the Payload, and the CRC. It excludes the length of the padding, Version, and Length fields. This allows the receiver to more easily stream the reception of packets by first reading 8 bytes and then reading the remaining bytes as specified by this field. Proper care to handle edge cases like timeouts and corrupt or invalid packets must be accounted for as well.

#### Tag Field
The Tag Field is used to match `response` packets to `request` packets and to avoid confusion or corruption when `event` packets occur during other operations. For Request packets this field is somewhat arbitrary. The sender can choose any value for this field and the responder must include the same value in the Response packet. It is suggested to simply start with 1 and increment for every request that the sender issues. The Tag field is entirely arbitrary for an event packet. It is suggested that 0xffff be reserved as a Tag value for all events and that senders never use 0xffff as the tag value for Request packets.

#### Type Field
Packets are one of three types: event, request, or response. 

| Packet Type | Description                                    |
| ----------- | ---------------------------------------------- |
| `event`     | A message that has no response        | 
| `request`   | Sender is expecting a `response` to the packet |
| `response`  | Reply to a previously send `request` packet    |

##### Event Type
The event type is intended for messages that are informational in nature. The sender doesn't expect a response. An example of an Event Type would be a trigger press event.

##### Request Type
The request type is intended for messages that require some form of response. Specific requests are outlined later in this document. An example request is a low power request. A sender could request that the receiver go into a low power state. The receiver must acknowledge the request, and may deny it.

##### Response Type
The response type is intended for responses to request type packets. The response is matched to the request by way of the Tag field. An example would be the sender requesting the receiver go into low power mode. The receiver must reply within a certain time period with a valid response and the Tag Field must match the Tag Field from the request packet.

#### Opcode Field
The Opcode Field is an enumeration, defined later in this document, that can be used to determine the contents of the Payload Field. 

#### Payload Field
The tuple of the protocol version, Type Field, and Opcode field determine the expected layout of the Payload field. Each payload is Opcode specific and is outlined later in this document.


## Events
### Heartbeat
The Heatbeat event message is sent every XX. The payload is a JSON string containing the following information

#### Payload
```json
{
	"type": "heartbeat",
	"battery_level": 11,
	"motor_temp_c": 42,
	...
}
```


### Profile State
A Profile State event message is sent any time the Haptics profile changes. The payload is a JSON string.

#### Payload
```json
{
	"type": "profile_state",
	"profile_previous": "jelly",
	"profile_current": "fart"
}
```


### Button State
A Button change event message is sent from the Haptics system to the ITPP any time the button state changes. The payload is a JSON string.

#### Payload
```json
{
	"type": "button_state",
	// maybe the positions ought to be an array?
	"main_position": 1,
	"secondary_position": 6,
	...
}
```


### NFC
An NFC event message is sent from the Haptics system to the ITPP when an NFC tag is written. The payload is a JSON string.

#### Payload
```json
{
	"type": "nfc",
	"nfc_id": "not sure what the data is",
	...
}
```



## Requests

### ITPP to Haptics

#### Get Operating Mode
This message is used to query the current operating mode of the Haptics module. See section XXX for a listing of possible operating modes. @todo determine if operating modes should have a numeric ID or just use strings.

##### Payload
```json
{
	"type": "get_operating_mode"
}
```

##### Haptics Response
The Haptics module will respond within 100 ms.

```json
{
	"type": "get_operating_mode_response"
	"operating_mode": "functional test"
}
```


#### Set Operating Mode
This message is used to request that the Haptics module enters a different operating mode. @todo determine if operating modes should have a numeric ID or just use strings. 

##### Payload
```json
{
	"type": "set_operating_mode"
	"new_mode": "functional_test"
}
```

##### Haptics Response

The Haptics module will respond within 100 ms. The initial response is an indication that the request has been accepted. A second message, an Event message, will be send when the new mode takes effect. 

```json
{
	"type": "set_operating_mode_response"
	"current_mode": "functional test"
	"next_mode": "game"
}
```


#### Get Info
This allows the ITPP to query a plethora of data from the Haptics module. 

##### Payload
```json
{
	"type": "get_info_request"
}
```

##### Haptics Response
The Haptics module will respond within 100ms. @todo determine the listing of info data and document what each field means.

```json
{
	"type": "get_info_response",
	"uptime": 42,
	"motor_temp_c": 20,
	"odometer": 8675309,
}
```


#### Get Diagnostics
The intent of this message is for debugging purposes. Developers can use this message to get debugging information (e.g. logs, core dumps, etc) from the Haptics module. @todo determine the payload of this message. This could potentially be a binary payload as that may make more sense than JSON. There are options for binary encoding of JSON too.

##### Payload
```json
{
	"type": "get_diagnostics"
}
```

##### Haptics Response
@todo This could be a JSON response but this may be better as a binary response. Maybe send a file or files?


#### Firmware Update Start
The ITPP sends this message to the Haptics module to request the initiation of a firmware update. 
##### Payload
```json
{
	"type": "firmware_start"
	"file_size": 0xdeadbeef
}
```

##### Haptics Response
The Haptics module will respond within 100ms if a firmware update can begin. A positive respond tells the ITPP that it is OK to continue with Firmware Update Continue messages.

```json
{
	"type": "firmware_start_response",
	"is_ok": true
	"chunk_size": 0xfeed
}
```

#### Firmware Update Continue
This request is only valid following a successful Firmware Update Start message. The intent of these messages is to send the contents of the firmware to the Haptics module. @todo determine the format of the payload. It is expected that multiple Firmware Update Continue messages will be required to fully send a firmware image.

##### Payload
```json
{
	"type": "firmware_continue",
	"hex_line": "eieio_cia_fbi_kgb_ati_svr_wuz_here"
}
```

##### Haptics Response
The Haptics module will respond within 500ms. @todo Determine exact semantics... is_ok and is_complete are suggestions but may be overkill.

```json
{
	"type": "firmware_continue_response",
	"is_ok": true,
	"is_complete": false
}
```


#### Firmware Update Complete
##### Payload
```json
{
	"type": "firmware_update_complete",
	"do_update": true
}
```

##### Haptics Response
The Haptics module will respond within 100ms if it will attempt the firmware update. A response doesn't mean that the firmware update was successful as the Haptics module will reboot after applying the firmware. This response only indicates an attempt will be made to write the update. The ITPP should wait a period of time for the Haptics module to complete the update and then query for the expected firmware version.

```json
{
	"type": "firmware_update_complete_response",
	"roger_that": true	
}
```

#### Reboot
This instructs the Haptics module to restart.
##### Payload
```json
{
	"type": "rebootski"
}
```
##### Haptics Response
The Haptics module will respond within 100ms if it will reboot or not.
```json
{
	"type": "rebootski_response",
	"yes_sir": true
	
}
```


### Haptics to ITPP

At this time there are no requested expected from the Haptics module to the ITPP.









