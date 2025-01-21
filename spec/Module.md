# OpenBotKernel Native Module Specification

Version: draft-01 <br>
Last updated: 2025-01-20

## 1. Overview

OpenBotKernel is an flexible chatbot framework that allow bot developer to 
extends its functionality using modules and user-friendly extension. 

Because of its flexible nature, there should be a specification to allow 
execution and communication with different modules. This specification aim to 
give modules a way to be executed and communicate using pre-defined format.

This specification will define the module container format, the in-band 
communication protocol between Kernel and Module, and the communication 
protocol between Modules using API.

## 2. Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", 
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this 
document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when,
and only when, they appear in all capitals, as shown here.

Commonly used terms in this specification are described below.

- OBK: abbreviation for OpenBotKernel.

- Kernel: The main process which handles most communication from/to different 
modules, and coordinate the execution of modules.

- Module: A process that is spawned from Kernel or a thread inside Kernel, 
handling most functionality needed for operation.

## 3. Module programming language and container format

To ensure the flexiblity of OBK, any programming language can be used to make 
Module, as long as they are able to communicate using the format described in
this specification.

All files required to run a module SHOULD be packaged in a single container.
The container format is RECOMMENDED to be a ZIP file, but other formats can be
implemented as needed (TODO: add more container format, or add a list here).

The container MUST contain a file named `module.json` which contains the module
metadata and configuration. The file MUST be in JSON format.

In the `module.json` file, the following fields are REQUIRED:

- `name`: The human name of the module. This field MUST be a string.

- `namespace`: The namespace of the module. This field MUST be a string. The 
  namespace MUST be unique to avoid conflicts with other modules, and cannot be
  the reserved namespace `kernel`.

- `version`: The version of the module. This field MUST be a SemVer-formatted 
  string.

- `implemented_interfaces`: An array of strings that represent the standard API 
  that the module implements. The values are defined in other specifications.

- `exec_type`: The type of execution of the module. This field MAY be one of 
  the following values:

  + `process`: The module is already compiled and can be executed as a native 
    process.

  + `js_script`: The module is a JavaScript script that can be executed by a
    JavaScript runtime, such as Node.js, Deno, or Bun.

  + `js_npm`: The module is a JavaScript module in npm package format. Its
    entry point is defined in the `main` field in the `package.json` file.

  + `wasi`: The module is a WebAssembly module that can be executed with a 
    compatible WebAssembly runtime implementing the WASI API. RECOMMENDED.

  + For other values, another module can register for the type and handle the
    execution.

The following fields are OPTIONAL but RECOMMENDED:

- `description`: A brief description of the module. This field MUST be a 
  string.

- `author`: The author of the module. This field MUST be a string.

- `license`: The license of the module. This field MUST be a string.

- `keywords`: An array of strings that can be used to search for the module.

- `config_schema`: A JSON schema that describes the configuration object of the
  module. This field MUST be an object. The schema is used to validate the 
  configuration object before passing it to the module, and is also used for 
  GUI and utilities to assist users writing config. OPTIONAL.

- `supported_languages`: An array of BCP 47 [RFC4647] [RFC5646] language tags 
  that the module supports. This array can include `*` to indicate that the 
  module dynamically supports any language. OPTIONAL.

Depending on the `exec_type`, the following fields are REQUIRED:

- If `exec_type` is `process`:

  + `bin`: An dictionary which keys are the platform name append with `-` and 
    the architecture name, and the value is the path to the executable file 
    inside the container.

    The platform name SHOULD be one of the following values:
    `darwin`, `linux`, `win32`, `*`

    The architecture name SHOULD be one of the following values:
    `armv7l`, `aarch64`, `ppc64le`, `s390x`, `x86_64`, `i686`, `riscv`, 
    `riscv64`, `*`

    The `*` value means that platform check before running the module is not
    required, it does not mean that the module can run on any platform. This 
    value MAY only be used if you are sure that users will not run the module
    on a platform that is not supported.

  + `args`: An array of strings that will be passed as arguments to the 
    executable file. OPTIONAL.

