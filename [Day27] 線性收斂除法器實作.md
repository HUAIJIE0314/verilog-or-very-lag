# [Day27] 線性收斂除法器實作
> 在加減乘除四個基本運算中，其中除法最為困難及複雜，因此除法也是最耗時的運算。

對於一個被除數為N，除數為D，商為Q，餘數R的除法運算中，我們可以得到這樣的算式：

`N/D = Q..R，其中|R|<D `

當然也可以看成是：

`N = D x Q + R`

> 除法運算與乘法很大的不同是：乘法中部分的乘積都可以並行算出來，而除法中商的每一位則要用一種"嘗試錯誤"的方法得出，而大多數的微處理器都是以`N = D x Q + R`的方式作為乘法的逆運算處理，因此假定分子是一個乘法的結果，所以會把分母及商的位寬擴充為兩倍當作是分子的位寬，但這樣做的結果則會導致必須使用很沒有效率的方式來確定我們的商是否會在有效的範圍內。


**因此我們來做一個有效的假定：**
我們假設 `N >= Q` 以及 `|R| < D`
也就是我們假定**商和分子**以及**分母和餘數**的位寬互相相等，如此一來我們就不用再檢查商的範圍是否有效了。

> 而會說線性收斂是因為這個除法算法所花的cycle數與被除數的bit數是一樣的，是呈線性關係的。


---

## 實作方法
先來假設商(q_out)和分子(dividend)的位寬是8bit，而分母(divisor)和餘數(r_out)是6bit。

接著我們需要兩個位寬和的兩個變數，一個有號(r)、一個無號(d)。

一開始我們先把divisor左移(8-1)次載入到d(如果移8次會超出範圍，要少一次)，而直接將dividend存入到r。

接著將r減去d，如果為負號，代表不夠減，我們在商的部分就左移入"0"並且將r還原為剛剛的值(因為不夠減，要保持剛剛的值然後看下一個bit夠不夠減)，如果減完為正的代表夠減，此時則將商左移入"1"，而r不需要還原，最後要把d除以2(右移一次)，再繼續接著剛剛的動作連續8次。

最後將q及r輸出則是我們最後的商跟餘數了！

**宣告狀態**

```verilog
/*-------localparam-------*/
localparam IDLE    = 2'd0;
localparam SUB     = 2'd1;
localparam RESTORE = 2'd2;
localparam FINISH  = 2'd3;
```

**宣告輸入輸出**

```verilog 
module divRes(
  clkSys,
  en,
  rst_n,
  dividend,
  divisor,
  r_out,
  q_out,
  done
);
/*-------ports declarations-------*/
input        clkSys;
input        en;
input        rst_n;
input  [7:0] dividend;
input  [5:0] ;
output [5:0] r_out;
output [7:0] q_out;
output       done;
reg    [5:0] r_out;
reg    [7:0] q_out;
reg          done;
```

**宣告所需變數**

```verilog
/*-------variables-------*/
reg    [1:0] fstate;
reg    [3:0] count;
reg    [7:0] q;
reg   [13:0] d;
reg signed [13:0] r;
```

**狀態邏輯**

```verilog
/*-------fstate state-------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)fstate <= IDLE;
  else begin
    case(fstate)
      IDLE:begin
        if(en)fstate <= SUB;
        else  fstate <= IDLE;
      end
      SUB:fstate <= RESTORE;
      RESTORE:begin
        if(count == 4'd8)fstate <= FINISH;
        else             fstate <= SUB;
      end
      FINISH:fstate <= IDLE;
      default:fstate <= IDLE;
    endcase
  end
end
```

**輸出邏輯**

```verilog
/*-------fstate output-------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    count <= 4'd0;
    r_out <= 6'd0; 
    q_out <= 8'd0;
    q     <= 8'd0;
    r     <= 14'd0;
    d     <= 14'd0;
    done  <= 1'b0;
  end
  else begin
    case(fstate)
      IDLE:begin
        count <= 4'd0;
        q     <= 8'd0;
        d     <= divisor << 7;//load divisor
        r     <= dividend;//Remainder = dividend
        done  <= 1'b0; 
      end
      SUB:begin
        r     <= r - d;
        count <= count + 4'd1;
      end
      RESTORE:begin
        if(r<0)begin
          r <= r + d;
          q <= q << 1;//left shift and LSB = 0
        end
        else begin
          r <= r;
          q <= {q[6:0], 1'b1};//left shift and LSB = 1
        end
        d <= d >> 1;
      end
      FINISH:begin
        q_out <= q;
        r_out <= r[5:0];
        done  <= 1'b1; 
      end
      default:begin
        count <= 4'd0;
        r_out <= 6'd0; 
        q_out <= 8'd0;
        q     <= 8'd0;
        r     <= 14'd0;
        d     <= 14'd0;
        done  <= 1'b0; 
      end
    endcase
  end
end
endmodule
```

---

## TestBench

```verilog

`timescale 1ns/1ns
module tb_divRes();

reg        clkSys;
reg        en;
reg        rst_n;
reg  [7:0] dividend;
reg  [5:0] divisor;
wire [5:0] r_out;
wire [7:0] q_out;
wire       done;

divRes UUT(
  .clkSys(clkSys),
  .en(en),
  .rst_n(rst_n),
  .dividend(dividend),
  .divisor(divisor),
  .r_out(r_out),
  .q_out(q_out),
  .done(done)
);

initial begin
  clkSys = 0;
  rst_n  = 0;
  repeat(2)@(posedge clkSys)rst_n = 0;
  rst_n = 1;
  test(234,50);
  test(196,20);
  test(97 ,15);
  test(200,11);
  test(19 ,50);
  test(192 ,1);
  test(64 ,2);
  #1000 $stop;
end

always #10 clkSys = ~clkSys;

task test;
input [7:0]a;
input [5:0]b;
  begin
    dividend = a;
    divisor  = b;
    #40 en = 1;
    #20 en = 0;

    wait (done == 1);
  end
endtask

endmodule

```

**Wave**

![](https://i.imgur.com/Iy7UeHC.png)

商與餘的結果也都有符合預期，那麼今天的教學也到這邊嘍~
