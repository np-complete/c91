# Chrome Extensionを作ってみよう

この章では、どのようにChrome Extensionの開発を進めていくか、実際に開発したExtensionを例に、ハンズオン形式で具体的に解説していきます。
作りたいものは既に**無限**に存在していると仮定[^1]しますが、一応見つけるヒントも書いておきます。

[^1]: プログラマなら当然ですよね?

## 作りたいものの探し方

Chrome Extensionは、ブラウザの様々な動作に影響を及ぼすことができます。
普段ブラウザを使っている時に、何かがめんどくさいとか、何かを自動化したいとか、特定のWebサイトが見づらいとか、ショートカットキーが欲しいとか、ブラウザの機能が足りないとか、痒いところに微妙に手が届かないとか、いろいろ不満を抱えているのではないでしょうか?
これらの不満の多くはChrome Extensionを作ることにより解決できるものがほとんどだと思われます。

おそらくみなさん**起きている時間の90%くらい**をChromeの上で過ごしているはずですが、
そのような人生の非常に大きなウェイトを占めるChromeの不満を、そのままにしておいては絶対にいけません。
**確実に人間の精神を蝕んでいきます**。
ブラウザを使っていて、少しでも「めんどくさい」とか「使いにくい」と感じ、
それが2回目もあるようならば、Chrome Extensionを作る価値があります。
自分専用でもいいのです。

## 作るもの

