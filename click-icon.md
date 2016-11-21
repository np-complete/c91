# アイコンに動作を設定しよう

Chrome Extensionをインストールすると、ツールバー[^1]にずらっとアイコンが並ぶことはもうご存知でしょう。
すでに、アイコンをクリックするとなにか反応があるExtensionがひとつは入っているのではないでしょうか。

例えば、[HTTP Headers](https://chrome.google.com/webstore/detail/http-headers/hplfkkmefamockhligfdcfgfnbcdddbg)のアイコンをクリックすると**ポップアップ**が開き、そのページの通信でやり取りされるHTTPヘッダを見ることができます。
[Adblock Plus](https://chrome.google.com/webstore/detail/adblock-plus/cfhdojbkjhnklbpkdaibdccddilifddb)のアイコンをクリックするとかなりかっこいい設定画面が表示されます。

その他にも、アイコンをクリックしたら何らかのスクリプトが動作するというパターンもあります。
[itolize](https://github.com/masarakki/itolize)では、アイコンをクリックするとページのcssが変化し、**どんなWebページも伊東ライフのあのフォント**で表示されるようになります。
[nico-play-all](https://github.com/masarakki/nico-play-all)は、niconicoの動画内検索で出てきた動画をワンクリックで一気に再生リストに突っ込むことができます。

[^1]: Omnibox (URL入力するところ) の右側

## Browser ActionとPage Action

アイコンに動作を設定するには、**Browser Action** か **Page Action** を使います。
アイコンが常に有効な時はBrowser Actionを、有効/無効が切り替わる場合はPage Actionを、という基準で使い分けます。
上記の例だと、`itolize` はどんなページにも使えるのでBrowser Action、`nico-play-all` はniconicoの動画内検索ページでしか動作しないのでPage Actionに設定します。

`yo` で雛形を作る場合、

    ? Would you like to use UI Action?
      No
    ❯ Browser Action
      Page Action

と聞かれるので上手く答えてあげてください。

## Popup

単純にポップアップを出したいときは、`yo` で雛形を作り `popup.html` を編集すればよいでしょう。
JavaScriptやStylesheetも同時に作られるので迷うこともないでしょう。

## Background Page

単純なポップアップ以外は、**Background Page** で設定することができます。
Backgroud PageはChrome Extensionの最も重要なパーツで、アプリケーションのエントリーポイントになります。
ExtensionがインストールされるとBackground Pageが一度だけ実行されます。
本質的にChrome Extensionは、Chrome JavaScript APIに用意された様々な**Event**に対し、リスナーを登録することでアプリケーションを構築します。
このリスナー登録を行うのがBackground Pageであり、その重要度は一目瞭然でしょう。

Background Pageは機能的にもExtensionの中心になります。
JavaScript APIの中には、Background Pageでしか使えないものも多く、そのような機能をWebページ内(=Content Script)で使いたい場合、Background Pageに依頼しなければいけません。
これを実現するために、Background PageとContent Scriptなどのパーツ同士が通信する、**メッセージ**の機能を使います。
逆に、Background PageからContent Scriptを触りたい場合も、Background PageからContent Scriptに対してメッセージを投げなければなりません。
このような関係から、Background PageはExtensionの中心であり、司令塔のような役割を持っています。

## nico-play-allを読んでみよう

**nico-play-all** はniconicoの動画検索ページにいる時にアイコンを押すことで、一括でプレイリストに登録するExtensionです。
デフォルトの一括再生ボタンと違うのは、

- 古い順に再生されるようにリストに登録する
- 見たことのある動画を除外する

という機能があることです。
すべては **MMD艦これ** 動画を古い順に全部見るという目的に最適化するように作られました。

それでは `background.js` を読んでみましょう。

一番重要なのは、

```js
chrome.pageAction.onClicked.addListener(tab => {
  chrome.tabs.sendMessage(tab.id, { action: 'playAll'});
});
```

というコードで、Page Actionのアイコンが押された時の動作を設定しています。
動作の中身も単純で、アイコンが押された時に見ているタブが引数で渡されるので、そのタブに対してメッセージを送るだけです。

送られたメッセージを受信する部分は `contentscript.js` にあります。

Content Scriptでは、

```js
chrome.runtime.onMessage.addListener(e => {
  playAll();
});
```

のコードでメッセージを待ち受けます。
変数 `e` には、メッセージの中身

```json
{ "action": "playAll" }
```

が入っているのですが、
今回は他にメッセージが来ないので特に分岐処理などはしていません。

`playAll()` では、DOMの中から動画を抽出しています。
前述のように、Content Scriptからサイトに作用するには[^2]、DOMに対してイベントを発行するしかないので、実際にリストに追加するイベントが設定されたオブジェクトを慎重に見つける必要があります。

見たことのある動画を除外するためには、**History API** を使わなければなりません。
さてこのHistory APIというやつ、**Background Pageでしか動作しません**。
そのため、今度はContent ScriptからBackground Pageにメッセージを投げます。

```js
chrome.runtime.sendMessage({url: url}, response => {
  if(!response.visited) {
    クリックする処理
  }
});
```

メッセージは例えば
```json
 { "url": "http://www.nicovideo.jp/watch/sm9" }
```
のようになります。

これを `background.js` で同じように受け取ります。

```js
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  chrome.history.getVisits({url: request.url}, visits => {
    sendResponse({ visited: visits.length > 0});
  });
});
```

`chrome.history.getVisits` は、渡されたurlにアクセスした歴史をすべて返します。
ひとつでもエントリーが存在するどうかを、第3引数の `sendResponse` を使ってリクエスト元に返しています。
Content Scriptはコールバックでレスポンスを受け取り、visited でなければクリックイベントを発行します。

このように、Content ScriptとBackground Pageが互いにメッセージをやりとりして、
それぞれの領分で機能します。
やり取りするメッセージの種類が多い場合は、メッセージの中身で場合分けする必要があるでしょう。

[^2]: この場合は再生リストに動画を登録すること

## Browser Action/Page Actionのデバッグ方法

`default_popup` で設定された単純なポップアップは、アイコンを右クリックして出てくるメニューから `[Inspect popup]` を選択することでDeveloper Toolsを起動できます。
それ以外の部分はContent ScriptやBackgroud Pageのデバッグ方法を使います。
