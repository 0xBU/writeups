# Quick Run
*(misc60, solved by 269)* 
> Someone sent me a file with white and black rectangles. I don't know how to read it. Can you help me?

This was a funny challenge where after unzipping it you're presented with a single file that contains a 
bunch of wonky looking text. If you base64 decode it, you're presented with a set of:
```
██████████████████████████████████████████████
██              ██    ██  ████              ██
██  ██████████  ██████  ██  ██  ██████████  ██
██  ██      ██  ██  ██████  ██  ██      ██  ██
██  ██      ██  ██████████████  ██      ██  ██
██  ██      ██  ██    ██    ██  ██      ██  ██
██  ██████████  ████  ████████  ██████████  ██
██              ██  ██  ██  ██              ██
██████████████████████  ██  ██████████████████
████  ████  ██  ████████      ████  ██  ██  ██
██  ██  ██  ████  ██  ██████      ██████    ██
██  ██          ██  ██      ████  ████████  ██
██  ████████  ██  ██  ██  ██  ██  ██  ██  ████
████  ██    ██  ██  ██  ██  ██  ██  ██  ██  ██
████████████████████  ██  ██  ██  ██  ██  ████
██              ██████  ██  ██  ██  ██  ██  ██
██  ██████████  ████  ██  ██  ██  ██  ██  ████
██  ██      ██  ██  ██  ██  ██  ██  ██  ██  ██
██  ██      ██  ████  ██  ██  ██  ██  ██  ████
██  ██      ██  ██████  ██  ██  ██  ██  ██  ██
██  ██████████  ██    ████████  ████  ██    ██
██              ██████          ██  ██  ██  ██
██████████████████████████████████████████████
```

These are actually QR codes, decoding each one reveals:
```
flagis:IW{QR_C0DES_RUL3}
```

We could've saved 5 minutes by realizing to start looking for 'IW' first, and not decoding 'flagis:'.


