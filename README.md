# grrcon2012

1. How was the attack delivered?
2. What time was the attack delivered?
3. What was that name of the file that dropped the backdoor?
4. What is the ip address of the C2 server?
5. What type of backdoor is installed?
6. What is the mutex the backdoor is using?
7. Whee backdoor placed on the filesystem?
8. What process name and process id is the backdoor running in?
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

Let's check the explorer process spawned by the suspicious userinit, since explorer is a commonly attacked process. Running `handles` and filtering for mutexes (-t Mutant) reveals a suspicious Mutant entry:

`)!VoqA.I4`

which some may recognize as a mutex for the Poison Ivy backdoor. 

TODO: quick research on Poison Ivy. 




Indicators:
- Userinit (1212). 

Questions?
- How did userinit get hooked?
- what's it doing?





To start, I decided to check out network connections to identify any suspicious activity (C2, download, exfil, etc.).

`$ vol.py –f memdump.img connscan`   

We see a few interesting connections:
- 221.54.197.32:443 from PID 1096. 
- 199.7.52.190:80 from PID 1796.
- Activity on ports 139 & 445 to 172.16.150.10



