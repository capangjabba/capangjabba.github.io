+++
title = 'ASEAN Notes.iso Mustang Panda [Malware Analysis]'
date = 2025-01-25T03:49:34+08:00
draft = false
categories = ["Malware Analysis"]
tags = ["apt", "mustang-panda", "dll-sideloading"]
+++

# Malware Analysis Report: ASEAN Notes.iso from Mustang Panda Campaign

```metadata
Author: Capang
Date: 24-01-2025
Analysis Environment: Windows 10 VM
Associated Campaign: Mustang Panda ASEAN Notes.iso  
Ref: https://csirt-cti.net/2024/01/23/stately-taurus-targets-myanmar/
```

## Executive Summary

The ASEAN Notes.iso file is a component of a Mustang Panda campaign targeting entities in Myanmar. The attack leverages DLL sideloading via a hijacked Microsoft GetCurrentRollback.exe process (renamed office.exe) to execute the malicious GetCurrentDeploy.dll. The malware establishes persistence through registry modification and attempts C2 communication with fallback infrastructure. Primary objectives include initial access, persistence, and command execution, with suspected espionage motivations.

## Case Details

### File Metadata

| **File Name**    | ASEAN Notes.iso                                                  |
| ---------------- | ---------------------------------------------------------------- |
| **File Size**    | 602,112 B                                                        |
| **File Type**    | ISO 9660                                                         |
| **MD5**          | 9832bd120aa506758b3c1850dc2f7e41                                 |
| **SHA1**         | 8e7dfe85c00f76c2525b0ea001b735b1240f3342                         |
| **SHA256**       | a00673e35eaccf494977f4e9a957d5820a20fe6b589c796f9085a0271e8c380c |
| **Created Time** | N/A                                                              |

| **File Name**    | GetCurrentDeploy.dll                                             |
| ---------------- | ---------------------------------------------------------------- |
| **File Size**    | 97,792 B                                                         |
| **File Type**    | Linker: Microsoft Linker(14.34**)\[DLL32]                        |
| **MD5**          | d901af6c326d9d6934d818beef214e81                                 |
| **SHA1**         | b78e786091f017510b44137961f3074fe7d5f950                         |
| **SHA256**       | 51d89afe0a49a3abf88ed6f032e4f0a83949fc44489fc7b45c860020f905c9d7 |
| **Created Time** | 16/1/2024 11:27:46 PM                                            |

| **File Name**    | office.exe                                                       |
| ---------------- | ---------------------------------------------------------------- |
| **File Size**    | 73,344 B                                                         |
| **File Type**    | PE32                                                             |
| **MD5**          | 823ce97af76ce9321f8ca58f126b3141                                 |
| **SHA1**         | aad6f04d8e4a511eb518df3c07a2094c8b558708                         |
| **SHA256**       | 0d0981941cf9f1021b07b7578c45ed4c623edb16ad03a256c4cd9aaf900d723d |
| **Created Time** | 16/1/2024 11:27:44 PM                                            |

| **File Name**    | ASEAN 2024.lnk  Mofa memo.lnk  MS.lnk   NS.lnk                   |
| ---------------- | ---------------------------------------------------------------- |
| **File Size**    | 1200                                                             |
| **File Type**    | MS Windows shortcut                                              |
| **MD5**          | 698382d42978ee9b86046682cacc76ab                                 |
| **SHA1**         | dd149a0c4a650df907557b3c0219fde81d339d11                         |
| **SHA256**       | e537c5da268c6a08d6e94d570e8efb17d0ca3f4013e221fadc4e0b3191499767 |
| **Created Time** | N/A                                     |

## Case Specific Requirement

##  Machine

- Windows Environment

## Tools

- hashmyfiles
- PEStudio
- DiffView
- RegShot
- Wireshark
- DirWatch

## Static Analysis

### ASEAN Notes.iso

![image](/images/aseannotesiso/1.png)


34/61 security vendors flagged this file as malicious. 

![image](/images/aseannotesiso/2.png)

Associated with Mustang Panda targeting Myanmar

When mounting the ISO file, the victim is shown a set of `LNK` files or Shortcut and a folder with numerous structure named `_` 

![image](/images/aseannotesiso/3.png)

All of the `LNK` files are programmed to display PDF icon. Each of the `LNK` / Shortcut has the same properties/command which is 
```bash
C:\Windows\System32\ScriptRunner.exe -appvscript _\_\_\_\_\_\_\_\_\_\_\_\office.exe
```

The `_` folder structure:

![image](/images/aseannotesiso/4.png)

At the end of folder `_` there are `office.exe` and `GetCurrentDeploy.dll`.

![image](/images/aseannotesiso/5.png)

### office.exe


![image](/images/aseannotesiso/6.png)

No security vendors flagged this file as malicious. However `office.exe` is not the real name of this file.

![image](/images/aseannotesiso/7.png)

`GetCurrentRollback.exe` is the original name and it is a legitmate file associated with Microsoft Corporation. Why does the `LNK` file execute a legitimate file? DLL-Side Loading is the current assumption that can be made.

### GetCurrentDeploy.dll

![image](/images/aseannotesiso/8.png)

49/72 security vendors flagged this file as malicious

