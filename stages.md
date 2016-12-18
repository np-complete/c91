# Chrome Extensionを構成する要素まとめ

ここまでで、 **Background Page**、 **Content Script**、 **Browser Action / Page Action** の3つの要素を見てきました。
実はChrome Extensionは、たったこれだけしか要素がありません。
たった3つの要素でアプリケーションが作られています。
おさらいしましょう。

## Content Script

Content Scriptは、既存の**ページにJavaScriptやStylesheetを追加する**機能です。
マッチするURLを指定して対象のページを限定することもできます。

通常のJavaScriptとほぼ同じように書くことが可能ですが、**元ページの関数や変数にアクセスすることはできません**。
DOMのイベントを発火することはできるので、イベントリスナに登録されたコードなら間接的にアクセスすることが可能と言えます。
主にDOMにアクセスして表示する要素を変更するなどの用途が多くなると思います。

Chrome JavaScript APIのほとんどのAPIは、ここで**直接使うことはできません**。
使えるAPIは[ドキュメント](https://developer.chrome.com/extensions/content_scripts)にリストアップされています。
それ以外のAPIを使いたい場合は、Background PageにMessageを投げ、依頼しなければなりません。

## Browser Action / Page Action

Browser Action / Page Actionは、アドレスバー横の**アイコンを押すと発動する**機能です。
**特定のページだけ有効**にする場合はPage Actionを、**常に有効**にする場合はBrowser Actionを使います。
できることに違いはありません。

単純なケースでは、**Popup**を表示するように指定することができます。
Popupは、単純にHTMLとJavaScriptとCSSで構成されています。
Content Scriptと違い、PopupのJavaScriptからは**すべてのChrome JavaScript APIが呼び出せます**。

Background Pageで`chrome.browserAction.onClick` にイベントを登録することで、
Popupを表示するのではなく、**単なるボタンとして利用する**ことも可能です。
また、アイコンを変更したりアイコンにバッジ(小さなテキスト)を付けたりすることも可能です。

## Background Page

Background Pageは、その名の通り**常にBackgroundに存在しつづけます**。
、Chrome Extensionを構成する要素の中で**最も重要**なもので、Extension全体をコントロールする立場にあります。
もちろんBackground Pageでは、**すべてのChrome JavaScript APIが呼び出せます**。
PopupでもAPIは使えますが、わざわざBackground Pageに依頼するのも定跡のひとつです。

Extensionのインストール時に必ずBackground Pageが実行されるので、このタイミングで様々な**イベントリスナを登録**します。
Chrome Extensionは基本的に、**ブラウザが発行する様々なイベント**をイベントリスナで受け取って、何かしらの反応をする、というモデルでプログラミングしていきます。
メッセージを受け取るのもイベント、タブが切り替わるのもイベント、ダウンロードが始まるのもイベント、あらゆる行動がイベントとしてフック可能になっています。
