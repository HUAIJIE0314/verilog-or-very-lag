# [Day23] I2C Master(Write)的實現

上一篇我們設計了I2C Master的狀態機，那麼我們今天要來引用上次完成的狀態機模塊來實現I2C Master模塊，但我們先一步一步來，先來完成write的部分，也就是先可以把輸入的值成功以I2C協定傳出去，再來完成讀入的部分

**先來看看這個I2C Master write模塊該有哪些輸入輸出腳吧：**

**輸入：**
- clk_sys
- rst_n
- en(外部給此模組的驅動信號(enable))
- data_R(27bit)
- data_W(18bit)
- R_W(Read/Write)

**輸出：**
- ACK1, ACK2, ACK3(BUS上的ACK接收後輸出)
- ACK(ACK = ACK1|ACK2|ACK3)
- rstACK(連接狀態機模組的腳)
- SCLK
- ldnACK1, ldnACK2, ldnACK3(連接狀態機模組ACK1, ACK2, ACK3的腳 )
- tick_I2C_neg(我由於我讀資料又會包在外面的模塊，這個訊號是要傳到外面並要在此訊號為"1"時取樣資料)

**輸入+輸出：**
- SDA

```verilog
module I2C_write_R_W(
  clk_sys, 
  rst_n, 
  en,
  data_R,
  data_W,
  ACK, 
  ACK1,
  ACK2,
  ACK3,
  rstACK, 
  SCLK,
  SDA, 
  ldnACK1,
  ldnACK2, 
  ldnACK3,
  R_W, 
  tick_I2C_neg
);
/*---------ports declaration---------*/
input        clk_sys;
input        rst_n; 
input        en;
input        R_W;
input [26:0] data_R;//read  27bit
input [17:0] data_W;//write 18bit
output       ACK1;
output       ACK2;
output       ACK3;
output       tick_I2C_neg;
output       ACK; 
output       rstACK;
output       SCLK;
output       ldnACK1;
output       ldnACK2;
output       ldnACK3;
reg          tick_I2C_neg;    
reg          ACK1;
reg          ACK2;
reg          ACK3;
wire         ACK; 
wire         rstACK;
wire         SCLK;
wire         ldnACK1;
wire         ldnACK2;
wire         ldnACK3;
inout        SDA;
```

**宣告變數：**
需要用到的變數有：
- count(計數18 or 27個bit用)
- cnt_I2C(計數用(0~499)，要做tick_I2C)
- regdata_R(load進來的資料先暫存在這裡(for Read))
- regdata_W(load進來的資料先暫存在這裡(for Write))
- 要做tick_I2C(100KHZ的tick)
- 其他與狀態機模組連接的變數
- SEL(SDA的暫存值(而SDA又是由regdata的LSB以及SD0做切換的)，再多一層是因為，SDA實際輸出只會有高阻抗以及低電位，因此再多用一條線去用多工器選)


```verilog
/*---------variables---------*/
reg  [4:0] count;
reg  [8:0] cnt_I2C;
reg        tick_I2C;
reg        SCLK_100k;
reg [26:0] regdata_R;
reg [17:0] regdata_W;
wire       SCLK_temp;
wire       SHEN;
wire       LDEN;
wire       SDO;
wire       countEN;
wire       rstcount;
wire       SEL;
```

```verilog
/*---------assign wire---------*/
assign SEL = (SHEN)?((!R_W)?(regdata_W[17]):(regdata_R[26])):(SDO);
assign SDA = (SEL)?(1'bz):(1'b0);
assign ACK = ACK1|ACK2|ACK3;
```

**引用狀態機模組**

```verilog
/*---------module instantiate---------*/
I2C_control_R_W U0(
  .clk_sys(clk_sys),
  .SCLK_100k(SCLK_100k),
  .tick_I2C(tick_I2C),
  .rst_n(rst_n),
  .en(en),
  .count(count),
  .countEN(countEN),
  .rstcount(rstcount),
  .ACK1(ldnACK1),
  .ACK2(ldnACK2),
  .ACK3(ldnACK3),
  .rstACK(rstACK),
  .SCLK(SCLK),
  .SCLK_temp(SCLK_temp),
  .SHEN(SHEN),
  .LDEN(LDEN),
  .SDO(SDO),
  .R_W(R_W)
);
```

