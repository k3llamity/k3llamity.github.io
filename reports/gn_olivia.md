# The student that wasn't one 

## Case summary
On March, 13 2025, during a routine threat hunt, we discovered a live threat on the network, that we were able to successfully shut down before too much damage was done.

Prior to the intrusion, effective on March, 8, the threat actor used social engineering to gain the trust of a company's scientist. Posing as a student, the threat actor first contacted the scientist on LinkedIn, then they both moved to emails to keep talking. The scientist even shared papers with the "student", including research belonging to the company, a major faux-pas and already a legal issue. Using a legitimate file-sharing service, the threat actor then shared a paper of their own. The file made use of the .svg format, which contained a malicious script. When accessed by the scientist, the script downloaded a .zip archive on his machine.

Once on the machine, the .zip archive was expanded to reveal a decoy file and an AutoIt script. The execution of the script downloaded a Remote Access Trojan (RAT) on the machine. The RAT was added as a startup program for persistence, and everything else was done through that RAT.

In the span of a couple days, the RAT collected the scientist's credentials, acted as a keylogger, copied the content of the clipboard, and stole documents, exclusively on the scientist's machine.

From the .svg file being accessed to the detection of the threat, five days have passed, placing this intrusion on the slow end of the spectrum. We can assume the threat actor was being cautious and intended to stay longer on the network, to steal as much data as they could. While we cannot be certain, we think it might be a case of cyber espionage.

## Reconnaissance
On March, 5 2025, the threat actor accessed a dozen webpages on the company's website. They used 5 different IP addresses that were linked to their infrastructure. The user agent remained constant: `Mozilla/5.0 (Windows NT 6.0) AppleWebKit/533.1 (KHTML, like Gecko) Chrome/54.0.821.0 Safari/533.1`.

Some pages were accessed directly from the website "search" function, others came from LinkedIn. They initially looked for scientists working on Brain Computer Interface (BCI), and then focused on a specific employee (that we will refer to as "RK"), a neuroscientist used to public speaking.

## Initial Access
The threat actor did not spearphish RK immediately. Instead, they started to built rapport with him to gain his trust. To do so, they connected with RK on LinkedIn using a sock puppet called "Olivia Octopus". That connection request was sent right after the reconnaissance ended. We do not have access to the communication that happened between them on LinkedIn, but we do have access to the emails they exchanged. From those, we understand that "Olivia" is a PhD student at Harvard, studying BCI. Note that the email address that "Olivia" use mimicks the real Harvard domain: `olivia.octopus@harvards.edu`.

RK and Olivia exchanged emails over a couple days, with RK freely and voluntarily sharing research papers with her. This is highly problematic, as RK is under NDA and those papers are the property of the company (except for the first one). This already becomes a case of data leak via an insider threat.

On March, 8, the threat actor shared a file via the legitimate service Dropbox. That file was named `Olivia Octopus shared Using_BCI_for_language_acquisition_in_children.pdf.svg`. It exploited the ability of .svg files to contain script to download a .zip archive (more on malicious .svg over at [Sophos](https://news.sophos.com/en-us/2025/02/05/svg-phishing/)). Using Powershell to expand the .zip archive created three new files on RK's machine: a decoy .pdf with the original name seen in the Dropbox email, a .com file with AutoIT, and a file containing a password.

![Screenshot of the filenames: olivia_bci_research.zip, Using_BCI_for_language_acquisition_in_children.pdf, autoit_loader.com, autoit_loader_password.txt](https://github.com/user-attachments/assets/2feea88a-c2bd-4837-8fe1-1c32475ee2a3)

The decoy .pdf file was spawned into the `\Documents` folder of the user, while the AutoIt-related files were created into RK's `\AppData\Roaming\` folder. 
AutoIt was then executed to download Nymeria, a RAT also known as Loda.

## Execution
AutoIt3.exe was used to execute the script found in `autoit_loader.com`. It downloaded Nymeria.exe from the domain `bigbrainssmallbrains[.]net` and installed it into the `\AppData\Roaming\` folder of the user. The `$INET_DOWNLOADBACKGROUND` option was added to the `InetGet` command to download the file silently in the background.

Nymeria was then used to carry on all other activities, exploiting `cmd.exe` and `powershell.exe` to execute any other commands.

## Persistence
The legitimate Windows tool `regedit.exe` was used to persist Nymeria:

`reg add HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v ZombieNomNom /t REG_SZ /d C:\Users\rikingsley\AppData\Roaming\nymeria.exe /f`

This means Nymeria will always run at startup when RK boots his machine.

## Credential Access
The threat actor uses a Base64 encoded Powershell command to steal the credentials stored into `cmdkey` and sends the results to the `\brain_dump\` folder on the server they own.

## Discovery
The threat actor uses basic discovery commands once they have access to RK's machine. It checks for system configuration, running processes, network connections and listening ports, IP configuration, and installed programs.

![Screenshot of the discovery commands: whoami, systeminfo, tasklist, netstat -an, ipconfig /all, wmic product get name](https://github.com/user-attachments/assets/c208805a-2eb6-443b-a297-86e984a51321)

## Collection
As with the Credential Access, any collection was made via Base64 encoded Powershell scripts and immediately exiltrated (see [Exfiltration](#exfiltration) for more details).

## Command & Control
Nymeria was used for the C2 operations, communicating directly with the `bigbrainssmallbrains[.]net` domain. 

## Exfiltration
Three more Base64 encoded Powershell scripts were executed on March, 12. All of them sent the data to `bigbrainssmallbrains[.]net`, in the `\brain_dump\` folder.
* The first powershell command records the keystrokes.
* The second gets the content of the clipboard.
* The last steals a very sensitive document they found in RK's `\Documents` folder.

## Impact
The exfiltration of the very sensitive document (proprietary research) is the worst the threat actor had time to do. Because they focused solely on RK to start with, we were able to find them as the situation was still developping, and shut them down as soon as possible. Thus, they were not able to escalate privileges or move through the network, nor steal anything else of importance.

## IOCs
### Domains
* harvards[.]edu
* bigbrainssmallbrains[.]net
### IP addresses
* 39[.]208[.]185[.]141
* 71[.]156[.]150[.]160
* 78[.]86[.]143[.]154
* 128[.]154[.]72[.]217
* 139[.]7[.]86[.]85
* 183[.]118[.]199[.]194
* 193[.]0[.]207[.]170
* 209[.]43[.]101[.]171
* 210[.]218[.]143[.]29
* 221[.]228[.]254[.]161
### Files
| Filename | Sha256 |
| :--- | :--- |
| olivia_bci_research.zip | 1c3ef0407d5714037504c52f7abfa86c081fd7a021b52e2abe8a669f92413252 |
| Using_BCI_for_language_acquisition_in_children.pdf | 8ced3a034e25ae9669aae44af738ce16510122a0c0e23a4f5fcc32720f493fe8 |
| autoit_loader.com | a1e9ed2b75820c3ba85b01a5d2192ebd30e4f8b7f9b38a17f7b64de09fb602e1 |
| autoit_loader_password.txt | c2154cc65f4ec3749bc631c253c4081304f3ce362917d12ccbd6e655adabf3dc |
