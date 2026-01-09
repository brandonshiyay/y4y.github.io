---
layout: "post"
title: "BambooFox CTF 2021 Writeup"
date: "2021-01-18 07:28:31"
date_gmt: "2021-01-18 07:28:31"
modified: "2021-01-18 07:37:15"
modified_gmt: "2021-01-18 07:37:15"
slug: "bamboofox-ctf-2021-writeup"
author: "brandonshi123"
status: "publish"
type: "post"
original_url: "https://y4y.space/2021/01/18/bamboofox-ctf-2021-writeup/"
guid: "http://y4y.space/?p=347"
wordpress_id: 347
parent_id: 0
categories: ["CTF Write Up", "Reverse Engineering"]
---

{% raw %}
Man, I suck.

For the first time, I've decided to actually include the challenge files. Hope the organizers don't DMCA me.

## Calc.exe Online (Web)

http://chall.ctf.bamboofox.tw:13377

author: splitline

### Solution

They gave us the source code, yay.

http://chall.ctf.bamboofox.tw:13377/?source

```html
 <?php
error_reporting(0);
isset($_GET['source']) && die(highlight_file(__FILE__));

function is_safe($query)
{
    $query = strtolower($query);
    preg_match_all("/([a-z_]+)/", $query, $words);
    $words = $words[0];
    $good = ['abs', 'acos', 'acosh', 'asin', 'asinh', 'atan2', 'atan', 'atanh', 'base_convert', 'bindec', 'ceil', 'cos', 'cosh', 'decbin', 'dechex', 'decoct', 'deg2rad', 'exp', 'floor', 'fmod', 'getrandmax', 'hexdec', 'hypot', 'is_finite', 'is_infinite', 'is_nan', 'lcg_value', 'log10', 'log', 'max', 'min', 'mt_getrandmax', 'mt_rand', 'octdec', 'pi', 'pow', 'rad2deg', 'rand', 'round', 'sin', 'sinh', 'sqrt', 'srand', 'tan', 'tanh', 'ncr', 'npr', 'number_format'];
    $accept_chars = '_abcdefghijklmnopqrstuvwxyz0123456789.!^&|+-*/%()[],';
    $accept_chars = str_split($accept_chars);
    $bad = '';
    for ($i = 0; $i < count($words); $i++) {
        if (strlen($words[$i]) && array_search($words[$i], $good) === false) {
            $bad .= $words[$i] . " ";
        }
    }

    for ($i = 0; $i < strlen($query); $i++) {
        if (array_search($query[$i], $accept_chars) === false) {
            $bad .= $query[$i] . " ";
        }
    }
    return $bad;
}

function safe_eval($code)
{
    if (strlen($code) > 1024) return "Expression too long.";
    $code = strtolower($code);
    $bad = is_safe($code);
    $res = '';
    if (strlen(str_replace(' ', '', $bad)))
        $res = "I don't like this: " . $bad;
    else
        eval('$res=' . $code . ";");
    return $res;
}
?>

<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@0.9.1/css/bulma.min.css">
    <script defer src="https://use.fontawesome.com/releases/v5.3.1/js/all.js"></script>
    <title>Calc.exe online</title>
</head>
<style>
</style>

<body>
    <section class="hero">
        <div class="container">
            <div class="hero-body">
                <h1 class="title">Calc.exe Online</h1>
            </div>
        </div>
    </section>
    <div class="container" style="margin-top: 3em; margin-bottom: 3em;">
        <div class="columns is-centered">
            <div class="column is-8-tablet is-8-desktop is-5-widescreen">
                <form>
                    <div class="field">
                        <div class="control">
                            <input class="input is-large" placeholder="1+1" type="text" name="expression" value="<?= $_GET['expression'] ?? '' ?>" />
                        </div>
                    </div>
                </form>
            </div>
        </div>
        <div class="columns is-centered">
            <?php if (isset($_GET['expression'])) : ?>
                <div class="card column is-8-tablet is-8-desktop is-5-widescreen">
                    <div class="card-content">
                        = <?= @safe_eval($_GET['expression']) ?>
                    </div>
                </div>
            <?php endif ?>
            <a href="/?source"></a>
        </div>
    </div>
</body>

</html> 1
```

So basically the application use `eval()` to evaluate the user input, which from every story told, this will not end well. That being said, developer of this application tried hard to avoid malicious inputs. Though lower case alphabets are allowed, they can only be used in some special cases like a function name. So if I enter `a` that'd be invalid, but `cos` will do. That won't stop us though, I'm not sure if it's a PHP thing or what, but we can use function names as strings, say `cos[0]` will evaluate `c`. Then that will solve some of the character issue.