**cnt_I2C**

```verilog
/*---------100k counter---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)cnt_I2C <= 9'd0;
  else begin
    if(cnt_I2C<9'd499)cnt_I2C <= cnt_I2C + 9'd1;//0-499
    else              cnt_I2C <= 9'd0;
  end
end
```

**tick_I2C**

```verilog
/*---------I2C tick---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)tick_I2C <= 1'b0;
  else begin
    if(cnt_I2C==9'd498)tick_I2C <= 1'b1;
    else               tick_I2C <= 1'b0;
  end
end
```

```verilog
/*---------I2C tick_neg---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)tick_I2C_neg <= 1'b0;
  else begin
    if(cnt_I2C==9'd249)tick_I2C_neg <= 1'b1;
    else               tick_I2C_neg <= 1'b0;
  end
end
```

**SCLK_100k**

```verilog
/*---------SCLK_100k---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)SCLK_100k <= 1'b0;
  else begin
    if(cnt_I2C==9'd100)     SCLK_100k <= 1'b0;
    else if(cnt_I2C==9'd400)SCLK_100k <= 1'b1;
    else                    SCLK_100k <= SCLK_100k;
  end
end

```
這裡來解釋一下為什麼是100跟400：

還記得前幾天的I2C的介紹內有提到，SDA只能在SCLK為"0"時更動值，因此我們的SCLK高電位的部分要抓短一點，所以才會出現100時下去400上來的情況(咦？這樣不是反過來嗎？因為我狀態機模組裡面有反向了，你們也可以100上去400下來，然後不要反向，當然你想用其他的高電位寬度也都可以！)，而SDA只會在cnt_I2C等於498時才更動值，如此一來就可以達到SDA只會在SCLK等於"0"時才更動值的效果！

**藉由rstcount&countEN來控制count**

```verilog
/*---------count---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)begin
    count <= 5'd0;
  end
  else begin
    if(tick_I2C)begin
      if(rstcount)    count <= 5'd0;
      else if(countEN)count <= count + 5'd1;
      else            count <= count;
    end
    else count <= count;
  end
end
```

**藉由LDEN&SHEN&R_W來控制我們的regdata_W&regdata_R**

```verilog
/*---------load data---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)begin
    regdata_R <= 27'd0;
    regdata_W <= 18'd0;
  end
  else begin
    if(!R_W)begin
      if(tick_I2C)begin
        if(LDEN)     regdata_W <= data_W;
        else if(SHEN)regdata_W <= {regdata_W[16:0],1'b0};
        else         regdata_W <= regdata_W;
      end
      else regdata_W <= regdata_W;
    end 
    else begin
      if(tick_I2C)begin
        if(LDEN)     regdata_R <= data_R;
        else if(SHEN)regdata_R <= {regdata_R[25:0],1'b0};
        else         regdata_R <= regdata_R;
      end
      else regdata_R <= regdata_R;
    end
  end
end
```

**讀取BUS上的ACK**

```verilog
/*---------ACK---------*/
always@(posedge clk_sys or negedge rst_n)begin
  if(!rst_n)begin
    ACK1 <= 1'b0;
    ACK2 <= 1'b0;
    ACK3 <= 1'b0;
  end
  else if(rstACK&&tick_I2C)begin
    ACK1 <= 1'b0;
    ACK2 <= 1'b0;
    ACK3 <= 1'b0;
  end
  else begin
    if(ldnACK1&&tick_I2C)ACK1 <= SDA;
    else                 ACK1 <= ACK1;
    if(ldnACK2&&tick_I2C)ACK2 <= SDA;
    else                 ACK2 <= ACK2;
    if(ldnACK3&&tick_I2C)ACK3 <= SDA;
    else                 ACK3 <= ACK3;
  end
end

endmodule
```

由於這篇的內容也算偏多，可以先讓你們消化一下，下一篇我們會繼續完成Read的部分，並且撰寫TestBench來驗證電路的正確性~~
