# GIT 基本操作
* 初始化
```
git init 
```
* 查看目前狀態，只會顯示跟上次commit不同的部分，如果以commit存檔過的檔案不會顯示，但如果兩個檔案跟上次commit相比有所變動時，會出現 **modified:file name**
```
git status
```
* 新增檔案到git
```
git add
```
當前資料夾都加進去
```
git add .
```
* 保存進度(類似遊戲存檔)
```
git commit 
```

* commit message 通常會寫你這次commit修改了什麼東西
```
git commit -m "commit message"
```
* 把repository連結到本地端
```
git remote add <自訂名稱> <網址>
```
* 列出所有remote
```
git remote
```
* 把master推到(上傳)至git
**-u:把預設的remote設為<指定git name>，未來push如果不指定remote對象都會推到此git name**
```
git push -u <git名稱> <branch名稱>
```
用過 **-u** 後不須再下git、branch name
```
git push
```
