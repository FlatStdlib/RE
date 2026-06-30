<div align="center">
    <h1>Reverse Engineer Tutorial</h1>
    <p>Educational Purposes Only</p>
</div>

# Inspecting an ELF Program

In this tutorial, I will be showing how to view an ELF program and use the following code for the example!

The following tools and//or documentation for information on syscalls
- x86_64 syscall table (https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
- Disassembler - objdump
- Hex Editor - HxD

I added comments to the functions wrapping syscalls behind them.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

char BUFFER[4096];

int main()
{
    FILE *fd = fopen("/etc/os-release", "r");       // sys_open
    if(!fd) {
            return 1;
    }

    fseek(fd, 0L, SEEK_END);                        // sys_lseek
    int sz = ftell(fd);
    fseek(fd, 0L, SEEK_SET);                        // sys_lseek

    fread(BUFFER, 1, sz, fd);                       // sys_read
    puts(BUFFER);                                   // sys_write
    fclose(fd);                                     // sys_close
    return 0;
}
```

Now lets compile it and inspect the disassembly output. Only using the main disassembled function

As you can see, The encoded bytes for the instructions listed are there. Some important instruction sets to look for when inspecting programs are;

- mov - Values being passed to EAX/RAX with syscall around value (Right Oprand)
- cmp - if-statement
- test - if-statement

```
00000000000011e9 <main>:
    11e9:       f3 0f 1e fa             endbr64
    11ed:       55                      push   rbp
    11ee:       48 89 e5                mov    rbp,rsp
    11f1:       48 83 ec 10             sub    rsp,0x10
    11f5:       48 8d 15 08 0e 00 00    lea    rdx,[rip+0xe08]        # 2004 <_IO_stdin_used+0x4>
    11fc:       48 8d 05 03 0e 00 00    lea    rax,[rip+0xe03]        # 2006 <_IO_stdin_used+0x6>
    1203:       48 89 d6                mov    rsi,rdx
    1206:       48 89 c7                mov    rdi,rax
    1209:       e8 e2 fe ff ff          call   10f0 <fopen@plt>
    120e:       48 89 45 f8             mov    QWORD PTR [rbp-0x8],rax
    1212:       48 83 7d f8 00          cmp    QWORD PTR [rbp-0x8],0x0
    1217:       75 07                   jne    1220 <main+0x37>
    1219:       b8 01 00 00 00          mov    eax,0x1
    121e:       eb 7b                   jmp    129b <main+0xb2>
    1220:       48 8b 45 f8             mov    rax,QWORD PTR [rbp-0x8]
    1224:       ba 02 00 00 00          mov    edx,0x2
    1229:       be 00 00 00 00          mov    esi,0x0
    122e:       48 89 c7                mov    rdi,rax
    1231:       e8 aa fe ff ff          call   10e0 <fseek@plt>
    1236:       48 8b 45 f8             mov    rax,QWORD PTR [rbp-0x8]
    123a:       48 89 c7                mov    rdi,rax
    123d:       e8 8e fe ff ff          call   10d0 <ftell@plt>
    1242:       89 45 f4                mov    DWORD PTR [rbp-0xc],eax
    1245:       48 8b 45 f8             mov    rax,QWORD PTR [rbp-0x8]
    1249:       ba 00 00 00 00          mov    edx,0x0
    124e:       be 00 00 00 00          mov    esi,0x0
    1253:       48 89 c7                mov    rdi,rax
    1256:       e8 85 fe ff ff          call   10e0 <fseek@plt>
    125b:       8b 45 f4                mov    eax,DWORD PTR [rbp-0xc]
    125e:       48 98                   cdqe
    1260:       48 8b 55 f8             mov    rdx,QWORD PTR [rbp-0x8]
    1264:       48 8d 3d d5 2d 00 00    lea    rdi,[rip+0x2dd5]        # 4040 <BUFFER>
    126b:       48 89 d1                mov    rcx,rdx
    126e:       48 89 c2                mov    rdx,rax
    1271:       be 01 00 00 00          mov    esi,0x1
    1276:       e8 35 fe ff ff          call   10b0 <fread@plt>
    127b:       48 8d 05 be 2d 00 00    lea    rax,[rip+0x2dbe]        # 4040 <BUFFER>
    1282:       48 89 c7                mov    rdi,rax
    1285:       e8 16 fe ff ff          call   10a0 <puts@plt>
    128a:       48 8b 45 f8             mov    rax,QWORD PTR [rbp-0x8]
    128e:       48 89 c7                mov    rdi,rax
    1291:       e8 2a fe ff ff          call   10c0 <fclose@plt>
    1296:       b8 00 00 00 00          mov    eax,0x0
    129b:       c9                      leave
    129c:       c3                      ret
```

Here we can see fopen call in the main function with 2 arguments being passed to the ABI registers and the file descriptor check if statement.

```
  401a3f:   48 89 d6                mov    rsi,rdx
  401a42:   48 89 c7                mov    rdi,rax
  401a45:   e8 16 3c 00 00          call   405660 <_IO_new_fopen>   ; call fopen
  401a4a:   48 89 45 f8             mov    QWORD PTR [rbp-0x8],rax  ; file descriptor returned
  401a4e:   48 83 7d f8 00          cmp    QWORD PTR [rbp-0x8],0x0  ; null/0 check
  401a53:   75 07                   jne    401a5c <main+0x37>       ; return (jump back to the loader)
```

If the null check was changed or removed this app then seg faults trying to process an invalid fopen and continue to execute further code relying on the file.

You can then use the byte to the if statement, (48 83 7d f8 00) with the last value being the null/0. (00). You can find and replace this in a hex editor directly

For runtime, You attach to the process and do the same thing by jumping around in it