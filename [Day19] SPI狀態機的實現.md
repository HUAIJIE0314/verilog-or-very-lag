# [Day19] SPI狀態機的實現

## SPI是什麼？
SPI(Serial Peripheral Interface)，是一種同步的傳輸協定，主要應用於單晶片系統中。類似I2C(之後會提到)，它的應用有快閃記憶體、EEPROM、SD卡與顯示器等等....

![](https://i.imgur.com/vjiueVR.png)

[圖片出處](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface)

---

## 介面
SPI協定規定了4個邏訊號介面(也有三線的，但那就是單工了)：
- SCLK(Serial Clock，會由master端發出)
- MOSI(Master Out，Slave In)
- MISO(Master In， Slave Out)
- CS(Chip Select，因為一個master可以跟數個slave做通訊，因此需要CS來選擇要通訊的slave，而通常CS為低電位致能)

---

## SPI 的 Timing Diagram
開始傳輸資資料時，CS要拉低，而我的模式是：mosi在SCLK的負緣時值會改變，而miso則是在SCLK正緣時值會改變，與Uart不同的是：SPI並沒有起始位元，slave單純靠SCLK的正負緣讀取資料(正或負要看如何設計的)，在這邊，資料位寬的話我是做16bit;傳輸頻率1MHZ，因為想說可以試著與max7219做通訊，有興趣的可以看一下max7219的datasheet。

---

## 設計SPI狀態機
跟UART一樣，先定義四個狀態：
```verilog
/*---------parameter---------*/
localparam IDLE  = 2'd0;
localparam START = 2'd1;
localparam SHIFT = 2'd2;
localparam STOP  = 2'd3;
```

**再來定義輸入輸出**
**輸入：**
- clk_sys
- rst_n
- SCLK_temp
- ready(狀態機 enable)
- tick_SPI
- count(計數移位16次的計數器，由外模組來數所以這裡是輸入)

**輸出：**
- CS(Chip Select)
- SCLK(總輸出的SCLK)
- LDEN(LoadEnable，告訴外部模塊要Load資料)
- SHEN(ShiftEnable，告訴外部模塊要移位)
- rstcount(reset count，告訴外部模塊將計數器歸零)
- countEN(CountEnable，告訴外部模塊將計數器往上計數)
- finish(告訴外部模塊現在傳輸結束了)

```verilog
module SPI(
  clk_sys, 
  SCLK_temp, 
  tick_SPI, 
  rst_n, 
  SCLK, 
  CS, 
  count, 
  countEN, 
  rstcount, 
  ready, 
  finish, 
  SHEN, 
  LDEN
);
/*-----------parameter-----------*/
parameter IDLE  = 2'd0;
parameter START = 2'd1;
parameter SHITF = 2'd2;
parameter STOP  = 2'd3;
/*-----------in/output-----------*/
input       clk_sys; 
input       SCLK_temp; 
input       rst_n;
input       ready;
input       tick_SPI;
input [3:0] count;
output      CS; 
output      countEN; 
output      rstcount; 
output      SHEN; 
output      LDEN;
output      finish;
output      SCLK;
reg         CS; 
reg         countEN; 
reg         rstcount; 
reg         SHEN; 
reg         LDEN;
reg         finish;
wire        SCLK;
```

```verilog
/*-----------finish-----------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)begin
    finish <= 1'b0;
  end
  else begin
    if(fstate==STOP&&tick_SPI)finish <= 1'b1;
    else                      finish <= 1'b0;
  end
end
```
**宣告跑狀態的變數：**

```verilog
/*---------variables---------*/	
reg [1:0] fstate;
```

**狀態邏輯：**
- 重置時要能回到IDLE狀態。
- IDLE(收到ready(enable)訊號後開始到下一個狀態)
- START(只有一個傳輸週期，用來Load資料用)
- SHIFT(count數滿15次後並且最後一次也要待滿一個傳輸週期，狀態才往STOP)
- STOP(再次回到IDLE，等待下一次傳輸)

```verilog
/*-----------state-----------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)fstate <= IDLE;
  else begin
    case(fstate)
      IDLE:begin
        if(ready)fstate <= START;
        else     fstate <= IDLE;
      end
      START:begin
        if(tick_SPI)fstate <= SHITF;
        else        fstate <= START;
      end
      SHITF:begin
        if(count==4'd14&&tick_SPI)fstate <= STOP;
        else                      fstate <= SHITF;
      end
      STOP:begin
        if(tick_SPI)fstate <= IDLE;
        else        fstate <= STOP;
      end
      default:fstate <= IDLE;
    endcase
  end
end
```

**輸出邏輯：**
- 重置時
  - CS = 1(不選晶片)
  - 其餘都是0
- IDLE
  - CS = 1(不選晶片)
  - 其餘都是0
- START
  - 此時要Load要傳的16 bit資料，所以LDEN = 1
  - 此時CS要拉到0，致能slave
  - 其餘都是0
- SHIFT
  - 這個狀態要開始數bit數，也要在最後一個bit將計數暫存器歸零。
  - countEN = 1
  - 在最後一bit時countEN = 1
  - 移位訊號SHEN = 1
- STOP
  - CS再拉回1(不選晶片)
  - 其餘都是0

```verilog
/*-----------output-----------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)begin
    CS       <= 1'b1;
    countEN  <= 1'b0;
    rstcount <= 1'b0;
    SHEN     <= 1'b0;
    LDEN     <= 1'b0;
  end
  else begin
    if(tick_SPI)begin
      case(fstate)
        IDLE:begin
          CS       <= 1'b1;
          countEN  <= 1'b0;
          rstcount <= 1'b0;
          SHEN     <= 1'b0;
          LDEN     <= 1'b0;
        end
        START:begin
          CS       <= 1'b0;
          countEN  <= 1'b0;
          rstcount <= 1'b0;
          SHEN     <= 1'b0;
          LDEN     <= 1'b1;
        end
        SHITF:begin
          CS       <= 1'b0;
          countEN  <= 1'b1;
          if(count==4'd14)    rstcount  <= 1'b1;
          else if(count<4'd14)rstcount  <= 1'b0;
          else                rstcount  <= 1'b0;//prevent latch
          SHEN     <= 1'b1;
          LDEN     <= 1'b0;
        end
        STOP:begin
          CS       <= 1'b1;
          countEN  <= 1'b0;
          rstcount <= 1'b0;
          SHEN     <= 1'b0;
          LDEN     <= 1'b0;
        end
        default:begin
          CS       <= 1'bx;
          countEN  <= 1'bx;
          rstcount <= 1'bx;
          SHEN     <= 1'bx;
          LDEN     <= 1'bx;
        end
      endcase
    end
    else begin
      CS       <= CS;
      countEN  <= countEN;
      rstcount <= rstcount;
      SHEN     <= SHEN;
      LDEN     <= LDEN;
    end
  end
end

endmodule
```

這裡也一樣，要數到15卻14就結束是因為外部模組會延後一個clk，如果15才停，那外部模組它數到16才停下，shift的字數也會多一次，這種情況是我們不樂見的~！

那麼今天的的教學就到這邊，下一篇我們會完成整個SPI模組~~~~
