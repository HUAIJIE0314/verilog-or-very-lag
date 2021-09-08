# [Day22] I2C Master狀態機的實現
由於上一篇已經接紹過了SPI的Timing Diagram，那麼今天就直接進入I2C Master狀態機的程式嘍~~

## 設計I2C狀態機
先來定義8個狀態(下面會解釋各個狀態的行為)
- IDLE
- GO
- START
- WAIT
- SHIFT
- STOP
- FINAL
- END

```verilog
/*---------parameter---------*/
parameter IDLE  = 3'd0;
parameter GO    = 3'd1;
parameter START = 3'd2;
parameter WAIT  = 3'd3;
parameter SHIFT = 3'd4;
parameter STOP  = 3'd5;
parameter FINAL = 3'd6;
parameter END   = 3'd7;
```

**再來定義輸入輸出**
**輸入：**
- clk_sys
- rst_n
- SCLK_100k(輸入已經生成好的SCLK)
- en(狀態機 enable)
- tick_I2C
- count(計數移位18 or 27次的計數器，由外模組來數所以這裡是輸入)
- R_W(我這次狀態機跑的次數沒有固定，分別為write模式的18次((8+1)bitx2)，read模式的27次((8+1)bitx3)，低電位是write，高電位是read)

**輸出：**
- ACK1, ACK2, ACK3(為一時，讓外部模組知道此時要收ACK)
- SCLK_temp(負責除了傳輸時以外的SCLK訊號控制，會利用SHEN判斷搭配assign去做切換)
- LDEN(LoadEnable，告訴外部模塊要Load資料)
- SHEN(ShiftEnable，告訴外部模塊要移位)
- rstcount(reset count，告訴外部模塊將計數器歸零)
- countEN(CountEnable，告訴外部模塊將計數器往上計數)
- SDO(除了資料的SDA以外的SDA控制，也是會利用SHEN判斷搭配assign去做切換)
- rstACK(因為是透過外部暫存器存放ACK狀態，因此也需要reset這些暫存器的時機，因此需要rstACK，會在最後通訊結束時拉到"1")
- SCLK(最後總SCLK輸出)

```verilog
module I2C_control_R_W(
  clk_sys, 
  SCLK_100k, 
  tick_I2C, 
  rst_n, 
  en, 
  count, 
  countEN,
  rstcount, 
  ACK1,
  ACK2, 
  ACK3, 
  rstACK, 
  SCLK, 
  SCLK_temp,  
  SHEN, 
  LDEN, 
  SDO, 
  R_W
);
/*---------ports declaration---------*/
input       clk_sys;
input       rst_n;
input       en;
input       SCLK_100k;
input       tick_I2C;
input       R_W;// W---->0, R---->1
input [4:0] count;
output      ACK1; 
output      ACK2; 
output      ACK3; 
output      SCLK_temp; 
output      SHEN; 
output      LDEN; 
output      SDO; 
output      countEN; 
output      rstcount; 
output      rstACK;
output      SCLK;
reg         ACK1; 
reg         ACK2; 
reg         ACK3; 
reg         SCLK_temp; 
reg         SHEN; 
reg         LDEN; 
reg         SDO; 
reg         countEN; 
reg         rstcount; 
reg         rstACK;
wire        SCLK;
```


**宣告跑狀態的變數：**

```verilog
/*---------variables---------*/	
reg [2:0] fstate;
```

**狀態邏輯：**
- 重置時要能回到IDLE狀態。
- IDLE(收到en(enable)訊號後開始到下一個狀態)
- GO(只有一個傳輸週期，用來Load資料用)
- START(只有一個傳輸週期，用來Load資料用，但這裡發生START訊號)
- WAIT(等待一個傳輸週期)
- SHIFT(count數滿18 or 27次後並且最後一次也要待滿一個傳輸週期，狀態才往STOP)
- STOP(在此狀態會停止所有動作)
- FINAL(這裡開始要產生STOP訊號)
- END(完成STOP訊號的產生，接著再回到IDLE)

```verilog
/*---------fstate state---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)begin
    fstate <= IDLE;
  end
  else begin
    case(fstate)
      IDLE:begin
        if(en) fstate <= GO;
        else   fstate <= IDLE;
      end
      GO:begin
        if(tick_I2C)fstate <= START;
        else        fstate <= GO;
      end
      START:begin
        if(tick_I2C)fstate <= WAIT;
        else        fstate <= START;
      end
      WAIT:begin
        if(tick_I2C)fstate <= SHIFT;
        else        fstate <= WAIT;
      end
      SHIFT:begin
        if(!R_W)begin//write-->18bit
        if(count==5'd16&&tick_I2C)fstate <= STOP;
        else                      fstate <= SHIFT;
      end
      else begin//read-->27bit
        if(count==5'd25&&tick_I2C)fstate <= STOP;
        else                      fstate <= SHIFT;
      end
      end
      STOP:begin
        if(tick_I2C)fstate <= FINAL;
        else        fstate <= STOP;
      end
      FINAL:begin
        if(tick_I2C)fstate <= END;
        else        fstate <= FINAL;
      end
      END:begin
        if(tick_I2C)fstate <= IDLE;
        else        fstate <= END;
      end
    endcase
  end
end
```

