
# [Day10] 模組化及引用模組
## 模組
在一個.V檔案裡面，可以有很多個module，但是Top Module只會有一個，所以檔名必須以Top Module.v來命名來辨別Top Module。

---

## 模組化的概念
- 每個模組的實際意義是一塊實際的硬體電路。
- 每一塊模組都有自己的功能，再透過連接各個模組來達成特定功能。
- 模組與模組之間也會是並行處理的。

---

## 引用模組
為甚麼會需要引用模組呢？

因為通常一個大模組往往會已由許多小模組所組成的，這樣便有以下好處：
- 一方面是可讀性比較好，往後在維護方面也比較方便，因為電路分為一塊一塊的比較好去知道各個模塊的功能，再針對特定的模塊做優化就好。
- 再來是除錯，合成為大電路前，如果可以先提前測試每個模塊功能是否正常，那debug起來也會比較快

---

## 引用模組的兩種方法
將模組的埠與其他模組連接的方法有兩種，分別是:

1.依照要引用之模組的埠列「**順序**」(**in order**)來連接，也就是如果要引用的模組是：
```verilog
module test(
  clkSys, 
  rst_n
);
..
...
....
endmodule
```
那麼引用時括號內的順序就會對應到該模組括號內的順序，例如：
```verilog
reg clk;
reg reset_n;
test U0(clk, reset_n);//module instantiation
```
那麼這個模組的{clk, reset_n}就會接到test模組的{clkSys, rst_n}

2.依「指定名稱」(by name)的方法來連接，會以" .該模組腳位(此模組腳位) "來引用，例如：
```verilog
reg clk;
reg reset_n;
//module instantiation
test U0(
  .clk(clk), 
  .reset_n(reset_n)
);
```
這樣也會是一樣的效果~~

而不管是哪一種引用法引用時都要給模組命名，像這邊就是命名為"U0"

>**對於一個較大的電路來說，可能接腳會非常的多，此時如果以in order的方式來連接，這個時候就算邊看邊打也相當不方便且容易犯錯，所以通常我們會以by name的方法來連接來避免不必要的失誤。**
