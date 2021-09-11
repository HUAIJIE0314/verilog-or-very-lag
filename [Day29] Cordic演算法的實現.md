# [Day29] Cordic演算法的實現

假設今天再做某種數位信號處理時，不小心用到了arctan(y/x)函數，那麼當然可以用泰勒展開得到多項式，化成一連串的乘法與加法運算，但是在這裡其實有另一個更有效率的方法，那就是基於座標旋轉式計算機(Coordinate Rotation Digital Computer, CORDIC)算法。

> 而Cordic算法也被運用在眾多場合當中，例如DSP、適應性濾波器、FFT(Fast Fourier Transform)、DCT(Discrete cosine Transform)、解調器及神經網絡上。

- step1:資料的預處理(要將不在-90度~90度內的座標旋轉至此角度內)
  - 若x為正(座標在右半部)：直接將x載入x暫存器，y載入y暫存器。
  - 若x為負，y為正(座標在第二象限)：將座標-x載入y暫存器，y載入x暫存器(意思是將座標向右旋轉90度至第一象限)。
  - 若x為負，y為負(座標在第三象限)將座標x載入y暫存器，-y載入x暫存器(意思是將座標向左旋轉90度至第四象限)。
- step2:
  - 如果x軸之上則將此座標向右旋轉45度(要有變數來記錄目前轉了多少角度了)。
    - x = x + y
    - y = y - x
  - 否則向左旋轉45度。
    - x = x - y
    - y = y + x
- step3:
  - 如果x軸之上則將此座標向右旋轉26度。
    - x = x + y/2
    - y = y - x/2
  - 否則向左旋轉90度。
    - x = x - y/2
    - y = y + x/2
- step4:
  - 如果x軸之上則將此座標向右旋轉14度。
    - x = x + y/4
    - y = y - x/4
  - 否則向左旋轉14度。
    - x = x - y/4
    - y = y + x/4
- step4:
  - 依照step2 ~ step4的規律做到達到所需精度即可。 
- step5:最後結果即是：
  - x為半徑也就是開號的(x<sup>2</sup> + y<sup>2</sup>) (但會是1.618倍)
  - 而記錄角度變數的則是arctan(y/x)的最後輸出
  - y則是誤差值


`註：角度在下圖中`