**輸出邏輯：**
- IDLE(閒置狀態，SDA及SCLK都為"1")
- GO(在這裡依舊SDA及SCLK都為"1"，但會在此狀態Load資料)
- START(這裡發生START訊號，SDA此時為"0"，SCLK則維持)
- WAIT(等待一個傳輸週期，SDA及SCLK維持，與上個狀態相同)
- SHIFT(在此狀態的SDA及SCLK值不影響，會assign到外部模塊，主要做資料的移位以及在第9、18、27 bit接收ACK)
- STOP(在此狀態SDA及SCLK都先拉為"0"，停止所有動作)
- FINAL(這裡開始要產生STOP訊號，因此SCLK先拉到"1")
- END(最後SDA拉為"1"，產生STOP訊號(完成一次通訊)，接著再回到IDLE)


```verilog
/*---------fstate output---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)begin
    SCLK_temp <= 1'b1;
    LDEN      <= 1'b0;
    SHEN      <= 1'b0;
    ACK1      <= 1'b0;
    ACK2      <= 1'b0;
    ACK3      <= 1'b0;
    SDO       <= 1'b1;//SDIN_temp control data[26]/raising/falling
    countEN   <= 1'b0;
    rstcount  <= 1'b0;
    rstACK    <= 1'b0;
  end
  else begin
    if(tick_I2C)begin
      case(fstate)
        IDLE:begin
          SCLK_temp <= 1'b1;//high
          LDEN      <= 1'b0;
          SHEN      <= 1'b0;
          ACK1      <= 1'b0;
          ACK2      <= 1'b0;
          ACK3      <= 1'b0;
          SDO       <= 1'b1;//high
          countEN   <= 1'b0;
          rstcount  <= 1'b0;
          rstACK    <= 1'b0;
        end
        GO:begin
          SCLK_temp <= 1'b1;//high
          LDEN      <= 1'b1;//load data
          SHEN      <= 1'b0;
          ACK1      <= 1'b0;
          ACK2      <= 1'b0;
          ACK3      <= 1'b0;
          SDO       <= 1'b1;//high
          countEN   <= 1'b0;
          rstcount  <= 1'b0;
          rstACK    <= 1'b0;
        end
        START:begin
          SCLK_temp <= 1'b1;//high
          LDEN      <= 1'b0;
          SHEN      <= 1'b0;
          ACK1      <= 1'b0;
          ACK2      <= 1'b0;
          ACK3      <= 1'b0;
          SDO       <= 1'b0;//falling(start)
          countEN   <= 1'b0;
          rstcount  <= 1'b0;
          rstACK    <= 1'b0;
        end
        WAIT:begin
          SCLK_temp <= 1'b1;//high
          LDEN      <= 1'b0;
          SHEN      <= 1'b0;
          ACK1      <= 1'b0;
          ACK2      <= 1'b0;
          ACK3      <= 1'b0;
          SDO       <= 1'b0;//low
          countEN   <= 1'b0;
          rstcount  <= 1'b0;
          rstACK    <= 1'b0;
        end
        SHIFT:begin
          SCLK_temp <= 1'b0;//don't care
          LDEN      <= 1'b0;
          SHEN      <= 1'b1;//shifting
          //ACK1
          if(count==5'd7) ACK1 <= 1'b1;
          else            ACK1 <= 1'b0;
          //ACK2
          if(count==5'd16)ACK2 <= 1'b1;
          else            ACK2 <= 1'b0;
          //ACK3
          if(count==5'd25)ACK3 <= 1'b1;
          else            ACK3 <= 1'b0;
          SDO       <= 1'b1;//don't care(data[26])
          countEN   <= 1'b1;//counting
          //rstcount
          if(!R_W)begin//write-->18bit
            if(count==5'd16)rstcount  <= 1'b1;
            else            rstcount  <= 1'b0;
          end
          else begin//read-->27bit
            if(count==5'd25)rstcount  <= 1'b1;
            else            rstcount  <= 1'b0;
          end
          rstACK    <= 1'b0;
        end
        STOP:begin
          SCLK_temp <= 1'b0;//stop the clock
          LDEN      <= 1'b0;
          SHEN      <= 1'b0;
          ACK1      <= 1'b0;
          ACK2      <= 1'b0;
          ACK3      <= 1'b0;
          SDO       <= 1'b0;//low
          countEN   <= 1'b0;
          rstcount  <= 1'b0;
          rstACK    <= 1'b0;
        end
        FINAL:begin
          SCLK_temp <= 1'b1;//high
          LDEN      <= 1'b0;
          SHEN      <= 1'b0;
          ACK1      <= 1'b0;
          ACK2      <= 1'b0;
          ACK3      <= 1'b0;
          SDO       <= 1'b0;//low
          countEN   <= 1'b0;
          rstcount  <= 1'b0;
          rstACK    <= 1'b0;
        end
        END:begin
          SCLK_temp <= 1'b1;//high
          LDEN      <= 1'b0;
          SHEN      <= 1'b0;
          ACK1      <= 1'b0;
          ACK2      <= 1'b0;
          ACK3      <= 1'b0;
          SDO       <= 1'b1;//raising
          countEN   <= 1'b0;
          rstcount  <= 1'b0;
          rstACK    <= 1'b1;//reset ACK
        end
      endcase
    end
    else begin
      SCLK_temp <= SCLK_temp;
      LDEN      <= LDEN;
      SHEN      <= SHEN;
      ACK1      <= ACK1;
      ACK2      <= ACK2;
      ACK3      <= ACK3;
      SDO       <= SDO;
      countEN   <= countEN;
      rstcount  <= rstcount;
      rstACK    <= rstACK;
    end
  end
end

endmodule
```


這次的I2C砍起來雖然好像複雜很多，但其實與前面幾篇的Uart及SPI都有很大的相似之處歐~~

下一篇我們會繼續來完成I2C的模組~！


