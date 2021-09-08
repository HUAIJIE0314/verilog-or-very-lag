# [Day20] SPI的實現
上一篇我們設計了SPI的狀態機，那麼我們今天要來引用SPI狀態機模塊來實現整個SPI的模塊，並撰寫testbench來驗證電路的正確性。

**先來看看這個SPI模塊該有哪些輸入輸出腳吧：**

**輸入：**
- clock_50M
- rst_n
- miso(master in, slave out)
- en(外部給此模組的驅動信號(enable))
- din(外部輸入給此模組要傳輸的16 bit資料)

**輸出：**
- mosi(master out, slave in)
- finish(讓外部知道資料傳輸結束了，可以繼續傳下一筆)
- SCLK
- dout(將miso的資料蒐集起來並輸出觀測是否接收能正常)
- CS(chip select)


```verilog
module uasge_SPI_4wire(
  en, 
  clock_50M, 
  rst_n, 
  CS, 
  mosi, 
  miso, 
  din, 
  dout, 
  SCLK, 
  finish
);
/*-------ports declaration------*/
input         clock_50M;
input         rst_n;
input         miso;
input         en;
input  [15:0] din;
output [15:0] dout;
output        CS; 
output        mosi;
output        SCLK;
output        finish;
reg    [15:0] dout;
wire          CS; 
wire          mosi;
wire          SCLK;
wire          finish;
```

**宣告變數：**
需要用到的變數有：
- counter(計數用16個bit用，數0~49)
- cnt_SPI(計數用，要做tick_SPI)
- regdata_i(load進來的資料先暫存在這裡)
- regdata_o(用來存放miso的資料，並於傳輸結束後輸出)
- tick_SPI(50MHZ的tick)
- 其他與狀態機模組連接的變數

```verilog
/*-------variables------*/
reg     [3:0] counter;
reg    [15:0] regdata_i;
reg    [15:0] regdata_o;
reg           tick_SPI;
reg           SCLK_temp;//mux of SCLK_temp or 1'b1
reg     [5:0] cnt_SPI;  //1MHZ counter
wire          countEN;
wire          rstcount;
wire          SHEN;
wire          LDEN;
```

**引用狀態機模組**

```verilog
/*-------module instantiate------*/
SPI U0(
  .clk_sys(clock_50M),
  .SCLK_temp(SCLK_temp), 
  .rst_n(rst_n),
  .tick_SPI(tick_SPI),
  .SCLK(SCLK),
  .CS(CS),
  .count(counter),
  .countEN(countEN),
  .rstcount(rstcount),
  .ready(en),
  .finish(finish),
  .SHEN(SHEN),
  .LDEN(LDEN)
);
```

**cnt_SPI**

```verilog
/*-------1M counter------*/
always@(posedge clock_50M or negedge rst_n)begin
  if(!rst_n)cnt_SPI <= 6'd0;
  else begin
    if(cnt_SPI<6'd49)cnt_SPI <= cnt_SPI + 6'd1;
    else             cnt_SPI <= 6'd0;
  end
end
```

**做出SCLK_temp**
```verilog
/*-------make SCLK------*/
always@(posedge clock_50M or negedge rst_n)begin
  if(!rst_n)begin
    SCLK_temp <= 1'b0;
  end
  else begin
    if(cnt_SPI == 6'd24)     SCLK_temp <= 1'b0;
    else if(cnt_SPI == 6'd49)SCLK_temp <= 1'b1;
    else                     SCLK_temp <= SCLK_temp;
  end
end
```

為什麼這邊是24跟49呢？因為0-24 & 25-49會剛好把clk切為一半，這樣寫就會分別在正緣且cnt為24&49時SCLK_temp反向。

**tick_SPI**

```verilog
/*-------tick_SPI------*/
always@(posedge clock_50M or negedge rst_n)begin
  if(!rst_n)tick_SPI <= 1'b0;
  else begin
    if(cnt_SPI==6'd48)tick_SPI <= 1'b1;
    else              tick_SPI <= 1'b0;
  end
end
```

來解釋一下位甚麼這邊是48，因為我想讓其他動作有在SCLK負緣時動作的效果，而SCLK會在cnt=49時發生負緣，所以如果我想讓其他事情也在cnt=49時動作，那麼我tick必須提前一個clk升起來才行，所以這裡才是48而非49。

**靠rstcount&countEN來控制counter**

```verilog
/*-------ctrl counter------*/
always@(posedge clock_50M or negedge rst_n)begin
  if(!rst_n)begin
    counter <= 4'd0;
  end
  else begin
    if(tick_SPI)begin
      if(rstcount)    counter <= 4'd0;
      else if(countEN)counter <= counter + 4'd1;
      else            counter <= 4'd0;
    end
    else counter <= counter;
  end
end
```

