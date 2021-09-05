# [Day2] Verilog 基本簡介
## verilog電路基本架構
舉個簡單電路的例子：
```
module adder(
  a, 
  b, 
  c
);

input a;  //輸入埠 敘述
input b;  //輸入埠 敘述
output c; //輸出埠 敘述
wire c;  //資料型態 敘述
assign c = a & b;//內部電路 敘述

endmodule
```
上方為一個AND邏輯閘，由此可知，一個完整的模組是由`module`以及`endmodule`包起來的，而adder那個位置就要放的則是`模組名稱`，而後面的括號內則是放`所有的輸入及輸出腳位`，接著裡面的最上方會宣告每隻腳位是輸出還是輸入，在來是宣告變數的資料型態。
在這裡，只要是輸入，一律都是`input`，輸出的話，如果沒有特別宣告則默認`wire`。

- input
  - 模組內只可接 wire
  - 模組外可接   wire、reg

- Output
  - 模組內可接   wire、reg
  - 模組外只可接 wire

- InOut（雙向埠）
  - 模組內&外只可接 wire

---

## 那甚麼時候才會用到inout呢？
舉個例子，在實現I2C protocol時就會用到這個好用的東西了，為甚麼這麼說呢？先來看看下面這張圖

![](https://i.imgur.com/4Ls8rP4.jpg)

[圖片出處](https://china.cypress.com/documentation/application-notes/an50987-getting-started-i2c-psoc-1)

I2C的運作機制是這樣的，I2C僅使用兩個雙BUS，串列資料線（SDA）和串列時鐘線（SCL），因此當master送資料給slave時此時master的SDA要設成out狀態，而slave的SDA要設成in的狀態，相反過來，當slave送資料給master時此時slave的SDA要設成out狀態，而master的SDA要設成in的狀態，所以會需要可輸入又可輸出的inout資料型態。
