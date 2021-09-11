# [Day30] Pipelined加法器

## 什麼是Pipelined？

先以RISC-V架構來舉例：

我們先來看看這張圖：

![](https://i.imgur.com/jFXKZqG.png)

圖片出處：源自[Computer Organization and Design RISC-V edition](http://home.ustc.edu.cn/~louwenqi/reference_books_tools/Computer%20Organization%20and%20Design%20RISC-V%20edition.pdf)這本書中的524頁

我們先來看上面那張圖：

一道指令從進來到執行結束一共得花800ps，而上一筆指令執行完下一道指令才會繼續進來，那麼Pipeline的意思就是把硬體共享最大化，例如一道指令要經歷五個步驟才能做完的話，那麼在前一道指令做到第二步驟時，其實第一個步驟的硬體是閒置的，這樣並沒有達到資源最大利用，也就是說，其實在此時下一道指令就可以先進來了，如此一來就會像是下圖那樣，先解釋一下為什麼上面800ps切成五份後會變成200ps，不是應該是160ps嗎？

因為每一階的執行時間必須一樣，也就是要以執行時間最久的那階為準(Critical Path)，可以看到Reg那階，其實只要100ps就執行完了，但還是得等滿200ps因此有了上圖800ps與下圖卻總共要1000ps的差異，所以做了pipeline並不會提高每一道指令的總執行時間，卻可以提高Throughput，可以看到沒有做pipeline的話要每800ps才能執行完一道指令，而有做pipeline則在1000ps以後就可以每過200ps就完成一道指令~~

那麼今天要來完成的pipeline加法器背後的想法也是類似這樣，因為越多位元的加法器可能所需的時間就越久，所以我們可以把它們拆成幾個等分，最後再一起輸出結果，如此一來就可以提高throughput。

但是做pipeline還有一點是，每一階之間都需要暫存器來存放上一階的執行結果。

那麼我們來試著寫個切一階pipeline的加法器吧~

**宣告輸入輸出**

```verilog
module pipelinedAdderOneStage#(
/*--------parameter-------*/
parameter width    = 19,
          widthLsb = 9,
          widthMsb = 10)
(
  clkSys, 
  rst_n, 
  x, 
  y, 
  sum
);
/*--------ports declaration-------*/
input             clkSys;
input             rst_n;
input  [width-1:0]x;
input  [width-1:0]y;
output [width-1:0]sum;
wire   [width-1:0]sum;
```

**宣告中間的暫存變數**

```verilog
/*--------variables-------*/
//lsb
reg    [widthLsb-1:0]lsbTemp1; //9bit
reg    [widthLsb-1:0]lsbTemp2; //9bit
reg      [widthLsb:0]sumTemp1; //10bits(9bits sum of lsb & 1-bit carry)
reg    [widthLsb-1:0]sumOutLsb;//9bit
//msb
reg    [widthMsb-1:0]msbTemp1; //10bit
reg    [widthMsb-1:0]msbTemp2; //10bit
reg    [widthMsb-1:0]sumTemp2; //10bit
reg    [widthMsb-1:0]sumOutMsb;//10bit
```

再lsb有一個10bit變數是因為可能低位元加完後會有carry項，不能漏掉了

**將輸入變數切成高位元及低位元**

```verilog
/*--------catch data-------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    lsbTemp1 <= 'd0;
    lsbTemp2 <= 'd0;
    msbTemp1 <= 'd0;
    msbTemp2 <= 'd0;
  end
  else begin
    lsbTemp1 <= x[widthLsb-1:0];//8~0 bits
    lsbTemp2 <= y[widthLsb-1:0];//8~0 bits
    msbTemp1 <= x[widthMsb+widthLsb-1:widthLsb];//18~9 bits
    msbTemp2 <= y[widthMsb+widthLsb-1:widthLsb];//18~9 bits
  end
end
```

**第一階加法器(LSB與MSB先各自相加)**

```verilog
/*--------first stage or the adder-------*/
//add lsb
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)sumTemp1 <= 'd0;
  else      sumTemp1 <= {1'b0, lsbTemp1}+{1'b0, lsbTemp2};
end
//add msb
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)sumTemp2 <= 'd0;
  else      sumTemp2 <= msbTemp1 + msbTemp2;
end
```

**第二階加法器**
- LSB的話直接拿掉carry項然後輸出
- MSB要加上剛剛LSB的carry項


```verilog
/*--------second stage or the adder-------*/
//sum of Lsb
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)sumOutLsb <= 'd0;
  else      sumOutLsb <= sumTemp1[widthLsb-1:0];
end
//sum of Msb
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)sumOutMsb <= 'd0;
  else      sumOutMsb <= sumTemp1[widthLsb] + sumTemp2;
end

endmodule
```


---

## TestBench

```verilog

`timescale 1ns/1ns
module tb_adder();

localparam width    = 19;
localparam widthLsb = 9;
localparam widthMsb = 10;

reg             clkSys;
reg             rst_n;
reg  [width-1:0]x;
reg  [width-1:0]y;
wire [width-1:0]sum;
integer i, j;

pipelinedAdderOneStage UUT(
  .clkSys(clkSys), 
  .rst_n(rst_n), 
  .x(x), 
  .y(y), 
  .sum(sum)
);
//variables
initial begin
  i = 0;
  j = 0;
end
//input port
initial begin
  x = 0;
  y = 0;
  rst_n = 0;
  clkSys = 0;
end

always #10 clkSys = ~clkSys;//50MHZ

initial begin
  repeat(5)@(posedge clkSys)rst_n = 0;
  rst_n = 1;
  for(i=0;i<=10;i=i+1)begin
    for(j=0;j<=10;j=j+1)begin
      y = y + 1;
      #100;
    end
    x = x + 1;
  end
  #100 $stop;
end

initial begin
  $monitor("time=%3d, %3d + %3d = %3d",$time, x, y, sum);
end

endmodule

```

**Wave**

![](https://i.imgur.com/LRZPY0J.png)

可以看到加法功能正確(你們可以去看看有carry的部分有沒有功能正常)

**RTL Viewer**

![](https://i.imgur.com/vDSkoHt.png)

你們也可以寫切更多層的試試看歐~~

那麼30天也很快的過去了，希望對你在學習verilog這條路上多少有些幫助~~!

