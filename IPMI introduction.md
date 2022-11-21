# IPMI introduction
## What's BMC solution? Why we need it? Who will use it?
• BMC (Baseboard Management Controller):
用來<font color="#f00">管理</font>和<font color="#f00">串接</font>主機板上<font color="#f00">不同功能的模組和元件</font>(包含電壓、溫度、風扇等元件)，使主機板上的元件經由BMC的管理能順暢運作。

• BMC是獨立主系統之外，<font color="#f00">獨立運作</font>，與OS、CPU<font color="#f00">無關</font>，不需要開啟作業系統才能做事，<font color="#f00">只要系統有電源時就可以開始做監管的動作</font>。

• BMC藉由蒐集基板個別元件與串接硬體的運作數據，若偵測到<font color="#f00">異常狀況</font>時發出警示訊息給網管人員，再<font color="#f00">依照BMC提供的資訊進行遠端診斷及問題排除</font>，進而提升伺服器自動化的運作機能，減少人工監控及故障故障排除的時間成本。


----
## 一般BMC具有以下功能：
1. 通過系統的串列連線埠進行訪問
2. 故障日志記錄和 SNMP 警報傳送
3. 訪問系統事件日志 
(System Event Log ,SEL) 和感測器狀況
4. 控制包括開機和關機
5. 獨立于系統電源或工作狀態的支持
6. 用于系統設定、基于文本公用程式和作業系統控製台的文本控製台重定向

* BMC Chip(目前常用的Ast2400, Ast2500, Pilot4…etc.)是ARM architecture的Chip，內部使用的Firmware主要是使用Embedded Linux為基礎，在上面開發各式各樣的功能，對外是透過I2C/IPMB, LPC, Uart/COM port, LAN跟其他設備或是外界溝通
* LPC:Low Pin Count Bus
* PCIE:PCI Express

![](/uploads/upload_f3e2855dc55659455c27af5e61194781.png)

---
## The basic IPMI introduction
* 各家廠牌BMC的設計架構、運作邏輯有所不同，在網管實施伺服器遠端管理時，常面臨不同廠牌BMC相容性和管理的問題；故Intel與戴爾電腦、NEC、惠普等伺服器廠商，共同建立「<font color="#f00">智慧平臺管理介面（Intelligent Platform Management Interface，IPMI）」</font>，可讓<font color="#f00">不同廠商的BMC具有高度相容性、互操作性</font>，藉此解決複雜的裝置及參數管理問題。
* IPMI由於可相容不同廠牌的BMC，因此網管可對<font color="#f00">多臺BMC下達相同指令</font>，並且蒐集不同BMC所回傳的系統運作參數，更有利網管人員針對不同的BMC做出診斷，排除不同廠牌BMC所發生的問題，減少後勤維護的困擾和成本。

![](/uploads/upload_56313a66a2d1805e458878e62071ded4.png)

----
## HOW to Use IPMI? 
* 在 Linux 系統下可以利用 ipmitool 來實現對伺服器的 ipmi 管理。
* Intel Spec. [(IPMI Specification, V2.0, Rev. 1.1: Document)](https://www.intel.com.tw/content/www/tw/zh/products/docs/servers/ipmi/ipmi-second-gen-interface-spec-v2-rev1-1.html)
* 如果要透過遠端管理控制 IPMI 這時候需要使用到 **RMCP (Remote Management Control Protocol)) 協定**
* 所有的IPMI功能都是**向BMC傳送命令來完成的**，BMC接收並在系統事件日誌中記錄事件訊息，維護描述系統中感測器情況的感測器資料記錄。而通過IPMI ，使用者可以主動監測組件的狀況，以確保不超出預置閾值，例如伺服器溫度。這樣，通過避免不定期的斷電，協助維護了 IT 資源的運行時間。 **IPMI的預告故障能力也有助于 IT 周期的管理**。通過檢查系統事件日志 (SEL)，可以更輕松的預先判定故障組件。
---
## IPMI 架構方塊圖

![](/uploads/upload_8364416b8081d66e2915a7004a7496e5.png)

---
![](/uploads/upload_4713a69c34116a5f4cea2e5069956bd7.png)
### *System Bus*:
> BMC透過下方的**System Interface**與 HOST(主機)相連是經由**System Bus (PCIE or USB or LPC Bus)** 做傳遞，然而有些系統、元件沒有這interface(Ex:DIMM、CPU、power board…)， 而使用**IPMB做management controllers間的溝通** ， IPMB 是在major system modules之間的匯流排，是以I2C-based serial bus為基礎通訊，因為遠離主要的BMC，又被稱為 **satellite controllers(衛星控制器)**。 

### *Field Replaceable Unit (FRU) Information*:
> 是一種**EEPROM**電子抹除式可複寫唯讀記憶體， IPMB 和管理控制器可存取系統中各種模塊的現場可更換單元 (FRU) 信息 ，**裡頭包含了製造商、序號和出廠日等data，作為一個”身分認證”，讓BMC知道是什麼元件及是否可安全溝通使用等**。
> 當主處理器的 FRU 存取機制故障時，允許**在故障條件下**獲取 FRU 信息。這有助於創建自動遠程庫存和服務應用程序。
> 使用FRU的元件包含Memory board(**DIMM**)、Processor board(**CPU**)、Redundant Power board(**power supply**) **Chassis Board** 等。
> IPMI 以兩種方式提供 FRU Information：
> 通過Management Controllers或通過 FRU SEEPROM。使用 IPMI command存取由管理控制器管理的 FRU Information 。

