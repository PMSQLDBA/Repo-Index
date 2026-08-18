Yes. For your lab, I would split it into **two scripts**: one for `pmlabs.com` and one for `dblabs.com`.

One correction first: Microsoft documents `Add-DnsServerConditionalForwarderZone` for scripting the DNS side. For creating a **forest trust**, `netdom trust` is not suitable; Microsoft specifically notes that `netdom trust` cannot create a forest trust between two AD DS forests. ([Microsoft Learn][1])

So the clean approach is: automate DNS and validation with PowerShell, then either create the forest trust with the AD GUI, or use .NET Active Directory APIs. Below is a PowerShell version that automates both sides using the .NET `System.DirectoryServices.ActiveDirectory` classes.

# PowerShell Forest Trust Setup

Environment:

```text
pmlabs.com  = 192.168.0.100
dblabs.com  = 192.168.0.10
```

Run PowerShell **as Administrator** on both domain controllers.

---

# Script 1 — Run on pmlabs.com

Run this on:

```text
Domain: pmlabs.com
DC/DNS: 192.168.0.100
```

```powershell
# ============================================================
# PMLABS.COM - DNS + Connectivity + Forest Trust Preparation
# ============================================================

$LocalDomain  = "pmlabs.com"
$RemoteDomain = "dblabs.com"
$LocalDNS     = "192.168.0.100"
$RemoteDNS    = "192.168.0.10"

Write-Host "============================================="
Write-Host "PMLABS.COM Trust Configuration"
Write-Host "============================================="

# ------------------------------------------------------------
# 1. Check required Windows modules
# ------------------------------------------------------------

Write-Host "`n[1] Checking PowerShell modules..."

Import-Module DnsServer -ErrorAction Stop
Import-Module ActiveDirectory -ErrorAction Stop

Write-Host "DnsServer and ActiveDirectory modules loaded."

# ------------------------------------------------------------
# 2. Test network connectivity to DBLABS
# ------------------------------------------------------------

Write-Host "`n[2] Testing connectivity to DBLABS..."

$Ports = @(53,88,135,389,445)

