# [Day24] I2C Master的實現及驗證(最終章)

今天，我們要來完成整個I2C的最後一個部份了！
**先來看看這個I2C Master write模塊該有哪些輸入輸出腳吧：**

**輸入：**
- clk_sys
- rst_n
- en(外部給此模組的驅動信號(enable))
- addr(8 bit)
- data_i(8 bit)

(如果是write模式，那前8 bit傳address，而後8 bit傳data_i)

**輸出：**
- data_o(16 bit)
- SCLK

(如果是read模式，那前8 bit傳address，而後面都傳"1"(高阻抗)，因為要接收data所以斷開，由slave去給信號)

**輸入+輸出：**
- SDA

```verilog
module usage_I2C_write_R_W(
  en, 
  clk_sys, 
  rst_n, 
  addr, 
  data_i, 
  SCLK, 
  SDA, 
  data_o
);
/*------ports declaration------*/
input         clk_sys;
input         rst_n;
input         en;
input   [7:0] addr;
input   [7:0] data_i;
inout         SDA;
output        SCLK;
output [15:0] data_o;
reg    [15:0] data_o;
wire          SCLK;
```

再來，因為我打算收資料也做個狀態機，先來定義三個狀態：

```verilog
/*------parameter------*/
parameter IDLE  = 2'd0;
parameter SHIFT = 2'd1;
parameter STOP  = 2'd2;
```

**宣告變數：**
- data_temp(蒐集read data，於狀態為STOP時輸出給data_o)
- fstate(狀態機變數)
- count(計數16 bit的data的counter)
- 其他與狀態機模組連接的變數

**引用上次寫好的模組**

```verilog
/*------module I2C_write instantiate------*/
I2C_write_R_W U0(
  .clk_sys(clk_sys),
  .rst_n(rst_n),
  .en(en), 
  .data_R({addr, 1'b1, 8'hff, 1'b0, 8'hff, 1'b1}),//27bit
  .data_W({addr, 1'b1, data_i, 1'b1}),//18bit
  .ACK(ACK),
  .ACK1(),
  .ACK2(),
  .ACK3(), 
  .rstACK(rstACK),
  .SCLK(SCLK),
  .SDA(SDA),
  .ldnACK1(ldnACK1), 
  .ldnACK2(ldnACK2),
  .ldnACK3(ldnACK3),
  .R_W(addr[0]),
  .tick_I2C_neg(tick_I2C_neg)
);
```

**狀態邏輯：**
- 重置時要能回到IDLE狀態。
- IDLE( ldnACK1 && (addr[0]) && tick_I2C_neg 訊號後開始到下一個狀態)(代表第九bit以及目前是read模式並且在訊號正中間開始)
- SHIFT(蒐集完16 bit資料後結束)
- STOP(回到IDLE)

```verilog
/*------fstate state------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)fstate <= IDLE;
  else begin
    case(fstate)
      IDLE:begin
        if(ldnACK1&&(addr[0])&&tick_I2C_neg)fstate <= SHIFT;
        else                                fstate <= IDLE;
      end
      SHIFT:begin
        if(count==4'd15&&tick_I2C_neg)fstate <= STOP;
        else                          fstate <= SHIFT;
      end
      STOP:begin
        fstate <= IDLE;
      end
      default:begin
        fstate <= IDLE;
      end
    endcase
  end
end
```

**輸出邏輯：**
- IDLE(變數都為0)
- SHIFT( !ldnACK2&&tick_I2C_neg 此條件成立才count加一以及存入資料)(第18 bit資料不收，因為是ACK，而tick_I2C_neg則是為了在資料正中間取樣，確保資料正確)
- STOP(將蒐集完的16 bit料輸出給data_o，並將count歸零)

```verilog
/*------fstate output------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)begin
    count     <= 4'd0;
    data_temp <= 16'd0;
    data_o    <= 16'd0;
  end
  else begin
    case(fstate)
      IDLE:begin
        count     <= 4'd0;
        data_temp <= data_temp;
      end
      SHIFT:begin
        if(!ldnACK2&&tick_I2C_neg)begin
          count     <= count + 4'd1;
          data_temp <= {data_temp[14:0], SDA};
        end 
        else begin
          count     <= count;
          data_temp <= data_temp;
        end
      end
      STOP:begin
        count  <= 4'd0;
        data_o <= data_temp;
      end
      default:begin
        count     <= 4'dx;
        data_temp <= 16'hxx;
        data_o    <= 16'hxx;
      end
    endcase
  end
end

endmodule
```

這樣就結束了我們的I2C整個模塊

再來是TestBench的部分~~~~

---

## TestBench

```verilog
`timescale 1ns / 10ps

module i2c_tb();

localparam durTime = 5*1000;
reg sysClk, rst_n, tick_tx, sda_i;
reg [7:0]addr_i, data_i;
wire [15:0]data_o;
wire SCL;
wire SDA;
integer i;
assign SDA = (sda_i==1'b1)?1'bz:1'b0;
pullup(SDA);

usage_I2C_write_R_W UUT(
  .en(tick_tx), 
  .clk_sys(sysClk), 
  .rst_n(rst_n), 
  .addr(addr_i),
  .data_i(data_i),
  .SCLK(SCL), 
  .SDA(SDA),
  .data_o(data_o)
);

always@(negedge SCL, negedge rst_n)begin
  if(!rst_n)sda_i <= 1'b1;
  else if(i==8||i==17||i==27||i==28||i==36||i==35||i==43)#2000 sda_i <= 1'b0;//<27 for write
  else#2000 sda_i <= 1'b1;
end

always@(posedge SCL, negedge rst_n)begin
  if(!rst_n)i <= 0;
  else i <= i + 1;
end

always #10 sysClk = ~sysClk;

initial begin
  sysClk  = 0;
  rst_n   = 1;
  tick_tx = 0;
  sda_i   = 1'b1;
  addr_i  = 8'hA6;
  data_i  = 8'h81;
  #50 rst_n   = 0;
  #50 rst_n   = 1;

  #10 tick_tx = 1;
  #10 tick_tx = 0;

  repeat(50) #durTime;

  addr_i  = 8'hA7;
  #10 tick_tx = 1;
  #10 tick_tx = 0;

  repeat(100) #durTime;
  
  $stop;

end

endmodule 
```

值得注意的是verilog本身就有"pullup"語法可以模擬上拉電阻的效果

**Wave**

![](https://i.imgur.com/FLeAcqj.png)


可以看到上半部的部分：
> 前半部是write模式，先傳送addr_i的10100110(最後一bit是"0"，所以是write)再來是ACK，再接著data_i的10000001，最後也是1 bit的ACK。

> 後半部是read模式，先傳送addr_i的10100111(最後一bit是"1"，所以是read)，再來是ACK，接著的16 bit是由tb發送的01111110和11111101，而最後再搭配一個master端的non-ACK完成通訊。

而由此波形也可以看見SDA確實有在SCLK為LOW時才改變值。

![](https://i.imgur.com/4UFhnlt.png)

而這裡可以看到tick_I2C_neg(紅色那條)位於SDA的正中間，以此來取樣SDA信號可以確保值的正確性~~~~

來看看State Machine Viewer是否如預期

![](https://i.imgur.com/woYtU2s.png)


![](https://i.imgur.com/KKkyez4.png)

狀態確實都有照著我們的想法去跑呢~

那麼I2C的教學也到這邊結束嘍~~~~

