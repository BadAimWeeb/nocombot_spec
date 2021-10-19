# NOCOM_BOT Module Specification

Version: v0r4 (draft)<br>
Last updated: 19/10/2021

## 1. Overview

NOCOM_BOT is a powerful, flexible chatbot framework that allow bot developer to extends its functionality using modules and plugins. 

Because of its flexible nature, there should be a specification to allow execution and communication with different modules. This specification aim to give modules a way to be executed and communicate using pre-defined format.

## 2. Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP
14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

Commonly used terms in this specification are described below.

- Core: The main process which handles all communication from/to different modules.

- Module: A process that is spawned from Core or a thread inside Core, used for all functionality of NOCOM_BOT.

## 3. Module protocol and format

### 3.1. Module programming language and format

To ensure the flexiblity of NOCOM_BOT, any programming language can be used to make Module.

If Module is a npm-compatible (Node.js) package (with `package.json`), Module will be spawned as a new Node.js-compatible process with entry point described in `main` in `package.json`.

If Module is a single Node.js script, then Module will be spawned as a thread in Core (Node.js worker).

If Module is source code in other languages and needs a transpiler (for example, Python), you MUST install that languages before you can use the Module. Additionally, module description MUST contains how to run the source code.

Module MUST be packed in ZIP with `module.json` describing the type of Module, and how it communicates, as below (Note: It's in TypeScript to write comments and possible value, but you MUST write JSON in the ZIP file).

```ts
{
    type: (
        "package" | // If Module is a npm-compatible package
        "script" | // If Module is a Node.js script
        "executable" | // If Module is an executable (usually compiled from other languages than JS)
        "code-src" // If Module is source code writen in other languages
    ),
    shortName: string,
    communicationProtocol: (
        "node_ipc" | // process.send, process.on("message", ...)
        "node_worker" | // require("worker_threads")
        "msgpack" // See 3.2
    ),
    scriptSrc?: string, // Only required if type = "script", describing the entry point location relative to ZIP file root.
    executable?: { // Only required if type = "executable"
        [platform: (
            "win32" | 
            "aix" | 
            "darwin" | 
            "freebsd" | 
            "linux" | 
            "openbsd" | 
            "sunos"
        )]: {
            [cpuArch: (
                "arm" |
                "arm64" |
                "ia32" |
                "mips" |
                "mipsel" |
                "ppc" |
                "ppc64" |
                "s390" |
                "s390x"
                "x32" | 
                "x64"
            )]: string // Path relative to ZIP file root.
        }
    }
    executableArgs?: string, // Only required if type = "executable"
    transplier?: string, // Only required if type = "code-src", describing how  to execute the source code.
    autoRestart: boolean
}
```

Please note that, when Module is being executed, all content inside the ZIP file will be extracted to `<Core's current working directory>/.data/temp/runtime-${random hex}/${module namespace}/`, and the Module's current working directory (if type = "package" or "executable" or "code-src") will be set to that directory.

### 3.2. Core-Module communication protocol

If Module is a Node.js-compatible process and support `process.send` and `process.on("message", ...)` then Module SHOULD use that to communicate with Core.

If Module is a thread inside Core (Node.js worker), it MUST get `parentPort` from `require('worker_threads')` and use `parentPort.on("message", ...)`, `parentPort.postMessage()` to communicate with Core.

Alternatively, Module (that isn't a thread inside Core) can read STDIN and write to STDOUT in a way that's described below to communicate with Core (it's in hexadecimal, but you MUST use raw bytes when communicating).

```
<Message header (string AAAA): 41 41 41 41> <Length in 4-byte integer, big-endian (not string!)> <MessagePack binary serialization of the object received/sent>
```

For example, this is a message containing the object `{foo:"bar"}`.

```
41 41 41 41 00 00 00 09 81 A3 66 6F 6F A3 62 61 72
```

### 3.3. Handshaking

To determine what the module should do, a handshake is required. Module MUST NOT do or send anything before receiving a handshake from Core.

Handshake MUST be done only once, and MUST start from Core.

The Core will send a message to Module with this data to initialize handshake:

```json
{
    "type": "handshake",
    "id": "Module's instance ID",
    "protocol_version": "1"
}
```

In return, Module MUST return a message with data described below in 30 seconds or else Module will be terminated.

```json
{
    "type": "handshake_success",
    "module": "<module type that Module is acting>",
    "module_displayname": "<user-friendly name of the module (example: MongoDB database, Discord interface, ...)>",
    "module_shortname": "<MUST match with the JSON>"
}
```

If Module cannot act as the requested type, 
Module SHOULD return a message describing error and request to be terminated.

```json
{
    "type": "handshake_fail",
    "error": "<... (for example: Unknown module type)>"
}
```

### 3.4. Keep-alive (Ping)

To prevent the process from hanging, an alive-or-dead test (challenge) will be issued from Core at random to determine whether if Module is hanging or not, and Module will be killed if hanging.

Module MUST response to a challenge in 30 seconds to keep running.

The Core will send a message described below to check if Module is alive.

```json
{
    "type": "challenge",
    "challenge": "<random string>"
}
```

If Module received a challenge, it MUST response back a message described below.

```json
{
    "type": "challenge_response",
    "challenge": "<random string from Core>"
}
```

### 3.5. API call/response

Module can call API commands of other modules. When Module want to call an API command, Module MUST send a message:

```json
{
    "type": "api_send",
    "call_to": "<target Module ID>",
    "call_cmd": "<target API command from target module>",
    "data": "...",
    "nonce": "<rolling number for each API call or random number>"
}
```

The target Module will receive a message indicating an API call from other modules:

```json
{
    "type": "api_call",
    "call_from": "<source Module ID>",
    "call_cmd": "...",
    "data": "...",
    "nonce": "..."
}
```

There are 3 possible responses for the target Module:

1. The API command don't exist and cannot be called
```json
{
    "type": "api_sendresponse",
    "response_to": "<source Module ID>",
    "exist": false,
    "nonce": "<exact nonce from request>"
}
```

2. The API command does exist, but the execution of it failed
```json
{
    "type": "api_sendresponse",
    "response_to":
                    
                 "<source Module ID>",
    "exist": true,
    "error": "...",
    "data": null,
    "nonce": "<exact nonce from request>"
}
```
> Note: Error MUST NOT be an Error object from JavaScript or the native implementation. It SHOULD be a string, or number, or even null (if there isn't an error message).

3. The API command does exist, and executed successfully
```json
{
    "type": "api_sendresponse",
    "response_to": "<source Module ID>",
    "exist": true,
    "error": null,
    "data": "...",
    "nonce": "<exact nonce from request>"
}
```

When the target Module has responded, source Module will receive an API response:

```json
{
    "type": "api_response",
    "response_from": "<target Module ID>",
    "exist": "...",
    "data": "...", // This will be null if there's an error.
    "error": "...", // This will be null if error doesn't occur.
    "nonce": "<exact nonce from request>"
}
```

## 4. Core API call

> Note: The Core's API will be available at module ID `core` to not complicate things up.

### 4.1. Get registered modules (`get_registered_modules`)

Data: none

Return: 
```ts
{
    moduleID: string,
    shortname: string,
    displayname: string
}[]
```

### 4.2. Kill this module (`kill`)

Data: none

Return: `null`

### 4.3. Shutdown the Core (`shutdown_core`)

Data: none

Return: `null`

### 4.4. Restart the Core (`restart_core`)

Data: none

Return: `null`

### 4.5. Register event hook (`register_event_hook`)

Data: 
```ts
{
    callbackFunction: string,
    eventName: string,
    downloadBufferFrom: number // timestamp
}
```

Return:
```ts
{
    success: boolean
}
```

For more information, see 6.

### 4.6. Unregister event hook (`unregister_event_hook`)

Data:
```ts
{
    callbackFunction: string,
    eventName: string
}
```

Return:
```ts
{
    success: boolean
}
```

For more information, see 6.

## 5. Application-specific API call

> Note: If you are creating an interface that is using the module types defined below, you MUST implement all API call to maintain compatibility. Additional API commands MAY be defined if Module needs that.

### 5.1. Interface handler (module type = "interface")

> Note: Only one instance per interface will be created. The Core will expect the interface handler to handle multiple accounts.

#### **5.1.1. Login (`login`)**

Data:
```ts
{
    interfaceID: number,
    interfaceData: "module defined"
}
```

Return: 
```ts
{
    success: boolean,
    interfaceID: number,
    accountName: string,
    rawAccountID: string,
    formattedAccountID: string,
    accountAdditionalData: any
}
```

#### **5.1.2. Logout (`logout`)**

Data:
```ts
{
    interfaceID: number
}
```

Return: `null`

#### **5.1.3. Get User info (`get_userinfo`)**

Data:
```ts
{
    userID: string
}
```

Return:
```ts
{
    userName: string,
    ... // additional data is module-defined.
}
```

#### **5.1.4. Get Thread/Channel info (`get_channelinfo`)**

Data:
```ts
{
    channelID: string
}
```

Return:
```ts
{
    channelName: string,
    ... // additional data is module-defined.
}
```

#### **5.1.5. Send message (`send_message`)**

Data:
```ts
{
    content: string,
    attachments: string[], // URL (file:// is allowed)
    threadID: string,
    replyMessageID?: string,
    ... // additional data is module-defined.
}
```

Return:
```ts
{
    success: boolean,
    messageID: string,
    ... // additional data is module-defined.
}
```

### 5.2. Database (module type = "database")

#### **5.2.1. Connect database**

Data: module-defined

Return:
```ts
{
    success: boolean
}
```

#### **5.2.2. Get data**

Data:
```ts
{
    table: string,
    key: string
}
```

Return: data in that location

#### **5.2.3. Set data**

Data:
```ts
{
    table: string,
    key: string,
    value: any
}
```

Return: 
```ts
{
    success: boolean
}
```

#### **5.2.4. Delete data**

Data: 
```ts
{
    table: string,
    key: string
}
```

Return:
```ts
{
    success: boolean
}
```

#### **5.2.5. Delete table**

Data:
```ts
{
    table: string
}
```

Return:
```ts
{
    success: boolean
}
```

#### **5.2.6. Disconnect database**

Data: none

Return: none

## 6. Events

Sometimes you need to broadcast to a lot of modules interested in a topic without knowing which modules subscribed. This is where Events come in.

Events is part of the Core module. To register/unregister an event, use API call 4.5 and 4.6.

TBD.
