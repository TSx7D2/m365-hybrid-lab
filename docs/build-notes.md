# Build notes

Things that failed, and what actually caused them. Kept because the diagnosis is the useful part. A
runbook where everything works first time teaches nothing.

---

## EDR onboarding stuck in `Conflict`

**Symptom.** The setting *Onboarding blob from Connector* reported `Conflict`, with the device report
naming only one source profile, which looks like a profile conflicting with itself.

**Cause.** Two onboarding policies existed. After connecting the Defender connector, the EDR
Onboarding Status tab offers **Deploy preconfigured policy**, which creates its own policy targeting
all Windows devices. Used in addition to a manually created profile rather than instead of it, both
deliver an onboarding blob for the same setting.

**Fix.** Delete one. Keeping the preconfigured one means correcting its defaults, because it arrives
with a generic name, no scope tag, and an all-devices assignment.

**Worth remembering.** The device report naming a single source profile does not mean there is only
one. Check the policy list, not the report.

---

## `New-NetIPAddress` refused to set a static address

**Symptom.** On a fresh Server Core install:

```
New-NetIPAddress : Inconsistent parameters PolicyStore PersistentStore and Dhcp Enabled
```

Disabling DHCP first with `Set-NetIPInterface -Dhcp Disabled` did not help.

**Cause.** The DHCP flag is held in both the ActiveStore and the PersistentStore.
`Set-NetIPInterface` without a `-PolicyStore` updates the active one, while `New-NetIPAddress`
writes to the persistent one, which still reads `Enabled`.

**Fix.** Use `netsh`, which writes both:

```powershell
netsh interface ipv4 set address name="Ethernet" source=static `
      address=10.0.0.10 mask=255.255.255.0 gateway=10.0.0.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

Alternatively, disable DHCP in the persistent store explicitly using `-PolicyStore PersistentStore`
together with `-AddressFamily IPv4`.

**Related, and benign.** Setting DNS with `netsh` warns *The configured DNS server is incorrect or
does not exist* when pointing a future domain controller at `127.0.0.1`. Nothing is listening on
port 53 until the DNS role is installed. The setting is applied regardless and the command returns
success, and `validate=no` suppresses the warning.

---

## Password length reported `Conflict` on an enrolled device

**Cause.** The security baseline sets *Device Lock, minimum password length* to 14. The Windows
Hello for Business policy sets *Minimum PIN length* to 6. Since the Anniversary Update these are the
same underlying value, so the two policies were fighting over one setting while appearing to
configure unrelated things.

**Fix.** Decide which profile owns it and set the other to *Not configured*.

---

## Smaller things

- **Renaming a domain-joined machine** needs `-DomainCredential`, or only the local name changes and
  the secure channel breaks. Renaming before joining avoids the problem entirely. Never do it on a
  domain controller.
- **Group Policy drive maps apply at logon**, not on `gpupdate`. Group membership changes need a new
  logon token as well, so testing item-level targeting means signing out and back in, twice.
- **`Get-ADRootDSE` needs RSAT.** Plain ADSI does not, and works on any domain member:
  `([ADSI]"LDAP://RootDSE").configurationNamingContext[0]`. The `[0]` matters, because the property
  is a collection rather than a string.
- **Connecting Entra Connect to the directory** needs **Enterprise Admins**, not Domain Admins. The
  service connection point lives in the forest configuration partition.