- If `exec_type` is `js_script`:

  + `script`: The path to the JavaScript script file inside the container.

- If `exec_type` is `wasi`:

  + `wasm`: The path to the WebAssembly module file inside the container.

  + `wasi_args`: An array of strings that will be passed as arguments to the 
    WebAssembly module. OPTIONAL.

Kernel MAY only support a subset of the `exec_type` values, and move the 
support for other values to another module that can handle the execution of
the requested module. This type of module is called an Alternative Module 
Resolver, and is defined in the 
[Alternative Module Resolver Specification](spec/Alternative_Module_Resolver.md).

## 4. In-band module communication

### 4.1. Module communication protocol and format

Modules MUST support communicating with Kernel using the standard input and 
output. For each message, the sender MUST write a raw binary message to the
standard output in the following format:

- Header: 4 bytes, the ASCII string "OBK\0" (raw bytes `0x4F 0x42 0x4B 0x00`).

- Packet type: 1 byte, the type of the packet.

- Length: 4 bytes, the length of the payload in big-endian format.

- Payload: The message payload. Depending on the packet type, its format can be
  different.

Data from Kernel to Module MUST be sent in the same format.

An out-of-band channel (outside of standard input and output) MAY be used for
communication between Kernel and Module or between Modules, after the initial
communication is established.

All communication is expected to be asynchronous.

#### 4.2. Module communication packet types

The following packet types are defined for communication between Kernel and
Module:

