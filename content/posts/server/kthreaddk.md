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

1. 找到排程中的異樣，在 `Command` 找到 `kthreaddk` 並且取得 `PID`
    ```
    htop
    ```
2. 查看 `PID` 位置，通常資料名稱會是沒有意義的亂數而且也已經被刪除
    ```sh
    ll /proc/<pid>/exe
    ```
3. 查看是否也有奇怪排程
    ```sh
    sudo crontab -l
    ```
    ![crontab](../images/crontab.png)
4. 找到奇怪的 `port`
    ``` sh
    netstat -lpnt
    ```
5. 拿到 `port` 並取得 `pid`
    ``` sh
    ps -ef | grep <port>
    ```
6. 刪除 `crontab` 和 `pid` 找到的不知名的路徑
    ``` sh
    rm -rf <path>
    ```
7. 刪除奇怪的 port 和 kthreaddk process
    ``` sh
    kill -9 <pid>
    ```
8. 用 `htop` 在等一兩分鐘看看有沒有跑起來

## 追查原因

再搜尋了一下 kthreaddk，找到了一篇[相關文章](https://www.freebuf.com/articles/system/282954.html)。在這篇文章中，它提到了這個病毒可能進來的方式，主要是透過 `横向传播`。

目前看來，最有可能的入侵途徑是 `Laravel Debug mode RCE（CVE_2021_3129）`這個漏洞。

原因是 `facade/ignition` 套件版本太舊導致破口產生 ([參考文章](https://tyskill.github.io/posts/cve_2021_3129/))

有目標後搜尋一下果然找到

![logs](../images/logs.png)

修正過這個問題後就沒有再跑出這個病毒了
