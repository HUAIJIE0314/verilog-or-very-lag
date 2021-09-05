
# [Day05] Gate Level
## 一些基本邏輯閘
![](https://i.imgur.com/OOIKYkm.jpg)
[圖片出處](https://frankcomputerscience.wordpress.com/chapter-3/)

---

## 語法
```verilog
<邏輯閘種類> <邏輯閘命名> (output, in1, in2);
```
- 邏輯閘種類：AND、OR、NOT....。
- 邏輯閘命名：賦予這個邏輯閘一個名子，`名子不可重複。`

EX:

**半加器：**

![](https://i.imgur.com/yqIuFAt.png)


```verilog
//Half adder
module Half_adder(
  a, 
  b, 
  carry, 
  sum
);

input a;
input b;
output carry;
output sum;
and andl(carry, a, b);//AND gate carry = a and b
xor xor1(sum, a, b);//XOR gate sum = a xor b

endmodule
```
**全加器：**

![](https://i.imgur.com/VScAo8t.png)



```verilog
//Full_adder
module Full_adder(
  x, 
  y, 
  c_in, 
  c_out, 
  s
);
input x;
input y;
input c_in;
output c_out;
output s;
wire sl;
wire cl;
wire c2;
Half_adder HAl(x, y, cl, sl);//instantiate Half_adder HAI
Half_adder HA2(c_in, sl, c2, s);//instantiate Half_adder HA2
or(c_out, cl, c2);//OR gate c_out = cl | c2

endmodule
```
**四位元加法器：**

![](https://i.imgur.com/z0oEZEM.png)
```verilog
module Full_adder_FourBits(
  a_in, 
  b_in, 
  c_in, 
  carry_out, 
  sum_out
);
input  [3:0] a_in;
input  [3:0] b_in;
input        c_in;
output [3:0] sum_out;
output       carry_out;
wire   [3:0] sum_out; 
wire carry_out;
wire c1;
wire c2;
wire c3;

Full_adder FA1(a_in[0], b_in[0], c_in, sum_out[0], c1);//instantate FA FA1
Full_adder FA2(a_in[1], b_in[1], c1, sum_out[1], c2);//instantate FA FA2
Full_adder FA3(a_in[2], b_in[2], c2, sum_out[2], c3);//instantate FA FA3
Full_adder FA4(a_in[3], b_in[3], c3, sum_out[3], carry_out);//instantate FA FA4

endmodule

```
