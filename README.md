# QuackC2
### About
My next Command and Control framework using lessons learned from [IronHelm](https://github.com/vlcakx90/IronHelm). 
- Will use the  [C2 Comms Spec: OST-C2-Spec](https://github.com/rasta-mouse/ost-c2-spec?tab=readme-ov-file#link-pass-thru) for communication specification.
- Will generally follow the design of [Cobalt Strike](https://www.cobaltstrike.com/)
- Will be partly or entirely written in Rust
- Serve as a project for me to learn and have fun :)

# Proposed Design
### Project Layout
| QuackC2 | Cobalt Strike | About |
| --- | --- | --- |
|  DuckHouse | TeamServer | Rust or C# Multi-player server with db|
|  Duck | Beacon | Rust agent with built in comms and commands built as DLL |
| PIC DLL Generators | NA | Tools to create PIC DLL |
|  Egg | Stager | Rust stager that loads the PIC DLL |
|  Client | Client | Cross-platform Rust Desktop client for Teamserver |
|  Waddle Script | Aggressor Script | Scripting new behavior (commands) using Lua |

### Flow
Detailed configuration of behavoir will be set via C2Profile and Operator inputs
1. Client connects to DuckHouse
2. Client creates Duck DLL and stamps with DuckHouse public RSA key
3. Client transforms Duck DLL into PIC DLL
4. Client stamps PIC DLL into an Egg (if stageless, otherwise PIC DLL is hosted on DuckHouse)
5. Egg is delivered to target
6. Egg hatches and loads Duck PIC DLL which registers to Duckhouse
7. Client tasks Duck

# First Version
### Comms
C2 Comm Specification will follow [C2 Comms Spec: OST-C2-Spec](https://github.com/rasta-mouse/ost-c2-spec?tab=readme-ov-file#link-pass-thru)
- specifies format of data (binary stream) for each type of transmission
	- agent registration, agent checkin (tasks), child agent registration, and child tasks
	- formats are specified for metadata, tasks and responses

### DuckHouse
- Attached to SQLite databse
- Manages listeners and hosted files
- If in Rust, will utilize the [Tokio](https://docs.rs/tokio/latest/tokio/runtime/)  runtime for asynchronous support

### Duck
Implements comms and commands from [C2 Comms Spec: OST-C2-Spec](https://github.com/rasta-mouse/ost-c2-spec?tab=readme-ov-file#link-pass-thru)
- Egress protocol HTTP
- Peer-to-Peer protocols SMB & TCP
- Built as DLL

### PIC DLL Loaders
Will utilitze both of these projects to turn a DLL into PIC DLL (PIC reflective loading)
- [Donut](https://github.com/TheWover/donut): good out-of-the-box utility
- [Crystal Palace](https://tradecraftgarden.org/index.html): provides great extensibility

### Egg
Mix of open-source and ports of them to Rust
-  C via [The Garden](https://tradecraftgarden.org/tradecraft.html)
- Varrious Rust ports of known techniques
- Built as EXE, DLL, and Service EXE

### Client
A Cobalt Strike style cross-platform desktop client using Rust with [Tauri](https://tauri.app/)
- GUI will be designed in [Figma](https://www.figma.com/) and a TBA AI tool will be utilitzed to create the HTML/CSS and some JavaScript (because I am allergic to HTML/CSS)
- availble Duck commands will be loaded from a YAML file

Commands in YAML may look like
```yaml
﻿- Alias: cp
  Description: cp file from src to dst with optional force
  Type: 3
  Code: 0
  Arguments:
    - Key: source
	  DataType: 0
      Optional: false
    - Key: destination
	  DataType: 0
      Optional: false
	- Key: force
	  DataType: 3
	  Optional: true

###### C2 Comms Spec: OST-C2-Spec ######
#FILE-COPY-REQ {
#  source       [1]  String
#  destination  [2]  String
#  force        [3]  Boolean  OPTIONAL
#}
```

Rust structs example for deserialization
```Rust
use serde::Deserialize;
use std::fs;

#[derive(Debug, Deserialize)]
struct Command {
    Alias: String,
    Description: String,
    Type: u32,
    Code: u32,
    Arguments: Vec<Argument>,
}

#[derive(Debug, Deserialize)]
struct Argument {
    Key: String,
	DataType: DataType,
    Optional: bool,
}

#[derive(Debug, Deserialize)]
enum DataType {
    String,
    Int32,
    UInt32,
    Bool,
}
```

#### Waddle Script: Teaching a Duck How to Walk
Use [rlua](https://docs.rs/rlua/latest/rlua/) to load Lua scripts to define new behavior (commands)
- scripts will be able to access specific Rust functions that have been exposed to Lua
- operator supplied commands will be searched in the loaded commands list that is initialized with a yaml file (explained above in Client section), this list will include loaded waddle scripts

Waddle Script Example (Lue)
```LUA
hello_world(42)
```

Waddle Script Loading and Running (Rust)
```Rust
use rlua::{Lua, Result};
use std::fs;
    // Lua instance
    let lua = Lua::new();
    // Exposed Rust Function
    let hello_world = lua.create_function(|_, num: i32| {
        println!("Hello, World! The number is: {}", num);
        Ok(())
    })?;
    // Set function in Lua env
    lua.globals().set("hello_world", hello_world)?;
	// Read Lua script
	et lua_script = fs::read_to_string("waddle.lua")
        .expect("Failed to read Waddle script file");
    // Run Lua script
	lua.load(&lua_script).exec()?;
```

Waddle Scripts can be added to commands list by executing a 'register' function in them (Similar to how Cobalt Strike does it) that can simply add it to an existing list
```C
# Cobalt Strike Example
beacon_remote_exploit_register("dcom", "x64", "Use DCOM to run a Beacon payload", &invoke_dcom);
```

#### C2Profile
YAML file to hold configuration for DuckHouse, Egg, and Duck
- set paths for listeners
- format of Get/Post requests and where data is placed
- sleep/jitter
- stage (ex. module -> dll load)
- post-ex (ex. amsi/etw disable, spawn-to )
- Windows API Interaction (Kernel32, re-map Kernel32, Ntdll, re-map Ntdll, Direct Syscalls, Indirect Syscalls, API hashing)

# Later Versions
#### More Comm Protocols
- DNS over HTTPS
- QUIC
- WebSocket
- ...

#### More Opsec
- Sleep Obfuscation
- AMI/ETW Bypass
- ...
