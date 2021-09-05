
# [Day07] Behavior Level
## always block
- always若超過一行要用begin、end包起來。
- always內的變數若要賦值(等號左邊的變數)必須是reg型態，而等號右邊可以是wire或reg。
- always的觸發條件若超過一項則以 ' , ' 或是' or '  區分。
- 正緣觸發用posedge，負原則是negedge。
- 若想以任意一個訊號b的變動來當作觸發原則是寫成 always @(b)
- 若想隨時執行這個block寫成 always @(*)

EX:
```
always@(posedge clkSys or negedge rst_n)begin
  if(!rst_n)begin
    .....
  end
  else begin
    .....
  end
end

```
---

## if-else語句
這裡的if-else跟C語言是相同的用法，不過值得一提的是，`在寫verilog時else最好要寫，避免電路的描述不完整，容易產生latch。`

EX:
```
if(...)begin
  if()begin
    ....
  end
  else begin
    ....
  end
end
else if(...)begin
  ....
end
else begin
  ....
end
```

---

## case、casex、casez語句
- 一般case中的item不能有'z'或是'x'，只能出現'0'、'1'。
- casez中的item值除了”0”、“1”外，'z'也可以出現。
- casex中的item值除了”0”、“1”外，'z'及'x'都可以出現。
- 最後記得endcase。
- 除外的狀況寫在default內。

EX:
```
case(...)
  item_1:begin
    ....
  end
  item_2:begin
    ....
  end
  item_3:begin
    ....
  end
  item_4:begin
    ....
  end
  default:begin
    ....
  end
endcase
```

`這邊的default跟if-else中的else一樣，不管有沒有用，最好都加上去，避免導致latch。`
