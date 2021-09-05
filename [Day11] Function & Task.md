
# [Day11] Function & Task
很多人對於Function以及Task有點混亂，這篇將帶你搞懂他們~

## 共有的特色
- 他們都會寫在Module內而不是外面。
- **都不能使用wire變數**(原因大概也是wire不具有記憶性吧)。
- 被使用時都會放在always內(所以適用Behavioral level)。
- 裡面都不能放always block。

---

## Function
- 可以引用其他的Function，但就是不能引用task。
- 至少要有一個以上的input。
- 最多只能有一個Output。
- 一定出現在等號右邊(有output值)。

EX:
`輸入數字計算需要的位寬大小`
```
function integer log2;
  input integer in ;
  for(log2=0; in>1; log2=log2+1) begin
    in = in >> 1 ;
  end
endfunction
```
引用 log2：
```
parameter width = 16;
reg [log2(width)]:0]a;//5 bit register a
```
`在這邊值得注意的是，上面雖然用了integer，你可能會想，這樣不是很耗硬體資源嗎，不如自己算好log2再打上數字還比較省資源，但其實那些數字會在 "前置處理器" 就先幫你算出來了，所以並不會合成出實際的電路~`
`(在quartus編譯好然後查看RTL Viewer就可以知道了~)`

---

## Task
- 可以引用其他的Function以及task。
- 不一定要宣告input、output，有的話可以有數個都沒問題。

EX:
`七段顯示解碼(共陰)`
```
task give_seg;
  input reg [13:0]in;
  output reg [7:0]out;
  casex(in)
    14'd0:out = 8'b11111100;
    14'd1:out = 8'b01100000;
    14'd2:out = 8'b11011010;
    14'd3:out = 8'b11110010;
    14'd4:out = 8'b01100110;
    14'd5:out = 8'b10110110;
    14'd6:out = 8'b10111110;
    14'd7:out = 8'b11100000;
    14'd8:out = 8'b11111110;
    14'd9:out = 8'b11100110;
    default:out = 8'b00000010;
  endcase
endtask
```
