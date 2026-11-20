# configs-and-logs

Sanitized evidence from the engagement: command outputs, scan results, and Wazuh alert samples.

> **Note:** `nightwatch.local` is a fully isolated lab with **fabricated** accounts and credentials. The hashes and keys here belong to throwaway lab objects and grant no access to any real system. Nothing in this folder is from a production environment.

| File | Contents |
| --- | --- |
| `nmap-dc01.txt` | Full TCP service scan of the domain controller |
| `secretsdump-ntds.txt` | DCSync NTDS.dit dump (lab accounts only) |
| `wazuh-7045-alert.json` | Sample level-12 PsExec service-creation alert |
