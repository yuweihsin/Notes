# LAN interface

## Ceate these queue (LAN_IFC_Q/LAN_RES_Q/LAN_MON_Q)
* Race condition:file system access
* Queue的特點:
  - 1. 非同步(send message不必檢查是否送到，可以做其他事情)
  - 2. 不必知道雙方address，只要send to queue
  - 3.queue不容易丟失、有暫存性，當某元件失效可提供data暫存，等元件恢復再重新上傳
  - 4.可以依需求add queue or reduce queue:
當web service 每秒須接受大量request，request不能被丟失，
但又要經過大量運算的function才能得到response，
所以需要有一個server可以保持接收request的狀態，不被request block住
--->在webservice和process service中加入message queue

## Behavior of these thread function (RecvLANPkt/LANTimer/LANMonitor)
* RecvLANPkt:
此功能接收IPMI LAN request packet，並將收到的packet post to LAN_IFC_Q。
* LANTimer:
handler socket timeout(能處理LAN的連接時間)
* LANMonitor:
監視網絡設備狀態變化，當發生變化時wake up並通知初始化socket(base on NCML configurations and POST_TO_Q by LAN_MON_Q)
 
## IPMI message communicate by queue (GetMsg(_Q)/SendMsg(_Q)) -> Message.c 
* **GetMsg(_Q)** : **Gets the message posted to this task** 
int getmsg(int fildes, struct strbuf *ctlptr, struct strbuf *dataptr, int flagsp);
Ex:Int GetMsg (_FAR_ MsgPkt_T* pMsgPkt, INT8 *Queuepath, INT16S Timeout, int BMCInst)

* **SendMsg(_Q)** : 
int sendmsg(int s, const struct msghdr *msg, int flags);

***Note:***
* VLAN : virtual LAN (Logic LAN) 實做於LAN switch上的網路管理技術(隔開廣播領域，提升頻寬利用率，降低延遲、增加安全性)
* MAC address : Media Access Control address，又稱LAN address，用於網路介面卡的識別碼(獨一無二) 由6組16進位數字組成。

---
## LANIfTask
![](/uploads/upload_b5c787a98b779e0e4e691401ce2c3add.png)

---
## IPMI LAN Interface
* 關於BMC與remote console之間的message handle，LAN連接方式包含IPv4、IPv6和UDP，其中UDP格式包含IPMI request and response，以及message的身分驗證。
* 有兩種方式可以使用LAN來建立IPMI LAN Interface : 
  - 1. 使用嵌入式的LAN controller。
  - 2. 在add-in card (擴充卡)上使用LAN controller。

* 兩者都有LAN controller ，允許IPMI LAN 訊息傳遞獨立於系統軟體和電源狀態，故即使斷電時仍可保持運行，並透過UDP讓訊息在Management Port 和 BMC之間流通。
* 第一種方式需要把LAN controller 嵌入system，其中BMC與LAN之間使用side band(ex:SMBus、I2C)做傳輸。
* 第二種方式可以不需要把LAN controller build在system中，方便進行LAN controller 的更新或新增。

![](/uploads/upload_53ab40e06087c044e0dd9397dc73c8b9.png)

---

## IPMI v1.5 LAN Session Startup 

![](/uploads/upload_cc9bd0d6170a02e7a6a5e4725c76d389.png)

![](/uploads/upload_d3a894972465aac59926a6d4214640f0.png)

![](/uploads/upload_9370e597a1413c73ed7f3d2902902990.png)

---

## RMCP
* 分佈式管理工作（DMTF) 定義了RMCP (遠程控制協議)來support pre-OS、OS-absent，RMCP是一個request-reponse的協議，使用UDP datagrams 做傳輸(使用RMCP版本1)；
* RMCP會嵌入一個訊息指示packet格式和類別(ASF、OEM)，ASF-class還可以使用Ping、Pong來發現網路上支援IPMI syste的platform。
* ASF(Alerting Standard Forum警報標準論壇) 針對non-intelligent管理硬體通過LAN來做到警報和恢復的控制的功能(EX:系統重置)。
* RMCP有兩個UDP port :
623(26Fh) : Aux Bus Shunt (主要的RMCP port)每個RMCP必須要有的port，訊息傳遞清晰，故容易被系統軟體尋找到並得知支援RMCP。
664(298h) : Secure Aux Bus (次要的RMCP port)只有當必須使用加密算法或規格的packet時才會使用此port，可以防止在同的port傳輸未加密的packet。

