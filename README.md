# Enterprise Active Directory Lab: Deployment, Networking Remediation, and Group Policy Enforcement

## Project Overview
This project documents the end-to-end deployment, network troubleshooting, and security hardening of a Windows Server Active Directory lab built inside Oracle VirtualBox. The objective was to configure a fully functional domain controller, resolve cross-VM routing and DNS issues over a physical wireless bridge, join a Windows 10 workstation to the domain, provision non-privileged user accounts, and enforce system restrictions through Group Policy Objects (GPOs).

---

## Technical Specifications & Architecture

| Component | Operating System | Hostname | IP Address | Network Role |
| :--- | :--- | :--- | :--- | :--- |
| **Domain Controller** | Windows Server 2022 | `DC01` | `192.168.1.200` (Static) | Primary DC, AD DS, DNS Server (`cyberlab.local`) |
| **Client Workstation** | Windows 10 Enterprise | `DESKTOP-BRE6DPH` | `192.168.1.201` (Static) | Domain Endpoint |
| **Virtual Switch** | VirtualBox v7.x | N/A | Bridged Adapter | Promiscuous Mode: *Allow All* |
| **Default Gateway** | Physical Wi-Fi Router | N/A | `192.168.1.1` | Local Gateway / External Route |

---

## Phase 1: Network Architecture & Static IP Configuration

### 1. Migrating from NAT to Bridged Networking
By default, VirtualBox isolates each virtual machine inside a private sandbox using standard NAT (`10.0.2.x`). While this provides outbound internet access, it prevents direct VM-to-VM communication. 

To allow the Windows 10 endpoint to reach the Domain Controller:
* Both virtual machines were switched to **Bridged Adapter**, bound directly to the host's physical network adapter.
* **Promiscuous Mode** was set to **Allow All** under VirtualBox Advanced Network Settings to prevent the wireless interface from dropping cross-VM traffic.

### 2. Server Static IP Allocation
A Domain Controller requires an unchanging IP address to handle directory requests and DNS queries. The primary network adapter was statically assigned:

* **IP Address:** `192.168.1.200`
* **Subnet Mask:** `255.255.255.0`
* **Default Gateway:** `192.168.1.1`
* **Preferred DNS Server:** `127.0.0.1` (Points to internal Active Directory DNS)
* **Alternate DNS Server:** `8.8.8.8` (Public DNS fallback)

<!-- Drag and drop 3.png here -->
![Adapter Properties]<img width="1039" height="837" alt="3" src="https://github.com/user-attachments/assets/7f1f1560-7c2d-4b2e-ac2a-1578515891f6" />
*Figure 1.1: Ethernet adapter configuration panel on Windows Server 2022.*

<!-- Drag and drop 4.png here -->
![Static IPv4 Configuration]<img width="1031" height="770" alt="4" src="https://github.com/user-attachments/assets/6e2760ad-e123-4d2a-857d-d7db1c87e1b4" />

*Figure 1.2: Static IP, Gateway, and DNS settings on the Domain Controller.*

<!-- Drag and drop 2.png here -->
![Network Connection Status]<img width="1021" height="830" alt="2" src="https://github.com/user-attachments/assets/979cf622-086f-4a2b-b9d0-ce95c75feb86" />

*Figure 1.3: Active interface verifying domain association with cyberlab.local.*

---

## Phase 2: DNS Troubleshooting & Resolution

### 1. The IPv6 Conflict
After assigning static IP `192.168.1.201` to the Windows 10 client, ICMP connectivity tests (`ping 192.168.1.200`) succeeded. However, testing domain lookup using `nslookup cyberlab.local` failed with a `Non-existent domain` error.

Inspection showed that the client machine automatically queried an IPv6 DNS server (`fe80::1213:31ff:fe1c:1b2`) handed out by the physical Wi-Fi router. Because the home router has no records for the lab's local Active Directory namespace, name resolution failed.