![image](/images/aseannotesiso/9.png)

11//77 Windows API being used flaged as malicious

Can be confirmed that DLL-Side Loading attack is being used. `office.exe` or `GetCurrentRollback.exe` might be using a shared library or DLL named `GetCurrentDeploy.dll`. The nature of importing functions from a shared library is to locate the file in a specific order. For windows this is the searching order:

1. **The directory from which the application loaded**
2. The system directory
3. The 16-bit system directory
4. The Windows directory
5. The current working directory (CWD)
6. The directories that are listed in the PATH environment variable

Knowing the `office.exe` and `GetCurrentDeploy.dll` is in the same folder indicates DLL-Side Loading attack is being used.

reference: https://techzone.bitdefender.com/en/tech-explainers/what-is-dll-sideloading.html

### Key Findings

- Victim will be tricked into mounting the `ISO` file and click one of the `LNK` file
- The `LNK` file will execute `C:\Windows\System32\ScriptRunner.exe -appvscript _\_\_\_\_\_\_\_\_\_\_\_\office.exe` 
- The file `office.exe` using `GetCurrentDeploy.dll` as shared library
- Malicious code is executed within the context of a legitimate application using **DLL Side Loading** method
### Attack Visualization

![image](/images/aseannotesiso/attack-flowdrawio.png)

## Dynamic Analysis

### Network Analysis

Clicking on one of the `LNK` files will execute `C:\Windows\System32\ScriptRunner.exe -appvscript _\_\_\_\_\_\_\_\_\_\_\_\office.exe`. Right off executing it, the malware will try to communicate with its C2, `openservername.com` with IP `103.159.132.80`.

DNS request for openservername.com
![image](/images/aseannotesiso/10.png)

Malware trying to communicate with the C2, but unfortunately the C2 is down.

![image](/images/aseannotesiso/11.png)
Upon multiple failed request, the malware switch into another C2 server with IP 37.120.222.19 

![image](/images/aseannotesiso/12.png)
This C2 is also down. After failed attempts to communicate with the C2 servers, the malware killed itself
![image](/images/aseannotesiso/13.png)
However upon execution, the malware replicate itself into `C:\Users\Public\` directory

![image](/images/aseannotesiso/14.png)

### Persistence
The malware established a persistence under the registry `gameestrto` with command line argument `StarWegameToyOU` at under `\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\` Key.

![image](/images/aseannotesiso/15.png)

The Run key makes a program run every time a user logs on.

### Execution Flow

1. **LNK Activation**: Triggers `ScriptRunner.exe` to execute `office.exe` from the nested `_` directories.
    
2. **DLL Hijacking**: `office.exe` loads `GetCurrentDeploy.dll` from its directory (DLL search order abuse).
    
3. **Persistence**: Copies itself to `C:\Users\Public\office.exe` and adds registry autorun entry.
    
4. **C2 Communication**:
    
    - **Primary C2**: `openservername.com` (103.159.132.80) 
        
    - **Fallback C2**: `37.120.222.19` 
        
5. **Self-Termination**: Exits if C2s are unreachable.

## Key Technical Findings

1. **DLL Sideloading Technique**:
    
    - Legitimate Process: `GetCurrentRollback.exe` (Microsoft-signed).
        
    - Malicious DLL: `GetCurrentDeploy.dll` (unsigned, high entropy sections).
        
2. **Persistence Mechanism**:
    
    - Registry: `HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\gameestrto` → `"C:\Users\Public\office.exe"`.
        
3. **C2 Resilience**: Dual C2 infrastructure with failover logic.
## Full Malware Events

![image](/images/aseannotesiso/statelytaurusdrawio.png)

## IOC

| IOC                                                    | Type                                                             |
| ------------------------------------------------------ | ---------------------------------------------------------------- |
| ASEAN Notes.iso (SHA256)                               | a00673e35eaccf494977f4e9a957d5820a20fe6b589c796f9085a0271e8c380c |
| ASEAN 2024.lnk, NS.lnk, MS.lnk, Mofa memo.lnk (SHA256) | e537c5da268c6a08d6e94d570e8efb17d0ca3f4013e221fadc4e0b3191499767 |
| office.exe (SHA256)                                    | 0d0981941cf9f1021b07b7578c45ed4c623edb16ad03a256c4cd9aaf900d723d |
| GetCurrentDeploy.dll (SHA256)                          | 51d89afe0a49a3abf88ed6f032e4f0a83949fc44489fc7b45c860020f905c9d7 |
| Primary C2 IP address                                  | 103.159.132.80                                                   |
| Backup C2 IP address                                   | 37.120.222.19                                                    |
| C2 Domain                                              | openservername.com                                               |
| Autorun key                                            | gameestrto                                                       |
| String                                                 | StarWegameToyOU                                                  |
## Reference

https://www.spyshelter.com/exe/microsoft-corporation-getcurrentrollback-exe/
https://csirt-cti.net/2024/01/23/stately-taurus-targets-myanmar/
https://techzone.bitdefender.com/en/tech-explainers/what-is-dll-sideloading.html
https://learn.microsoft.com/en-us/windows/win32/setupapi/run-and-runonce-registry-keys
