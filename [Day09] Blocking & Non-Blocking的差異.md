
# [Day09] Blocking & Non-Blocking的差異
## Blocking vs Non-Blocking
在寫一般軟體語言時，都與verilog中的blocking語句相同，是一行一行由上至下執行的，但verilog又有一個non-blocking，而non-blocking則是同步執行的，而撰寫時也要把握以下原則，才不會產生誤用的情形。
- always用clock觸發的區塊要使用nonblocking。
- always沒有用clock觸發(組合邏輯)的區塊要使用blocking。
- assign語句一律使用blocking。
- 在同一個always block中不可同時出現nonblocking及blocking。

### 來看下面這個例子
`non-blocking:`
```
module blockingVSnonblocking(
  clkSys, 
  rst_n
);
input clkSys;
input rst_n;
reg [1:0]a;
reg [1:0]b;
reg [1:0]c;
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    a <= 2'd1;
    b <= 2'd2;
    c <= 2'd3;
  end
  else begin
    a <= c;
    b <= a;
    c <= b;
  end
end
endmodule

```
`testbench:`
```
`timescale 10ns/1ns
module tb();
reg clkSys;
reg rst_n;

blockingVSnonblocking UUT(
  .clkSys(clkSys), 
  .rst_n(rst_n)
);

initial begin
  clkSys = 0;
  rst_n = 0;
  repeat(3)@(posedge clkSys)rst_n = 0;
  rst_n = 1;
  #100 $stop;
end

always #5 clkSys = ~clkSys;

endmodule

```
上方的執行結果會是
![](https://i.imgur.com/G5L0rGh.png)

>reset後，clk正緣來臨時，程式會同步的執行，所以{a, b, c}的值會同步到{c, b, a}。

---

`blocking:`
```
module blockingVSnonblocking(
  clkSys, 
  rst_n
);
input clkSys;
input rst_n;
reg [1:0]a;
reg [1:0]b;
reg [1:0]c;
always@(*)begin
  if(!rst_n)begin
    a = 2'd1;
    b = 2'd2;
    c = 2'd3;
  end
  else begin
    a = c;
    b = a;
    c = b;
  end
end
endmodule
```
而上方的執行結果則會是
![](https://i.imgur.com/mJ5k6AT.png)

>可以看出{a, b, c}最後都變成了2'd3，這就是使用了non-blocking，使程式變得有順序性，第一行a=c，導致第二行b=a時也等同b=c了，最後一行也是如此。