### *IPMI massaging*
> 可視為一個**匯流排**，根據傳輸的需要，**‘wrapped’(包)成不同的Interface並更改底層的傳輸‘driver’**，讓user可在不同的Sensor、board上用IPMB、ICMB等Bus以**Message Queue** 做訊息的交換。

---

![](/uploads/upload_b8a0eccfd59d065bf9c342ceec36ec14.png)
### *System Interface*
> IPMI 定義了用於將 IPMI messages傳輸到 BMC 的三個standardized system interfaces 。 為了支援各種microcontrollers ，IPMI 提供了多種System Interface選擇。 使用這些 Inerface 是啟用cross-platform software的關鍵。 System Interface非常相似，因此可以創建一個支援所有 IPMI system interfaces的驅動程序。 system interfaces 連接到可由主處理器驅動的system bus ，當前的 IPMI system interfaces可以是 I/O or memory mapped ，可以使用任何允許主處理器訪問指定 I/O memory locations並滿足時序規範的System Bus。 因此，IPMI 系統接口可以連接到 X-Bus、PCI、LPC 或 a proprietary bus off the baseboard chip set. 。
IPMI 系統接口包括：
**Keyboard Controller Style (KCS) 
System Management Interface Chip (SMIC) 
Block Transfer (BT) 	
SMBus System Interface (SSIF)**

---
![](/uploads/upload_c0c29df3c7da09a5a8d4877c1abe594a.png)
### *Non-volatile Storage (非揮發性記憶體)包括:*
### *SEL(System Event Log)*
> 用於**存儲系統平台事件訊息**以供檢索，IPMI command允許讀取和清除 SEL，並將Event添加到 SEL，用於向 SEL 添加事件的request message （command）稱為**Event Message**， Event Message可**以通過 IPMB 發送到 BMC**。這為satellite controllers(IPMB)提供了檢測事件並將它們登錄到 SEL 的機制。通過 IPMB 向另一個控制器:
> **生成事件消息的控制器稱為 IPMB Event Generator 。
> 接收事件消息的控制器稱為 IPMB Event Receive 。**
> Generic(通用) Event Receiver是一個控制器，它通過連接到它的任何媒體以及內部生成的事件消息接受平台事件消息命令。 
> **BMC 通常是System中唯一的Generic Event Receive** 。
> Management Controllers that generate Event Messages
> 必須知道Sensor、Event的類型，以便它可以將該訊息放入Event Messages中。這確保Event Messages可以攜帶的重要的訊息，而無需訪問Sensor的Data Record。


### *SDR(Sensor Data Record)*
> 提供platform管理Sensor類型、位置、Event生成和存取資訊，通過 **‘out-of-band’** (用獨立管理通道進行裝置維護)來retrieved SDR，可獨立於main processors, BIOS,system management software, and the OS，SDR還會記錄**連接到IPMB的device數量、FRU的類型和位置**。
> SDR 中包含的信息表明傳感器**正在監控哪個系統實體**（例如內存板），還提供指向該實體的 FRU 信息的鏈接。 
> SDR 使用一組代碼來指定哪個控制器持有Sensor、Sensor類型（例如溫度）、Sensor的Event和讀取類型（例如離散或基於閾值），相同的代碼和位字段**直接映射到在Event Message中傳遞並記錄在 SEL 中的信息**。
> SEL 可以**表示與Event相關聯的控制器、Sensor類型和Event類型**。此訊息可將Event與 FRU 關聯，可以幫助User引導到問題區域，甚至可以用來識別他們應該更換的部件。

---

![](/uploads/upload_c4229c3bb148e2b8ebc230f85fb68bf0.png)
### *LAN Interface* 
> LAN Interface 定義**如何將 IPMI messages發送到封裝在 RMCP（遠程管理控制協議）數據包中的 BMC**。此功能也稱為 **“IPMI over LAN”**。 IPMI 還定義了和特定於LAN有關的configuration interfaces ，用於設置諸如 IP 地址等其他選項，以及用於發現IPMI-based systems的Command。
> 分佈式管理任務組 (DMTF) 指定 RMCP 格式，通過 DMTF 的Alert Standard Forum(警報標準論壇) “ASF”規範，這種相同的packet格式用於non-IPMI messaging 。
> 使用 RMCP packet格式可以在包括 IPMI 和 ASF 系統的環境中運行的管理應用程序之間實現更多的通用性。
> **IPMI v2.0** 定義了一種擴展的數據包格式和功能，統稱為“RMCP+”。 RMCP+ 實際上是在 RMCP 數據包的 IPMI 特定部分下定義的。 RMCP+ 使用的身份驗證算法與 ASF 2.0 規範所使用的機制更加一致。此外，RMCP+ 增加了數據機密性（加密）和“payloads”功能。