ダウンロードしたファイルの名前を変えるExtensionを作ります。
[iwara.tv](http://iwara.tv/)というMMD専用の動画サービス[^2]では、動画ファイルのダウンロードが可能なのですが、**ユーザがアップロードしたファイル名のまま**[^3]ダウンロードされてしまいます。

これでは見たい動画を探すのに不便なのは明らかで、以前は**手作業で** `ユーザ名/動画タイトル.mp4` というファイル名にリネームして保存していました。
一度にたくさんのタブを開き同時ダウンロードするので、ダウンロードした動画をひとつひとつ再生し、サイト上でも再生し、一致するタブを探しリネームするというかなり手間のかかる作業です。
この作業のめんどくささのため、動画を**使う**までのハードルが非常に高くて困っていました。
あるとき**いい加減めんどくさい!!**と思い、Extension化を決断しました。

[^2]: !警告! 職場等で視聴しないほうがよい類の動画を集めるサービスです
[^3]: しかも漢字を中国語読みしてアルファベットに置き換える処理が入る

## 実現可能性を探る

まず重要なことに、サイトのDOMから**ユーザ名** や **動画タイトル**を拾わなければならないので、**Content Scriptは必須**だというのが判明しました。
動画をダウンロードさせる方法は、この段階ではまだ決めていませんが、候補はいくつかに絞られます。

- サイトのダウンロードボタンを再利用する
- Page Actionのボタンを使う

サイトに新たにダウンロードボタンを追加することもできますが、
それよりもPage Actionを使ったほうがスジが良さそうです。
サイトのダウンロードボタンを再利用できるのが最良です。

ダウンロード手段も、

- ブラウザのダウンロードを使い
  - 始まる前にリネームする
  - 終わった後にリネームする
- それ以外のなにかしらの方法でダウンロードする

くらいしかパターンはないでしょう。

次は、[API一覧](https://developer.chrome.com/extensions/api_index)を見て、使えそうなAPIがないか調べてみます。
[chrome.downloads](https://developer.chrome.com/extensions/downloads)というAPIが見つかったので、もしかしたらこれが使えるのではないかと期待します。
APIを見てみると `download` というメソッドと、 ``onCreated` や `onDeterminingFilename` などのイベントがあることがわかります。
**なんとかなりそうな気がした**ので、このAPIを試してみることにします。

`iwara-downloader` のディレクトリを作り、`yo` で雛形を生成します。

## このイベントはなんじゃ?

とりあえず、Chrome Extensionでは**イベントリスナを登録する**手法のほうが自然なので、まず**イベントの方に注目**します。
`onCreated` より `onDeterminingFilename`、 直訳すると「**名前を決定するとき**」というイベントが気になります。

ドキュメントを読むと

> ファイル名を決定するとき、Extensionは `DownloadItem.filename` を上書きする機会が与えられます

まさにドンピシャの機能です。
これを試してみましょう。
`background.js` にこんなコードを書いてみます。

```js
chrome.downloads.onDeterminingFilename.addListener((download, suggest) => {
  console.log(download);
  suggest({filename: 'hello/world.mp4'});
});
```

このExtensionを登録し、適当なサイトでファイルをダウンロードしてみると、`~/Downloads/hello/world.mp4` というファイル名[^4]で保存されます。
大成功ですね。
**勝ったな! ガハハ!!**という気分ですが、間違いなくこれは敗北フラグです。

さてこれを

- iwara.tvで
- HTMLを見て適切な名前を付ける

を満たそうとするととたんに難しくなります。

まずBackground Pageはブラウザ全体に影響するので、iwara.tvだけに限定するならダウンロードURLなどで判断するしかありません。
そして何より問題なのは、**どのタブでダウンロードされたか**がわからないので、Content Scriptに問い合わせることが難しいのです。

ひとつだけ手はあるかもしれません。
[chrome.tabs](https://developer.chrome.com/extensions/tabs) APIの `onActivated` を使い、常にアクティブなタブを監視します。
今見ているタブがiwara.tvの動画ページかどうか、そうであればユーザ名/動画タイトルは何か、という情報を取得しておけば、**ほぼ**上手くいきます。
ほぼというのは、ダウンロードボタンを押し、`onDeterminingFilename` が呼ばれる前にタブを移動してしまった場合、上手く行かない可能性があるということで、これは実際の作業では**おそらく頻繁に**起こるでしょう。
人間の操作スピードと気の短さは、**ダウンロードの準備ができる**すなわち**Webページがレスポンスを返す**時間よりも圧倒的に短いからです。

ここまで試してみた結果、**イベントを使うのは難しそうだ**と考え方針を転換しました。

[^4]: ~/Downloads はChromeに設定されたダウンロードのディレクトリです

## download メソッドを使ってみる

`chrome.downloads` APIには、もうひとつ `download` メソッドという気になるものがあります。
こちらも試してみましょう。
ダウンロードするという機能を考えると、Content Scriptでやるのが一番相応しそうに見えますが、やはりBackground Pageでしか動きません。

Background Pageにコードを書いて試す場合、何かしらの**イベントの中でやらないと検証しにくい**ということが多々あります。
そのため、いったんアイコンを押すとダウンロードが発動するようなコードを書いてみます[^5]。

```js
chrome.browserAction.onClicked.addListener(tab => {
  chrome.downloads.download({url: 'https://google.com/', filename: 'hello/world.txt'});
});
```

アイコンをクリックすると、`https://google.com/` のHTMLが `~/Downloads/hello/world.txt` に保存されるでしょう。
これで任意のURLを任意のファイル名でダウンロードできることが判明しました。
`download` メソッドを使うという方針が確定します。

[^5]: manifest.jsonからdefault_popupを消す必要があります

## 本物のイベントを探る

仮に置いた `chrome.browserAction.onClicked` のことはいったん忘れて、次はContent Script側を試します。
というのも、Content Scriptの中で上手くMessageを送るタイミングを見つけられたのなら、おそらくこのコードは `chrome.runtime.onMessage` に置き換えられるからです。

まず、Content Scriptは対象URLを限定できるので、 `http://*.iwara.tv/videos/*` にマッチするURLだけで読み込まれるようにします。

Content Scriptは普通にサイトにJavaScriptを追加するように書けるので、**自分がiwara.tvの管理者になった気分で**コードを書いていきます。
インスペクタでHTMLを見ながら、本物のダウンロードボタンの仕様を確認しましょう。

```html
<div class="node-buttons">
  <a href="/download/12345" target="_blank" class="btn btn-primary">
    <span class="glyphicon glyphicon-arrow-down"></span>
    ダウンロード
  </a>
</div>
```

となっています。 単純な `<a>` タグであることがわかります。
Content Scriptでクリックイベントを追加してみましょう。

```js
$('.node-buttons > a').on('click', e => {
  e.stopPropagation();
  e.preventDefault();
});
```

この状態のExtensionを読み込み、おもむろにダウンロードボタンを押してみます。
**何も起こりません**。
なんと**デフォルトのイベント伝搬はExtensionからでも止められる**ことが判明しました。

この段階でやっと、今度こそ本気で**勝ったなガハハ!!**の確信が持てました。
`chrome.runtime.sendMessage` でBackground Pageにメッセージを送りましょう。
メッセージの中身は、`href` に指定されたファイルのURLと、DOMからタイトルとユーザ名を探し、保存したいファイル名を指定します[^6]。


```js
const username = sanitize($('.node-info .username').text()).trim();
const title = sanitize($('.node-info .title').text()).trim();

$('.node-buttons > a').on('click', e => {
  e.stopPropagation();
  e.preventDefault();

  chrome.runtime.sendMessage({
    url: url.resolve(location.toString(), e.target.href),
    filename: path.join('iwara', username, `{filename}.mp4`}
  });
});
```

もちろん、Background Page側も `chrome.runtime.onMessage` に差し替えです。

こうしてできあがったのが [iwara-downloader](https://github.com/masarakki/iwara-downloader)です。

[^6]: ファイル名に使えない文字を削除したり余計な空白を除く処理も必要です

## ユニットテストを~~しよう~~

開発するからにはテストが書きたくてウズウズしちゃうんじゃないでしょうか?
でも残念、**テストはほとんどできません**。
諦めましょう。

なぜなら**Chrome JavaScript APIの動作が予測不可能**だからです。
例えば、 `chrome.downloads.download` が本当にファイルをダウンロードしてくれるのか分かりません。
`download` メソッドに、想定する通りの引数が渡るかどうかSpyを設定することはできるでしょう。
しかし、Content Scriptからメッセージを飛ばすところを再現するのは非常に困難です。
そしておそらく、Chrome以外で実際にAPIを叩ける環境は無いでしょう。

結局、テストする方法は今のところなさそうなので、あっさりと諦めたほうが心は安泰です。


## ふりかえる

まず最初に、どんなAPIを使うか考えます。
ブラウザを構成する要素にはほとんど対応するAPIが用意されています。
作りたいものを**手動でやった場合**に使う要素を考えれば、おそらく必要なAPIが見つかるでしょう。
今回はダウンロードしたファイルをどうにかする話なので、**download** とか、もしかしたら **file** とかそんな名前かもと当たりをつけて探します。
偶然 `chrome.downloads` が見つかったのでそれを試してみました。

おそらくそのAPIを使える場所は、Background Pageになるはずです。
次は、Background PageでそのAPIが想定通りに使えるかどうかを試します。
試すときはBrowser Actionの `onClicked` を使うと楽でしょう。
今回は、最終的に `chrome.downloads.download` メソッドを使えば上手く行くことが判明しました。

次に、実際にどんなイベントでその機能が発動するか考えます。
Webページ内から操作したければContent Scriptとメッセージをやり取りするし、
例えばタブが切り替わった時に発動したいのなら、同じようにAPI一覧から**tab**っぽいAPIを探し、イベントを選びます。
すぐに `chrome.tabs.onActivated` が見つかるでしょう。

この段階で、おそらく欲しい機能を満たすことはできているでしょう。
いくつかの機能が必要なら、同じことを繰り返すだけです。
あとは使いやすさを向上させるために、Content Script側をがんばれば、いいものができあがるのではないでしょうか
