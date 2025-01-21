# NOCOM_BOT A-Type Plugin Specification

Version: v1r2p0<br>
Last updated: 12/05/2023

## 1. Overview

NOCOM_BOT is a powerful, flexible chatbot framework that allow bot developer to extends its functionality using modules and plugins. To make things easier for bot developers, this specification will define a new format that is more friendly than writing code that access the Core directly.

The A-Type plugin format is designed to be used C3CBot v1.x, but other bot developers can install the handler as well.

A-Type plugin format does have encrypted plugin variant, and bot supporting encrypt variant MUST enable ATBAN banning system included in the handler (see section 5 for more information).

## 2. Terms and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP
14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

Commonly used terms in this specification are described below.

- Core: The main process which handles all communication from/to different modules.

- Module: A process that is spawned from Core or a thread inside Core, used for all functionality of NOCOM_BOT.

- Plugin handler: A module that is used to parse/execute A-type plugin securely.

- Plugin: Code/file that can be parsed/executed by plugin handler. In this case, it's specifically in A-type plugin format.

## 3. Plugin format

A-Type plugin MUST use ZIP format as the container for all the required files (not including files must be downloaded from Internet on-demand).

The structure of the ZIP file SHOULD be like the tree below.

```
plugin.zip (root)
├─ assets/
│  ├─ anything...
├─ entry.js (!)
├─ jsfunc1.js
├─ jsfunc2.js
├─ tsconfig.json (subclass 1 only)
├─ package.json (*)
├─ pnpm-lock.json (*)
```

Note: 
- If a file/folder is marked (*), then that file/folder MUST be present, and the format of the file marked is in this specification.
- If a file/folder is marked (!), then that file/folder MUST be present, but can be renamed (should be following the specification).
- If a file/folder is not marked, then that file/folder is optional, but recommended.

## 4. File content format

### 4.1. package.json + pnpm-lock.json

These 2 files MUST be present.

Plugin resolver will automaticially install **all** dependencies found in these two files, and the packages installed can be used like usual (`import something from "package"`)

Additionally, package.json MUST contains plugin metadata under key `NOCOM_AType_Metadata`:

```ts
{
    formatVersion: 1,
    subclass: (0 | 1), // See note [1]
    entryPoint: string, // can be redefined, but SHOULD be entry.js / entry.ts
    author: string,
    pluginName: string,
    pluginNamespace: string,
    pluginVersion: string // MUST be SemVer-compatible
}
```

[1]: Subclass MUST be 0 if you are writing in JavaScript (A0), or MUST be 1 if you are writing in TypeScript (A1).

Example of package.json file can be found [https://github.com/NOCOM-BOT/plugin_base_A0/blob/main/package.json](here).

Note that the A-type format specifically uses pnpm, and we recommended that you also uses pnpm in every project, even projects not related to this, as pnpm can saves your disk space.

### 4.2. entry.js / entry.ts

Depending on the subclass, you MUST write either JavaScript or TypeScript code in ES module format on this file. CommonJS format is disallowed in this plugin format.

The `import` keyword can also be used to import code in other files relative to the current file root.

Some features that are in stage 3, and/or not enabled by default (as of 24/06/2022 - ES2022) are **enabled**:

- JSON modules importing (since node 17.5)
```ts
import pluginParam from "./param.json" assert { type: "json" }
```

Before using, you MUST import the function module by inserting this command on the top of entry file (other files are not required to import this file, unless it also needs to use function from this file):

```ts
import * as NOCOM_AType from "@nocom_bot/nocom-atype-support";
```

If you are using TypeScript (to compile to JavaScript A0, or use directly with A1), install `@nocom_bot/types_ts_plugin_a1` and add types using this command:

```ts
import "@nocom_bot/types_ts_plugin_a1";
```

The `NOCOM_AType` variable will have these keys:

```ts
function verifyPlugin(allow: boolean): void
function callFuncPlugin(namespace: string, funcName: string, ...args: any): Promise<any>
function registerFuncPlugin(funcName: string, callback: Function): Promise<boolean>
function callAPI(moduleID: string, cmd: string, value: any): Promise<any>
function registerCommand(
    commandName: string,
    commandInfo: {
        args: {
            fallback: string,
            [ISOLanguageCode: string]: string
        },
        argsName?: string[],
        description: {
            fallback: string,
            [ISOLanguageCode: string]: string
        }
    },
    commandCallback: (data: {
        interfaceID: number,
        interfaceHandlerName: string,

        cmd: string,
        args: string[],
        attachments: {
            filename: string,
            url: string // http(s)/file/data URI
        }[],
        mentions: {
            [formattedUserID: string]: {
                start: number,
                length: number
            }
        },
        originalContent: string,
        prefix: string,

        messageID: string,
        formattedMessageID: string,
        channelID: string,
        formattedChannelID: string,
        senderID: string,
        formattedSenderID: string,
        guildID: string,
        formattedGuildID: string,
        isDM: boolean,

        language: string,
        isOperator: boolean,
        additionalInterfaceData?: any
    }) => Promise<{
        content: string,
        attachments: {
            filename?: string,
            data: Buffer | string // if string then point to url, http(s)/file/data URI allowed
        }[],
        additionalInterfaceData?: any
    }>, 
    compatibility?: string[] // if this is an empty array then this indicates every messages platform is supported, otherwise indicates that this command only supports specific platform.
): Promise<boolean>
function registerCommandFuncPlugin(
    commandName: string,
    commandInfo: {
        args: {
            fallback: string,
            [ISOLanguageCode: string]: string
        },
        argsName?: string[],
        description: {
            fallback: string,
            [ISOLanguageCode: string]: string
        }
    },
    funcName: string, 
    compatibility?: string[]
): Promise<boolean>
function exit(exit_code: number, exit_reason?: string): void
function waitForModule(moduleNamespace: string, timeout?: number): Promise<boolean>
const log = {
    critical: (...data) => Promise<void>,
    error: (...data) => Promise<void>,
    warn: (...data) => Promise<void>,
    info: (...data) => Promise<void>,
    debug: (...data) => Promise<void>,
    verbose: (...data) => Promise<void>
}

function database(databaseID: number): {
    get(table: string, key: string): Promise<any>,
    set(table: string, key: string, data: any): Promise<boolean>,
    delete(table: string, key: string): Promise<boolean>,
    deleteTable(table: string): Promise<boolean>
}
```

**Note 1**: `args` in command info SHOULD be in this standardized format: 
```
<required arg1> <required arg2> [optional arg3]
```

For example (the entire command):
```
/rps <amount> [rock/paper/scissor]
```

**Note 2**: argsName is used for Discord slash command (or equivalent). It MUST only contain English character only, no spaces allowed.

### 4.3. tsconfig.json (subclass 1 only)

See [this](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html) for the format of this file.

## 5. Encryption and ATBAN banning system

> Note: Only closed-source version of handler support this feature.

Plugins can be encrypted if plugin developers don't want other users to see and modify plugin code. If an attempt to tamper protected plugins is detected, ATBAN banning system will take place and make protected plugins refuse to work.

Due to how sensitive this is, this specification will not describe how it is implemented.

## 6. Notes

### 6.1. Localization: language code/locales

All language code MUST follow BCP 47 [RFC4647] [RFC5646] standard (also known as [IETF language tag](https://en.wikipedia.org/wiki/IETF_language_tag)). Some examples of IETF language tag format is at below:

- English (United States): `en-US`
- English (United Kingdom): `en-GB`
- Vietnamese: `vi`
- Japanese: `ja`
