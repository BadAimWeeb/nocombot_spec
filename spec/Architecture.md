# OpenBotKernel Architecture Specification

Version: draft-01 <br>
Last updated: 2025-01-20

## 1. Overview

OpenBotKernel is an flexible chatbot framework that allow bot developer to 
extends its functionality using modules and user-friendly extension. 

The architecture of OpenBotKernel is designed to be modular and extensible.
Because of that, the entire system is divided into multiple components that are 
designed to perform specific tasks, and can work together to provide a complete 
chatbot experience.

## 2. Typical architecture of OpenBotKernel

The typical architecture of OpenBotKernel is shown in the following diagram:

```
+----------+             +-------------------+                                 
|  Kernel  | <=========> |  Database Module  | - - - - - [External DB]          
+----------+      |      +-------------------+                                 
                  |                                                            
                  |      +--------------------------+                          
                  |====> |  User Command Processor  |                          
                  |      +--------------------------+                          
                  |                                                            
                  |      +-------------------------------+                
                  |====> |  Alternative Module Resolver  |                 
                  |      +-------------------------------+                 
                  |                       |                                    
                  |                 +----------+                              
                  |===============> | Module X |
                  | (out-of-band)   +----------+
                  |                 | Module Y |
                  |                 +----------+
                  |
                  |      +-------------------------------------+
                  |====> | Developer-friendly Extension Module |
                  |      +-------------------------------------+      
                  |              |
                  |         +--------------+
                  |         |  "Plugin" A  |
                  |         +--------------+
                  |         |  "Plugin" B  |
                  |         +--------------+
                  |                                             [Facebook]
                  |      +----------------------+               [Discord]
                  |====> | User-input Interface | - - - - - - - [Telegram] ...
                         +----------------------+               [Zalo]
```

### 2.1. Kernel

The Kernel is the core component of OpenBotKernel. It is responsible for 
managing the entire system, and provides the necessary interfaces for other 
components to interact with each other.

Modules can interact with the Kernel through the Kernel API, which is a set of 
functions and data structures that the Kernel provides to modules. It is defined
in [Kernel specification](Kernel.md).

### 2.2. Database Module

Database Module is a special module that provides an interface to an external
(or internal) database. It is responsible for storing and retrieving data that 
is needed by other modules.

This type of module has a predefined set of API functions that other modules can
use to interact with the database. The API functions are defined in the
[Database specification](Database.md).

### 2.3. User Command Processor

User Command Processor is a module that is responsible for processing user raw 
input and converting it into a structured command that can be understood by 
other modules.

This can be done by using natural language processing techniques (LLM, NLP,
etc.), or by using a predefined set of rules to parse the input (such as slash
commands).

UCP will listen to user input from event sent by User-input Interface, process
the result with predefined rules (can be passed to another module for further
processing), and then pass the result back to User-input Interface for sending
the response back to user.

The most common type of UCP is slash command, defined in 
[Slash Command specification](Slash_Command.md).

### 2.4. Alternative Module Resolver

Alternative Module Resolver (AMR) is a module that is responsible for resolving 
and executing type of module that kernel cannot understand. AMR is strictly
forbidden from transforming the communication between modules and kernel. Module
executed by AMR is considered to be the same as module executed natively by 
kernel.

For example: Suppose that there is a module that is written in Python, but the
kernel is written in JavaScript and does not know how to execute Python code.
In this case, the Alternative Module Resolver will be responsible for executing
the Python code (preferably in a separate process), and establishing a separate
communication channel from the Python module to the Kernel.

This type of module may not be needed at all, but if implemented, it must follow
the [Alternative Module Resolver specification](Alternative_Module_Resolver.md).

### 2.5. Developer-friendly Extension Module

Developer-friendly Extension Module (DFEM) is a module that is responsible for 
providing a better, more user-friendly API interface for users to write their 
own modules.

It is different from the Alternative Module Resolver in that it is designed to
abstract away the complexity of interacting directly with the Kernel and other
modules, and provide a more high-level, user-friendly interface for developer to
write their own modules, extending the overall functionality of chatbot. 

Kernel considers module executed by DFEM to be a pseudo-module (can commonly be 
called a plugin). Psuedo-module does not need to implement the OBK protocol, 
all requests are handled by DFEM.

The container format and protocol accepted by User-friendly Extension Module 
may be vastly different from the container format accepted by the Kernel.

DFEM needs to follow the [Psuedo-module specification](Psuedo-module.md) to 
register plugin and communicate with kernel, and implement the 
[DFEM Plugin Registration specification](DFEM_Plugin_Registration.md).

### 2.6. User-input Interface

User-input Interface is a module that is responsible for receiving user input
from various sources (such as Facebook, Discord, Telegram, etc.), and passing
it to the User Command Processor for processing. It is also responsible for 
sending the response back to the user.

This type of module may do a little bit of preprocessing on the user input
before passing it to the User Command Processor, especially if the input is not
a standard text message (for example: Discord Slash Commands).

This type of module is defined in the 
[User-input Interface specification](User-input_Interface.md).

## 3. Distribution

OpenBotKernel can be repackaged and distributed as a distribution (distro),
which is a collection of modules that are bundled together with the Kernel to
provide a complete chatbot experience.

A distro can be customized to include only the modules that are needed for a
specific use case, and can be extended with additional modules that are not
included in the base distro.

Usually, the difference between distros is the DFEM that is bundled with the
Kernel.