***NOTE:***
UDP 不需要唯一識別碼和序號就能完成傳輸工作。這種協定以串流方式傳送資料，發送端不會等待接收端的確認信號，會繼續不斷發送封包資料，所以UDP 幾乎沒有錯誤修正功能，也不在乎封包遺失，因此很容易出錯，但傳輸速度比 TCP 更快。串流媒體、VoIP 語音、網路遊戲等服務經常使用 UDP 協定，這網路應用不太需要可靠性機制，封包遺失不會導致服務中斷。

---

## RMCP+

1. **Payload**：一個standard payload 裡面包含了IPMI Message的類型、RMCP+的設定訊息，並且允許BMC連接到遠端的BMC user 的相同PMI中啟用payload，BMC還可以要求遠程console為payload建立獨立的一個session。
2. **更強的身分驗證**：與RMCP(IPMI 1.5)相比，RMCP+有更可靠的session set-up 和 key handling algo。
3. **支援加密功能**：IPMI Message 和 payload可以在IPMI session 下進行加密，增強遠程操作(SOL、user passwords)的安全性。
4. **ASF 2.0**：RMCP+遵循ASF2.0規範的格式和身分驗證element，建立session message和packet的格式略有不同，但還是可以基於遠程管理連接來建立支援ASF2.0和IPMI2.0的applications。
5. **可以讓支援加密/未加密功能和身分驗證/無身分驗證功能之間做傳輸/溝通**：
加密和身分驗證功能在”IPMI Message Class” Level，所以可以在任何UDP port(26Fh)上建立經過加密和身分驗證的sessions (與ASF2.0不同，因為ASF2.0 有身分驗證和無身分驗證功能需要使用不同的port做message handle)；
IPMI允許一個BMC只有在privilege level和payload下配置身分驗證和加密功能，所以不需要所有的message都要加密或身分驗證，只有小部分的high level才需要這些功能，大大提高了BMC使用廉價的微控制器的性能。

---

## IPMI v2.0/RMCP+ Session Activation 
* Get Channel Authentication Capabilities request / response
與remote console 做訊息交換以發現IPMI version is supported.(BMC是否supports IPMI V2.0 / RMCP+格式)。
它提供了remote console 確認是否使用匿名，以及使用“one-key” or “two-key” 來做登錄，還可以提供用戶查詢user name、password。
這是BMC在IPMI v1.5和v2.0/rmcp+格式中的‘session-less’ command 。 
* RMCP+ Open Session Request, RMCP+ Open Session Response
此功能用於啟用remote console，來發現可以使用哪些密碼套件來使用最大權限建立session。
這些messages 還用於傳輸BMC希望activated session 的 ID和 remote console 的session ID，並track 建立seeion期間所做的exchange of messages。
+ RAKP Message 1, RAKP Message 2、RAKP Message 3, RAKP Message 4 
  - 這些messages用於BMC和remote console之間認證訊息和交換random number，v2.0/RMCP+和IPMI v1.5不同，v2.0的challenge/response是對稱式傳遞random numbe和username/privilege(特權) information。
  - remote console request (RAKP Message 1) 傳遞隨機數和用戶名/特權資訊給BMC，BMC利用這些資訊來‘sign’ a response message 來做身分驗證和request/response exchange，manage system利用安全驗證和session ID來建立session。

  - BMC做完驗證後再reponse (RAKP Message 2) 一個隨機數和GUID(唯一ID)回remote console，remote console根據此訊息來確認session是否啟用並根據GUID來和相對應的sesssion做匹配，remote consol再驗證 Key Exchange Authentication Code 並傳至BMC(RAKP Message 3) 。 
  - Activation session是經由remote console 和 BMC 交換訊息並sign完成的(這些sign的訊息是根據身分驗證以及先前傳遞的參數組成)
RAKP Message 3是remote console 傳遞至BMC的sign message，待BMC收到後再回傳RAKP Message 4回remote console 。
 
 ---
## Payload 包含以下資訊：
* Message Tag：用來match response and request ，還可以區分是new message 或是來自remote console message。
* Requested Maximum Privilege Level (Role)：權限級別的分類
* Reserved：保留用
* Remote Console Session ID：remote console 確認收到 packet 時給定的session ID
* Authentication Payload：標示manage system 想要用於session的身分驗證類型
* Integrity Payload：標示manage system 想要用於session的資料完整性類型
* Confidentiality Payload ：定義機密演算法

## RMCP+ Session Termination 
* 兩種方式來close session
1. 下close session command
2. Session 長時間暫停：終止session會連同payload一起停用，但終止一個session不會影響其他的session，故remote console需要單獨終止每個session。

