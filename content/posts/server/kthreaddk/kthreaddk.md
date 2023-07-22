---
date: 2023-07-22T13:20:12+08:00
title: "Kthreaddk 挖礦病毒處理"
draft: false
tags: ["kthreaddk"]
---

近期家裡當伺服器的電腦突然很 Lag

想查個元因因此用 htop 來看，看到 `kthreadd` 後 google 搜尋以為是系統程式在跑差點把它忽略，仔細一下原來是 `kthreaddk` 在用 google 搜尋就跳出一堆病毒的結果

![image2](./Screen%20Shot%202023-07-25.png)

## 解決
每個人的解決方法都很像不同，這邊提供我自己的，目前觀察是沒有再出現

可能需要注意的是，因為是cron排程每分鐘執行，所以盡量在下一分鐘前刪完

1. 使用找到奇怪的排程
```sh
sudo crontab -l
```
![image](Screen%20Shot%202023-07-24.png)
2. 找到奇怪的 port
``` sh
netstat -lpnt
```
3. 刪除奇怪的排程
``` sh
rm -rf xxx
```
4. 刪除奇怪的 port、kthreaddk process
``` sh
kill -9 xxx
```

