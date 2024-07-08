# Shell Shell Shell

I started to dive a bit deeper into shellcodes recently, this post is some of the things i picked up. The code are in x86-64 intel syntax.

# Shells dropping SUID rights

Consider this simple shell code skeleton that calls a "/bin/sh" shell assuming there are no fancy stuff going on like filtering etc.

```assembly
mov al, 59 # execve()
lea rdi, [rip + sh] # i know i used null bytes dont come at me yet
xor rsi, rsi # argv[]
xor rdx, rdx # envp

.sh
  .string "/bin/sh"
```

