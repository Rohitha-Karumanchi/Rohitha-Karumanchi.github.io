---
title: LSA Protection in Windows: How to Prevent Credential Theft with RunAsPPL
description: >-
  Protecting Windows credentials is more critical than ever. LSA Protection (RunAsPPL) is a powerful but often overlooked defense that helps prevent attackers from stealing sensitive credentials stored in LSASS. This guide explains what it is, why it matters, and how to enable it safely in your environment.
author: rk
date: 2025-07-08 20:55:00 +0800
categories: [Credential Protection, Hardening, Windows Security]
tags: [Windows Hardening]
pin: true
media_subpath: '/posts/20180809'
---

# 🛡️ LSA Protection in Windows: What It Is, Why It Matters, and Why It’s Too Rarely Enabled

If you work in security or IT and care about **credential protection**, you’ve likely heard of LSA Protection (RunAsPPL). But how often is it actually enabled in your environment/organisation?

**In this blog, we'll look into:**

- 🔍 What is LSA and what does "RunAsPPL" really mean?
- ⚠️ Why LSA protection is crucial for defending credentials
- 🛠️ How attackers try to bypass it (real CVEs and techniques)
- 🧩 Why many orgs still leave it disabled (risks & tradeoffs)
- 🚀 How to check, enable, and audit it across your estate

---

## 🔐 What is LSA and What is RunAsPPL?

**LSA (Local Security Authority Subsystem Service)**  
The `lsass.exe` process in Windows is responsible for:

- Authenticating logins
- Validating users
- Storing/managing decrypted credentials (e.g., password hashes, Kerberos tickets)
- Applying local security policies and auditing settings

This makes LSASS a prime target for attackers following initial compromise.

**RunAsPPL ("Run as Protected Process Light")**  
Introduced in Windows 8.1 / Windows Server 2012 R2, **RunAsPPL** makes `lsass.exe` a “protected process.”  
It means only other protected processes or kernel-level code can access LSASS memory—blocking Mimikatz and similar user-mode tools, even if run with admin rights.

**Enabling RunAsPPL reduces the risk of credential dumping via common attack tools.**

> **Note:** Kernel-mode malware or vulnerable/signed drivers can still bypass LSA Protection.

---

## ⚠️ Why LSA Protection Matters

- **Credential theft** from LSASS is a key move for advanced attackers.
  - **Mimikatz** and its clones extract passwords, hashes, or Kerberos tickets.
  - Enables **lateral movement**: attackers “Pass-the-Hash” or “Pass-the-Ticket” to move deeper into the network.
- 💪 With **LSA Protection**, attackers can no longer dump credentials using user-mode tools—they’ll need to find a kernel exploit or weak/legacy driver instead.

> It’s not a cure-all, but dramatically limits credential theft through LSASS memory.

---

## 🔦 Real Attacks and CVEs Involving LSASS

**Relevant CVEs/examples:**
- `CVE-2022-26966`: Elevation of privilege via LSA spoofing
- `CVE-2022-37987`, `CVE-2022-37989`: Bypass of Credential Guard/LSA defenses
- Credential dumping seen in many ransomware/APT campaigns (Conti, FIN6, TrickBot, etc.), using:
    - Tools: `procdump`, `taskmgr`, `mimikatz`, direct LSASS injection

---

## ❓ Why Many Orgs Don’t Enable RunAsPPL

| Reason                         | Description                                                                                |
|--------------------------------|--------------------------------------------------------------------------------------------|
| 💥 Compatibility Issues        | Older drivers or third-party security products may fail, crash, or lose functionality      |
| 🛠️ Tool Limitations            | Password recovery or forensics tools may stop working                                      |
| 📆 Deployment Risk             | Sudden change can disrupt critical workflows or lock out users/services                    |
| 🤔 Lack of Awareness           | Admins may not know this control exists or what it blocks                                  |

> Even Microsoft does **not** enable RunAsPPL by default on upgrades—only on clean installs of newer Windows releases.

---

## 📝 How to Check If LSA Protection is Enabled

**To check on a local machine:**
```
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "RunAsPPL"
```
- **Value `1`**: Enabled
- **Value `0` or missing**: Disabled

---

## 🔄 How to Enable RunAsPPL Safely

**Update the registry (reboot required):**
```
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa]
"RunAsPPL"=dword:00000001
```

### **Best Practices Before Deployment:**
- **Pilot:** Test on a non-critical subset of systems.
- **Check Compatibility:** Verify all endpoint security, management, and forensics tools function correctly.
- **Combine with:** [Windows Defender Credential Guard](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard) for stronger protection.

---

## ⚙️ Audit LSA Protection Across Your Domain

**PowerShell Audit Script:**
```
$computers = Get-ADComputer -Filter * | Select -ExpandProperty Name
foreach ($computer in $computers) {
try {
$value = Invoke-Command -ComputerName $computer -ScriptBlock {
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "RunAsPPL" -ErrorAction SilentlyContinue
}
if ($value.RunAsPPL -ne 1) {
Write-Output "$computer does NOT have LSA protection enabled."
}
} catch {
Write-Output "Could not connect to $computer"
}
}
```

*Tip: For large-scale and repeatable settings, use Group Policy, Intune, or SCCM to enforce configuration.*

---

## 💡 Troubleshooting Tips

- **Startup failures after enabling RunAsPPL:** Some security agents or management tools may be incompatible—consult vendor documentation and update drivers.
- **Memory dump tool failures:** This is expected. Use **kernel-debugging** tools or protected auditing software for incident response.
- **Unexpected application issues:** Revert the registry value, reboot, and contact your CTI/security engineering team for compatibility review.

---

## 🧐 Final Thoughts

- **LSA Protection isn’t a silver bullet**, but it is a critical baseline for Windows hardening.
- **If an attacker gets admin,** LSA Protection may be the barrier that blocks lateral movement via credential theft.
- **Always pair with:**
    - Credential Guard
    - Attack Surface Reduction (ASR) rules
    - Event log monitoring (e.g., Sysmon Event ID 10 for LSASS access attempts)
- **Recommended workflow:** Awareness → Testing → Gradual Deployment → Wide-scale enforcement

## 📚 References & Further Reading

- [Microsoft: Configure LSA Protection](https://learn.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection)
- [Microsoft: Credential Guard Overview](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard)
- [Mimikatz and LSASS Attacks](https://attack.mitre.org/techniques/T1003/)
- [Sysmon: Event ID 10 for LSASS Access](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)

---

*Have questions, or need assistance? Let me know via GitHub Issue or Discussion or Comment Below*

