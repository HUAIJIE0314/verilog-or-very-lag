# [Day25] One-Wire protocol
## One-Wire
One-Wire是一種只需要一條線即可傳輸資料的傳輸協定，而通常這種傳輸協定會用於與小型裝置溝通，例如數位溫度、濕度感測器。

`那我們在這邊會以AM2302的規格為主去撰寫我們的程式`

---

## AM2302 Datasheet

![](https://i.imgur.com/dPg8A6i.png)

![](https://i.imgur.com/PE9BhOR.png)

![](https://i.imgur.com/5ZD5O1m.png)

![](https://i.imgur.com/RD5mVvb.png)

`注意：Tbe為master拉低`

圖片出處：[AM2302 Datasheet](https://files.seeedstudio.com/wiki/Grove-Temperature_and_Humidity_Sensor_Pro/res/AM2302-EN.pdf)

這邊我們由圖一及圖二可以看到在接收AM2302的資料時，一共會有一連串的Start Signal以及40 bit的data，分別是濕度的前8 bit、後8bit，溫度的前8 bit、後8 bit，還有最後的8 bit較驗位(較驗位=前面四筆8 bit資料做相加)

而圖三及圖四則告訴了我們種信號所會花的時間長

而datasheet內容也有提到這裡的SDA線也是有pull-up的~

先來第一步吧！

**先定義狀態機**

```verilog
//===== state machine =====//
localparam IDLE     = 3'd0;
localparam Be       = 3'd1;
localparam GO       = 3'd2;
localparam REL      = 3'd3;
localparam REH      = 3'd4;
localparam DATA     = 3'd5;
localparam CHECKOUT = 3'd6;
```

**定義每個間隔時間**

```verilog
//===== during time =====//
localparam durBe       = 16'd50000;//1ms
localparam durGo       = 16'd1500; //30us
localparam durRel      = 16'd2555; //80us
localparam durReh      = 16'd2560; //80us
localparam durDataHIGH = 16'd2580; //70us
localparam durDataMax  = 16'd5000; //100us
```

這裡的間隔周期以datasheet的typical為準，而這裡的時脈是以50M為主，舉個例子：如果要1ms那麼counter就要數1ms/20ns = 50000，其他以此類推。

再來比較有趣的是這裡可以不用精準定義"0"訊號以及"1"訊號的HIGH時間長，因為只要在資料線一變LOW時去檢查在HIGH時數到的數字有沒有超過一定的值就可以視為訊號"1"了(這裡是抓70us，因為 "0" HIGH的typical值為30us，"1" HIGH的typical值為75us)

**再來定義輸入輸出**
**輸入：**
- clk_sys
- rst_n
- en(狀態機 enable)

**輸出：**
- dataReady(資料正確時升為一，表示資料可讀取走)
- data_o

**inout**
- SDA


```verilog
module AM2302_controller(
  clkSys, 
  en, 
  rst_n, 
  SDA, 
  dataReady, 
  data_o
);
/*---------Ports Declarations---------*/
input         clkSys;
input         en;
input         rst_n;
inout         SDA;
output        dataReady;
output [39:0] data_o;
reg           dataReady;
reg    [39:0] data_o;
```

**宣告變數：**
- fstate(狀態機用)
- counter(計數用)
- databuf(蒐集的資料暫存的地方)
- counterEN(counter的致能)
- flag_get(以此flag抓負緣)
- sdaReg(sda的暫存)

```verilog
/*---------variables---------*/
reg     [2:0] fstate;
reg    [15:0] counter;
reg    [39:0] databuf;
reg           counterEN;
reg           flag_get;
reg           sdaReg;
```

**SDA輸出**
```verilog
/*---------assign wire---------*/
assign SDA = (sdaReg)?(1'bz):(1'b0);
```

**counter致能控制counter的計數**

```verilog
/*---------counter---------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)        counter <= 16'd0;
  else if(counterEN)counter <= counter + 16'd1;
  else              counter <= 16'd0;
end
```

**狀態機狀態邏輯**
- 每個狀態都數滿週期後才往下個狀態跑。
- 特別的是，我這邊的DATA狀態是讓它數到超過，才往下狀態走(因為"1"的HIGH時間不會到100us這麼長，如果超過代表資料傳送完畢了)，如此一來可以省掉許多判斷。
- 最後一個狀態是來檢查校驗碼的。

```verilog
/*---------fstate_state---------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)fstate <= IDLE;
  else begin
    case(fstate)
      IDLE:begin
        if(en)fstate <= Be;
        else  fstate <= IDLE;
      end
      Be:begin
        if(counter == durBe)fstate <= GO;
        else                fstate <= Be;
      end
      GO:begin
        if(counter == durGo)fstate <= REL;
        else                fstate <= GO;
      end
      REL:begin
        if(counter == durRel)fstate <= REH;
        else                 fstate <= REL;
      end
      REH:begin
        if(counter == durReh)fstate <= DATA;
        else                 fstate <= REH;
      end
      DATA:begin
        if(counter == durDataMax)fstate <= CHECKOUT;
        else                     fstate <= DATA;
      end
      CHECKOUT:fstate <= IDLE;
      default: fstate <= IDLE;
    endcase
  end
end
```

**狀態機輸出邏輯**
- sda只有在be需要拉低(datasheet的Tbe為master拉低)。
- 當counter小於該狀態應計數值的值時counterEN = 1，這樣是為了讓counter可以在進入下個狀態前歸零。

```verilog
/*---------fstate_output---------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    counterEN <= 1'b0;
    sdaReg    <= 1'b1;
  end 
  else begin
    case(fstate)
      IDLE:begin
        counterEN <= 1'b0;
        sdaReg    <= 1'b1;
      end
      Be:begin
        if(counter < durBe)counterEN  <= 1'b1;
        else               counterEN  <= 1'b0;
        sdaReg <= 1'b0;
      end
      GO:begin
        if(counter < durGo)counterEN  <= 1'b1;
        else               counterEN  <= 1'b0;
        sdaReg <= 1'b1;
      end
      REL:begin
        if(!SDA && counter < durRel)counterEN  <= 1'b1;
        else                        counterEN  <= 1'b0;
        sdaReg <= 1'b1;
      end
      REH:begin
        if(SDA && counter < durReh)counterEN  <= 1'b1;
        else                       counterEN  <= 1'b0;
        sdaReg <= 1'b1;
      end
      DATA:begin
        if(SDA && counter < durDataMax)counterEN  <= 1'b1;
        else                           counterEN  <= 1'b0;
        sdaReg <= 1'b1;
      end
      CHECKOUT:begin
        counterEN <= 1'b0;
        sdaReg    <= 1'b1;
      end 
      default:begin
        counterEN <= 1'b0;
        sdaReg    <= 1'b1;
      end 
    endcase
  end
end
```

**flag_get**

```verilog
/*---------flag_get---------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)flag_get <= 1'b0;
  else      flag_get <= SDA;
end
```

把SDA延後一個clk給flag_get，往後只要讀到SDA=0;flag_get=1，就知道SDA此時下降了(負緣)

**蒐集資料**

```verilog
/*---------put data in databuf---------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)databuf <= 39'd0;
  else begin
    if(fstate > REH)begin
      if(!SDA && flag_get && (counter > durDataHIGH))     databuf <= {databuf[38:0],1'b1};    
      else if(!SDA && flag_get && (counter < durDataHIGH))databuf <= {databuf[38:0],1'b0};
      else                                                databuf <= databuf;
    end
    else databuf <= 39'd0;
  end 
end
```

在SDA剛下降時去檢查剛剛在SDA為HIGH時維持了多少時間，如果大於70us則將1左移移入databuf，反之則移入"0"。

**檢查較驗碼**

```verilog
/*---------check data and sent dataReady---------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    data_o    <= 39'd0; 
    dataReady <= 1'b0;
  end 
  else begin
    if(fstate==CHECKOUT)begin
      if(databuf[7:0]==databuf[39:32]+databuf[31:24]+databuf[23:16]+databuf[15:8])begin
        data_o    <= databuf;
        dataReady <= 1'b1;
      end 
      else begin
        data_o    <= data_o;
        dataReady <= 1'b0;
      end 
    end
    else begin
      data_o    <= data_o;
      dataReady <= 1'b0;
    end
  end
end

endmodule
```

在CHECKOUT狀態檢查較驗碼，如果正確則輸出至data_o，並且將dataReady升為"1"。

如此以來我們就完成了這個AM2302 controller模組瞜，是不是比上次的I2C更簡單了一點呢~~~~
