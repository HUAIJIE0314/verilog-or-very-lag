
# [Day04] Verilog 資料型態(下)
## 各種進制表示法
```
<位元長度> ’ <b、o、d、h> <數值>
```
- 位元長度：以十進制表示幾個bit數
- 進制表示：二進制(b)、八進制(o)、十進制(d)、十六進制(h)，不打則預設為十進制
- 數值資料：可用底線 _ 來增加可讀性

EX:
```verilog
a = 32'd0;
b = 6'b000101;
c = 8'h0A;
d = 8'b0101_1010;
```
`錯誤用法舉例：`
```verilog
a = 32'b0;//打了b就要把所有位打出來
b = 6'd000101;//數值使用錯誤
c = 8'h10A;位元長度錯誤
```

---

## 向量(陣列)表示法

EX:
```verilog
reg [7:0]a;//一個8bit的reg
reg [3:0]b[31:0];//32個4bit的reg--->又稱記憶體表示法
```
**EX:用向量表示法寫一個有讀寫及致能的簡易memory**
```verilog
module memory(
  Enable,
  ReadWrite,
  Address, 
  DataIn,
  DataOut
);
/*----------ports declarations----------*/
input        Enable;
input        ReadWrite;
input   [5:0]Address; 
input   [3:0]DataIn;
output  [3:0]DataOut;
reg     [3:0]DataOut;
/*----------variables----------*/
reg     [3:0]mem[63:0];
/*----------memory----------*/
always@(Enable or ReadWrite)begin
  if(Enable)begin
    if(ReadWrite)DataOut      = mem[Address];
    else         mem[Address] = DataIn;
  end
  else DataOut = 4'bzzzz;
end

endmodule
```
**TestBench**
```verilog
`timescale 1ns/1ns
module tb_memory();
/*----------variables----------*/
reg      Enable;
reg      ReadWrite;
reg [5:0]Address; 
reg [3:0]DataIn;
wire[3:0]DataOut;
integer i = 0;
/*----------module memory instantiation----------*/
memory UUT(
  .Enable(Enable),
  .ReadWrite(ReadWrite),
  .Address(Address), 
  .DataIn(DataIn),
  .DataOut(DataOut)
);
/*----------control signal----------*/
initial begin
  Enable    = 1'b0;
  ReadWrite = 1'b0;
  Address   = 6'd0;
  DataIn    = 4'd0;
end

initial begin
  for(i=0;i<=63;i=i+1)begin
    #100;
    Enable    = 1'b1;
    ReadWrite = 1'b0;
    Address   = Address + 6'd1;
    DataIn    = DataIn  + 4'd1;
    #10;
    Enable    = 1'b0;
  end
  Address   = 6'd0;
  for(i=0;i<=63;i=i+1)begin
    #100;
    Enable    = 1'b1;
    ReadWrite = 1'b1;
    Address   = Address + 6'd1;
    DataIn    = 4'dz;
    #10;
    Enable    = 1'b0;
  end
#1000;
$stop;
end

always@(negedge Enable)begin
  if(ReadWrite)$display("time = %3d, dataRead  = %x", $time, DataOut);
end

endmodule
```

**Wave**
![](https://i.imgur.com/sgFNz36.png)
![](https://i.imgur.com/Hbn57P0.png)

`這邊可以看到前面是寫入值到各個位址，下圖則是成功從該位址讀出先前存入的值。`

(這邊ReadWrite是'0'寫入，'1'讀取)

有趣的是quartus竟然可以辨識那是一顆mem!
![](https://i.imgur.com/CSU5QJI.png)


---

在這裡可以順便提到integer的使用，*integer好比一個32bit的register(但integer視為有符號數)*，所以不要輕易用它來宣告變數，否則會無形中多使用了很多硬體資源，它通常會被宣告來當for-loop的迴圈變數。

EX:
```verilog
reg [3:0]a[31:0];

always@(posedge clk or negedge rst_n)begin
  if(!rst_n)begin
    for(i=0;i<32;i=i+1)begin
      a <= 4'd0;
    end
  end
  else begin
  .
  .
  .
  end
end
```

---

## 參數parameter
- 是一個宣告了就`無法更動的常數`
- 常常會用來指定`資料位寬(Width)`或是`狀態機的值`

EX:
```verilog
parameter width = 32;
reg [width-1:0]a;//一個32bit的reg
```