## RMCP ACK Messages ( Acknowledge Message確認訊息)
ACK訊息用來確認收到具有0-254 (RMCP編號)-->
 normal RMCP message(有收到message)
如果收到255(FFH)，則不會生成RMCP ACK Messages (沒收到message)

---
## IPMI V1.5和2.0 差異
1. IPMI v1.5 允許sessio中的packe具有不同的身分驗證，為簡化處理packet只能有兩種身分驗證類型：在open session command 或是 none，並刪除身分驗證類型的字段，而是使用”integrity Data”(完整性資料)字段的存在不存在來表示packet是否經過認證。
2. IPMI v1.5 使用單向收發機制來進行用戶身分認證(BMC發出challenge，remote console 收到並發出response)，IPMI v2.0 /RMCP+使用的是對稱式challenge，BMC和remote console都必須response for session to be activated。
3. ASF 2.0 對身分驗證定義了”Roles”(角色)，例如user 和管理員，每個Roles都有自己的passward，而IPMI v1.5的身分驗證為每個passward of user name配置了“privilege level”(特權級別例如User、管理員)，以上兩種方法IPMI 2.0/RMCP+都可以支援，而如果在配置BMC時把User設為“null”，這時key(passward) for session 的查找方式為”privilege level only”，而其他方式就要把user name 設為 non-null 。
4. IPMI v1.5使用單一的session ID for BMC and remote console。
IPMI v2.0允許BMC and remote 使用兩個不同的session ID，使得session 在傳輸過程中的更容易識別。
6. IPMI v1.5 使用單一的金鑰(user key/password)，該金鑰既可用於身分驗證，還可以用來計算integrity(Authcode)的計算。IPMI v2.0/RMCP+ 有分為單一金鑰或雙金鑰，其中單金鑰和v1.5相似，可用於身分驗證還可生成一個session用於計算integrity(Authcode)，而雙金鑰的其一key 用於身分驗證，另一key則是使用單獨的“BMC Key”來create a session in integrity(Authcode)。

## RMCP.c
![](/uploads/upload_87cb08dbce362102202d9e613b8a66b1.png)


---
## UDSIfc.c
![](/uploads/upload_295f952c1e56e9befd77b05f8367ddc3.png)

![](/uploads/upload_ee4f4cfab7ef574e5fb37e6d27927a68.png)

---
## Out-Of-Band & In-band：
* Out-Of-Band : 使用固定的網路協議(乙太網路)，控制設備的訊號和資料傳輸訊號公用相同的網路傳輸管道，屬於應用層管理，管理設備必須處於開機狀態並進入OS才可進行管理，當設備系統損壞或網路中斷時不能快速排除故障，造成服務中斷。

* In-band : 資料傳輸訊號獨立於主要的網路傳輸，被管理的設備即使在關機、甚至故障的狀態下，都可以進行修復開機或是日誌監控等管理作業。

* IPMI中的Inband(In-Band):
1.Console Redirection-->透過”serial port(comport) sharing” 
2.System interface-->OS透過KCS,SMIC,BT,SSIF下IPMI command

* Outband (Out-of-Band):
LAN Interface-->使用獨立的RJ45埠給IPMI做 : 
遠程系統管理(電源控制、系統診斷、FRU管理)、IPMI管理(韌體升級、Serial-over-LAN、SDL、SDR) 

---
## WebUI：
* 後端：
Restful API--> 回傳 JSON format data
PHP/ASP/ASP.NET…etc. 網頁程式--> 回傳 html format data
以上兩者都是執行在WEB server上，只是回傳給瀏覽器的格式 & 內容不一樣
也可以用PHP/ASP/ASP.NET/JSP寫RESTful API
RESTful API 是一個抽象概念，可用現今的網頁程式技術實作，其實也是走HTTP的GET/POST/PUSH…etc.
* 前端：
瀏覽器Browser：Edge/IE/Firfox/Chrome/Safari…etc.
瀏覽器會執行網頁上的Java script去跟後端RESTful API 做溝通，
取回資料(JSON格式) 或 執行某些動作(POST新增、GET = 讀取、PUT = 更新、DELETE = 刪除)

* 假設web server 是 M$的IIS
-->Browser (Web Page + JavaScript) <----------- HTTP ------------> Web Server (IIS + ASP . NET)
我們的BMC為
-->Browser (Web Page + JavaScript) <----------- HTTP ------------> Web Server (lighttpd + FastCGI)

* FastCGI --> spx_restservice-src
Web Page + JavaScript --> webui_html5-src
-->RESTful API實作是在spx_restservice-src
    前端網頁跟JavaScript是webui_html5-src

