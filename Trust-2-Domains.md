Assuming `pmlabs.com` and `dblabs.com` are **two separate Active Directory forests** and you want a **two-way trust**, do the work in this order.

## 1. Work on `pmlabs.com`

First, log in to a Domain Controller in `pmlabs.com` with Domain Admin/Enterprise Admin permissions.

### Step 1 — Check connectivity to the DBLabs DC

From the PMLabs DC:

```powershell
ping <DBLABS-DC-IP>
```

Example:

```powershell
ping 10.20.0.10
```

Also test the DNS/DC hostname if DNS is already partially configured:

```powershell
ping dc01.dblabs.com
```

If the name does not resolve yet, that is expected until the DNS forwarder is configured.

### Step 2 — Create a DNS Conditional Forwarder

Open:

**Server Manager → Tools → DNS**

Then:

**DNS Server → Conditional Forwarders → right-click → New Conditional Forwarder**

Enter:

```text
DNS Domain: dblabs.com
IP Address: <DBLABS DNS/DC IP>
```

For example:

```text
dblabs.com
10.20.0.10
```

If the option is available, select:

```text
Store this conditional forwarder in Active Directory
```

Choose the replication scope appropriate for your environment, normally all DNS servers in the forest or domain.

Click **OK**.

### Step 3 — Verify DNS from PMLabs

Run:

```powershell
nslookup dblabs.com
```

Then verify the DBLabs DC:

```powershell
nslookup dc01.dblabs.com
```

Also verify AD service records:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.dblabs.com
```

You should receive one or more DBLabs Domain Controllers.

Test DC discovery:

```powershell
nltest /dsgetdc:dblabs.com
```

If this command succeeds, DNS and basic AD discovery are working.

---

## 2. Work on `dblabs.com`

Now log in to a Domain Controller in `dblabs.com`.

### Step 4 — Check connectivity to the PMLabs DC

Run:

```powershell
ping <PMLABS-DC-IP>
```

Example:

```powershell
ping 10.10.0.10
```

### Step 5 — Create the reverse DNS Conditional Forwarder

Open:

**Server Manager → Tools → DNS**

Then:

**Conditional Forwarders → New Conditional Forwarder**

Configure:

```text
DNS Domain: pmlabs.com
IP Address: <PMLABS DNS/DC IP>
```

Example:

```text
pmlabs.com
10.10.0.10
```

Select:

```text
Store this conditional forwarder in Active Directory
```

Then click **OK**.

### Step 6 — Verify DNS from DBLabs

Run:

```powershell
nslookup pmlabs.com
```

Then:

```powershell
nslookup dc01.pmlabs.com
```

Verify AD records:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.pmlabs.com
```

And:

```powershell
nltest /dsgetdc:pmlabs.com
```

At this stage the important requirement is:

```text
PMLabs can resolve DBLabs
AND
DBLabs can resolve PMLabs
```

---

# 3. Verify time synchronization

Kerberos authentication is very sensitive to clock differences.

On the `pmlabs.com` DC:

```powershell
w32tm /query /status
```

On the `dblabs.com` DC:

```powershell
w32tm /query /status
```

The clocks should be closely synchronized.

You can also check:

```powershell
Get-Date
```

on both systems.

---

# 4. Check firewall/network ports

The two environments must allow Active Directory traffic between the Domain Controllers.

Typical required services include:

```text
DNS        TCP/UDP 53
Kerberos   TCP/UDP 88
RPC        TCP 135
LDAP       TCP/UDP 389
LDAPS      TCP 636
SMB        TCP 445
Kerberos Password TCP/UDP 464
Global Catalog TCP 3268
GC SSL     TCP 3269
RPC Dynamic Ports TCP 49152-65535
```

Do not continue to the trust wizard until basic DNS and connectivity work correctly.

---

# 5. Create the trust from `pmlabs.com`

Now go back to the PMLabs Domain Controller.

Open:

**Server Manager → Tools → Active Directory Domains and Trusts**

You should see:

```text
Active Directory Domains and Trusts
└── pmlabs.com
```

Right-click:

**pmlabs.com → Properties**

Select:

**Trusts**

Click:

**New Trust**

The New Trust Wizard opens.

Click **Next**.

### Step 7 — Enter the other forest/domain

Enter:

```text
dblabs.com
```

Click **Next**.

### Step 8 — Select trust type

If these are two independent AD forests and you want forest-wide access, select:

```text
Forest trust
```

Then click **Next**.

If Forest Trust is not offered, verify the forest functional levels and DNS configuration.

### Step 9 — Select trust direction

For access in both directions, choose:

```text
Two-way
```

This means:

```text
pmlabs.com users → can potentially access dblabs.com resources
dblabs.com users → can potentially access pmlabs.com resources
```

Actual access still depends on NTFS/share/application permissions.

Click **Next**.

### Step 10 — Select sides of trust

The wizard normally asks whether you want to configure:

```text
This domain only
```

or:

```text
Both this domain and the specified domain
```

Choose:

```text
Both this domain and the specified domain
```

This is the easiest option if you have administrator credentials for `dblabs.com`.

Click **Next**.

### Step 11 — Enter DBLabs credentials

Provide an administrator account from `dblabs.com`.

For example:

```text
DBLABS\Administrator
```

or:

```text
administrator@dblabs.com
```

Enter its password.

Click **Next**.

### Step 12 — Authentication scope

For a normal trusted lab environment, you will commonly choose:

```text
Forest-wide authentication
```

This allows authenticated users from the trusted forest to be recognized throughout the forest, subject to resource permissions.

The alternative:

```text
Selective authentication
```

provides tighter control but requires you to explicitly grant "Allowed to authenticate" permissions on servers/resources.

For your lab, assuming standard access requirements:

```text
Forest-wide authentication
```

is simpler.

Configure the option for both directions when prompted.

Continue through the wizard.

### Step 13 — Confirm the trust

Review the configuration.

You should have approximately:

```text
Trust name: dblabs.com
Trust type: Forest
Direction: Two-way
Sides: Both domains/forests
Authentication: Forest-wide
```

Click:

**Next / Finish**

---

# 6. Confirm from `dblabs.com`

Even if you selected **Both this domain and the specified domain**, log in to a DBLabs Domain Controller and verify the trust.

Open:

**Server Manager → Tools → Active Directory Domains and Trusts**

Then:

**dblabs.com → Properties → Trusts**

You should see an entry similar to:

```text
Domains trusted by this domain:
pmlabs.com

Domains that trust this domain:
pmlabs.com
```

For a two-way trust, PMLabs should appear in both appropriate trust directions.

---

# 7. Validate from PMLabs

On `pmlabs.com`:

**Active Directory Domains and Trusts → pmlabs.com → Properties → Trusts**

Select:

```text
dblabs.com
```

Click:

**Properties → Validate**

Choose:

```text
Yes, validate the incoming trust
```

when appropriate.

Provide DBLabs credentials if requested.

You should receive a successful validation message.

---

# 8. Validate from DBLabs

Do the same from `dblabs.com`.

Open:

**Active Directory Domains and Trusts → dblabs.com → Properties → Trusts**

Select:

```text
pmlabs.com
```

Then:

**Properties → Validate**

The validation should succeed.

---

# 9. Verify using command line

From the PMLabs DC:

```powershell
nltest /domain_trusts
```

You should see `dblabs.com`.

You can also run:

```powershell
Get-ADTrust -Filter *
```

Expected output should contain something similar to:

```text
Name       : dblabs.com
Direction  : BiDirectional
TrustType  : Forest
```

From the DBLabs DC:

```powershell
Get-ADTrust -Filter *
```

You should see:

```text
Name       : pmlabs.com
Direction  : BiDirectional
TrustType  : Forest
```

---

# 10. Test cross-domain authentication

Suppose you have:

```text
PMLABS user:
pmlabs\testuser

DBLABS server:
dbserver01.dblabs.com
```

From the PMLabs side, test whether the DBLabs domain can locate and authenticate users.

For example:

```powershell
runas /user:dblabs\Administrator cmd
```

Or from DBLabs:

```powershell
runas /user:pmlabs\Administrator cmd
```

If authentication succeeds, the trust is functioning.

---

# 11. Give users access to resources

A trust **does not automatically give users access**.

For example, suppose DBLabs has this share:

```text
\\dbserver01\DBShare
```

and you want a PMLabs user to access it.

On the DBLabs file server:

**Folder → Properties → Security → Edit → Add**

Click **Locations**.

You should now be able to choose:

```text
pmlabs.com
```

Then add:

```text
PMLABS\TestUser
```

and assign the required permissions.

Then from a PMLabs workstation, test:

```text
\\dbserver01.dblabs.com\DBShare
```

---

## Your final configuration should look like this

```text
                DNS Conditional Forwarders

     pmlabs.com ----------------------> dblabs.com
        DNS        forward dblabs.com      DNS
         |                                  |
         |                                  |
         |          TWO-WAY TRUST           |
         +==================================+
         |                                  |
         |                                  |
     PMLABS DC                           DBLABS DC
         ^                                  |
         |                                  |
         +----------------------------------+
              forward pmlabs.com


PMLABS DNS:
dblabs.com → DBLabs DNS/DC IP

DBLABS DNS:
pmlabs.com → PMLabs DNS/DC IP


Trust:
pmlabs.com <===========> dblabs.com

Type:       Forest Trust
Direction:  Two-way
Auth:       Forest-wide authentication
```

The safest execution order is **PMLabs DNS → DBLabs DNS → verify name resolution both ways → verify time/network → create trust from PMLabs using “Both this domain and specified domain” → verify from DBLabs → validate both sides → test user/resource access**.
