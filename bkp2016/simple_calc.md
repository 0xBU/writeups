# Simple Calc
*(pwn, solved by 186)*  
> what a nice little calculator!   
https://s3.amazonaws.com/bostonkeyparty/2016/b28b103ea5f1171553554f0127696a18c6d2dcf7  
simplecalc.bostonkey.party 5400

I teamed up with [@kierk](https://github.com/kierk) for this challenge.

We're given an ELF64 binary. Opening it up in IDA and running the decompiler reveals to us:

```
int __cdecl main(int argc, const char **argv, const char **envp)
{
  void *v3; // rdi@1
  int result; // eax@3
  const char *v5; // rdi@4
  __int64 v6; // rax@4
  char v7; // [sp+10h] [bp-40h]@14
  int v8; // [sp+38h] [bp-18h]@5
  int v9; // [sp+3Ch] [bp-14h]@1
  __int64 v10; // [sp+40h] [bp-10h]@4
  int i; // [sp+4Ch] [bp-4h]@4

  v9 = 0;
  setvbuf(stdin, 0LL, 2LL, 0LL);
  v3 = stdout;
  setvbuf(stdout, 0LL, 2LL, 0LL);
  print_motd(v3);
  printf((unsigned __int64)"Expected number of calculations: ");
  _isoc99_scanf((unsigned __int64)"%d");
  handle_newline("%d", &v9);
  if ( v9 <= 255 && v9 > 3 )
  {
    v5 = (const char *)(4 * v9);
    LODWORD(v6) = malloc(v5);
    v10 = v6;
    for ( i = 0; i < v9; ++i )
    {
      print_menu(v5);
      v5 = "%d";
      _isoc99_scanf((unsigned __int64)"%d");
      handle_newline("%d", &v8);
      switch ( v8 )
      {
        case 1:
          adds("%d");
          *(_DWORD *)(v10 + 4LL * i) = dword_6C4A88;
          break;
        case 2:
          subs("%d");
          *(_DWORD *)(v10 + 4LL * i) = dword_6C4AB8;
          break;
        case 3:
          muls("%d");
          *(_DWORD *)(v10 + 4LL * i) = dword_6C4AA8;
          break;
        case 4:
          divs("%d");
          *(_DWORD *)(v10 + 4LL * i) = dword_6C4A98;
          break;
        default:
          if ( v8 == 5 )
          {
            memcpy(&v7, v10, 4 * v9);
            free(v10);
            return 0;
          }
          v5 = "Invalid option.\n";
          puts("Invalid option.\n");
          break;
      }
    }
    free(v10);
    result = 0;
  }
  else
  {
    puts("Invalid number.");
    result = 0;
  }
  return result;
}
```

It seems the program is infact, a simple calc. The program can add, subtract, multiply, or divide for us. 
Looking deeper into the program's functions reveals that there's nothing really too interesting, summarized 
here:

* print_motd - a simple 1 line `printf`, nothing exploitable
* handle_newline - does `getchar` until a `\0` or a `\n`, perhaps buggy, but doesn't seem exploitable
* print_menu - simple `printf`s, nothing exploitable
* adds, subs, muls, divs - do basic arithmetic of two arguments, the arguments are limited to > 39, for 
some reason. It seems you can cause integer overflows here perhaps. The interesting thing in these is that 
result of the functions seems to get put into the global data section, `.bss`.

The `main` function allows us 5 options, the 4 basic math operators, and a save and exit. The save and exit 
is particularly interesting. The save and exit option does a `memcpy` of global memory onto the stack. 
However, there was only 11 dwords allocated for the copy operation by the stack. If we ask main for more 
than 11 operations, prompted here: `printf((unsigned __int64)"Expected number of calculations: ");`, then 
we'll have a simple stack buffer overflow. With that, we can overwrite the return address, and initiate a 
ropchain. 

There is one other interesting point here, and that's the call to `free()`. Initially we ran into the 
problem of `free()` failing, and causing a segmentation fault when we were playing with the program with 
over 11 operations. That's because we kept spamming the global memory with the values "40..41..42..43..44", 
and `free()` would try to free the memory located at `0x0000000000000059`. We were stuck, but we guessed 
perhaps `free()` doesn't fail for 0 or null, and that turned out to be true, as documented here:

>The free function causes the space pointed to by ptr to be deallocated, that is, made available for 
further allocation. If ptr is a null pointer, no action occurs.

So we can simply make sure the 19th value copied onto the stack is `0x0000000000000000`.

So let's create this ropchain, luckily the binary is gigantic, and has everything we need. Running 
[ropgadget](https://github.com/JonathanSalwan/ROPgadget) (also included in my 
[sec-tools](https://github.com/eugenekolo/sec-tools)) on the binary produces for us an already complete 
ropchain:

```
ropgadget --binary b28b103ea5f1171553554f0127696a18c6d2dcf7 --ropchain
> https://gist.github.com/eugenekolo/d07741d6e7b261b3c339#file-ropchain-py
```

Now, we have to get that chain onto the stack, and we're set. We have control of a global array in memory 
that gets copied onto the stack. Getting the exact bytes we want into that global array is a bit tricky. We 
have to use one of the operations of the calculator (`sub` is my favourite), with specially crafted 
operands. The result of that calculation will be put into the global array. *This part is a bit tricky in 
that it takes 2 operations of the calculator to fill 64-bit blocks in memory that the program uses (due to 
it being a 64-bit elf) - the calculations return 32-bit numbers.* So, we end up with a chain of operations 
that looks like this:

```
2          # Subtract operation
4602868
100        # Result = 4602768

2          # Subtract operation
100
100        # Result = 0

2          # Subtract operation
4602868
100        # Result = 4602768

2          # Subtract operation
100
100        # Result = 0

...

5 # Save and exit operation
```

That will result in `0x0000000000463B90` and `0x0000000000463B90` quad-words in the global memory.

The global array is then copied onto the stack with a `save and exit`. The copying overwrites the return 
address on the stack, and begins our ropchain.

Now we have to somehow get the entire ropchain into memory with some calculator operations. I went with a 
low-tech pwning script for this exploitation. I wrote a 
[pwn\_script](https://gist.github.com/eugenekolo/d07741d6e7b261b3c339#file-pwn-py) that outputted the above 
chain of operations, newlines included. When the 
[pwn_header](https://gist.github.com/eugenekolo/d07741d6e7b261b3c339#file-pwn_header-py) (to initiate 254 
operations, and make sure `free()` frees `0x0`) and the output of the 
[pwn\_script](https://gist.github.com/eugenekolo/d07741d6e7b261b3c339#file-pwn-py) is copied into the 
remote session of the challenge, `nc simplecalc.bostonkey.party 5400`, a series of subtraction operations 
with different operands will occur. Finally a `5` will be passed to the remote session, and cause a `save 
and exit`. That will kick off the ropchain now in the `.bss` section, revealing to us this tasty shell:

```
$ ls
key
run.sh
simpleCalc
simpleCalc_v2
socat_1.7.2.3-1_amd64.deb
$ cat key
BKPCTF{what_is_2015_minus_7547}
```