**靠LDEN&SHEN來控制我們的regdata_i**

```verilog
/*-------data in------*/
always@(posedge clock_50M or negedge rst_n)begin//SCLK's negedge
  if(!rst_n)begin
    regdata_i <= 16'd0;
  end
  else begin
    if(tick_SPI)begin
      if(LDEN)     regdata_i <= din;
      else if(SHEN)regdata_i <= {regdata_i[14:0], 1'b0};
      else         regdata_i <= regdata_i;
    end
    else regdata_i <= regdata_i;
  end
end
```

這裡使用左移是因為SPI為MSB先傳。

**蒐集miso的data**

```verilog
/*-------collect miso data------*/
always@(posedge clock_50M or negedge rst_n)begin//SCLK's posedge
  if(!rst_n)regdata_o <= 16'd0;
  else begin
    if(tick_SPI)begin
      if(SHEN)regdata_o <= {regdata_o[14:0], miso};
      else    regdata_o <= regdata_o;
    end
    else begin
      regdata_o <= regdata_o;
    end 
  end
end
```

這邊來解釋一下為什麼用SCLK的負緣收資料，因為我設計的模組是預想外部輸入會在SCLK正緣時改變資料，所以為了確保資料的正確，要在SCLK的負緣收比較恰當。

**於傳輸結束後將dout送出**

```verilog
/*-------data out------*/
always@(posedge clock_50M or negedge rst_n)begin
  if(!rst_n)begin
    dout <= 16'd0;
  end
  else begin
    if(finish)dout <= regdata_o;
    else      dout <= dout;
  end
end

endmodule
```

**還有最後的mosi還沒處理**

```verilog
/*-------assign wire------*/
assign mosi = (SHEN)?(regdata_i[15]):(1'b0);
```
當沒有要傳資料時其實給什麼值都行，因為此時SCLK沒有在動作，也意味著slave端並不會收取資料。

**Quartus的State Machine Viewer**

![](https://i.imgur.com/ScDeeZ2.png)

---

## TestBench

```verilog
`timescale 1ns / 1ns

module spi_tb();
reg  sysClk, rst_n, tick_tx, miso;
reg  [7:0]address, txdata;
reg  [15:0]rxdata;
wire [15:0]data_get;
wire sclk, cs_n, mosi;
wire finish;
localparam Freq_i = 50000000;
localparam durTime = 1000;
integer i;

uasge_SPI_4wire UUT(
  .en(tick_tx),
  .clock_50M(sysClk),
  .rst_n(rst_n),
  .CS(cs_n),
  .mosi(mosi),
  .miso(miso),
  .din({address,txdata}),
  .dout(data_get),
  .SCLK(sclk),
  .finish(finish)
);

always@(posedge sclk, negedge rst_n)begin
  if(!rst_n)         miso <= 1'b0;
  else if(i>=0&&i<16)miso <= rxdata[15-i];
  else               miso <= 1'b0;
end

always@(posedge sclk, negedge rst_n)begin
  if(!rst_n)i <= 0;
  else      i <= i + 1;
end

always #10 sysClk = ~sysClk;

initial begin
  sysClk  = 0;
  rst_n   = 0;
  tick_tx = 0;
  address = 8'h7A;
  txdata  = 8'h81;
  rxdata  = 16'h4689;
  rst_n   = 0;
  #1000 rst_n   = 1;
  #1000;
  #1000 tick_tx = 1;
  #1000 tick_tx = 0;
  repeat(20) #durTime;
  $stop;
end

endmodule 
```
這裡的輸入資料16 bit是由8 bit的address以及8 bit的txdata所組成，
而testbench會將rxdata於SCLK的正緣依序由MSB傳出。

![](https://i.imgur.com/KDMz6S3.png)

> 這邊可以看到din為7a81，同樣為水藍色信號線也確實有在SCLK負緣時改變輸出值，而且值為0111_1010_1000_0001，確實為7a81。

> 再來是收的部分，可以看到粉紅色信號線是在SCLK正緣傳送0100_0110_1000_1001(4689)，而在data_get也確實在資料傳送結束時吐出4689。

最後也來可以看一下RTL Viewer

![](https://i.imgur.com/7t2mn4V.png)

![](https://i.imgur.com/kBLOcH5.png)

看起來沒有合成出過多冗餘的部分，還可以~

以上就是我們的4線SPI教學嘍~~~~
