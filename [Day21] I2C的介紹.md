# [Day21] I2C的介紹

## I2C是什麼？
I2C，又稱I²C（Inter-Interated Circuit)，在I2C的通訊協定中，收發資料只單純靠兩條線就能完成，分別為SCL(serial clock)以及SDA(serial data)，但比較特別的是I2C bus上所連接的裝置都是open-drain的方式來驅動信號的，那跟一般的數位邏輯輸出電路有著甚麼樣的差異呢？

一般的數位邏輯輸出電路會有分別把電位拉到high及low的兩顆電晶體，但是open-drain驅動的電路則是只有負責把電位拉到low的電晶體，而高電位則是由外部的上拉電阻負責輸出。

那這樣一條clk一條data要如何溝通呢？就是因為只有一條資料線，所以每個裝置端的SDA都可以是雙向溝通的，它既可以當輸入也可以當輸出，需要"0"時打電位拉到0，需要"1"時輸出高阻抗，此時裝置與BUS端有如斷路，讓上拉電阻到Vcc來提供"1"。

![](https://i.imgur.com/t7PQ5Jp.png)

[圖片來源](https://www.analog.com/en/technical-articles/i2c-primer-what-is-i2c-part-1.html)

---

## I2C 的 Timing Diagram

![](https://i.imgur.com/zchONpC.png)

[圖片來源](https://www.analog.com/en/technical-articles/i2c-primer-what-is-i2c-part-1.html)

**I2C bus 上的邏輯訊號有幾個原則：**

- 當未傳輸時(IDLE狀態)，SCL和SDA都會維持在high電位。
- SCL為high時，表示SDA上的資料為有效，此時SDA的值不能改變，以確保可以收到到正確的SDA狀態
- SCL為low時，SDA的狀態可以改變
- 當SCL為high時，如果SDA變動，只有兩種情況：
  - SDA由 "1" 變 "0" ---> START
  - SDA由 "0" 變 "1" ---> STOP

**Acknowledge**
當每8 bit資料傳輸完後，會跟隨著1 bit的ACK，這是在確保兩端的接收狀況是否正常。
- 如果是master傳資料給slave：此時ACK會來自slave端，以此告訴master端它有收到資料(通常用於對slave下instructions)。
- 如果是master想要向slave端索取資料時，舉個例子，例如讀取記憶體資料：master會先slave傳送想讀取的記憶體"位址"，接著slave端會先ack回去表示有收到此位址，接著傳出記憶體位址上的data給master，那麼傳了8 bit後，master會ACK回去，以表示有收到資料，可以繼續傳，那麼slave端就會繼續傳，再過了8 bit，此時如果傳輸要結束的話則master不回ACK回去，而這個動作也稱non-ACK，以此告訴slave端這次的傳輸到此為止結束了，完成這次通訊。


## I2C如何選擇晶片的呢？
在master要與BUS上的其中一個slave溝通時，會先將要溝通chip的address透過SDA傳輸到BUS上，此時如果address沒錯的話，那麼該slave會在下第八個SCL週期結束後把SDA拉下來，而master端會在SCL進入第九個週期前去讀取SDA是否為low，為low的話就會開始一連串handshake式的傳輸動作，若沒有拉為low(non-ACK)，則出現問題，可能slave根本不在BUS上或是slave端的address打錯了都有可能。
