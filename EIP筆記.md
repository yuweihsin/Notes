
# 燒錄rom.ima檔步驟

## 架設tftp (Ubuntu)
1. 下載 tftpd-hpa
```
sudo apt-get install tftpd-hpa
```
2. 設定分享的目錄, 以下是 /home/yuweihsin/tftp
```
sudo mkdir -p /home/yuweihsin/tftp
sudo chmod -R 777 /home/yuweihsin/tftp
```
3. 配置 /etc/default/tftpd-hpa
```
vim /etc/default/tftpd-hpa
```
```
# /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/home/yuweihsin/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="-l -c -s"

#-c: Allow new files to be created
#-s: Change  root  directory  on startup.
#-l:  Run the server in standalone (listen) mode, rather than run from inetd.
```
4. 重啟 tftpd-hpa
```
sudo service tftpd-hpa restart
```


## 進uboot
如果沒有設定MAC address, 會拿不到IP，下print可以查MAC address
有ethaddr跟eth1addr, 這兩個就是MAC address的參數
```
print
```
設定MAC address
```
setenv ethaddr 00:ab:cd:dd:ee:01
setenv eth1addr 00:ab:cd:dd:ee:02
```
```
saveenv
```
```
reset
```

---
## 載入rom.ima並燒錄至開發板

1. 進入uboot
2. 設定tftp server IP
``` 
setenv serverip 172.31.100.227
```
3. help 開發板取得IP
```
dhcp
```
4. 打開PRJ檔案檢查 
`CONFIG_SPX_FEATURE_GLOBAL_^AST2500EVB^_FLASH_START` 的數值是不是 `0x20000000`?
**(kernal3-->`0x20000000`，kernal5-->`0x0`)**
5. 從tftp下載`rom.ima`
`0x83000000`是uboot起來的時候所做的一個mapping, 
這個位置是mapping到AST2500EVB上的DRAM,
這一步驟是把`rom.ima`下載到記憶體的`0x83000000`的位址
```
tftpboot 0x83000000 rom.ima
```
6. Unlock SPI flash

### 查Kernal版本
```
cat /proc/ractrends/Helper/FwInfo
```
### *Kernal3*：
```
protect off all
```
```
erase all
```
現在要把剛剛下載到`0x83000000`的`rom.ima`拷貝至SPI flash, 而SPI flash位址是在`0x20000000`
先確認`rom.ima`的大小是否為`32MB`
```
cp.b 0x83000000 0x20000000 0x2000000
```

### *Kernal 5*
懶人包：
```
setenv serverip 172.31.100.61 && dhcp && tftpboot 0x83000000 rom.ima  && sf probe 0 && sf update 0x83000000 0 0x4000000
```
sf是SPI flash的縮寫，probe是指初始化指定的SPI上的設備
指定連接 `0 flash` (BMC)
```
sf probe 0
```
DRAM裡面有很多section，`update`功能包含` write`, `erase`, `protect of all`(unclock用)
```
sf update 0x83000000 0 0x4000000
```
最後 `reset` 進入 `uboot` 並重新設定 `MAC address`


***Note：***
MAC address：MAC位址，直譯為媒體存取控制位址，也稱為區域網路位址，乙太網路位址或實體位址，它是一個用來確認網路裝置位置的位址。在OSI模型中，第三層網路層負責IP位址，第二層資料鏈結層則負責MAC位址。MAC位址用於在網路中唯一標示一個網卡，一台裝置若有一或多個網卡，則每個網卡都需要並會有一個唯一的MAC位址。
(作業系統看的是IP跟Port Number，網路卡是看MAC address)

sf probe [[bus:]cs] [hz] [mode] - init flash device on given SPI bus and chip select

sf read addr offset len              - read \`len' bytes starting at\`offset' to memory at \`addr'

sf write addr offset len             - write \`len' bytes from memor at \`addr' to flash at \`offset'

sf erase offset [+]len                - erase \`len' bytes from \`offset' \`len' round up \`len' to block size

sf update addr offset len         - erase and write \`len' bytes from memory at \`addr' to flash at \`offset'


ipmitool -I lanplus -H 127.0.0.1 -U admin -P admin123 raw 0x0 0xb 0xf 

