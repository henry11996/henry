---
date: 2023-07-22T12:55:00+08:00
title: "關閉 Capacitorjs Webview Android Autosuggestion"
draft: false
tags: ["capacitor", "webview", "android", "autosuggest", "keyboard"]
---

最近我們在進行開發工作時，遇到了一個有趣的問題。我們正在嘗試將 Web 專案使用 Capacitorjs 打包成 Android App，但在登入頁面輸入帳號密碼時，Gboard 鍵盤竟然會給出下一個字的提示，這實在是有點令人意外。

雖然這本來不算什麼大問題，但在提交資安審查時，審查人員強調鍵盤不能有任何提示，這讓我們的專案陷入了一個有趣的境地。

起初我們試圖使用 HTML input 元素的 disabled 屬性來禁用鍵盤提示，但效果並不如預期。因此，我們開始對 Capacitor 的打包機制進行深入研究，希望找到更靈活、專業的解決辦法。

``` html
 // 這樣是沒有用的
 <input  type="text" autocomplete="false" autocomplete="off">
 ```

正因為我們打包成 Android App 後的畫面是由 Webview 元件在渲染網頁，所以我們的解決方案主要朝這個目標著手。

## 修改
修改方式如下 (以下`com/example/app`需要依照你的app去替換名稱)
1. 在 `android/app/src/main/java/com/example/app` 下面建立一個檔案 `NoSuggestionsWebView.java`
2. 在裡面加入
``` java
package com.example.app;

import android.content.Context;
import android.text.InputType;
import android.util.AttributeSet;
import android.view.inputmethod.EditorInfo;
import android.view.inputmethod.InputConnection;
import com.getcapacitor.CapacitorWebView;

public class NoSuggestionsWebView extends CapacitorWebView {

    public NoSuggestionsWebView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public InputConnection onCreateInputConnection(EditorInfo outAttrs) {
        InputConnection ic = super.onCreateInputConnection(outAttrs);
        outAttrs.inputType &= ~EditorInfo.TYPE_MASK_VARIATION; /* clear VARIATION type to be able to set new value */
        outAttrs.inputType |= InputType.TYPE_TEXT_VARIATION_WEB_PASSWORD; /* WEB_PASSWORD type will prevent form suggestions */
        outAttrs.inputType |= InputType.TYPE_TEXT_FLAG_NO_SUGGESTIONS; 
        outAttrs.inputType |= InputType.TYPE_CLASS_TEXT; 
        return ic;
    }
}
```
3. 在 `android/app/src/main/res/layout/activity_main.xml` 新增
``` java
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <com.example.app.NoSuggestionsWebView  //新增這個
        android:id="@+id/webview"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent" />
</androidx.coordinatorlayout.widget.CoordinatorLayout>

```
4. 最後在 `android/app/src/main/java/com/example/app/MainActivity.java` Overrid `onCreate` 這個方法 (官網有[提到](https://capacitorjs.com/docs/android/custom-code#mainactivityjava))
``` java
package com.example.app;

import android.os.Bundle;
import android.webkit.WebView;

import com.getcapacitor.BridgeActivity; // 是使用 capacitor 的 activity
import com.getcapacitor.Logger;
import com.getcapacitor.PluginLoadException;
import com.getcapacitor.PluginManager;

public class MainActivity extends BridgeActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState); // 可以透過此內部去查看boot的順序
        setContentView(R.layout.activity_main); // load 進自定義的 layout
        this.load(); // 重新 load webview
    }
}
```
5. 這樣一來，當我們打開 Android App 時，可以發現所有的鍵盤提示都已經被關閉了。這主要歸功於我們在 NoSuggestionsWebView 的 inputType 全域新增了不需要鍵盤提示的屬性。