### 2. Resolution Steps
1. Opened the client adapter settings (`ncpa.cpl`) and disabled **Internet Protocol Version 6 (TCP/IPv6)**.
2. Verified the **Preferred DNS** was pointed strictly to the Domain Controller (`192.168.1.200`).
3. Cleared local resolver cache:
   ```cmd
   ipconfig /flushdns


 4. Retested with `nslookup cyberlab.local`, which successfully resolved the domain record to `192.168.1.200`.

---

## Phase 3: Joining the Workstation to the Active Directory Domain

With name resolution verified, the workstation was joined to the domain via the Windows System Properties interface:

1. Executed `sysdm.cpl`, navigated to **Computer Name**, and selected **Change...**.
2. Switched membership from `WORKGROUP` to Domain: `cyberlab.local`.
3. Authenticated using domain administrative credentials (`CYBERLAB\Administrator`).
4. Received confirmation dialog: *"Welcome to the cyberlab.local domain"*.

<img width="1022" height="849" alt="1" src="https://github.com/user-attachments/assets/18a4f032-f041-4a3e-90b4-64586401542e" />



*Figure 3.1: Workstation registered to the cyberlab.local domain.*

The client was restarted to establish the domain security trust relationship and initialize machine Kerberos tickets.

---

## Phase 4: User Provisioning in Active Directory

To validate user-tier access controls and simulate a corporate environment, a dedicated non-administrative user account was provisioned in directory services.

### 1. Account Creation
1. Launched **Active Directory Users and Computers** (`dsa.msc`) on `DC01`.
2. Expanded the domain partition `cyberlab.local` and selected the **Users** container.
3. Initialized the **New Object - User** wizard:
   * **Full Name:** `lab user`
   * **User Logon Name:** `labuser@cyberlab.local` / `CYBERLAB\labuser`
4. Set a compliant password with *Password never expires* checked to support continuous lab testing.

<img width="885" height="865" alt="5" src="https://github.com/user-attachments/assets/8426b42a-3c71-4495-805c-ec1423fad4e6" />



*Figure 4.1: Active Directory Users and Computers console hierarchy.*

<img width="1022" height="814" alt="6" src="https://github.com/user-attachments/assets/5aea5c68-f7a5-465a-87b8-b9db782404ad" />



*Figure 4.2: Initializing the User creation wizard.*

<img width="901" height="798" alt="7" src="https://github.com/user-attachments/assets/f1c043b2-9325-4816-8c88-dab11f6b418a" />



*Figure 4.3: Identity mapping and logon credentials defined for labuser.*

### 2. Client Authentication Validation
On the Windows 10 client, the existing session was signed out, and the workstation was unlocked using the newly provisioned domain account:

<img width="1006" height="760" alt="8" src="https://github.com/user-attachments/assets/a51e06bf-d73a-4506-8a04-0ccfab3d19cb" />



*Figure 4.4: Authenticating as CYBERLAB\labuser on the client lock screen.*

Running `whoami` verified the user was operating under standard user rights without membership in the local Administrators group.

---

## Phase 5: Group Policy Hardening & Endpoint Enforcement

To simulate enterprise system hardening, a Group Policy Object was deployed to restrict critical administrative settings for standard users.

### 1. GPO Configuration
1. Opened the **Group Policy Management Console** (`gpmc.msc`) on the Domain Controller.
2. Created and linked a new Group Policy Object named **`Block-ControlPanel`** to the domain root.
3. Configured the administrative template:
   * **Path:** `User Configuration` > `Policies` > `Administrative Templates` > `Control Panel`
   * **Policy:** **Prohibit access to Control Panel and PC settings**
   * **State:** Set to **Enabled**

<img width="990" height="779" alt="9" src="https://github.com/user-attachments/assets/264c0245-9393-4386-82cd-1fb5c1feb7a0" />



*Figure 5.1: Creating and linking the GPO in Group Policy Management.*

<img width="976" height="773" alt="10" src="https://github.com/user-attachments/assets/9eccc47c-5d95-483a-857b-183b97b37e61" />



*Figure 5.2: Enforcing the restriction policy within the Group Policy Management Editor.*

### 2. Client-Side Policy Convergence & Verification
To force immediate policy processing on the Windows 10 workstation without waiting for the background refresh cycle:

1. Opened Command Prompt as `labuser`.
2. Forced policy retrieval:
   ```cmd
   gpupdate /force


<img width="1030" height="877" alt="11" src="https://github.com/user-attachments/assets/72ba1520-98ca-435e-af24-68449281ad1e" />



*Figure 5.3: Successful policy update on the Windows 10 endpoint.*

3. Executed `control` via the Run prompt and attempted to open Settings (`Win + I`). The operating system immediately intercepted the request and displayed an access restriction dialog:

<img width="1030" height="797" alt="12" src="https://github.com/user-attachments/assets/0cc924f6-8c4d-4c16-b6fb-392378a929c9" />



*Figure 5.4: Access restriction enforced by Group Policy.*

---

