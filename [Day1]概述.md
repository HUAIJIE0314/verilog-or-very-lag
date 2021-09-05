# [Day1] 概述
## Verilog是什麼？
Verilog是一種硬體描述語言(**Hardware Description Language, HDL**)，用於數位電路的系統設計，是一種描述數位電路的語言，設計者設計數位電路時可以透過這種語言來描述自己的電路設計想法，利用EDA Tool來幫你完成電路設計。
目前HDL分為兩大類，有Verilog及VHDL兩種，在歐洲國家以VHDL較為普遍，而亞洲國家則是以Verilog較為多人使用。

---

## 撰寫Verilog時應該注意的那些事
前面說過Verilog是硬體描述語言，描述硬體的語言，雖然語法與C相似，但概念卻不太相同，c語言是由上至下一行一行的執行，而verilog是每個always block都會同步執行的，因此在設計時就要特別注意，並且要以硬體的角度去寫，否則寫出來的程式可能不會那麼理想。

Verilog主要有四種層級的描述方法：
- Behavioral level:
  - Verilog HDL中的最高層，我們只需針對電路的功能來做設計，不需要考慮底層硬體架構。
- Dataflow level:
  - 這裡必須指明訊號處理的方法，在這裡會使用assign語法來對訊號做處理。
- Gate level:
  - 這裡是由邏輯閘所連接而成。
- Switch level:
  - 這裡是由電晶體元件所連接而成。(*現在幾乎沒什麼在用了*)

`不過通常寫Verilog 時只會用到 Behavioral level 以及 Dataflow level。`