![](https://i.imgur.com/T8VaffL.png)

[圖片出處](https://blog.csdn.net/lz0499/article/details/100002361)

**這邊來寫一個簡易的Cordic電路**

**宣告狀態**

```verilog
/*------parameter------*/
localparam IDLE    = 3'd0;
localparam LOAD    = 3'd1;
localparam FIRST   = 3'd2;
localparam SECOND  = 3'd3;
localparam THIRD   = 3'd4;
localparam FINISH  = 3'd5;
```
**定義輸入輸出**

```verilog
module Cordic#
(
  parameter width = 7
)
(
  clkSys, 
  rst_n,
  en,
  x_in,
  y_in,
  radius,
  phase,
  error,
  ready
);
/*------ports declaration------*/
input                   clkSys;
input                   rst_n;
input                   en;
input  signed [width:0] x_in;
input  signed [width:0] y_in;
output        [width:0] radius;
output        [width:0] phase;
output        [width:0] error;
output                  ready;
reg                     ready;
reg    signed [width:0] radius;
reg    signed [width:0] phase;
reg    signed [width:0] error;
```

**宣告變數**

```verilog
/*------variables------*/
reg    signed [width:0] x[3:0];
reg    signed [width:0] y[3:0];
reg    signed [width:0] z[3:0];
reg               [2:0] fstate;
```

**狀態邏輯**

```verilog
/*------fstate_machine_state------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)fstate <= 2'd0;
  else begin
    case(fstate)
      IDLE:begin
        if(en)fstate <= LOAD;
        else  fstate <= IDLE;
      end
      LOAD:    fstate <= FIRST;
      FIRST:   fstate <= SECOND;
      SECOND:  fstate <= THIRD;
      THIRD:   fstate <= FINISH;
      FINISH:  fstate <= IDLE;
      default :fstate <= IDLE;
    endcase
  end
end
```

**輸出邏輯**

```verilog
/*------fstate_machine_output------*/
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    x[0]<=0;x[1]<=0;x[2]<=0;x[3]<=0;
    y[0]<=0;y[1]<=0;y[2]<=0;y[3]<=0;
    z[0]<=0;z[1]<=0;z[2]<=0;z[3]<=0;
    ready  <= 1'b0;
    radius <= 0;
    phase  <= 0;
    error  <= 0;
  end
  else begin
    case(fstate)
      IDLE:begin
        x[0]<=0;x[1]<=0;x[2]<=0;x[3]<=0;
        y[0]<=0;y[1]<=0;y[2]<=0;y[3]<=0;
        z[0]<=0;z[1]<=0;z[2]<=0;z[3]<=0;
        ready <= 1'b0;
      end
      LOAD:begin
        if(x_in >= 0)begin
          x[0] <= x_in;
          y[0] <= y_in;
          z[0] <= 8'd0;
        end
        else if(y_in >= 0)begin
          x[0] <= y_in;
          y[0] <= -x_in;
          z[0] <= 8'd90;
        end
        else begin
          x[0] <= -y_in;
          y[0] <= x_in;
          z[0] <= -8'd90;
        end
      end
      FIRST:begin
        if(y[0] >= 0)begin
          x[1] <= x[0] + y[0];
          y[1] <= y[0] - x[0];
          z[1] <= z[0] + 8'd45;
        end
        else begin
          x[1] <= x[0] - y[0];
          y[1] <= y[0] + x[0];
          z[1] <= z[0] - 8'd45;
        end
      end
      SECOND:begin
        if(y[1] >= 0)begin
          x[2] <= x[1] + ({y[1][width], y[1][width:1]});//y[1]>>>1
          y[2] <= y[1] - ({x[1][width], x[1][width:1]});//x[1]>>>1
          z[2] <= z[1] + 8'd26;
        end
        else begin
          x[2] <= x[1] - ({y[1][width], y[1][width:1]});
          y[2] <= y[1] + ({x[1][width], x[1][width:1]});
          z[2] <= z[1] - 8'd26;
        end
      end
      THIRD:begin
        if(y[2] >= 0)begin
          x[3] <= x[2] + ({{2{y[2][width]}}, y[2][width:2]});//y[2]>>>2
          y[3] <= y[2] - ({{2{x[2][width]}}, x[2][width:2]});//x[2]>>>2
          z[3] <= z[2] + 8'd14;
        end
        else begin
          x[3] <= x[2] - ({{2{y[2][width]}}, y[2][width:2]});
          y[3] <= y[2] + ({{2{x[2][width]}}, x[2][width:2]});
          z[3] <= z[2] - 8'd14;
        end
      end
      FINISH:begin
        radius <= x[3];
        phase  <= z[3];
        error  <= y[3];
        ready  <= 1'b1;
      end
      default:begin
        x[0]<=0;x[1]<=0;x[2]<=0;x[3]<=0;
        y[0]<=0;y[1]<=0;y[2]<=0;y[3]<=0;
        z[0]<=0;z[1]<=0;z[2]<=0;z[3]<=0;
        ready  <= 1'b0;
        radius <= 0;
        phase  <= 0;
        error  <= 0;
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
module tb_Cordic();
localparam width = 7;
reg                    clkSys;
reg                    rst_n;
reg                    en;
reg   signed [width:0] x_in;
reg   signed [width:0] y_in;
wire  signed [width:0] radius;
wire  signed [width:0] phase;
wire  signed [width:0] error;
wire                   ready;
Cordic UUT (
  .clkSys(clkSys), 
  .rst_n(rst_n),
  .en(en),
  .x_in(x_in),
  .y_in(y_in),
  .radius(radius),
  .phase(phase),
  .error(error),
  .ready(ready)
);

always #5 clkSys = ~clkSys;

initial begin
  clkSys = 0;
  rst_n = 0;
  x_in = 0;
  y_in = 0;
  en = 0;
  repeat(5)@(posedge clkSys)rst_n = 0;
  rst_n = 1;
  test(-41,55);
  test(30, 40);
  #1000 $stop;
end
/*----------task---------*/
task test;
input signed [width:0] in1;
input signed [width:0] in2;
  begin
    #30;
    x_in = in1;
    y_in = in2;
    #10 en = 1;
    #10 en = 0;
    wait (ready == 1);
  end
endtask

endmodule

```

**Wave**

![](https://i.imgur.com/v4cQyAU.png)

可以看到(-41,55這個座標)：
> r理論值為1.618x根號((-41)<sup>2</sup> + (55)<sup>2</sup>) = 110.995，而此電路算出來為111，再來是arctan的部分，arctan(55/-41) = 126度，此電路算出來為123度，雖然有點誤差(畢竟是簡易版)。

那麼這次的教學就在這邊告一個段落了歐~~
