---
title: "ChipWhisperer 出廠設置報告分析"
date: 2026-03-14
draft: false
tags: ["fault-injection"]
---

## 電壓故障注入(Voltage Glitching)
簡短地説，就是在晶片運算的瞬間，把電壓瞬間拉低再恢復

當你要glitch 的時候，ChipWhisperer 送一個訊號讓 mosfet 瞬間打開。Mosfet 一開，就等於在電源和地線之間開了一條捷徑，電流不走晶片了，直接從這條捷徑泄到地。

幾十奈秒後，mosfet 關閉，捷徑消失，電壓恢復正常

### High-Power 和 Low-Power 的差別
ChipWhisperer 提供兩種模式,分別是 High-Power 和 Low-Power

拉低電壓的方式就是接一顆 mosfet,把電源瞬間短路到地。但目標晶片的電路板上通常有旁路電容，會自動補電，抵抗電壓下降。耗電量大的晶片，板上的旁路電容大，用 Low-Power 根本拉不下來（只能打簡單的小晶片,IOT 裝置），所以 High-Power 用電流承載能力更大的 mosfet ，才打的動旁路電容強的目標

![High-Power Graph vs Low-Power Graph](power-glitch.jpeg)

High-Power: 一開始電壓穩定在 3.3V。然後大約0.3*10^-7 秒的位置，電壓突然被猛拉到大約1V。這就是 mosfet 打開，把電源短路到地的那一瞬間。之後電壓迅速彈回。但也因爲吸力過猛，關閉後，電路會產生震蕩

Low-Power: mosfet 電流承載能力小，吸電流的速度慢，所以電壓是慢慢滑下去的，不是像 High-Power 這種跳崖式的。恢復的時候也慢


## CPA and Bandwidth Test  

![CPA and Bandwidth Test](cpa_bandwidth.jpeg)

### CPA Test
X軸是 trace number。每次AES 加密時，就用示波器記錄下晶片的耗電變化，這就是一條trace。
Y軸: Average PGE(Partial Guessing Entropy)，簡單講就是離猜對金鑰還有多遠。PGE 代表還猜不到; PGE=0，代表金鑰已經排在第一名，攻擊成功。


### Bandwidth Test
上面的綫量測到的是真實訊號。我們可以看到隨著訊號頻率越來越快，板子的硬體極限開始出現，捕捉能力慢慢下降
下面的綫是底噪(Noise Floor),這代表沒有任何輸入訊號時，電路板本身(因爲電流移動，環境電磁波等) 自然產生的雜訊

這張圖要表達的概念是雜訊比(Signal-to-Noise,SNR)。要在含有雜音的環境中破解密碼，關鍵不在於訊號有多强，而是訊號比雜音强上多少。圖中兩者的差距相差了足足50dB。

## 其他資訊

![Other Info](chipwhisperer_info.jpeg)

報告上有一區叫做 LNA Gain Adjustment Test。LNA 的全名是 Low Noise Amplifier(低雜訊放大器)，它在板子上的功用，就跟麥克風放大器一樣，只是它錄得不是聲音，而是目標晶片極其微弱的耗電量變化。

### ChipWhisperer 的放大功能兩層控制
Hi/Low: 決定要不要接入一段額外的放大電路。就像音響的强力模式按鈕，按下去，所有聲音直接大一截

Gain Setting(0~60): 在上面的基礎做微調

### 高增益和低增益
- 什麽時候用高增益(High Gain)
當你攻擊目標是一顆非常省電，功耗極低的小晶片時，它運算的電流起伏非常非常小。這時你必須在設定裏把Gain 調高，把微小的波動放大，板子的上的ADC 才能看清楚訊號的長相

- 什麽時候用低增益(Low Gain):
大型設備(例如 SSD 控制器,這裏的大型是指耗電量大)，這種晶片本身耗電量比較大，電流起伏已經很明顯了。如果這時候還開高增益，訊號就會 "clipping"，失去所有破解密碼需要的細節。這時你就必須把 Gain 調低

### 爲什麽會做 AVR/STM32/XMEGA 等的 Programming Test
Chipwhisperer 出廠時都會測試說能不能把程式正常燒錄去目標板的微控制器的Flash 記憶體裏

### FGPA: Chipwhisperer 的核心
Chipwhisperer 板子上有一顆 FGPA,它負責所有需要奈秒級進準度的工作(偵測加密開始的瞬間，同步取樣，打出 glitch)。

