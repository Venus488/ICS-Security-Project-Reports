# **ICS Security Lab — Week 1 Report**

**Lab Operator:** Sebastian  
 **Hardware:** Mac Mini (Ubuntu Server) \~ CachyOS Laptop \~ Dedicated Switch  
 **Goal:** Build a simulated industrial control system to learn Modbus protocol, packet analysis, and ICS attack/defense concepts

## **What Was Built**

By end of week 1, the following was done:

* OpenPLC installed and running as a systemd service on the Mac Mini (port 8080 web UI, port 502 Modbus TCP (default))  
* Isolated network segment: Mac Mini at `192.168.50.20`, laptop at `192.168.50.10`, connected via dedicated switch with no router uplink  
* Netplan configured with both DHCP and static IP so the Mac Mini can move between home router and isolated switch without manual reconfiguration  
* pymodbus installed in isolated Python virtual environments on both machines  
* A blank PLC program loaded and running in OpenPLC  
* A synchronous ModbusTcpClient script on the laptop that reads coil 1 from OpenPLC and returns `False` (expected result from a blank program)  
* Wireshark installed and capturing on the isolated segment interface   
* Modbus TCP traffic visible in Wireshark on port 502

## **Concepts Learned/Used**

**Linux/Systems:**

* `apt` package manager: `update` refreshes package index, `install` adds packages, `remove` uninstalls. System-level requires `sudo`.  
* `git clone` vs one-line installer scripts: cloning gives you source code you can inspect before running, prolly an important habit for security-focused work.  
* `systemctl` verbs: `start`, `stop`, `status`. Services managed by systemd run independently of user sessions so closing SSH does not kill them.  
* Environment variables: set with `export KEY=value`, inherited by all child processes, no spaces around `=` it caused a headache.  
* `less` for reading files: search with `/term`, navigate with `n`, exit with `q`.  
* `cat` dumps full file output; `less` makes pages.  
* `echo $VARNAME` verifies a variable is set.  
* Bash is silent on success, no output sometimes means it worked.  
* `ip addr` lists network interfaces and their assigned addresses.  
* `usermod -aG wireshark username` gives Wireshark capture permissions without running as root. Required full logout/reboot to work.

**Python/pymodbus:**

* `pip install` on some things like python is blocked on modern Debian/Ubuntu (PEP 668)? to prevent breaking OS-managed Python dependencies.  
* Virtual environments (`python3 -m venv`) create isolated Python installations that don’t have my pc yell at me.  
* Laptop named fish shell requires `source venv/bin/activate.fish` — the standard `activate` script is bash-only and will fail in fish.  
* `ModbusTcpClient` is the synchronous Modbus TCP client class in pymodbus (vs `AsyncModbusTcpClient` for async code).  
* IP addresses must be passed as strings (in quotes) in Python, bare numbers with dots make the syntax evil.  
* `result.bits[0]` — `read_coils` always returns a list even for a single coil; `[0]` extracts the first value.

**Networking:**

* Static IPs don't auto-adapt between networks, this the tradeoff vs DHCP.  
* Netplan can be configured with both `dhcp4: true` and a static `address` DHCP is used when a DHCP server is present, the static address is always applied regardless. This allows the Mac Mini to work on both the home router and the isolated switch without reconfiguration.  
* Switches do traffic between connected devices without needing a router — two machines on the same switch can communicate over IP directly.  
* Isolated segments have no DNS resolution by default so `apt` and other tools that use hostnames will fail with "temporary failure resolving" until internet access is restored.  
* Attack surface reduction: fewer entry points means fewer things to defend. Isolation is a good thing ig.

**Modbus Protocol:**

* Modbus is an old industrial protocol, originally serial, now commonly TCP. No authentication, no encryption by design but Ai said it’s historically significant for ICS security, so I used that for babies first project.  
* Four data types: Coils (R/W, 1 bit), Discrete Inputs (read-only, 1 bit), Holding Registers (R/W, 16 bit), Input Registers (read-only, 16 bit).  
* Master/Slave (Client/Server) model: master initiates all communication, slave responds passively. A PLC is typically the slave; an HMI or script acts as master.  
* Function codes identify Modbus operations (e.g. FC1 \= Read Coils, FC3 \= Read Holding Registers). Visible as raw fields in Wireshark captures.  
* Modbus TCP rides on top of a standard TCP connection — a three-way handshake precedes any Modbus data exchange.  
* Port 502 is the IANA-registered default Modbus TCP port.  
* The modbus documentation contains most of this, I found it useful enough to keep for reference

**ICS/OT Security Concepts:**

* OpenPLC stores passwords in plaintext in `webserver/openplc.db` (SQLite). An attacker with file read access gets credentials just from that. Contrast that with proper password hashing.  
* Default credentials (`openplc`/`openplc`) not smart irl, but im at home so idc.  
* Segmented OT networks exists in the real world, so money was well spent, because mixing ICS protocols with home/IT networks is prolly what good architecture avoids.  
* "Reducing attack surface" is the underlying principle behind network isolation, least privilege, and minimizing open services.

**OpenPLC Internals:**

* OpenPLC uses `bison`/`flex` to build its own IEC 61131-3 compiler (matiec) — program upload isn't a file copy, it's a compile-then-run operation. (this is scary voodoo)  
* OpenPLC's web interface is Python-based (Flask/Werkzeug stack), using SQLite for state storage.  
* `webserver/openplc.db` contains the Users table with credentials, Programs table, Settings, and Slave\_dev entries, or just check users on the browser client if you have a brain.  
* OpenPLC supports multiple protocols simultaneously: Modbus TCP (502), EtherNet/IP (44818), DNP3 (optional, I failed to build — see below).

