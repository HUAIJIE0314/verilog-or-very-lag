# [Day18] Uart_TX的實現
既上一篇我們設計了Uart_TX的狀態機，我們今天要來引用狀態機模塊來實現這個Uart_TX的模塊。

**輸入：**
- clk_50M
- write_value
- write(外部給此模組的驅動信號(enable))
- write_value(外部輸入給此模組要傳輸的8 bit 資料)

**輸出：**
- uart_txd(data線)
- busy(讓外部知道資料傳輸未結束，還不能繼續傳下一筆)


```verilog
module usage_Uart_TX(
  clk_50M, 
  reset_n, 
  write, 
  write_value, 
  uart_txd, 
  busy
);
/*--------ports declarations--------*/
input       clk_50M;
input       write;
input       reset_n;
input [7:0] write_value;
output      uart_txd;
output      busy;
wire        uart_txd;
wire        busy;
```
**宣告變數：**
需要用到的變數有：
- counter(計數用8 bit用)
- cnt_9600(計數用，要做tick_uart)
- regdata(load進來的資料先暫存在這裡)
- 與狀態機模組連接的變數

```verilog
/*---------variables---------*/
reg [7:0]  regdata;
reg [2:0]  counter;
reg [12:0] cnt_9600;
reg        tick_uart;
wire       TX_D;
wire       LDEN;
wire       SHEN;
wire       rstcount;
wire       countEN;
```

**引用狀態機模組**

```verilog
/*---------Uart_TX instantiation---------*/
Uart_TX U0(
  .clk_50M(clk_50M),
  .rst_n(reset_n),
  .count(counter),
  .rstcount(rstcount),
  .countEN(countEN),
  .tick_uart(tick_uart),
  .TX_D(TX_D),
  .LDEN(LDEN),
  .SHEN(SHEN),
  .en(write),
  .busy(busy)
);
```

**cnt_9600 & tick_uart：**
因為我是以鮑率9600為主，所以需除出9600HZ，首先，輸入頻率為50MHZ，50M/9600 = 5208.333，取5208，因此我們要每計數到5207(因為0~5207就會數5208次)就產生一個clk的tick

```verilog
/*---------9600HZ counter---------*/
always@(posedge clk_50M or negedge reset_n)begin
  if(!reset_n)cnt_9600 <= 13'd0;
  else begin
    if(cnt_9600<13'd5207)begin
      cnt_9600  <= cnt_9600 + 13'd1;
      tick_uart <= 1'b0;
    end 
    else begin
      cnt_9600 <= 13'd0;
      tick_uart <= 1'b1;
    end 
  end
end
```

**靠rstcount&countEN來控制counter**

```verilog
/*---------counter---------*/
always@(posedge clk_50M or negedge reset_n)begin
  if(!reset_n)counter <= 3'd0;
  else begin
    if(tick_uart==1'b1)begin
      if(rstcount)    counter <= 3'd0;
      else if(countEN)counter <= counter + 3'd1;
      else            counter <= counter;
    end
    else counter <= counter;
  end
end
```
if(tick_uart==1'b1)是因為一個count要待滿一個鮑率的時間週期。

**靠LDEN&SHEN來控制我們的regdata**

```verilog
/*---------write_value---------*/
always@(posedge clk_50M or negedge reset_n)begin
  if(!reset_n)regdata <= 8'd0;
  else begin
    if(tick_uart==1'b1)begin
      if(LDEN)     regdata <= write_value;
      else if(SHEN)regdata <= regdata>>1;
      else         regdata <= regdata;
    end
    else regdata <= regdata;
  end
end

endmodule
```
regdata>>1 這邊右移是因為uart是LSB先傳。

**最後的最後**
要在非SHIFT狀態時tick_uart切到狀態機的TX_D，而在SHIFT狀態時切回regdata的LSB。

```verilog
/*---------assign wire---------*/
assign uart_txd = (SHEN)?(regdata[0]):(TX_D);
```

**這樣我們就完成了Uart_TX的實作了唷~**

