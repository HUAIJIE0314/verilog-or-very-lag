
# [Day13] Latch的生成條件以及如何避免(下)
## Latch的生成條件
上一篇講解了什麼是latch，其又與flip-flop差在哪，也解釋了我們要去避免它的原因，那麼這篇將會告訴你有哪幾種情況會產生latch。

---

### 第一種：if語句結構不完整
在組合邏輯中，不完整的if-else語句會導致latch的生成，那怎樣是不完整呢？
就是最後缺少了else，正是因為沒有寫else，會被系統認定為該值不需要改變，而自動生成latch來幫你儲存

**會生成latch的例子：**
```
module latch_test(
  in,
  en, 
  out
);
input  in;
input  en;
output out;
reg    out;

always@(*)begin
  if(en) out = in;//no else
end

endmodule
```

解決方法：

**SOL1：補齊else語句**
```
module latch_test(
  in,
  en, 
  out
);
input  in;
input  en;
output out;
reg    out;

always@(*)begin
  if(en) out = in;
  else   out = 1'b0;//add
end

endmodule
```
**SOL2：在always內最上方加上初值**
```
module latch_test(
  in,
  en, 
  out
);
input  in;
input  en;
output out;
reg    out;

always@(*)begin
  out = 1'b0;//initialization
  if(en) out = in;
end

endmodule
```
**但這裡有一點該注意，其實講補齊if-else算是攏統的說法，因為以下這個例子其實也會產生latch，原因就在同一個變數在不同if條件下都要敘述完整，否則還是會生成latch。**
```
module latch_test(
  in1,
  in2,
  en, 
  out1,
  out2
);
input  in1;
input  in2;
input  en;
output out1;
output out2;
reg    out1;
reg    out2;

always@(*)begin
  if(en) out1 = in1;
  else   out2 = in2;
end

endmodule
```
**應改成**
```
module latch_test(
  in1,
  in2,
  en, 
  out1,
  out2
);
input  in1;
input  in2;
input  en;
output out1;
output out2;
reg    out1;
reg    out2;

always@(*)begin
  if(en)begin
    out1 = in1;
    out2 = 1'b0;//add
  end 
  else begin
    out1 = 1'b0;//add
    out2 = in2;
  end 
end

endmodule
```



`這裡值得注意的是，在循序邏輯中(clock觸發)，不完整的if-else並不會生成latch，因為register具有儲存前態的功能，並只有在正負緣才會改變值~`


---

### 第二種：case語句結構不完整
邏輯跟if-else相似，就是最後缺少了default，正是因為沒有寫default，進入到未被定義的case時會被系統認定為該值不需要改變，而自動生成latch來幫你儲存。

**會生成latch的例子：**

```
module latch_test(
  in1,
  in2,
  sel, 
  out
);
input       in1;
input       in2;
input [1:0] sel;
output      out;
reg         out;

always@(*)begin
  case(sel)
    2'd0:out = in1;
    2'd1:out = in2;
  endcase
end

endmodule
```
**應改成**
```
module latch_test(
  in1,
  in2,
  sel, 
  out
);
input       in1;
input [1:0] sel;
output      out;
reg         out;

always@(*)begin
  case(sel)
    2'd0:out = in1;
    2'd1:out = in2;
    default:out = 1'b0;//add
  endcase
end

endmodule
```
**或是補上初值也是可以的！**

---

### 第三種：使變數等於自己時或是判斷元素有自己時

因為會用到變數的前態而自動生成latch來幫你儲存。

**會生成latch的例子：**
```
reg a;
reg b;

always@(*)begin
  if(a | b)a = 1'b0;
  else     a = 1'b1;
end
```
**以及**
```
reg a;
reg b;
always@(*)begin
  if(b)a = a;
  else a = a + 1'b1;
end
```
**當然wire也不例外**
```
wire a;
wire c;
reg b;
assign a = (a & b)?(1'b0):(1'b1);
assign c = (a & b)?(c):(1'b0);
```
**這種其實不太能解，就是盡量不要這樣寫**

或是，如果輸出沒有非常及時，也可以延後一個clk將值鎖在register內
EX:
```
reg a;
reg b;
reg a_temp;

always@(posedge clkSys)begin
  a_temp <= a;
end

always@(*)begin
  if(a_temp | b)a = 1'b0;
  else          a = 1'b1;
end
```
用先前儲存的a_temp來當判斷元素，也可以避免。

---

### 第四種：always觸發條件不完整
`等號右邊的所有變數或判斷元素都應蓋列入alway觸發條件中`

**會生成latch的例子：**
```
module latch_test(
  in1,
  in2,
  sel, 
  out
);
input  in1;
input  in2;
input  sel;
output out;
reg    out;

always@(in1 or in2)begin
  if(sel)out = in1;
  else   out = in2;
end

endmodule
```
**應改成**
```
module latch_test(
  in1,
  in2,
  sel, 
  out
);
input  in1;
input  in2;
input  sel;
output out;
reg    out;

always@(in1 or in2 or sel)begin//add sel
  if(sel)out = in1;
  else   out = in2;
end

endmodule
```

以上四種就是常見的latch生成原因以及相對應解決的方法~~