## **Hurdles Cleared**

### **1\. `git` not installed on fresh Ubuntu Server**

**What broke:** `git clone` failed with `command not found`.  
 **Why:** Ubuntu Server was a minimal install that excludes non-essential packages.  
 **Fix:** `sudo apt update && sudo apt install git`

### **2\. OpenPLC install failed — CMake 4.2.3 / OpenDNP3 incompatibility**

**What broke:** `./install.sh linux` failed with `CMake Error at CMakeLists.txt:1 (cmake_minimum_required): Compatibility with CMake < 3.5 has been removed`.  
 **Why:** Ubuntu's repos shipped CMake 4.2.3, which dropped backward-compatibility shims for old `cmake_minimum_required()` declarations. OpenPLC's vendored OpenDNP3 dependency has a stale `CMakeLists.txt` that hasn't been updated to declare modern CMake compatibility.  
 **Fix:** `export CMAKE_POLICY_VERSION_MINIMUM=3.5` before re-running the installer. CMake 4.0+ supports this as both a `-D` cache variable and an environment variable, blessing me with not having to write code for it.  
 **Note:** I need to file as a GitHub Issue on the OpenPLC\_v3 repo. Be a good boy cuz others will hit the same villainy as CMake 4.x becomes standard.

### **3\. Default OpenPLC credentials unknown**

**What broke:** `admin`/`admin` (from a Google search) did not work on the web login.  
 **Fix:** asking the SQLite database directly: `sqlite3 webserver/openplc.db` → `SELECT * FROM Users;`  credentials are `openplc`/`openplc`. 

### **4\. Mac Mini lost internet access on isolated switch**

**What broke:** `sudo apt install python3-pip` failed with `Temporary failure resolving 'us.archive.ubuntu.com'` — no DNS on the isolated segment, no route to the internet.  
 **Fix:** Temporarily reconnected Mac Mini to home router via ethernet to install packages, then returned to isolated segment. Long-term: get all of you need before switching to the switch box.

### **5\. Netplan static IP does not change automatically (duh.)**

**What broke:** Moving the Mac Mini between the isolated switch and home router required manual netplan edits each time since the static `192.168.50.20` address was incompatible with the home network's DHCP range.  
 **Fix:** Edited `/etc/netplan/00-installer-config.yaml` to set `dhcp4: true` while keeping the static `addresses: [192.168.50.20/24]` entry. Both are applied simultaneously: DHCP succeeds on the home network, static address is always present for the isolated segment. Applied with `sudo netplan apply` as it didn't seem to work after saving in nano.

### **6\. `pip install pymodbus` blocked on both machines**

**What broke:** `sudo pip install pymodbus` returned `externally-managed-environment` error (PEP 668).  
 **Why:** Modern Debian/Ubuntu locks system Python to prevent pip from breaking OS-managed dependencies.  
 **Fix:** Created virtual environments (`python3 -m venv venv`). Fish shell on the laptop required `source venv/bin/activate.fish` instead of the standard bash `activate` script.

### **7\. Wireshark capture permission denied**

**What broke:** Wireshark launched but could not capture on any interface.  
 **Fix:** `sudo usermod -aG wireshark sebastian` followed by a reboot. Group membership changes require a new login session to take effect.

### **8\. `read_coils()` argument error**

**What broke:** `TypeError: ModbusClientMixin.read_coils() takes 2 positional arguments but 3 were given`  
 **Why:** pymodbus must’ve changed the function so that being `count` is no longer a standard argument in the way the docs led me to believe.  
 **Fix:** Adjusted the call to use keyword arguments or drop the count argument. Script returned `False` — correct response from a blank OpenPLC program.

---

## **Current Script (`modbus_read.py`)**

python  
from pymodbus.client import ModbusTcpClient

client \= ModbusTcpClient('192.168.50.20')  
client.connect()  
result \= client.read\_coils(1)  
print(result.bits\[0\])  
client.close()

Lives at `~/ics-lab/modbus_read.py` on the CachyOS laptop. Run from inside the activated venv.

## **Lab Network State**

| Device |  | IP | Role |
| ----- | :---- | ----- | ----- |
| Mac Mini |  | `192.168.50.20` | OpenPLC (PLC/Slave) |
| CachyOS Laptop |  | `192.168.50.10` | Attack/Analyst (Master) |
| Both connected via dedicated switch, no router uplink on isolated segment |  |  |  |

## **Parked Tangents (Revisit Later)**

1. **Understanding the netplan config** — the DHCP+static config was AI assisted during initial setup. It's worth going back and trying to understand each field.  
2. **Filing a GitHub Issue** — the CMake 4.2.3 / OpenDNP3 breakage should be documented on the OpenPLC\_v3 repo for the fellas. The fix (`CMAKE_POLICY_VERSION_MINIMUM=3.5`) is worth sharing.

## **Week 2 Direction**

* Load a minimal IEC 61131-3 program into OpenPLC (a single always-ON coil) so Modbus registers have a known, non-zero value to read back  
* Re-run `modbus_read.py` against the new program and confirm `True` is returned  
* Examine the Wireshark capture: identify the TCP handshake, the Modbus request packet (function code, address), and the response — read and explain it, not just observe that it exists  
* Begin thinking about write operations and what it would mean to send a command to OpenPLC rather than just read from it