* AMI是採用Backbone.JS的框架
呼叫RESTful API 是使用AJAX (Asynchronous JavaScript and XML)
前端網頁的JavaScript 主要是使用 Backbone.JS + AJAX

* RESTful API 實作部分：透過 libipmi 的 API 去執行對應的IPMI command
EX：要在WebUI上power on host ，相關網頁的javascript會去呼叫對應的power on 的 RESTful API (走HTTP)，後端的Web server 就會執行對應的FastCGI，這個對應的FastCGI就是power on 的 RESTful API實作，裡面會去呼叫 IPMI 的host power on 的IPMI command。


* RESTful API：
把RESTful 想成是建立在HTTP協定上的架構或設計模式，利用HTTP協定特性來處理數據，好處是可以讓資源和操作做分離，讓對資源的管理有更好的規範，並讓串接API或使用API的人可以很快速的了解API的內容，省去不必要的溝通。
缺點：
安全性問題：一個用戶可以任意對 Database 操作CRUD，或得到一個URL為/api/users/1/，可以try到/api/users/100/來得到其他用戶資料，所以在回應數據時應該要先對用戶做身分驗證，再決定是否有權限存取資料，或對資料做其他加密的方式。

---
## REST須符合六條件：
1. Uniform interface (介面一致，API邏輯一致)
(開發者只要熟悉你的一支API，他應該能用同樣的邏輯，來理解你開出來的其他API)
所有資源都應該可以通過類似的通用方法訪問，並使用一致的方法進行修改。
EX：GET https://xx.com/api/articles -> 可拿到xx的文章
GET http://xx.com/api/articles/1/title -> id=1 的文章標題
DELETE http://xx.com/api/articles/1 -> 刪除id=1 的文章
2. Stateless (無狀態的Server端)
(在 server 端 不儲存 client 端的狀態資訊(context)，而是由client 端來掌握app的狀態)
EX：登入購物網站後要結帳，系統要求再登入一次->發出不必要的request，因為登入時結帳時的server1和結帳時的server2 不相同，造成server2拿不到server1 的資訊，故讓所有能夠證明用戶身分的資訊都包含在一個request哩，而不是儲存在server中，好處是因為狀態資訊不儲存在server，這樣serve就可以擴大增長(分散式系統)。
3. Client-server (Client端和Server端分離)
( 只要中間的 API 沒有變，Server 和 Client 可以個別開發，相互不受影響 )
4. Cachable (可暫存的資源)
( 好的 Caching 能部分或完全消滅 client 和 server 間的互動，更能夠改善 scalability 和 performance )
把資源拆成獨立的部分，針對不同常用的資源做暫存，當重複使用到這些資源時就直接從暫存中拿取，不必進到server中拿取資料。
5. Layered System (分層的系統架構)
( REST 允許使用分層的系統架構，舉例來說，你把 API 放在 ServerA、資料放在 ServerB、處理驗證在 ServerC，對於 client端 而言，無法分辨到底是跟哪一個 Server 在對話 )
Server分成多層，不同Server 間各司其職，EX：有三個不同server，當一client只需用到其中兩個server，直接到指定server即可，不需要因為不需要某一server而更動到server的系統結構。

6. Code on demand (Optional) (視情況決定要不要遵守，Server 端可以傳送可執行的程式碼給Client端)
(Optional，隨需求傳可執行的程式碼，可依情況選擇要不要遵守這條規則，多數情況下是由 server 傳靜態資源給 client，但如果有需要，也可以傳可執行的程式碼給 client 去執行 )
EX：html上的<script>標籤--> 因為有些網頁沒有動畫或不需用到動態執行javascript，只是純文字的結構，故可選擇性使用。

---

## CGI 概念：
* Common Gateway Interface (共通閘道介面) 描述了**client 和 server 之間傳輸數據的標準**，可以讓一個client 端從網頁瀏覽器向執行在Web server上的程序請求數據。原始的HTML語言只能展現靜態資料，但**如果要讓資料有動態的展現以及可時常更新、互動就需要CGI**，透過CGI可以秀出server上”**最新的資料**”，達到與使用者互動的效果(EX：目前股票市場行情)。

![](/uploads/upload_65363054befd4f486146a7a8dad0efb3.png)
1. 使用者透過Browser access server 並發出request URL 
2. Server 收到數據並解析
3. Server將一些無法處理的登錄數據送至CGI-->server會創立一個CGI process載入配置並獲取數據
4. CGI 經邏輯處理後得到結果在發送至server再退出銷毀
* 以上為CGI的一個流程，server只要有動態數據的request就**必須不斷創建、銷毀CGI，造成成本大、效率低**

