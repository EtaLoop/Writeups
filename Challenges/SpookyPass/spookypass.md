# HackTheBox - SpookyPass
Category: Reversing
Difficulty: Very easy

## Analysis & Steps

1. Inspect the file type:
```bash
$ file pass
pass: ELF 64-bit LSB pie executable
```

2. Search readable strings (fast way to find hardcoded passwords):
```bash
$ strings pass
# Welcome to the 
# [1;3mSPOOKIEST
# [0m party of the year.
# Before we let you in, you'll need to give us the password: 
# s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
# Welcome inside!
# You're not a real ghost; clear off!
```
The presence of a clearly formatted password in the binary indicates the check is done against a hardcoded value.


3. Run the binary and provide the password:
```bash
$ ./pass
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: [enter password]
Welcome inside!
[flag output]
```

4. Retreive the flag
