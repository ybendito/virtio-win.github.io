# VirtIO Socket Driver for Windows

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
- [Build System](#build-system)
- [Installation](#installation)
- [Usage](#usage)
- [API Reference](#api-reference)
- [Testing](#testing)
- [License](#license)

---

## Overview

The **VirtIO Socket (viosock)** driver provides Windows support for **AF_VSOCK** address family, enabling fast and efficient communication between virtual machines and their host systems. This implementation is based on the Linux VM Sockets specification and provides a complete Windows driver stack for VirtIO socket devices.

### Key Features

- **Full AF_VSOCK Support**: Complete implementation of the virtio-vsock protocol
- **Winsock Service Provider (WSP)**: Seamless integration with Windows Sockets API
- **Kernel Socket (WSK) Provider**: Kernel-mode socket support
- **Multi-Architecture**: Supports x86, x64, and ARM64 platforms
- **TCP Bridge Utility**: VSock-to-TCP proxy for SSH and other protocols
- **Windows 10/11 Compatible**: Full support for modern Windows versions

### Use Cases

- VM-to-host communication without network configuration
- Fast inter-VM communication on the same host
- SSH access to VMs via VSock (bypassing network stack)
- Container and virtualization platforms
- Secure, isolated communication channels

---

## Architecture

### Driver Stack

The VirtIO Socket driver consists of multiple layers:

```
┌─────────────────────────────────────┐
│   Applications / User-Mode Tools    │
├─────────────────────────────────────┤
│   Winsock API (WS2_32.DLL)          │
├─────────────────────────────────────┤
│   VirtIO Socket WSP Service         │
│   (viosockwspsvc.exe)               │
├─────────────────────────────────────┤
│   VirtIO Socket Library             │
│   (viosocklib.dll)                  │
├─────────────────────────────────────┤
│   Kernel Socket Provider (WSK)      │
├─────────────────────────────────────┤
│   VirtIO Socket Kernel Driver       │
│   (viosock.sys)                     │
├─────────────────────────────────────┤
│   VirtIO Infrastructure             │
└─────────────────────────────────────┘
```

### Address Family

VirtIO Sockets use the `AF_VSOCK` address family with the following address structure:

```c
typedef struct sockaddr_vm {
    ADDRESS_FAMILY svm_family;  // AF_VSOCK
    USHORT svm_reserved1;
    UINT svm_port;              // Port number (host byte order)
    UINT svm_cid;               // Context ID (host byte order)
} SOCKADDR_VM, *PSOCKADDR_VM;
```

### Context IDs (CIDs)

- **VMADDR_CID_ANY (-1)**: Bind to any address
- **VMADDR_CID_HYPERVISOR (0)**: Reserved for hypervisor
- **VMADDR_CID_RESERVED (1)**: Reserved
- **VMADDR_CID_HOST (2)**: Host system
- **Guest VMs**: Assigned unique CID values (typically 3+)

---

## Components

### 1. Kernel Driver (`sys/`)

**Location**: `D:\Projects\virtio-win\master\viosock\sys\`

The kernel-mode driver (`viosock.sys`) implements the core VirtIO socket protocol.

#### Key Files

- **`Driver.c`**: Driver entry point and initialization
- **`Device.c`**: Device management and PnP handling
- **`Socket.c`**: Socket creation and management
- **`Rx.c`**: Receive path implementation
- **`Tx.c`**: Transmit path implementation
- **`Loopback.c`**: Local loopback support
- **`Evt.c`**: Event handling
- **`IsrDpc.c`**: Interrupt service routine and DPC handling

#### Features

- VirtIO device initialization and configuration
- Socket lifecycle management (create, bind, connect, listen, accept, close)
- Asynchronous I/O with IRP completion
- Interrupt-driven packet processing
- Loopback optimization for local connections
- MSI/MSI-X interrupt support

### 2. User-Mode Library (`lib/`)

**Location**: `D:\Projects\virtio-win\master\viosock\lib\`

The user-mode library (`viosocklib.dll`) provides the Winsock Service Provider interface.

#### Key Files

- **`viosocklib.c`**: WSP implementation and function table
- **`native.c`**: Native socket operations
- **`utils.c`**: Utility functions
- **`install.c`**: WSP installation and registration

#### WSP Functions

Implements all standard Winsock Service Provider functions:
- Socket creation: `WSPSocket`
- Connection: `WSPConnect`, `WSPBind`, `WSPListen`, `WSPAccept`
- Data transfer: `WSPSend`, `WSPRecv`, `WSPSendTo`, `WSPRecvFrom`
- Socket options: `WSPGetSockOpt`, `WSPSetSockOpt`
- Event handling: `WSPEventSelect`, `WSPEnumNetworkEvents`
- Asynchronous operations: `WSPAsyncSelect`
- I/O completion: `WSPGetOverlappedResult`

### 3. WSK Provider (`wsk/`)

**Location**: `D:\Projects\virtio-win\master\viosock\wsk\`

Kernel-mode socket provider for WSK (Windows Sockets Kernel) consumers.

#### Key Files

- **`viowsk.c`**: WSK provider implementation
- **`provider.c`**: Provider registration and dispatch
- **`socket.c`**: Socket operations
- **`socket-internal.c`**: Internal socket management
- **`wsk-completion.c`**: Completion handling
- **`wsk-utils.c`**: Utility functions
- **`wsk-workitem.c`**: Work item processing

### 4. WSP Service (`wspsvc/`)

**Location**: `D:\Projects\virtio-win\master\viosock\wspsvc\`

Windows service (`viosockwspsvc.exe`) that manages the Winsock Service Provider.

#### Features

- Auto-start service (SERVICE_AUTO_START)
- WSP protocol registration and management
- Service control handling (start, stop, pause)

### 5. TCP Bridge (`tcp-bridge/`)

**Location**: `D:\Projects\virtio-win\master\viosock\tcp-bridge\`

**Executable**: `vstbridge.exe`

Windows alternative to Linux `systemd-ssh-proxy` for SSH over VSock.

#### Purpose

Bridges VSock connections to TCP, enabling SSH and other TCP-based protocols to work over VSock transport.

#### Key Files

- **`main.cpp`**: Entry point and command-line parsing
- **`bridge.cpp`**: Bridge logic
- **`service.cpp`**: Windows service implementation
- **`socket.cpp`**: Socket abstraction

#### Usage Example

```bash
# SSH to Windows VM using VSock
ssh -o ProxyCommand="socat - VSOCK-CONNECT:<vm_id>:22" <user>@0.0.0.0
```

Replace:
- `<vm_id>`: VSock CID of the guest VM
- `<user>`: Username in the Windows guest

### 6. Test Tools

#### viosock-test (`viosock-test/`)

User-mode test utility for VirtIO socket operations.

**Key Files**:
- `viosock-test.c`: Main test program
- `socket.c`: Socket test implementations

#### viosocklib-test (`viosocklib-test/`)

Test program for the viosocklib.dll library.

**Key Files**:
- `viosocklib-test.c`: Library function tests

#### viosock-wsk-test (`viosock-wsk-test/`)

Kernel-mode test driver for WSK provider.

**Key Files**:
- `viosockwsk-test.c`: WSK test implementation
- `test-messages.c`: Test message handling

### 7. Python Test Scripts

#### `vioconnect.py`

Client-side test script that connects to a VSock server.

```python
import socket

client = socket.socket(socket.AF_VSOCK, socket.SOCK_STREAM)
client.connect((3, 401))  # CID=3, Port=401
client.send(data)
packet = client.recv(65536)
client.close()
```

#### `violisten.py`

Server-side test script that listens for VSock connections.

```python
import socket

server = socket.socket(socket.AF_VSOCK, socket.SOCK_STREAM)
server.bind((-1, 401))  # VMADDR_CID_ANY, Port=401
server.listen(1)
conn, addr = server.accept()
packet = conn.recv(65536)
conn.send('Recv OK')
conn.close()
```

---

## Build System

### Build Scripts

#### `buildAll.bat`

Builds all components for all platforms and configurations.

#### `build_AllNoSdv.bat`

Builds without Static Driver Verifier (SDV) analysis.

#### `cleanAll.bat`

Cleans all build artifacts.

### Visual Studio Solution

**File**: `viosock.sln`

The solution contains the following projects:

1. **viosock** (`sys/viosock.vcxproj`): Kernel driver
2. **viosocklib** (`lib/viosocklib.vcxproj`): User-mode library
3. **wsk** (`wsk/wsk.vcxproj`): WSK provider
4. **wspsvc** (`wspsvc/viosockwspsvc.vcxproj`): WSP service
5. **tcp-bridge** (`tcp-bridge/tcp-bridge.vcxproj`): TCP bridge utility
6. **viosock-test** (`viosock-test/viosock-test.vcxproj`): User-mode test
7. **viosocklib-test** (`viosocklib-test/viosocklib-test.vcxproj`): Library test
8. **viosock-wsk-test** (`viosock-wsk-test/viosock-wsk-test.vcxproj`): WSK test
9. **ViosockPackage** (`ViosockPackage/ViosockPackage.vcxproj`): Driver package

### Supported Platforms

- **x86** (32-bit)
- **x64** (64-bit)
- **ARM64** (64-bit ARM)

### Build Configurations

- **Win10Debug**: Windows 10+ debug build
- **Win10Release**: Windows 10+ release build

---

## Installation

### Prerequisites

- Windows 10/11 or Windows Server 2019/2022/2025
- Administrator privileges
- VirtIO-capable virtualization platform (QEMU/KVM, etc.)

### Installation Files

The driver package (`Install/Win10/`) includes:

- **`viosock.sys`**: Kernel driver
- **`viosock.inf`**: Driver installation information
- **`viosock.cat`**: Catalog file (driver signature)
- **`viosocklib.dll`** (x86/x64/ARM64): User-mode library
- **`viosockwspsvc.exe`**: WSP service
- **`vstbridge.exe`**: TCP bridge utility (x64/ARM64 only)
- **`viosock-test.exe`**: Test utility
- **`viosocklib-test.exe`**: Library test utility

### Installation Steps

1. **Install the driver package**:
   ```powershell
   pnputil /add-driver viosock.inf /install
   ```

2. **Verify installation**:
   ```powershell
   Get-Service VirtioSocket
   Get-Service VirtioSocketWSP
   ```

3. **Start services** (if not auto-started):
   ```powershell
   Start-Service VirtioSocket
   Start-Service VirtioSocketWSP
   ```

### Hardware IDs

The driver supports the following VirtIO device IDs:

- `PCI\VEN_1AF4&DEV_1012` (VirtIO 0.9.5 transitional)
- `PCI\VEN_1AF4&DEV_1053` (VirtIO 1.0+ modern)

---

## Usage

### C/C++ Programming

#### Include Headers

```c
#include <ws2def.h>
#include "vio_sockets.h"
```

#### Get VSock Address Family

```c
ADDRESS_FAMILY af = ViosockGetAF();
if (af == AF_UNSPEC) {
    // VSock not available
}
```

#### Get Local CID

```c
VIRTIO_VSOCK_CONFIG config;
if (ViosockGetConfig(&config)) {
    printf("Local CID: %u\n", config.guest_cid);
}
```

#### Create Socket

```c
SOCKET sock = socket(af, SOCK_STREAM, 0);
if (sock == INVALID_SOCKET) {
    // Handle error
}
```

#### Server Example

```c
#include <winsock2.h>
#include "vio_sockets.h"

int main() {
    WSADATA wsaData;
    WSAStartup(MAKEWORD(2, 2), &wsaData);
    
    ADDRESS_FAMILY af = ViosockGetAF();
    SOCKET listen_sock = socket(af, SOCK_STREAM, 0);
    
    SOCKADDR_VM addr = {0};
    addr.svm_family = af;
    addr.svm_cid = VMADDR_CID_ANY;
    addr.svm_port = 1234;
    
    bind(listen_sock, (SOCKADDR*)&addr, sizeof(addr));
    listen(listen_sock, SOMAXCONN);
    
    SOCKET client = accept(listen_sock, NULL, NULL);
    // Handle connection
    
    closesocket(client);
    closesocket(listen_sock);
    WSACleanup();
    return 0;
}
```

#### Client Example

```c
#include <winsock2.h>
#include "vio_sockets.h"

int main() {
    WSADATA wsaData;
    WSAStartup(MAKEWORD(2, 2), &wsaData);
    
    ADDRESS_FAMILY af = ViosockGetAF();
    SOCKET sock = socket(af, SOCK_STREAM, 0);
    
    SOCKADDR_VM addr = {0};
    addr.svm_family = af;
    addr.svm_cid = VMADDR_CID_HOST;  // Connect to host
    addr.svm_port = 1234;
    
    connect(sock, (SOCKADDR*)&addr, sizeof(addr));
    
    send(sock, "Hello", 5, 0);
    
    closesocket(sock);
    WSACleanup();
    return 0;
}
```

### Python Programming

Python 3.7+ supports AF_VSOCK on Windows when the driver is installed.

#### Server

```python
import socket

# Create VSock socket
server = socket.socket(socket.AF_VSOCK, socket.SOCK_STREAM)

# Bind to any CID, port 1234
server.bind((socket.VMADDR_CID_ANY, 1234))
server.listen(1)

# Accept connections
conn, addr = server.accept()
print(f"Connection from CID {addr[0]}")

data = conn.recv(1024)
conn.send(b"Response")
conn.close()
server.close()
```

#### Client

```python
import socket

# Create VSock socket
client = socket.socket(socket.AF_VSOCK, socket.SOCK_STREAM)

# Connect to host (CID 2), port 1234
client.connect((socket.VMADDR_CID_HOST, 1234))

client.send(b"Hello")
data = client.recv(1024)

client.close()
```

---

## API Reference

### Socket Options

#### VSock-Specific Options

- **`SO_VM_SOCKETS_BUFFER_SIZE` (0x6000)**  
  Get/set buffer size (unsigned long long)

- **`SO_VM_SOCKETS_BUFFER_MIN_SIZE` (0x6001)**  
  Get/set minimum buffer size (unsigned long long)

- **`SO_VM_SOCKETS_BUFFER_MAX_SIZE` (0x6002)**  
  Get/set maximum buffer size (unsigned long long)

- **`SO_VM_SOCKETS_CONNECT_TIMEOUT` (0x6006)**  
  Get/set connection timeout (STREAM sockets)

#### Example

```c
unsigned long long buffer_size = 65536;
setsockopt(sock, SOL_SOCKET, SO_VM_SOCKETS_BUFFER_SIZE, 
           (char*)&buffer_size, sizeof(buffer_size));
```

### Device I/O Controls

#### `IOCTL_GET_AF` (0x0801300C)

Retrieves the address family value for VSock.

```c
HANDLE hDevice = CreateFileW(L"\\??\\Viosock", 
                             GENERIC_READ, FILE_SHARE_READ,
                             NULL, OPEN_EXISTING, 
                             FILE_ATTRIBUTE_NORMAL, NULL);
DWORD af;
DWORD returned;
DeviceIoControl(hDevice, IOCTL_GET_AF, NULL, 0, 
                &af, sizeof(af), &returned, NULL);
```

#### `IOCTL_GET_CONFIG` (0x08013004)

Retrieves VSock configuration (local CID).

```c
VIRTIO_VSOCK_CONFIG config;
DeviceIoControl(hDevice, IOCTL_GET_CONFIG, NULL, 0,
                &config, sizeof(config), &returned, NULL);
```

#### `IOCTL_VM_SOCKETS_GET_LOCAL_CID` (_IO(7, 0xb9))

Alternative method to get local CID.

### Header Files

#### `vio_sockets.h`

Core definitions for VSock address family:
- `SOCKADDR_VM` structure
- Address family constants
- CID and port definitions
- Socket options
- Helper functions (`ViosockGetAF`, `ViosockGetConfig`)

#### `vio_wsk.h`

WSK provider definitions for kernel-mode consumers.

#### `install.h`

WSP installation and configuration functions.

#### `debug-utils.h`

Debug and tracing utilities.

---

## Testing

### User-Mode Tests

#### Run viosock-test.exe

```powershell
cd Install\Win10\amd64
.\viosock-test.exe
```

#### Run viosocklib-test.exe

```powershell
cd Install\Win10\amd64
.\viosocklib-test.exe
```

### Python Tests

#### Terminal 1 (Server)

```powershell
python violisten.py
```

#### Terminal 2 (Client)

```powershell
python vioconnect.py
```

### VSock TCP Bridge

#### Install vstbridge as a service

```powershell
vstbridge.exe --install
Start-Service vstbridge
```

#### SSH over VSock

From the host system:
```bash
ssh -o ProxyCommand="vstbridge.exe <vm_cid> 22" user@localhost
```

### Kernel-Mode Testing

The WSK test driver (`viosock-wsk-test`) can be loaded for kernel-mode socket testing:

```powershell
sc create viosock-wsk-test type= kernel binPath= <path>\viosock-wsk-test.sys
sc start viosock-wsk-test
```

Check debug output with tools like DebugView or WinDbg.

---

## License

The VirtIO Socket driver is licensed under the **BSD 3-Clause License**.

### Copyright

```
Copyright 2019 Virtuozzo International GmbH
Copyright 2025 Red Hat, Inc. and/or its affiliates.
```

### License Text

```
Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

1. Redistributions of source code must retain the above copyright
   notice, this list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright
   notice, this list of conditions and the following disclaimer in the
   documentation and/or other materials provided with the distribution.

3. Neither the name of the copyright holder nor the names of its
   contributors may be used to endorse or promote products derived from
   this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

---

## Additional Resources

### Related Projects

- **Linux virtio-vsock**: [Kernel Documentation](https://www.kernel.org/doc/html/latest/networking/virtio_net.html)
- **QEMU VSock**: [QEMU Documentation](https://www.qemu.org/docs/master/)
- **Libvirt SSH Proxy**: [Documentation](https://libvirt.org/ssh-proxy.html)

### References

- VirtIO Specification: [OASIS VirtIO TC](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html)
- Windows Driver Development: [Microsoft Docs](https://docs.microsoft.com/windows-hardware/drivers/)
- Winsock Service Provider: [Microsoft Docs](https://docs.microsoft.com/windows/win32/winsock/winsock-service-provider-interface-2)

---

## Troubleshooting

### Driver Not Loading

1. Check if VirtIO device is present:
   ```powershell
   Get-PnpDevice | Where-Object {$_.FriendlyName -like "*VirtIO*"}
   ```

2. Verify driver installation:
   ```powershell
   pnputil /enum-drivers | Select-String -Pattern "viosock"
   ```

3. Check event logs:
   ```powershell
   Get-WinEvent -LogName System | Where-Object {$_.ProviderName -like "*VirtIO*"}
   ```

### Service Not Starting

1. Check service status:
   ```powershell
   Get-Service VirtioSocket*, VirtioSocketWSP
   ```

2. View service dependencies:
   ```powershell
   sc qc VirtioSocket
   sc qc VirtioSocketWSP
   ```

3. Check for conflicting address families:
   ```powershell
   netsh winsock show catalog
   ```

### Connection Issues

1. Verify local CID:
   ```powershell
   viosock-test.exe --get-cid
   ```

2. Test loopback:
   ```powershell
   viosock-test.exe --loopback
   ```

3. Enable debug tracing:
   ```powershell
   tracelog -start viosock -guid viosock.guid -f viosock.etl
   # Run your test
   tracelog -stop viosock
   tracefmt viosock.etl
   ```

---

## Project Structure Summary

```
viosock/
├── sys/                    # Kernel driver (viosock.sys)
│   ├── Driver.c           # Driver entry point
│   ├── Device.c           # Device management
│   ├── Socket.c           # Socket operations
│   ├── Rx.c               # Receive path
│   ├── Tx.c               # Transmit path
│   └── Loopback.c         # Loopback support
│
├── lib/                    # User-mode library (viosocklib.dll)
│   ├── viosocklib.c       # WSP implementation
│   ├── native.c           # Native operations
│   └── install.c          # WSP installation
│
├── wsk/                    # WSK provider
│   ├── viowsk.c           # WSK implementation
│   └── provider.c         # Provider registration
│
├── wspsvc/                 # WSP service (viosockwspsvc.exe)
│   └── wspsvc.c           # Service implementation
│
├── tcp-bridge/             # TCP bridge (vstbridge.exe)
│   ├── main.cpp           # Entry point
│   ├── bridge.cpp         # Bridge logic
│   └── service.cpp        # Service support
│
├── inc/                    # Public headers
│   ├── vio_sockets.h      # Main API header
│   └── vio_wsk.h          # WSK header
│
├── viosock-test/           # User-mode test
├── viosocklib-test/        # Library test
├── viosock-wsk-test/       # WSK test driver
├── ViosockPackage/         # Driver package project
│
├── Install/                # Built driver packages
│   └── Win10/
│       ├── amd64/
│       ├── x86/
│       └── ARM64/
│
├── vioconnect.py           # Python client test
├── violisten.py            # Python server test
├── buildAll.bat            # Build script
├── cleanAll.bat            # Clean script
├── viosock.sln             # Visual Studio solution
└── LICENSE                 # BSD 3-Clause License
```

---

**Last Updated**: September 2025  
**Version**: Based on master branch analysis  
**Platforms**: Windows 10/11, Windows Server 2019/2022/2025  
**Architectures**: x86, x64, ARM64
