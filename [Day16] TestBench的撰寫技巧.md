
# [Day16] TestBench的撰寫技巧
> 透過Verilog完成一個具有特定功能的電路後，並不代表你的工作已經完成了，TestBench(tb)在電路設計中也是一個非常重要的環節，往往驗證電路所花的時間還會比較開發來的多。

> 而TestBench則是另一個.v檔，TestBench的工作就是產生待測模塊的輸入信號，用這些訊號**模擬**真實的情況，接著透過波型來觀察輸出結果是否如自己預期，如果不如預期，也可以再透過把待測電路中的各個訊號叫出來觀察，以此debug也會比自己一直盯著程式碼來除錯來的有效率唷！

**我們可以先來看[Day15]中的範例程式**

```verilog
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
.
.
.
.
endmodule

```
可以看到這個模塊有三個輸入，分別是clkSys, rst_n, en，那麼我們在testbench那邊就會需要有對應這三個訊號的register(通常我名字會取一樣)，而輸出則是一個3 bit的out，那麼對應testbench，就會有一個3 bit 的wire對應它。

再來稍微解釋一下為甚麼testbench那邊一定要模塊的輸入用reg，而輸出用wire，因為reg屬於**過程性賦值(Procedural Assignment)**，意思是值是可以一直隨心所欲變動的，而wire則屬於**連續性賦值（Continuous Assignment）**，通常用於連接訊號用。


**所以就會有對應的這區塊程式：**

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
```


## timescale
先來談談最上面那行 `timescale 10ns/1ns`
timescale是Verilog中的一種**時間預編譯指令**，它用來定義模組模擬的**時間單位**以及**時間精度**。

格式長這樣：
```verilog
`timescale 時間單位 / 時間精度
```
`需要注意到的是：`以上兩者的數字只能是**1** or **10** or **100**，而且時間單位必須大於時間精度。

以下來舉個例子：
```verilog
`timescale 1ns/10ps
reg clk_50M;
initial begin
  clk_50M = 0;
end
always #10 clk_50M = ~clk_50M;
```
現在時間單位設為1ns那麼#1就會delay1ns，#10則是delay10個ns，所以delay10個ns則clk_50M會反向一次，所以反向兩次可以得到1周期的clk，所以一個clk總共delay (1ns)x(delay 10次)x(反向2次) = 20ns，得到此頻率為50MHZ。

---

現在回到testBench那個程式，宣告完reg以及wire後，剩下的就是以by name的方式來引用模組了。

`註：TestBench中引用的測試模塊通常命名為UUT(Unit Under Test)`

**再接著：**
```verilog
initial begin
  clkSys = 0;
  rst_n = 0;
  en = 0;
  repeat(2)@(posedge clkSys)rst_n = 0;//push rst_n
  rst_n = 1;//release rst_n
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
initial begin是用於TestBench中初始化變數值的語法，而且initial block中的程式只會執行一遍，並不會像always一直重複執行，在initial block內初始化訊號後，就要開始對輸入訊號的變數賦值了，而$stop語法則可以停止波形的模擬(否則會一直跑下去，會一直佔用電腦的資源)。

---

## 如何在Quartus完成對TestBench的設定以及跑模擬
Assignment/setting

![](https://i.imgur.com/3YvNOIT.png)

Simulation/Compile test bench--->點選右側Test Bench..按鈕

![](https://i.imgur.com/dbqOAwZ.png)

New...

![](https://i.imgur.com/6vsAc2g.png)

上面兩行打上module名稱，下方選擇tb的.v檔並add

![](https://i.imgur.com/TxdMiQ3.png)

接著連串的ok ok...

---

## 設定modelsim路徑

tools/options

![](https://i.imgur.com/Kxv5UqR.png)

EDA Tool Options，並選擇路徑

![](https://i.imgur.com/JTbEpSL.png)

## 模擬

Tools/Run Simulation Tool/RTL Simulation

![](https://i.imgur.com/KJCBeft.png)

這樣好好看的波形就會出來囉！！

順帶解釋一下RTL與Gate-level Simulation的差異：
RTL模擬是所謂的**功能**上的模擬，也就是不考慮硬體電路中實際的delay，所有情況皆為理想，所以適合拿來純測功能，而Gate-level模擬則是**時序**上的模擬，此時會考慮到所有的不理想因素，因此會有delay的情況發生，所以跑模擬的順序都會是先跑RTL再跑Gate-level。

