# [Day26] 快速乘法器的實作(Booth演算法)
## 為什麼要自己寫乘法器？

這篇會來教大家寫一個乘法器，那麼你可能會想：為什麼會需要乘法器呢？像我在quartus或Vivado裡打乘號也可以有乘法器用啊

那是因為在EDA tool裡面已經幫你在你使用乘號時合成出了他們已經寫好的乘法器了，那為什麼有這個不用還要另外設計呢？因為乘法器的種類有很多種，每一種也都有著不同的優缺點，所以通常會根據自己的需求來去設計一個最適合的乘法器，就例如pipelined乘法器，雖然把乘法拆成了數個步驟算，但是卻可以增加throughput。

那麼我們先來看看Booth這個演算法

---

## Booth演算法

例如nbit X nbit(P = a X b)：
- Step1:宣告一個2xn+1位寬的暫存器P。
- Step2:一開始先將b載入至P[n:1]中(其餘bit皆為"0")。
- Step3:判斷最後兩bit
  - 2'b01:P[2n:n+1]bit加上A
  - 2'b10:P[2n:n+1]bit減去A
  - 2'b00 or 2'b11:P保持不變
- Step4:右移1 bit(算術位移)
- Step5:重複Step3及Step4步驟n次
- Step6:拿掉最後1bit即是最後乘法結果

那麼第一步先來宣告狀態吧

**定義狀態**
```verilog
/*---------parameter----------*/
localparam IDLE   = 2'd0;
localparam CAL    = 2'd1;
localparam SHIFT  = 2'd2;
localparam FINISH = 2'd3;
```

**定義輸入/輸出**
**輸入：**
- clkSys
- rst_n
- en(狀態機致能)
- a(被乘數)
- b(乘數)

**輸出：**
- product(乘法結果)
- done(運算完成訊號)

```verilog
module booth_multiplier(
  clkSys, 
  rst_n, 
  en, 
  a, 
  b, 
  product, 
  done
);
/*---------ports declarations----------*/
input         clkSys; 
input         rst_n; 
input         en; 
input  [31:0] a; 
input  [31:0] b;
output [63:0] product; 
output        done;
reg    [63:0] product; 
reg           done;
```

**宣告變數**
- aReg(存放a以用來往後相加用)
- sReg(存放-a以用來往後相減用)
- pReg(product register)
- fstate(狀態變數)
- times(計數次數)

**狀態邏輯**
- reset後可以回到IDLE
- IDLE(收到en後往下一個狀態)
- CAL(停留1clk計算用，再往下個狀態)
- SHIFT(判斷是否計算夠多次，達到次數後結束計算，否則回到CAL繼續計算、移位)
- FINISH(輸出結果

```verilog
/*---------state----------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)fstate <= IDLE;
  else begin
    case(fstate)
	   IDLE:begin
		  if(en)fstate <= CAL;
        else  fstate <= IDLE;
      end
      CAL:begin
        fstate <= SHIFT;
      end
      SHIFT:begin
        if(times == 6'd32)fstate <= FINISH;
        else              fstate <= CAL;
      end
      FINISH: fstate <= IDLE;
      default:fstate <= IDLE;
    endcase
  end
end
```

**輸出邏輯**
- IDLE
  - 將a存入aReg
  - 將a補數存入sReg
  - 將b載入至pReg[n:1]，其餘bit為"0"
  - 計數器歸零
- CAL
  - 判斷pReg尾兩bit來判斷pReg[2n:n+1]是加a還是減a
  - times在此狀態加1(因為判斷次數是否足夠是在下個狀態，因此如果在下個狀態內才加一會來不及抓到)
- SHIFT
  - 將pReg算術右移1 bit
- FINISH(輸出結果
  - 將pReg[2n:1]輸出給product

`註：這裡我是寫32bit X 32bit的乘法器`


```verilog
/*---------output----------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    aReg    <= 32'd0;
    sReg    <= 32'd0;
    pReg    <= 65'd0;
    times   <= 6'd0;
    done    <= 1'b0;
    product <= 64'd0;
  end
  else begin
    case(fstate)
      IDLE:begin
        aReg  <= a;
        sReg  <= (~a + 32'd1);
        pReg  <= {32'd0, b, 1'b0};
        done  <= 1'b0;
        times <= 6'd0;
      end
      CAL:begin
        if(pReg[1:0] == 2'b01)     pReg <= {pReg[64:33]+aReg ,pReg[32:0]};//20 bit of MSB add aReg
        else if(pReg[1:0] == 2'b10)pReg <= {pReg[64:33]+sReg ,pReg[32:0]};//20 bit of MSB minus aReg
        else                       pReg <= pReg;
        times <= times + 6'd1;
      end
      SHIFT:begin
        pReg  <= {pReg[64],pReg[64:1]};
      end
      FINISH:begin
        done    <= 1'b1;
        product <= pReg[64:1];
      end
      default:begin
        aReg    <= 32'd0;
        sReg    <= 32'd0;
        pReg    <= 65'd0;
        times   <= 6'd0;
        done    <= 1'b0;
        product <= 64'd0;
      end
    endcase
  end
end
endmodule
```

---

## TestBench

```verilog
`timescale 10ns/10ns
module tb_booth();
reg                clk;
reg                rst_n;
reg  signed [31:0] a; 
reg  signed [31:0] b;
reg                en;
wire               done_sig;
wire signed [63:0] product;

booth_multiplier UUT(
  .clkSys(clk), 
  .rst_n(rst_n), 
  .en(en), 
  .a(a), 
  .b(b), 
  .product(product), 
  .done(done_sig)
);
initial begin
  clk = 0;
  rst_n = 0;
  a = 0;
  b = 0;
  en = 0;
  repeat(5)@(posedge clk)rst_n = 0;
  #10 rst_n = 1;
  test(32'd1, -32'd1);
  test(-32'd2, -32'd2);
  test(32'd3, -32'd3);
  test(32'd4, -32'd4);
  test(32'd5, -32'd5);
  test(32'd6, 32'd6);
  test(-32'd7, -32'd7);
  test(32'd8, -32'd8);
  test(32'd9, -32'd9);
  test(32'd10,-32'd10);
  
  test(32'd101, -32'd110);
  test(-32'd299, -32'd289);
  test(32'd330, -32'd3);
  test(32'd4986, -32'd4452);
  test(32'd5243, -32'd575);
  test(32'd2, 32'd67896);
  test(-32'd7, -32'd1334);
  test(32'd812, -32'd802);
  test(32'd0, -32'd9);
  test(32'd1000,-32'd123);
  
  #1000 $stop;
end
always #10 clk = ~clk;

task test;
  input  signed [19:0] in1;
  input  signed [19:0] in2;
  begin
    a = in1;
    b = in2;
	 #40 en = 1;
	 #20 en = 0;
    wait (done_sig == 1);
  end
endtask

endmodule

```

> 像這邊在寫TestBench時有很重複性動作時就可以寫成一個task，可以大大提升效率，而task之間總要等電路計算完才給下一筆測試資料嘛，所以在這邊也有用到wait語法，讓每一個task都收到模組的done時才繼續往下。

**Wave**

![](https://i.imgur.com/mfJudoX.png)

可以看到計算結果都完全正確，那麼今天的教學就到這邊~
