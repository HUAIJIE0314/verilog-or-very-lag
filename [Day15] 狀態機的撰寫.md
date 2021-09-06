# [Day15] 狀態機的撰寫

## 什麼是狀態機呢？
狀態機，其實是`有限狀態機(finite-state machine(FSM))`的簡稱，字面上來看可以知道它是有**有限**個狀態，並且可以按照著特定的順序來完成各個狀態內的動作，這使數位電路也能按著想要的順序完成特定的動作，例如某種演算法、某種傳輸協定等等。

---

## 狀態的編碼方式
常見的編碼有以下三種：
- 二進制編碼
- Gray code(格雷码)
- One Hot

---

## 三種編碼方式各自的優缺點
### 二進制編碼
優點：
- 可使用較少bit數的register來跑狀態，n個狀態只需要log2(n)個bit的register，在狀態很多時能有效節省register數量，有助於面積的減小。
- 編碼規則較簡單，採用數字遞增的方式來編碼即可。

缺點：
- 會綜合出較多的組合邏輯。
- 在狀態與狀態之間轉跳時，因為有多bit同時變化的情況發生，如此一來容易在次態的生成邏輯上產生glitch(毛刺)。
- 加上輸出與狀態有關，又二進制編碼複雜的關係造成，容易成為整體電路的critical path。

### Gray code(格雷碼)
優點：
- 一樣與二進制編碼相同，可使用較少bit數(log2(n) bit)的register來跑狀態。
- 格雷碼的特點是，上一個數字與下一個數字之間只差1bit的變化，可以改善二進制編碼中glitch的問題。

缺點：
- 雖然可以改善二進制編碼的缺點，但前提是狀態的變化是`依照順序的`，否則就失去這優勢了，但實際上幾乎不可能狀態都按著順序跳，所以這的編碼方法幾乎與二進制編碼效果差不多。
### One Hot
EX:
```verilog
//One-Hot
localparam IDLE  = 4'b0001;
localparam START = 4'b0010;
localparam SHIFT = 4'b0100;
localparam STOP  = 4'b1000;
```
優點：
- 狀態與狀態之間的變化最少1bit而最多只會有2bit的變化。
- 因為bit變化少，組合邏輯資源也會相對較少。

缺點：
- 跑狀態的register需要花費較多的bit數，若n個狀態則要n個bit的register。

**下面來看一個簡單狀態機的範例~**

```verilog=
module testFSM(
  clkSys, 
  rst_n,
  en,
  out
);
/*--------localparam--------*/
localparam one   = 2'd0;
localparam two   = 2'd1;
localparam three = 2'd2;
localparam four  = 2'd3;
/*--------ports declarations--------*/
input       clkSys;
input       rst_n;
input       en;
output [2:0]out;
reg    [2:0]out;
/*--------variables--------*/
reg    [1:0]fstate;
/*--------state--------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)fstate <= one;
  else begin
    case(fstate)
      one:begin
        if(en)fstate <= two;
        else  fstate <= one;
      end
      two:begin
        if(en)fstate <= three;
        else  fstate <= two;
      end
      three:begin
        if(en)fstate <= four;
        else  fstate <= three;
      end
      four:begin
        if(en)fstate <= one;
        else  fstate <= four;
      end
      default:fstate <= one;
    endcase
  end
end
/*--------output--------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)out <= 3'd0;
  else begin
    case(fstate)
        one:    out <= 3'd1;
        two:    out <= 3'd2;
        three:  out <= 3'd3;
        four:   out <= 3'd4;
        default:out <= 3'd0;
    endcase
  end
end
endmodule

```

**TestBench**
```verilog

`timescale 10ns/1ns
module tb_testFSM();
reg       clkSys;
reg       rst_n;
reg       en;
wire [2:0]out;
testFSM UUT(
  .clkSys(clkSys), 
  .rst_n(rst_n),
  .en(en),
  .out(out)
);

initial begin
  clkSys = 0;
  rst_n = 0;
  en = 0;
  repeat(2)@(posedge clkSys)rst_n = 0;
  rst_n = 1;
  en = 1;
  #20  en = 0;
  #40;
  en = 1;
  #20  en = 0;
  #40;
  en = 1;
  #20  en = 0;
  #40;
  en = 1;
  #20  en = 0;
  #100 $stop;
end

always #10 clkSys = ~clkSys;

endmodule

```

**Wave**

![](https://i.imgur.com/yggFHbt.png)

**state machine**

![](https://i.imgur.com/TO4UA5r.png)


> 上面的例子，狀態是以二進制編碼來表示的，我自己的狀態機是以兩段式為主，並且輸出邏輯為了timing更好，在output logic會再多敲一級clk(循序邏輯)，可以看到一個always處理狀態的轉移，而另一個always負責在該狀態輸出對應的信號，在這個例子中，在每一個狀態內都要在clk為正緣時並且讀到en為"1"時才會進入下一個狀態，否則留在原狀態等待en的來臨。