foreach ($Port in $Ports) {

    $Result = Test-NetConnection `
        -ComputerName $RemoteDNS `
        -Port $Port `
        -WarningAction SilentlyContinue

    if ($Result.TcpTestSucceeded) {
        Write-Host "Port $Port : OK"
    }
    else {
        Write-Warning "Port $Port : FAILED"
    }
}

# ------------------------------------------------------------
# 3. Create Conditional DNS Forwarder
# ------------------------------------------------------------

Write-Host "`n[3] Configuring DNS conditional forwarder..."

$ExistingForwarder = Get-DnsServerZone `
    -Name $RemoteDomain `
    -ErrorAction SilentlyContinue

if ($ExistingForwarder) {

    Write-Host "Conditional forwarder already exists for $RemoteDomain"

}
else {

    Add-DnsServerConditionalForwarderZone `
        -Name $RemoteDomain `
        -MasterServers $RemoteDNS `
        -ReplicationScope "Forest"

    Write-Host "Conditional forwarder created:"
    Write-Host "$RemoteDomain -> $RemoteDNS"
}

# ------------------------------------------------------------
# 4. Clear DNS cache
# ------------------------------------------------------------

Write-Host "`n[4] Clearing DNS cache..."

Clear-DnsServerCache -Force
Clear-DnsClientCache

# ------------------------------------------------------------
# 5. Test DNS resolution
# ------------------------------------------------------------

Write-Host "`n[5] Testing DNS resolution..."

Resolve-DnsName $RemoteDomain -ErrorAction Continue

# Check AD SRV records
Write-Host "`nTesting remote AD SRV records..."

Resolve-DnsName `
    "_ldap._tcp.dc._msdcs.$RemoteDomain" `
    -Type SRV `
    -ErrorAction Continue

# ------------------------------------------------------------
# 6. Test remote Domain Controller discovery
# ------------------------------------------------------------

Write-Host "`n[6] Testing DBLABS Domain Controller discovery..."

nltest /dsgetdc:$RemoteDomain

# ------------------------------------------------------------
# 7. Display existing trusts
# ------------------------------------------------------------

Write-Host "`n[7] Current trusts visible from PMLABS..."

Get-ADTrust -Filter * |
    Format-Table Name,Direction,TrustType,ForestTransitive -AutoSize

Write-Host "`nPMLABS DNS preparation completed."
Write-Host "Do not create the trust until the DBLABS script is also completed."
```

Microsoft supports creating AD-integrated conditional forwarders using `Add-DnsServerConditionalForwarderZone`, including forest-wide replication. ([Microsoft Learn][2])

---

# Script 2 — Run on dblabs.com

Run this on:

```text
Domain: dblabs.com
DC/DNS: 192.168.0.10
```

```powershell
# ============================================================
# DBLABS.COM - DNS + Connectivity + Forest Trust Preparation
# ============================================================

$LocalDomain  = "dblabs.com"
$RemoteDomain = "pmlabs.com"
$LocalDNS     = "192.168.0.10"
$RemoteDNS    = "192.168.0.100"

Write-Host "============================================="
Write-Host "DBLABS.COM Trust Configuration"
Write-Host "============================================="

# ------------------------------------------------------------
# 1. Check required Windows modules
# ------------------------------------------------------------

Write-Host "`n[1] Checking PowerShell modules..."

Import-Module DnsServer -ErrorAction Stop
Import-Module ActiveDirectory -ErrorAction Stop

Write-Host "DnsServer and ActiveDirectory modules loaded."

# ------------------------------------------------------------
# 2. Test network connectivity to PMLABS
# ------------------------------------------------------------

Write-Host "`n[2] Testing connectivity to PMLABS..."

$Ports = @(53,88,135,389,445)

foreach ($Port in $Ports) {

    $Result = Test-NetConnection `
        -ComputerName $RemoteDNS `
        -Port $Port `
        -WarningAction SilentlyContinue

    if ($Result.TcpTestSucceeded) {
        Write-Host "Port $Port : OK"
    }
    else {
        Write-Warning "Port $Port : FAILED"
    }
}

# ------------------------------------------------------------
# 3. Create Conditional DNS Forwarder
# ------------------------------------------------------------

Write-Host "`n[3] Configuring DNS conditional forwarder..."

$ExistingForwarder = Get-DnsServerZone `
    -Name $RemoteDomain `
    -ErrorAction SilentlyContinue

if ($ExistingForwarder) {

    Write-Host "Conditional forwarder already exists for $RemoteDomain"

}
else {

    Add-DnsServerConditionalForwarderZone `
        -Name $RemoteDomain `
        -MasterServers $RemoteDNS `
        -ReplicationScope "Forest"

    Write-Host "Conditional forwarder created:"
    Write-Host "$RemoteDomain -> $RemoteDNS"
}

# ------------------------------------------------------------
# 4. Clear DNS caches
# ------------------------------------------------------------

Write-Host "`n[4] Clearing DNS cache..."

Clear-DnsServerCache -Force
Clear-DnsClientCache

# ------------------------------------------------------------
# 5. Test DNS resolution
# ------------------------------------------------------------

Write-Host "`n[5] Testing DNS resolution..."

Resolve-DnsName $RemoteDomain -ErrorAction Continue

Write-Host "`nTesting remote AD SRV records..."

Resolve-DnsName `
    "_ldap._tcp.dc._msdcs.$RemoteDomain" `
    -Type SRV `
    -ErrorAction Continue

# ------------------------------------------------------------
# 6. Test remote Domain Controller discovery
# ------------------------------------------------------------

Write-Host "`n[6] Testing PMLABS Domain Controller discovery..."

nltest /dsgetdc:$RemoteDomain

# ------------------------------------------------------------
# 7. Display existing trusts
# ------------------------------------------------------------

Write-Host "`n[7] Current trusts visible from DBLABS..."

Get-ADTrust -Filter * |
    Format-Table Name,Direction,TrustType,ForestTransitive -AutoSize

Write-Host "`nDBLABS DNS preparation completed."
```

---

# Script 3 — Create the Two-Way Forest Trust

After **Scripts 1 and 2 succeed**, run this script on the `pmlabs.com` Domain Controller.

It will ask for credentials from `dblabs.com`.

```powershell
# ============================================================
# CREATE TWO-WAY FOREST TRUST
# Run on PMLABS.COM
# ============================================================

$LocalForest  = "pmlabs.com"
$RemoteForest = "dblabs.com"

Write-Host "============================================="
Write-Host "Creating Forest Trust"
Write-Host "$LocalForest <--> $RemoteForest"
Write-Host "============================================="

# Ask for DBLABS administrator credentials
$RemoteCredential = Get-Credential `
    -Message "Enter DBLABS administrator credentials, for example Administrator@dblabs.com"

# Extract username and password
$RemoteUser = $RemoteCredential.UserName
$RemotePassword = $RemoteCredential.GetNetworkCredential().Password

# Load .NET AD assembly
Add-Type -AssemblyName System.DirectoryServices

# Get local forest
$LocalForestObject =
    [System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest()

Write-Host "Local forest detected:"
Write-Host $LocalForestObject.Name

# Create both sides of the trust
Write-Host "`nCreating two-way forest trust..."

$LocalForestObject.CreateTrustRelationship(
    $RemoteForest,
    [System.DirectoryServices.ActiveDirectory.TrustDirection]::Bidirectional,
    $RemoteUser,
    $RemotePassword
)

Write-Host "`nTrust creation command completed."
```

If your Windows/.NET version does not expose the exact overload required for `CreateTrustRelationship`, use the GUI for this single step instead. The DNS scripts and verification scripts remain valid.

---

# Script 4 — Verify Trust from pmlabs.com

Run on:

```text
192.168.0.100
pmlabs.com
```

```powershell
$RemoteDomain = "dblabs.com"

Write-Host "============================================="
Write-Host "PMLABS Trust Verification"
Write-Host "============================================="

Write-Host "`nDNS Test:"
Resolve-DnsName $RemoteDomain

Write-Host "`nDomain Controller Discovery:"
nltest /dsgetdc:$RemoteDomain

Write-Host "`nConfigured AD Trust:"
Get-ADTrust -Identity $RemoteDomain |
    Format-List *

Write-Host "`nAll Trusts:"
Get-ADTrust -Filter * |
    Format-Table Name,Direction,TrustType,ForestTransitive -AutoSize

Write-Host "`nNLTEST Trust Information:"
nltest /domain_trusts
```

`Get-ADTrust` is Microsoft's supported Active Directory PowerShell cmdlet for retrieving trust objects. ([Microsoft Learn][3])

Expected result should look roughly like:

```text
Name              : dblabs.com
Direction         : BiDirectional
ForestTransitive  : True
```

---

# Script 5 — Verify Trust from dblabs.com

Run on:

```text
192.168.0.10
dblabs.com
```

```powershell
$RemoteDomain = "pmlabs.com"

Write-Host "============================================="
Write-Host "DBLABS Trust Verification"
Write-Host "============================================="

Write-Host "`nDNS Test:"
Resolve-DnsName $RemoteDomain

Write-Host "`nDomain Controller Discovery:"
nltest /dsgetdc:$RemoteDomain

Write-Host "`nConfigured AD Trust:"
Get-ADTrust -Identity $RemoteDomain |
    Format-List *

Write-Host "`nAll Trusts:"
Get-ADTrust -Filter * |
    Format-Table Name,Direction,TrustType,ForestTransitive -AutoSize

Write-Host "`nNLTEST Trust Information:"
nltest /domain_trusts
```

---

# Expected Final Configuration

On `pmlabs.com`:

```text
DNS Conditional Forwarder

dblabs.com
    |
    +--> 192.168.0.10
```

On `dblabs.com`:

```text
DNS Conditional Forwarder

pmlabs.com
    |
    +--> 192.168.0.100
```

Trust:

```text
                 Two-Way Forest Trust

pmlabs.com  <============================>  dblabs.com
192.168.0.100                               192.168.0.10

     PMLABS users                    DBLABS users
           |                              |
           +-------- Authentication ------+
```

---

# Recommended Execution Order

```text
1. PMLABS
   Run Script 1

          ↓

2. DBLABS
   Run Script 2

          ↓

3. Confirm from PMLABS:
   nltest /dsgetdc:dblabs.com

          ↓

4. Confirm from DBLABS:
   nltest /dsgetdc:pmlabs.com

          ↓

5. PMLABS
   Run Script 3 / create forest trust

          ↓

6. PMLABS
   Run Script 4

          ↓

7. DBLABS
   Run Script 5

          ↓

8. Test actual user/resource access
```

Do **not** continue to trust creation if either of these fails:

```powershell
nltest /dsgetdc:dblabs.com
```

from PMLABS, or:

```powershell
nltest /dsgetdc:pmlabs.com
```

from DBLABS. A working trust depends on reliable DNS/DC discovery between the forests.

One detail worth checking before running Script 3: the servers should resolve the remote forest's **AD SRV records**, not merely the domain's A record. That is why I included `_ldap._tcp.dc._msdcs.<domain>` testing. DNS conditional forwarding is the Microsoft-supported way to send queries for the other namespace to its DNS server. ([Microsoft Learn][4])

[1]: https://learn.microsoft.com/en-us/powershell/module/dnsserver/add-dnsserverconditionalforwarderzone?view=windowsserver2025-ps&utm_source=chatgpt.com "Add-DnsServerConditionalForwarderZone (DnsServer)"
[2]: https://learn.microsoft.com/en-us/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-configure-dns-forwarding-and-trust?utm_source=chatgpt.com "Configure DNS forwarding and domain trust"
[3]: https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-adtrust?view=windowsserver2025-ps&utm_source=chatgpt.com "Get-ADTrust (ActiveDirectory)"
[4]: https://learn.microsoft.com/en-us/windows-server/networking/dns/forwarding?utm_source=chatgpt.com "DNS Forwarding in Windows Server"
