# grrcon2012

1. How was the attack delivered?
2. What time was the attack delivered?
3. What was that name of the file that dropped the backdoor?
4. What is the ip address of the C2 server?
221.54.197.32:443
5. What type of backdoor is installed?
Poison Ivy RAT
6. What is the mutex the backdoor is using?
)!VoqA.I4
7. Whee backdoor placed on the filesystem?
8. What process name and process id is the backdoor running in?
explorer.exe (1096)
9. What additional tools do you believe were placed on the machine?
10. What directory was created to place the newly dropped tools?
11. How did the attacker escalate privileges?
12. What level of privileges did the attacker obtain?
13. How was lateral movement performed?
14. What was the first sign of lateral movement?
15. What documents were exfiltrated?
16. How and where were the documents exfiltrated?
17. What additional steps did the attacker take to maintain access?
18. How long did the attacker have access to the network?
19. What is the secret code inside the exfiltrated documents?
20. What is the password for the backdoor?


# Approach

Noob
Didn't see the questions (they guide what to look for). Get whole view.

# Solution

First, I wanted to get some basic information on the image, such as profile for Volatility commands. 

```   
$ vol.py –f memdump.img imageinfo
```   
From the results, we learn a few (potentially) important things:
- Profile: WinXP, Service Pack 3, x86
- KDBG: 0x8054cde0L (we probably won't use this, but good to know)
- Image date/time: 2012-04-28 02:23:21 UTC
- Image local date/time: 2012-04-27 22:23:21 -0400 (good for just-in-case timestamp referencing)

## Approach

Something about SANS courses

## Identify Rogue Processes

`pslist` + `pstree`

We find two instances of userinit that start 30 min after boot, one of which (PID 1212) a number of processes including Adobe Reader and a command prompt which runs mdd. A quick check of the process' SID with `getsids` reveals that this process was run by the Admin user. 

Let's check the explorer process spawned by the suspicious userinit, since explorer is a commonly attacked process. `dlllist` didn't turn up anything obvious, so I tried running `handles` and filtering for mutexes (-t Mutant) reveals a suspicious Mutant entry:

`)!VoqA.I4`

which some may recognize as a mutex for the Poison Ivy backdoor. 


### So what did the explorer process do? 
Looking for network artifacts, we run `connscan | grep 1096` to search for related network artifacts. 

`172.16.150.20:1424      221.54.197.32:443     1096`

So explorer.exe initiated a network connection to a remote host (221.54.197.32) over port 443. 

Dumped the file with vaddump. 
Searched for bits that contained 221.54.197.32, searched with grep -C 10. Found 'tigers' and 'svchosts.exe.'

Explorer.exe also initiated a command prompt that ran `mdd.exe` and two Adobe-related processes. Analyzing those processes for malicious DLLs and handles, we see a few more results:

TODO: what is adobe processes.
AdobeARM is an auto-update utility  'that notifies you, downloads, and installs new updates for these products.' https://www.bleepingcomputer.com/startups/AdobeARM.exe-25493.html

Means makes network connections? 
Reader_SL is a preloader
https://www.neuber.com/taskmanager/process/reader_sl.exe.html

AdobeARM (1796) made several connections to 199.7.59.190:80. So this could be a legtimiate auto-update, or a malicious process masquerading as an auto-update, or a hijacked process. 
File mutant: NamedPipe\Router? - look at registry keys for certificates

MDD.exe (1396)
- `dlllist`: Reveals command line of `mdd.exe -o memdump.img`

cmd.exe (?)
- `consoles`: 
net use z: \\DC01\response
connects to domain controller. 
copies mdd.exe to current system
- `cmdscan`:
Cmd:Otp 66.32.119.38
Mdd: Out 1.txt, Out 2.txt

If attacker utilizes simply-named .txt files, let's do a quick scan for with filescan:

`Windows\system32\h323log.txt`
`Windows\system32\systems\f.txt`


In summary:
- Attacker used `net use` to remotely access the Domain Controller
- Attacker copied `mdd.exe` to the current system
- Attacker executed `mdd.exe` to dump memory
- Attacker exfil-ed the image to 66.32.119.39
- At some point, attacker create `Out 1.txt` and `Out 2.txt`




Indicators:
- Userinit (1212)
- Explorer (1096)
- 221.54.197.32 (explorer)
- 199.7.59.190:80 (1796)

Questions?
- How did explorer get hooked?
- How did the backdoor get dropped?
- What is it doing?

TODO
- net use: rdp/ps. 
- 




Timeline:
2012-04-28 02:20:54   explorer.exe (1096)


## Timeline Analysis

Searched timeline for svchosts:
less compromised.timeline | grep svchosts
- Created 
- Executed once (prefetch)

Pivot off time of svchosts.exe creation. 

Find. SWING-MECHANICS.DOC[1].EXE executed (prefetch). WTF
Search for swing-mechanics, only found the prefetch file ??
Right before, looks like running internet explorer. 

Run strings on memory image:
- TODO: find it. 

After swing mechanics, see IPCONFIG, NET and PING activity. Likely something has been downloaded and is communicating back to C2 (compare to timeline).

Also see creation of C:\Windows\system32\systems. Interesting... 
Then see f.txt, g.exe, p.exe, r.exe, sysmon.exe, w.exe.

What are they? Are they run?

FTP?
- executed
what was exfil-ed?


TODO: quick research on Poison Ivy. 




