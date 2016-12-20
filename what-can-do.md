# Chrome Extensionでなにができるの?

Chrome Extensionは、Chromeの持つ[JavaScript API](https://developer.chrome.com/extensions/api_index)を利用してアプリケーションを作ります。
もちろん利用する言語は**JavaScript**です。

APIにはブラウザを操作する様々な機能が用意されています。
タブを操作する `tabs` やブックマークを操作する `bookmarks` などは想像しやすいでしょう。
Extension自体を情報を取得したり操作する[^1] `browserAction` や `pageAction` などのメタメタしいAPI、
Extensionのメインプロセスと言える `chrome.runtime` というとても重要なAPIもあります。
他には、`identity`[^2] や `gcm`[^3] などの、スマートフォンアプリ方面に起源があるAPIなども多数輸入されています。

もちろん、それ以外にもchromeのJavaScriptで元々できることはなんでもできます。
例えば、DOM操作のAPIはExtensionのAPI一覧には用意されていませんが、
普通に `document.getElementById` などが使えます。

[^1]: 例えば状況に応じてアイコンを変えるなど
[^2]: 端末ベースの個人特定の仕組み
[^3]: いわゆるPush通知

何ができるかは非常に多岐にわたるので、実例やサンプルコードを見ながら紹介していくことにします。
