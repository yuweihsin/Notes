# Redfish Introduction
## 前身:IPMI
* 由intel和HP主推IPMI標準，引入單獨的帶外(out-of-band)管理芯片BMC:
所謂帶外，是指在CPU這個主計算資源外，有了這個ARM based芯片加持，和開放免費的IPMI標準，服務器的管理上了一個台階。IPMI定義了一個所謂的服務器管理的最小集，並標準化了相關命令集合，IPMI可以架構在網路、串行/Moderm接口、IPMB(I2C)、KCS、SMIC、SMBus等不同接口上。
* IPMI的管理網路是有一個專有的網路，只有授權的用戶才能訪問，這導致其在開始的時候就對安全性考慮有所欠缺，在爆出安全漏洞後，IPMI2.0增加了增強身分認證(RAKP+、SHA-1等)，但其後更有別的漏洞爆出，故業界呼喚一種新的標準，一勞永逸的解決這些問題，**Redfish**才因此誕生，IPMI也在2015年公布2.0 v1.1標準後，不再更新，被Redfish永久代替，Intel也宣布不在維護，號召大家轉戰Redfish。
## Redfish
* Redfish 是在2015年由DMTF(Distributed Management Task Force) 這個組織開始著手建立的伺服器管理標準，官方的描述是:
**A standard, Redfish is designed to deliver simple and secure management for converged, hybrid IT and the Software Defined Data Center (SDDC). 
-->作為一項標準，Redfish 旨在為converged， hybrid IT 和 SDDC 提供簡單而安全的管理**
這邊的SDDC就是"**Software Defined Data Center(SDDC)-軟體定義數據中心**"，簡單來說就是希望未來能**提供一軟體工具及來管理這些虛擬化資源**，但在伺服器的領域，長期發展且成熟的協定一直是IPMI，對於數據中心的管理者/客戶端的反饋是他們並不了解IPMI這個協定，他們人員都需要重新學習，而且很多現代化管理工具並不能直接應用在IPMI上面，所以Redfish誕生的契機就出現了。
* **What is Redfish**
  - 用於 IT 基礎架構的行業標準 RESTful API
  - 基於 Odata v4 的 JSON 格式的 HTTPS
  - 應用程式、GUI 和腳本同樣可用
  - Schema-backed但可讀性高
* 再標準訂立之初，就設定了以下目標:
  - 1.安全
  - 2.高可擴展管理(Scalable)
  - 3.人類可讀數據介面(Human readable data)
  - 4.基於現有硬件可實現
  - 第四條十分淺顯易懂，也就是現在support IPMI的BMC上，不需要(或者很小)硬件改動，就可以support Redfish，也就是硬件兼容
  - 安全性依賴TLS-Secured HTTP，也就是HTTPS來保證
  - 高擴展通過定義所有的API為RESTful形式的API來完成。