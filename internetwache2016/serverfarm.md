# ServerfARM
*(rev70, solved by 186)*  
> Someone handed me this and told me that to pass the exam, I have to extract a secret string. I know 
cheating is bad, but once does not count. So are you willing to help me?

I teamed up with [@kierk](https://github.com/kierk) for this challenge.

After unzipping the challenge, we're presented with a single ELF32 ARM binary.
```
file serverfarm
serverfarm: ELF 32-bit LSB  executable, ARM, EABI5 version 1 (SYSV), dynamically linked (uses shared libs), 
for GNU/Linux 2.6.32, not stripped
```

Opening it up in IDA:  

![serverfarm1](/blog/content/images/2016/02/serverfarm.png)

Ignoring what it even does, jumping around to the subroutine it's calling (renamed to `handle_task`) and 
pressing `tab` to get IDA pseudocode.

![serverfarm2](/blog/content/images/2016/02/serverfarm2.png)

At this point, we're able to statically look over the code, with the knowledge of how the flag is supposed 
to look and piece together:

`IW{S.E` + `.R.V.E` + `.R>=F:` + `A:R:M}`
```
IW{S.E.R.V.E.R>=F:A:R:M}
```
