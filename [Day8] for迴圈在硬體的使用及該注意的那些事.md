
# [Day8] for迴圈在硬體的使用及該注意的那些事
## for-loop
在C/C++語言中，我們經常用到for迴圈語句，但在verilog中for語句的使用上會有很大的區別。

Verilog的for迴圈會經常在TestBench上做使用，一方面是TestBench只是拿來測試電路功能的正確性，而且在TestBench中生成信號用for迴圈也比較方便，不需一行一行打。

但是在RTL(Register-Transfer-Level)中卻很少使用for迴圈，其原因是，for迴圈會被綜合器展開為所有變數情況，每個變數獨立佔用暫存器資源，且每條執行語句並不能有效地複用硬體邏輯資源，而這樣的情況會造成消耗大量硬體資源，迴圈次數越多次，整體合成面積會越大，for迴圈雖然方便，但換來的則是速度上的下降。

雖然for迴圈不是那麼理想，但還是有使用的場合，例如`初始化二維陣列`時就需要，因為在verilog中不能直接對整個二維列賦值，此時就需要用index去跑每一個變數。

EX:
```
reg [63:0]mem[255:0];
integer i;

always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    for(i=0;i<256;i=i+1)begin//verilog沒有i++的用法
      mem[i] <= 64'd0;
    end
  end
  else begin
    ......
  end

end
```

另一個情況就是就是在時間與面積的trade-off過後，真的有需要在單獨一個clk完成一些處理的需求，也可以適當使用for迴圈，例如雙迴圈，雙迴圈可能耗的硬體資源太多，則可以將外迴圈拆開，一個clk後index才加一，而內迴圈依舊保持。

`總結：雖然for迴圈是可以綜合的，而且效率很高。但所消耗的邏輯資源較大。在對clk數要求不是很高的情況下，可以多用幾個時鐘週期來取代for迴圈的使用。`
