---
date: 2023-07-22T12:55:00+08:00
title: "關閉 Capacitorjs Webview Android Autosuggestion"
draft: false
tags: ["capacitor", "webview", "android", "autosuggest", "keyboard"]
---

最近工作在開發時其他團隊有遇到使用 [Capacitorjs](https://capacitorjs.com/) 將 Web 專案打包成 Android App 後，
在登入頁面輸入帳號密碼時會出現Gborad鍵盤下個字的提示。

這是本身是沒問題的，但是再送審資安時被要求鍵盤不能顯示提示任何提示

再研究了一陣子後發現無法使用 html 本身 input 內的屬性 disable 後
轉而去研究 capacitor 打包成 android 的機制
``` html
 // 這樣是沒有用的
 <input  type="text" autocomplete="false" autocomplete="off">
 ```

因為打包成 Android App 後畫面呈現是由 Webview 這個元件在渲染網頁，所以朝此目標下手

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
5. 這樣就完成了，打開 android app 就發現所有的提示都被關閉了，主要是因為我們在 `NoSuggestionsWebView` 的 `inputType` **全域**修改掉他的行為。