But that's not the end yet, we now can only represent some lower case letters, but special characters are still a no. Luckily, in PHP you can just XOR characters, and I remember doing one of those problems in the FWordCTF. You can test things out on your own, just type `php -a` in terminal to invoke a interactive PHP shell.

Then with all the character we've found, we can start to attack the service. It sounded easy, but it actually took me hours to do this problem. I probably should have automated things in the beginning so that I wouldn't be trying characters out like a idiot.

Essentially I got a reverse shell, and read the flag that way.

![](https://y4y.space/wp-content/uploads/2021/01/screenshot-from-2021-01-16-14-41-33.png?w=1024)

I thought the flag should be in the root directory, but for some reason the command `ls -la /` just wouldn't work, and I had to get an actual reverse shell. It's a CTF so who cares.

![](https://y4y.space/wp-content/uploads/2021/01/screenshot-from-2021-01-16-14-42-53.png?w=1024)

![](https://y4y.space/wp-content/uploads/2021/01/screenshot-from-2021-01-16-14-49-43.png?w=1024)

I actually don't know this is a real world one, but I can imagine.

### Flag

flag{d0_y0u_kn0w_th1s_15_a_rea1_w0rld_cha11enge}

## Flag Checker (Reverse)

A zip file was given.

[upload](https://y4y.space/wp-content/uploads/2021/01/upload.zip)

[Download](https://y4y.space/wp-content/uploads/2021/01/upload.zip)

### Solution

This thing is written in Verilog, and I'd recommend install `iverilog` on you machine to make thing easier.

There are three files in the zip file.

#### magic.v

```verilog
module magic(
  input clk,
  input rst,
  input[7:0] inp,
  input[1:0] val,
  output reg[7:0] res
);
  always @(*) begin
    case (val)
      2'b00: res = (inp >> 3) | (inp << 5);
      2'b01: res = (inp << 2) | (inp >> 6);
      2'b10: res = inp + 8'b110111;
      2'b11: res = inp ^ 8'd55;
    endcase
  end
endmodule
```

#### chall.v

```verilog
`include "./magic.v"

module chall(
  input clk,
  input rst,
  input[7:0] inp,
  output reg[7:0] res
);
  wire[1:0] val0 = inp[1:0];
  wire[1:0] val1 = inp[3:2];
  wire[1:0] val2 = inp[5:4];
  wire[1:0] val3 = inp[7:6];
  wire[7:0] res0, res1, res2, res3;

  magic m0(.clk(clk), .rst(rst), .inp(inp), .val(val0), .res(res0));
  magic m1(.clk(clk), .rst(rst), .inp(res0), .val(val1), .res(res1));
  magic m2(.clk(clk), .rst(rst), .inp(res1), .val(val2), .res(res2));
  magic m3(.clk(clk), .rst(rst), .inp(res2), .val(val3), .res(res3));

  always @(posedge clk) begin
    if (rst) begin
      assign res = inp;
      $display(inp);
    end else begin
      assign res = res3;
    end
  end
endmodule
```

#### t_chall.v

```verilog
`timescale 1ns/10ps

`include "./chall.v"

module t_chall();
  reg clk, rst, ok;
  reg[7:0] inp, idx, tmp;
  reg[7:0] res[32:0];
  wire[7:0] out;
  wire[7:0] target[32:0], flag[32:0];

  assign {target[0], target[1], target[2], target[3], target[4], target[5], target[6], target[7], target[8], target[9], target[10], target[11], target[12], target[13], target[14], target[15], target[16], target[17], target[18], target[19], target[20], target[21], target[22], target[23], target[24], target[25], target[26], target[27], target[28], target[29], target[30], target[31]} = {8'd182, 8'd199, 8'd159, 8'd225, 8'd210, 8'd6, 8'd246, 8'd8, 8'd172, 8'd245, 8'd6, 8'd246, 8'd8, 8'd245, 8'd199, 8'd154, 8'd225, 8'd245, 8'd182, 8'd245, 8'd165, 8'd225, 8'd245, 8'd7, 8'd237, 8'd246, 8'd7, 8'd43, 8'd246, 8'd8, 8'd248, 8'd215};

  // change the content of the flag as you need
  assign flag[0] = 102;
  assign flag[1] = 108;
  assign flag[2] = 97;
  assign flag[3] = 103;
  assign flag[4] = 123;
  assign flag[5] = 116;
  assign flag[6] = 104;
  assign flag[7] = 105;
  assign flag[8] = 115;
  assign flag[9] = 95;
  assign flag[10] = 105;
  assign flag[11] = 115;
  assign flag[12] = 95;
  assign flag[13] = 102;
  assign flag[14] = 97;
  assign flag[15] = 107;
  assign flag[16] = 101;
  assign flag[17] = 95;
  assign flag[18] = 102;
  assign flag[19] = 97;
  assign flag[20] = 107;
  assign flag[21] = 101;
  assign flag[22] = 95;
  assign flag[23] = 102;
  assign flag[24] = 97;
  assign flag[25] = 107;
  assign flag[26] = 101;
  assign flag[27] = 95;
  assign flag[28] = 33;
  assign flag[29] = 33;
  assign flag[30] = 33;
  assign flag[31] = 125;

  chall ch(.clk(clk), .rst(rst), .inp(inp), .res(out));

  initial begin
    $dumpfile("chall.vcd");
    $dumpvars;

    clk = 1'b0;
    #1 rst = 1'b1;
    #1 rst = 1'b0;
    inp = flag[0];
    tmp = target[0];

    ok = 1'b1;
    for (idx = 0; idx < 32; idx++) begin
      inp = flag[idx];
      tmp = target[idx];
      #4;
    end

    if (ok) begin
      $display("ok");
    end else begin
      $display("no");
    end

    $finish;
  end

  always @(posedge clk) begin
    #1 ok = ok & (out == tmp);
  end

  always begin
    #2 clk = ~clk;
  end
endmodule
```

It's exactly a flag checker, and it does a bunch of bit manipulations on each characters.

So when a character is being checked, it takes its ASCII value as input, let's say letter `l` which has the binary representation of `01101100`. Then the input reader read the input in little endian format, so let's also change that. Now the input is `00110110`, note I just reverse the binary string. Next in the `chall.v`, it takes the every two bit in the input and assign them to `val0`, `val1`, `val2`, and `val3`. But the two bit reading format is also little endian, which means from right to left, so the order correspond to the value `vals` are `00`, `11`, `10`, `01`. If you look at the un-reversed bit string again, we can put the slice into the format of

```ini
val0 = bit_string[6:8]
val1 = bit_string[4:6]
val2 = bit_string[2:4]
val3 = bit_string[0:2]
```

Now that's clear, let's look at the `magic` file. If we translate that into a language we are more familiar with, let's Python, here is how it's gonna look like.

```python
oz = lambda x: (x + int('110111', 2)) & 0xff
oo = lambda x: x ^ 55
zz = lambda x: ((x >> 3) & 0xff) | ((x << 5) & 0xff)
zo = lambda x: ((x << 2) & 0xff) | ((x >> 6) & 0xff)
def magic(inp, val):
    res = 0
    if val=='00':
        res = zz(inp)
    elif val=='01':
        res = zo(inp)
    elif val=='10':
        res = oz(inp)
    elif val=='11':
        res = oo(inp)
    return res
```

The `& 0xff` implies the operation should be done in a 8-bit format.

So now we have a translated Python version of the file, we can bruteforce the flag. I don't know how you are supposed to reverse the flag completely, so that's how I'm going to do it.

There is one problem while I was working on the bruteforcing tho, some characters will yield the same result as the actual one. So some guessing is also involved.

But here it is, the final script.

```python
target = [182, 199, 159, 225, 210, 6, 246, 8, 172, 245, 6, 246, 8, 245, 199, 154, 225, 245, 182, 245, 165, 225, 245, 7, 237, 246, 7, 43, 246, 8, 248, 215]

bbin = lambda x: format(x, 'b').zfill(8)
ubin = lambda x: int(x[::-1], 2)

oz = lambda x: (x + int('110111', 2)) & 0xff
oo = lambda x: x ^ 55
zz = lambda x: ((x >> 3) & 0xff) | ((x << 5) & 0xff)
zo = lambda x: ((x << 2) & 0xff) | ((x >> 6) & 0xff)

def bshl(inp, bit):
    res = (inp << bit) & 0xff
    return res

def bshr(inp, bit):
    res = (inp >> bit) & 0xff
    return res

def magic(inp, val):
    res = 0
    if val=='00':
        res = zz(inp)
    elif val=='01':
        res = zo(inp)
    elif val=='10':
        res = oz(inp)
    elif val=='11':
        res = oo(inp)
    return res

def check(inp):
    bits = bbin(inp)
    val3 = bits[0:2]
    val2 = bits[2:4]
    val1 = bits[4:6]
    val0 = bits[6:8]

    res0 = magic(inp, val0)
    res1 = magic(res0, val1)
    res2 = magic(res1, val2)
    res3 = magic(res2, val3)

    return (res3)

def bf():
    flag = ''
    for i in target:
        for n in range(33, 176):
            if check(n)==i:
                flag += chr(n)
                break

    print(flag)

bf()

def check_result(flag, target):
    for i in range(32):
        if check(ord(flag[i]))!=target[i]:
            print('!!!!!!!!!!!')
    return print(f'Correct flag: {flag}')

check_result('flag{v3ry_v3r1log_f14g_ch3ck3r!}', target)
```

![](https://y4y.space/wp-content/uploads/2021/01/screenshot-from-2021-01-16-15-12-07.png?w=929)

I mean you can probably that one.

### Flag

flag{v3ry_v3r1log_f14g_ch3ck3r!}

## Flag Checker Revenge (Reverse)

[task](https://y4y.space/wp-content/uploads/2021/01/task.zip)

[Download](https://y4y.space/wp-content/uploads/2021/01/task.zip)

### Solution

This challenge gives an executable binary. After load this into Ghidra, I realized it has a million functions calling each other. And I know it wouldn't be elegant to manually reverse this one. It'd be nice if we have a way to solve this for us.

![](https://y4y.space/wp-content/uploads/2021/01/screenshot-from-2021-01-17-22-59-16.png?w=232)

And we have just the thing, this challenge is pretty to the `Beginner` in the 2020 Google CTF. And in that challenge, most people used a Python library `angr` to automatically solve the check. I basically copied [Dvd848's writeup](https://github.com/Dvd848/CTFs/blob/master/2020_GoogleCTF/Beginner.md) and modified it a bit myself.

Also here is the decompiled main function of the binary.

```c
undefined8 main(void)

{
  size_t sVar1;
  undefined8 uVar2;
  long in_FS_OFFSET;
  byte local_58 [72];
  long local_10;

  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  __isoc99_scanf(&DAT_0010a004,local_58);
  sVar1 = strlen((char *)local_58);
  if (sVar1 == 0x2b) {
    uVar2 =func_1ef462fb1a985242d6ac0c03891f65b3(local_58);
    if ((int)uVar2 != 0) {
      puts("OK, you win!");
      goto LAB_00109ad1;
    }
  }
  puts("NO, it\'s wrong!");
LAB_00109ad1:
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

And we know the flag has the length of `0x2b` which is 43. Fun fact, when I worked on this problem, I misread the length to be `0x28` and I was wondering why it didn't give me any result. After my teammate solved this one, I asked for his solution and I realized the only thing I missed was the flag length.

Then we can just start bruteforce this.

```python
import angr
import claripy

filename = './task'
FLAG_LEN = 0x2b

proj = angr.Project(filename, main_opts={'base_addr': 0x00100000})

flag_chars = [claripy.BVS('flag_%d' % i, 8) for i in range(FLAG_LEN)]
flag = claripy.Concat( *flag_chars + [claripy.BVV(b'\n')])

state = proj.factory.full_init_state(
        args = [filename],
        add_options = angr.options.unicorn,
        stdin = flag,
        )

for k in flag_chars:
    state.solver.add(k >= ord('!'))
    state.solver.add(k <= ord('~'))

simgr = proj.factory.simulation_manager(state)
good_addr = 0x00109abe
bad_addr = 0x00109acc

simgr.explore(find=good_addr, avoid=bad_addr)

if (len(simgr.found) > 0):
    for found in simgr.found:
        print(found.posix.dumps(0))
```

Notice the `good_addr` and the `bad_addr`, we need this because `angr` wants to know what is the address to find and what to avoid. I found this address by clicking on the code in Ghidra. The `good_addr` is memory address where the binary prints "OK, you win!", and the `bad_addr` is memory address of instruction `puts("No, it's wrong!")`. After that, run the script and wait.

![](https://y4y.space/wp-content/uploads/2021/01/screenshot-from-2021-01-17-23-12-09.png?w=1024)

Yep, automation is the way.

### Flag

flag{4ll_7h3_w4y_70_7h3_d33p357_v4l1d4710n}
{% endraw %}