## FastCGI概念：
* Fast Common Gateway Interface (快速共通閘道介面) 是一種常駐型CGI，目的是**減少Web server 和 CGI 之間的互動開銷，讓server 可以同時處理更多的Web 請求**
1. Web server 啟動時載入**FastCGI進程管理器**(IIS ISAPI或Apache Module)
2. FastCGI 進程管理器自身初始化，啟動多個**CGI 解釋器process** (多個php-cgi) 並等待Web server的連接
3. 當request到達Web server時，FastCGI 進程管理器選擇並連接到一個CGI 解釋器。(server發送CGI環境變數和標準輸入至FastCGI子進程php-cgi)
4. FastCGI subprocess 完成處理後將數據返回web server，當subprocess 關閉連接時表示request 處理完成，即可等待web server 發送下一個request data。

---
## JSON 格式
![](/uploads/upload_c07fded73d882331aaad0405ea3af0ce.png)


## Backbone Introduction

![](/uploads/upload_d100ee7ee21a4276608033dd2c4e90c8.png)

---
![](/uploads/upload_3805a844378ef52ee97ad8b3ff0fa496.png)

* Backbone.js ：為MVC的一種，MVC框架可以讓JavaScript 為WebUI 做好架構上規劃，而Backbone.js 包含了Model、View、Controller來讓使用者操作，Model提供了key-value 結構，以及可以binding 大量event (將事件綁定在HTML的元件上)，開發者可以透過 RESTful JSON interface 來跟Backbone.js的 Model 及Collection 搭配。

  - Model：可表示應用中所有數據，當Client 端要使用變數來暫存，可以把變數存在Model 物件內，把Model當作一般物件做資料儲存。本身內建 fetch、save 等method，只要設定好相關參數(url) Model 即可自行與Server溝通取得資料，也就是當由Server取得資料暫存在Client時，會將取得的資料以Model物件方式儲存在Client 上。除了對Model的基本get/set操作之外，我們還可以透過Listen一些event，像是view、model的變化。![](/uploads/upload_f603ae7f71ee94fccb484f8f2ef53a1e.png)

  - View：View 中可以綁定model和客戶端事件。Html就是通過views的render方法渲染出來的，當新建一個view的時候通過要傳進一個model作為數據，例如：var view = new EmployeeView ((model：employee));而render 是view 內建的method，可以透過重新定義(override) 這個method讓我們可以操作改變 model的外觀。

  - Collection：是Model的一個有序的集合，如果有一堆Model 需要被當做陣列(或類似有序資料結構 –ordered set) 來操作，而更重要的是 Collection 可以無縫的整合RESTful web service，大大減低與後端溝通的阻礙。此外，Collection 可以直接對監聽Collection 本身或 Collection 裡的Model 所發出的各種事件，例如 change、add、remove。![](/uploads/upload_984a0ac9e2d2486069cf85824621b4a5.png)

* 結論：實際使用來說最好應用backbone.js的場景是單頁面應用，且在頁面上有大量數據模型，模型之間需要進行複雜的訊息溝通，Backbone.js能很好的實現事件驅動，以下為Backbone在此方面的優勢：
  - View 的劃分將頁面上的視圖元素一一切割。View 間通過事件和Model 通訊，避免DOM(Document Object Model)事件的濫用。
  - Model 和Restful 的通訊方式對於後端維護有相當的幫助。
  - MVC 架構清晰。
  - Collection/Model 抽象了以前雜亂的AJAX 請求，CRUD 請求變得方便。
  - 檔案輕量。

* 相對的，若應用在較複雜的數據關係如One-To-One或者One-To-Many時便顯得困難，原因是Model 只有基本的CRUD 操作，不能很好的擴展；此外View 層沒有很強的Page 管理機制，比如通過URL 切換改變整個頁面時，頁面上尚存的View 應該如何處理對開發者而言是一大挑戰。

--- 
## AJAX：
* Asynchronous JavaScript and XML（非同步的JavaScript 和XML） 又稱非同步傳輸、非同步呼叫，使用XML Http Request 物件來跟伺服器溝通，並能傳送、接收各種格式的資訊，包括JSON、XML、HTML與文字檔案。

* 它是一個可以與伺服器溝通、交換資料和更新頁面，將資料載入至網站部分頁面區段中，讓網站不需要重新更新整理、讀取整個頁面的技術，還能夠更加快速、即時更動介面內容，而網站頁面上的資料，也能因此各司其職，大幅節省等待資料的時間。









