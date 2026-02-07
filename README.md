# azure-
Hub-and-Spoke Architecture
# Hub-and-Spoke Lab 

![azp.JPG](attachment:df5779e0-f2f5-45d6-ac71-f225b02dfe9f:azp.jpg)

## 1) Create a Resource Group

- In the Azure Portal search bar, type **Resource groups**
- Click **Create**
- **Name:** `RG-HubSpoke-Lab`
- **Region:** Use the same region for all resources
- Click **Review + create** → **Create**

---

## 2) Create the Hub VNet (Central Network)

- Search for **Virtual networks**
- Click **Create**

### Basics

- **Subscription:** Select your subscription
- **Resource group:** `RG-HubSpoke-Lab`
- **Name:** `VNET-HUB`
- **Region:** Same region

### IP Addresses

- **IPv4 address space:** `10.0.0.0/24`
- **Subnets → Add subnet**
    - **Subnet name:** `HUB-SUBNET`
    - **Subnet range:** `10.0.0.0/26`
- Click **Review + create** → **Create**

---

## 3) Create the Jump VM inside the Hub (Only VM with Public IP)

- Search for **Virtual machines**
- Click **Create → Azure virtual machine**

### Basics

- **Resource group:** `RG-HubSpoke-Lab`
- **VM name:** `VM-JUMP`
- **Region:** Same region
- **Image:** Windows Server (or Windows 11 if preferred)
- **Size:** Any cost-effective size
- **Username / Password:** Create credentials

### Networking

- **Virtual network:** `VNET-HUB`
- **Subnet:** `HUB-SUBNET`
- **Public IP:** Create new (Enabled)
- **NIC network security group:** Basic
- **Inbound ports:** Allow selected ports
- **Selected inbound ports:** RDP (3389)
- Click **Review + create** → **Create**

---

### Secure RDP Access (Important)

After deployment:

- Go to **VM-JUMP → Networking**
- Edit the existing RDP rule or create a new one:
    - **Source:** IP Addresses
    - **Source IP:** Your public IP only
    - **Destination port:** 3389
    - **Action:** Allow
- **Delete or disable** any rule that allows **Any → 3389**

---

## 4) Create Spoke VNet 1 (LAN1)

- **Virtual networks → Create**
- **Name:** `VNET-LAN1`
- **IP address space:** `192.168.1.0/24`

### Subnets

- `WEB-SUBNET` → `192.168.1.0/26`
- `FILE-SUBNET` → `192.168.1.64/26`
- Click **Create**

---

## 5) Create Spoke VNet 2 (LAN2)

- **Virtual networks → Create**
- **Name:** `VNET-LAN2`
- **IP address space:** `192.168.2.0/24`

### Subnet

- `WORKSTATION-SUBNET` → `192.168.2.0/26`
- Click **Create**

---

## 6) Create VMs in LAN1 and LAN2 (No Public IP)

### A) Web Server VM

- **Virtual machines → Create**
- **Name:** `VM-WEB`
- **VNet:** `VNET-LAN1`
- **Subnet:** `WEB-SUBNET`
- **Public IP:** None
- **Inbound ports:** None
- **Create**

### B) File Server VM

- Same steps
- **Name:** `VM-FILE`
- **Subnet:** `FILE-SUBNET`
- **Public IP:** None

### C) Workstation VM (LAN2)

- **Name:** `VM-WS1`
- **VNet:** `VNET-LAN2`
- **Subnet:** `WORKSTATION-SUBNET`
- **Public IP:** None

---

## 7) Configure VNet Peering (Hub ↔ LAN1 and Hub ↔ LAN2)

### Hub ↔ LAN1

- Open **VNET-HUB**
- Go to **Peerings → Add**
- **Peering name:** `HUB-to-LAN`
- **Remote VNet:** `VNET-LAN1`
- Enable:
    - Allow virtual network access
    - Allow forwarded traffic
- Click **Add**

Verify:

- Open `VNET-LAN1 → Peerings`
- Ensure `LAN1-to-HUB` is **Connected**

### Repeat the same steps for LAN2

- `HUB-to-LAN2`
- `LAN2-to-HUB`

---

## 8) NSG Rules (Network Isolation and Security)

### Goal

- LAN1 and LAN2 allow access **only from the Hub**
- Block all other inbound traffic

### 8.1 Create NSG for LAN1

- Search **Network security groups → Create**
- **Name:** `NSG-LAN1`
- Create

### Inbound Rule

- **Source:** IP Addresses
- **Source IP:** `10.0.0.0/24`
- **Destination:** Any
- **Service:** RDP (or Any depending on lab)
- **Action:** Allow
- **Priority:** 100

(Optional) Add Deny rule:

- **Source:** Any
- **Destination:** Any
- **Service:** Any
- **Action:** Deny
- **Priority:** 200

Attach NSG:

- `VNET-LAN1 → Subnets`
- Attach `NSG-LAN1` to:
    - `WEB-SUBNET`
    - `FILE-SUBNET`

---

### 8.2 NSG for LAN2

- Create `NSG-LAN2` using the same logic
- Attach it to `WORKSTATION-SUBNET`

---

## 9) Connectivity Testing

### 9.1 Connect to Jump VM

- Go to `VM-JUMP → Connect → RDP`
- Download and open the RDP file
- Sign in

### 9.2 Test from Jump VM

- Find Private IPs of:
    - `VM-WEB`
    - `VM-FILE`
    - `VM-WS1`

Test connectivity:

- Ping (may fail due to Windows Firewall)

---

## 10) Enable Monitoring (Log Analytics + Azure Monitor)

### Create Log Analytics Workspace

- Search **Log Analytics workspaces**
- Click **Create**
- **Name:** `LAW-HubSpoke`
- **Resource Group:** `RG-HubSpoke-Lab`
- **Region:** Same region
- **Create**

### Connect VMs to Workspace

For each VM:

- Open VM → **Monitoring → Insights** or **Diagnostic settings**
- Enable monitoring
- Select **LAW-HubSpoke**
- Enable log collection

---

## 11) (Optional – Advanced) Forced Tunneling / Route Table

If you want all outbound traffic to go through the Hub:

- Search **Route tables → Create**
- **Name:** `RT-LANs-to-HUB`

### Add Route

- **Address prefix:** `0.0.0.0/0`
- **Next hop type:** Virtual appliance
- **Next hop address:** Firewall/NVA private IP in Hub