- `0x01`: Module initialization handshake packet. The payload is a MessagePack
  object, its content is defined in 
  [section 4.3](#43-module-initialization-handshake).

- `0x02`: Public event packet. The payload is a MessagePack object, its content
  is defined in [section 4.5](#45-public-event-packet). Modules MUST subscribe to the
  events they are interested in to receive these packets.

- `0x03`: API packet. The payload is a MessagePack object, its content
  is defined in [section 4.6](#46-api-packet).

- `0x04`: Keep-alive packet. The payload is a MessagePack object, its content 
  is defined in [section 4.4](#44-keep-alive-packet).

- `0x05`: Out-of-band negotiation packet. It is defined in 
  [Out-of-band Module Communication Specification](spec/OOB_Module_Comm.md).

- `0x06`: Psuedo-module API packet. It is defined in 
  [User-friendly Extension Specification](spec/User_friendly_Extension.md).

### 4.3. Module initialization handshake

All messages sending with the packet type `0x01` MUST be sent in MessagePack 
format.

Before doing anything, Module MUST send an array consisting of only one 
element, which is a constant number `0x01` to Kernel. This indicates that the
Module is compatible with the OBK protocol and is ready to receive the 
parameters for initialization.

Kernel MUST respond with an array consisting of two, which is a constant number
`0x02` and a MessagePack object. The object MUST contain the following fields:

- `runtime_id`: A unique identifier for the runtime of the Module. This field 
  MUST be a number. This is only used internally by Kernel and Module.

- `config`: User-defined configuration for the Module. This field MUST be an 
  object. Its content is defined by the Module.

- `system-wide_language`: The language tag of the system-wide language. This 
  field MUST be a string. Usually, if not specified by the user, this field
  will be English (`en`). Usage of this field is RECOMMENDED.

Module then MUST respond with an array consisting of two element, which is a 
constant number `0x03` and a MessagePack object. The object MUST contain the
following fields:

- `s`: A boolean value that indicates whether the initialization is successful.
  If this value is `false`, the Module is REQUIRED to be terminated by Kernel.
  Module MAY self-terminate early after sending this if wanted.

- `runtime_id`: The runtime ID that was sent by Kernel. This field MUST be a 
  number.

If the value of `s` is `false`, the object MAY contain an additional field:

- `error`: A string that describes the error. This field MUST be a string.

If the value of `s` is `true`, the object MUST contain the following fields:

- `available_interfaces`: An array of strings that represent the standard API 
  that the module implements. The values are defined in other specifications.
  This values MUST be a subset of the `implemented_interfaces` field in the
  `module.json` file.

- `namespace`: The namespace of the module. This field MUST be a string, and
  MUST be the same as the `namespace` field in the `module.json` file.

Additionally, the object MAY contain the following fields, used for
verification of values in `module.json` file:

- `name`: The human name of the module. This field SHOULD be a string, and it
  MUST match the `name` field in the `module.json` file.

- `version`: The version of the module. This field SHOULD be a string, and it
  MUST match the `version` field in the `module.json` file.

- `description`: A brief description of the module. This field SHOULD be a
  string, and it MUST match the `description` field in the `module.json` file.

- `author`: The author of the module. This field SHOULD be a string, and it
  MUST match the `author` field in the `module.json` file.

- `license`: The license of the module. This field SHOULD be a string, and it
  MUST match the `license` field in the `module.json` file.

- `supported_languages`: Currently supported languages by the module. This 
  field SHOULD be an array of strings. It does not need to match the
  `supported_languages` field in the `module.json` file, but it is RECOMMENDED
  to match.

### 4.4. Keep-alive packet

Kernel MAY send a keep-alive packet to the Module to ensure that the Module is
still alive. The packet type is `0x04`, and the payload is random bytes. The
Module MUST respond with the same payload.

### 4.5. Public event packet

In some cases, a Module may wish to broadcast an event to other Modules,
without knowing the identity of the recipients. The packet type is `0x02`, and
the payload is a MessagePack object.

To receive public events, a Module MUST subscribe to the events it wants to
receive. The subscription is done by sending a Kernel API request defined
(TODO: add link to kernel API specification).

Event will be returned to the Module in this format:

- `event`: The event name. This field MUST be a string.

- `data`: The event data. This field MUST be a MessagePack object.

- `timestamp`: The timestamp of the event. This field MUST be a number.

- `source`: The source of the event. This field MUST be a string, and it is the
  namespace of the Module that sent the event.

To send a public event, a Module MUST send a packet with the type `0x02` to
Kernel. The payload MUST be a MessagePack object with the following fields:

- `event`: The event name. This field MUST be a string.

- `data`: The event data. This field MUST be a MessagePack object.

### 4.6. API packet

All API calls are sent and received using the packet type `0x03`. Payload MUST
be a MessagePack object. Request and response format for both direction are the same.

To call an API, or when receiving an API call, the payload MUST contain the 
following fields:

- `r`: The boolean value `false`. This field MUST be a boolean.

- `namespace`: This field MUST be a string.

  If sending, this field is the namespace of the Module that the API call is 
  intended for. 
  
  If receiving, this field is the namespace of the Module that sent the API 
  call. 

- `cmd`: The command to invoke. This field MUST be a string.

- `data`: The data to pass to recipient. This field can be any type, but is 
  RECOMMENDED to be an object.

- `nonce`: A unique identifier for the API call. This field MUST be a string,
  and it MUST be unique for each API call. It is RECOMMENDED to prepend 
  `runtime_id` to the nonce to ensure uniqueness between modules.

When responding to an API call, the payload MUST contain the following fields:

- `r`: The boolean value `true`. This field MUST be a boolean.

- `namespace`: This field MUST be a string.

  If sending response, this field is the namespace of the Module that sent the
  API call.

  If receiving response, this field is the namespace of the Module that is
  responding to the API call.

- `success`: The success status of the API call. This field MUST be a boolean.

If the API call is successful, the payload MUST contain the following fields:

- `data`: The data to pass to the sender. This field can be any type, but is 
  RECOMMENDED to be an object.

If the API call is unsuccessful, the payload MUST contain the following fields:

- `error`: The error message. This field MUST be a string.
