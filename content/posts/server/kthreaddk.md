---
date: 2023-07-22T13:20:12+08:00
title: "Kthreaddk 挖礦病毒處理"
draft: false
tags: ["kthreaddk"]
---
最近我家裡的伺服器電腦突然變得很卡，所以我決定用 htop 來查找原因。

一開始看到 kthreadd，我還以為是系統程式在運行，差點忽略它。

## 解決
每個人的解決方法都很像不同，這邊提供我自己的，目前觀察是沒有再出現

需要注意的是，因為是 crontab 排程是每分鐘執行會執行一次，所以盡量在下一分鐘前刪完避免他又換資料夾為自

1. 找到排程中的異樣，通常資料名稱會是沒有意義的亂數
```sh
sudo crontab -l
```
![crontab](../images/crontab.png)
2. 找到奇怪的 port
``` sh
netstat -lpnt
```
``` sh
ps -ef | grep xxx
```
3. 刪除 crontab 找到的不知名排程的路徑
``` sh
rm -rf xxx
```
4. 刪除奇怪的 port 和 kthreaddk process
``` sh
kill -9 <pid>
```

