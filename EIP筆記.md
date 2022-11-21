# EIP筆記

## 711606：

- LNV：動態更改調試控制台用戶名似乎很難，但是我們可以在工廠更改用戶名並在發貨後修復嗎？如果不是，我們可以在 BMC 固件版本中更改用戶名嗎？
**[AMI]**  “sysadmin”為BMC Linux 用戶，配置為root 訪問權限，默認為Linux 系統中唯一的管理員角色。我們不建議在運行時更改此用戶名，但我們可以在固件構建時配置此用戶名。

- LNV：OEM更改的密碼是否持久？這意味著即使用戶恢復出廠設置，密碼仍然有效。
**[AMI]** BMC Linux用戶“sysadmin”的新密碼，保存在“/conf/passwd”和“/conf/shadow”文件中，如果客戶想在出廠後保留密碼，我們需要將相關文件添加到保留文件列表。

## 710259：
*Auto-recovery function sup SD and TFTP*
1. 如果SPI_MAIN損壞，SPI_MAIN可以從SPI_BAK恢復嗎？
**[AMI]**：如果你有雙圖像支持，一旦 SPI_MAIN 被破壞，SPI_BAK 就會運行。您可以使用 webui 、 yafuflash 從 SPI_BAK 升級 SPI_MAIN 固件。
2. BMC設置是否可以備份到SPI_BAK並從SPI_BAK恢復？（如果 SPI_BAK 大小足夠大）
**[AMI]**：一旦您啟用雙映像支持，我們在 PRJ 中有“同步兩個配置功能”選項。此功能將在 2 個圖像之間同步兩種配置。
3. 如果更新了SPI_MAIN，SPI_BAK會自動同步到SPI_MAIN嗎？
**[AMI]**：我們有升級 webui 的“雙圖像”選項，yafuflash 升級。
4. 如果開啟雙鏡像，讀卡器中沒有TF卡，是否還有本地鏡像需要重定向？
**[AMI]**：雙圖像和自動恢復功能不能共存，因為它們都由 WDT2 觸發。
---
1. WDT2 定時器，在加載內核後 UBoot 時將默認啟用定時器計數器為 5 分鐘，並期望在 BMC 啟動完成時禁用定時器計數器。這意味著我們使用 WDT2 來識別 BMC 啟動結果。
**[Roy]**：是的，沒錯。但是如果您在 PRJ 中啟用“定期刷新硬件看門狗”，則看門狗定時器在啟動完成後仍會繼續倒計時。在這種情況下，進程管理器將每 20 秒刷新一次看門狗定時器。
2. 如果 BMC 啟動失敗且支持雙映像功能，將重置為 UBoot 階段，然後從輔助 SPI 閃存啟動。即使用 UBoot_ENV 中的 boot SPI source 變量來切換 boot SPI。
**[Roy]**：更具體地說，軟件故障安全功能使用 UBoot_ENV 中的引導 SPI 源變量來切換引導 SPI，而硬件故障安全功能使用硬件機制來切換引導源。
3. 如果 BMC 啟動失敗且 Auto-recovery 功能開啟，Uboot 會在 UBoot_ENV 中記錄失敗計數，默認最大為 3，UBoot 會嘗試從 TFTP/SD 恢復 BMC SPI flash，這取決於源設置在 UBoot_ENV 中。
**[Roy]**：是的，沒錯。

---
1. SPI_MAIN存有BMC FW，SPI_BAK也存有BMC FW，存的是SPI_MAIN的bakcup image
**[AMI]**：SPI_BAK會作為存儲的方向去評估。
2. 正常情況下，BMC從SPI_MAIN讀取代碼和數據。
**[AMI]**: ok
3. SPI_MAIN裡的FW如果從SPI_MAIN的故障機制不成功或崩潰，將SPI_BAK MC FW恢復到SPI_MAIN，然後啟動
**[AMI]** 可以否接受當前AMI已有的自動恢復，那是觸發的wdt2 timeout就啟動recover 機制，情況可能包BMC hang，kernel panic...等。
4. 如果更新了SPI_MAIN的BMC FW，則也更新SPI_BAK的BMC FW
**[AMI]**：需要評估更新方法嗎？例如：webui、yafu 或 redfish 升級？
5. 要求執行備份鏡像（非SPI_MAIN crash或BMC.用戶不成功），則將MAIN的FW備份到SPI_BAK
**[AMI]**: 需要評估哪些備份方法？例如： webui 、 yafu 或 redfish 備份方法？
以webui為例，需要是否是新增一個備份圖片頁面。當用戶點擊備份圖片按鈕時，就會讀取當前SPI_MAIN的數據存儲到SPI_BAK的存儲。

---
## 714938
* Question : 使用**eth0** IPV4登錄BMC Web
1. 使用bond0 IPV4登錄BMC Web；
2. 不要勾選網絡設置->網絡綁定配置中的啟用綁定框；
3. 再次使用eth0 IPV4登錄BMC Web；
4. **Dashboard上無法顯示IPV4和IPV6**

* 發生原因是由於api/settings/network資料回傳的內容，在web按F12可查看網路回應的message (JSON格式)，在解除Bonding後，network產生eth0及eth1兩組資料，後一組為空因此資料被覆蓋過去，若刪除一組資料，可以正常輸出，不過Network IP Settin頁面會僅剩一組eth0。

* 測試方式：
在BMC路徑 /var/下產生test.log 就只輸出一組eth資料，
進入檔案spx_restservice-13.30.38.9.0-src\data\settings_network.c
修改函數START_AUTHORIZED_COLLECTION (getNetwork, GET, "/settings/network", 1, matches, true)--> 讓他抓到就離開while迴圈，針對指定函數break
如此一來#deshboard就能正常輸出，但Network IP Setting 頁面會僅剩一組eth0，建議換成跟隨資料組數呈現的網頁方式(有eth0、eth1可選擇的方式)。
