
# [Day14] verilog中的可綜合語句
> 我們都知道verilog是一種硬體描述語言，所以目的就是要能綜合出實際的電路，但實際上在verilog中並不是所有語句都是可綜合的，因為有些語句是用來驗證(TestBench)的關鍵字，屬於那些驗證用的語句只能在驗證時被使用，例如initial、time、wait...等等，所以在設計數位電路時，一定要特別注意電路的可綜合性~~

**所有綜合工具都支持的語法：**
> always, assign, begin, end, case, wire, tri, aupply0, supply1, reg, integer, default, for, function, and, nand, or, nor, xor, xnor, buf, not, bufif0, bufif1, notif0, notif1, if, inout, input, instantitation, module, negedge, posedge, operators, output, parameter。

**有些綜合工具支持但有些不支持的語法：**
> casex, casez, wand, triand, wor, trior, real, disable, forever, arrays, memories, repeat, task, while。

**所有綜合工具都`不`支持的語法：**
> time, defparam, $finish, fork, join, initial, delays, UDP, wait。

---

## 撰寫可綜合電路應該要保持的幾項原則：
- 不應該使用initial語句來初始化電路的信號。
  - 所有的初始化動作都應該靠reset或是reset_n來達成。
  - 寫MUC並燒錄進去後要按下reset鍵才會動作的原因也是為此，就是為了初始化電路內的各個信號。
  - 由於信號應由reset或reset_n來初始化，所以宣告reg時也不會先給初始值，那是沒有必要的。
- 整體電路應該為同步式設計。
  - 意思是always內的觸發信號應當只有clk以及reset信號。
  - 由電路內自己產生的信號當作是always的觸發是不算同步式設計的。
- 對同一個變數不可同時使用blocking及non-blocking。
  - 前幾篇有提到組合邏輯用blocking，而循序邏輯用non-blocking，依照此原則也不會誤用~
- 不使用UDP(User Defined Primitives)
  - 是一種允許用戶自己定義的元件，通過UDP，可以把一塊組合邏輯電路或者循序邏輯電路封裝在一個UDP內，並把這個UDP作為一個基本的元件來引用，但UDP不能綜合，只能用於仿真。
- 不該在程式內使用delay語句(ex: #100)。
- 同一個變數不可由兩個不同的always來賦值。
- 同一個變數不應該由多個不同的clk來做觸發。

