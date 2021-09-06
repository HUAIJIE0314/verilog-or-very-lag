## Uart是什麼？

UART(Universal Asynchronous Receiver/Transmitter)，是一種非同步的傳輸協定，非同步傳輸的意思是，不管是接收端還是傳送端都有自己傳輸資料的速度(鮑率(Baud Rate))，傳輸時兩邊必須以同樣的包率來收發資料才不會出錯。

UART它的好處是線路簡單，僅兩條線路(RX&TX)，但缺點是只能一對一連接，以及速度不是很快，一般而言最高為115.2kbps(常規鮑率為9600)

---

## Uart 的 Timing Diagram
![](https://i.imgur.com/uxKwVRk.png)

[圖片出處](https://zh.wikipedia.org/wiki/UART)

從圖片中可以看到，Uart的閒置狀態(IDLE)是高電平的，而起始位元則是1 bit的低電平，再來是8 bit資料的傳輸，而且是由LSB開始傳輸的，最後則是
1 bit的高電平當作是結束位元。

例如現在要傳輸一個0x55(0101_0101)的資料：

![](https://i.imgur.com/PT4RuA8.gif)

[圖片出處](https://www.fpga4fun.com/SerialInterface1.html)

---

## 設計Uart狀態機
我們剛剛看完了timing diagram後發現，Uart大致可以切成四個狀態，分別為IDLE(閒置狀態)、START(起始位元)、SHIFT(8bit的輸出移位)以及STOP(停止位元)。

定義四個狀態：
```verilog
/*---------parameter---------*/
localparam IDLE  = 2'd0;
localparam START = 2'd1;
localparam SHIFT = 2'd2;
localparam STOP  = 2'd3;
```
再來定義輸入輸出，記得!這邊還不是最外層的Uart模塊，別搞混了(想整個寫在一起也很好！)。

**輸入：**
- clk_50M
- rst_n
- en(狀態機 enable)
- tick_uart(9600鮑率的tick(因為是同步式設計，所以不能用自己除出來的頻率來當作always觸發)，由外模塊發送)
- count(計數移位8次的計數器，由外模組來數所以這裡是輸入)

**輸出：**

- LDEN(LoadEnable，告訴外部模塊要Load資料)
- SHEN(ShiftEnable，告訴外部模塊要移位)
- rstcount(reset count，告訴外部模塊將計數器歸零)
- countEN(CountEnable，告訴外部模塊將計數器往上計數)
- busy(告訴外部模塊現在處於非IDLE狀態)
```verilog
module Uart_TX(
  tick_uart, 
  clk_50M, 
  rst_n, 
  count, 
  rstcount, 
  countEN, 
  TX_D, 
  LDEN, 
  SHEN, 
  en, 
  busy
);
/*---------ports declaration---------*/
input       clk_50M;
input       rst_n;
input       en;
input       tick_uart;
input [2:0] count;
output      TX_D;
output      LDEN;
output      SHEN;
output      rstcount;
output      countEN;
output      busy;
reg         TX_D;
reg         LDEN;
reg         SHEN;
reg         rstcount;
reg         countEN;
wire        busy;
```

```verilog
/*---------assign wire---------*/
assign busy = (fstate!=IDLE);
```

**宣告跑狀態的變數：**

```verilog
/*---------variables---------*/	
reg [1:0] fstate;
```
**狀態邏輯：**
- 重置時要能回到IDLE狀態。
- IDLE(收到enable訊號後開始到下一個狀態)
- START(只有一個鮑率的週期(1 bit的低電平)，所以等待9600的tick來後就往SHIFT走)
- SHIFT(count數滿八次後並且最後一次也要待滿一個鮑率的時間，狀態才往STOP走)
- STOP(跟START相同，要待滿一個鮑率週期，再回到IDLE)

```verilog
/*---------fstate state---------*/
always@(posedge clk_50M or negedge rst_n)begin
  if(!rst_n)fstate <= IDLE;
  else begin
    case(fstate)
      IDLE:begin
        if(en)fstate <= START;
        else  fstate <= IDLE;
      end
      START:begin
        if(tick_uart==1'b1)fstate <= SHIFT;
        else               fstate <= START;
      end
      SHIFT:begin
        if(tick_uart==1'b1&&count==3'd7)fstate <= STOP;
        else                            fstate <= SHIFT;
      end
      STOP:begin
        if(tick_uart==1'b1)fstate <= IDLE;
        else               fstate <= STOP;
      end
      default:fstate <= IDLE;
    endcase
  end
end
```

**輸出邏輯：**
- 重置時
  - TX_D = 1(閒置狀態為1，故重置也是1)
  - 其餘都是0
- IDLE
  - TX_D = 1(閒置狀態為1，故重置也是1)
  - 其餘都是0
- START
  - 此時要Load要傳的8 bit資料，所以LDEN = 1
  - 其餘都是0
- SHIFT
  - 這個狀態要開始數bit數，也要在最後一個bit將計數暫存器歸零。
  - countEN = 1
  - 在最後一bit時countEN = 1
  - 移位訊號SHEN = 1
  - 此時TX_D不重要，此模塊沒有8 bit的傳輸資料，移位時會assign到外部模塊上
- STOP
  - 結束位元是一，TX_D = 1
  - 其餘都是0
```verilog
/*---------fstate output---------*/
always@(posedge clk_50M or negedge rst_n)begin
  if(!rst_n)begin
    TX_D     <= 1'b1;
    SHEN     <= 1'b0;
    LDEN     <= 1'b0;
    rstcount <= 1'b0;
    countEN  <= 1'b0;
  end 
  else begin
    if(tick_uart==1'b1)begin
      case(fstate)
        IDLE:begin
          TX_D     <= 1'b1;
          SHEN     <= 1'b0;
          LDEN     <= 1'b0;
          rstcount <= 1'b0;
          countEN  <= 1'b0;
        end
        START:begin
          TX_D     <= 1'b0;
          SHEN     <= 1'b0;
          LDEN     <= 1'b1;
          rstcount <= 1'b0;
          countEN  <= 1'b0;
        end
        SHIFT:begin
          TX_D     <= 1'b0;
          SHEN     <= 1'b1;
          LDEN     <= 1'b0;
          if(count==3'd7)    rstcount <= 1'b1;
          else if(count<3'd7)rstcount <= 1'b0;
          else               rstcount <= 1'b0;//prevent latch
          countEN  <= 1'b1;
        end
        STOP:begin
          TX_D     <= 1'b1;
          SHEN     <= 1'b0;
          LDEN     <= 1'b0;
          rstcount <= 1'b0;
          countEN  <= 1'b0;
        end
        default:begin
          TX_D     <= 1'b1;
          SHEN     <= 1'b0;
          LDEN     <= 1'b0;
          rstcount <= 1'b0;
          countEN  <= 1'b0;
        end
      endcase
    end
    else begin
      TX_D     <= TX_D;
      SHEN     <= SHEN;
      LDEN     <= LDEN;
      rstcount <= rstcount;
      countEN  <= countEN;
    end
  end
end
endmodule
```
以上就是我整個Uart_TX狀態機設計的方法~~
