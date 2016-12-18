# Webサイトをカスタマイズしよう

まずは一番わかりやすく、一番身近な例からいきましょう。
普段利用しているWebサイトを、あたかも自分のサイトかのように自由にいじくり回すことが可能です。

**Content Script** という機能を使えば、自分のサイトにJavaScriptのファイルをひとつ追加で読み込ませるような感覚で、どんなサイトにもJavaScriptを追加することができます。

ものすごく使いづらいサイトなのに、絶対使わないといけない、という時などに非常に高い効果を発揮します。
例えば、**コミケWebカタログ** なんかはとても良い例なのではないでしょうか。
控えめに言って**超絶クソ**なので、自分の使いやすいようにカスタマイズできると非常に有用です。

コミケWebカタログをカスタマイズするChrome Extensionが [fuck-catalog](https://github.com/masarakki/fuck-catalog) です。
コードを眺めていきましょう。

## まずはManifestを読む

Chrome Extensionのコードを読むときは、 まず最初に **manifest.json**[^1] に書かれたアプリケーションの概要を見ましょう。
この例では、 `content_scripts` という項目が見つかるでしょう。
この場合、 `https://webcatalog.circle.ms/CircleRapid/*` (サークル一覧ページ)に、`scripts/contentscript.js` を適応することが書いてあります。
このように、`manifest.json` を読むと、どこにどのファイルを読み込むかがわかります。

`manifest.json` は、 `yo chrome-extension` で聞かれる質問に適切に応えることでほとんど自動生成できます。
もちろんあとから手で編集することもできます。

[^1]: 実際には app/manifest.json にある

## Content Script を読む

manifestに書かれている `app/scripts/contentscript.js` ではなく、
変換前のファイルである `app/scripts.babel/contentscript.js` を読みます。

クソコードがダラっと書かれていますが、
読んで一目瞭然、皆さんには非常に馴染みが深いであろうjQueryでDOMを弄る処理が書かれています。
自分のサイトでJavaScriptを書いてDOMを操作するのと全く一緒の感覚なのが解るでしょう。
ひとつ明らかに違うところは、 `manifest.json` の `"run_at": "document_end"` の指定により、必ずページが読み込まれたあとに実行されることが保証されている点です。
`$(document).ready` 的なコードは必要ありません。

## Content Scriptのデバッグ方法

Content Scriptをデバッグするには、該当するWebページを開き、Developer Toolsを起動します。
Sourcesタブを選択すると、ファイル一覧が表示されるゾーンに `[Content scripts]` のタブがあることに気づくでしょう。
その中から該当するExtensionを選び、デバッグしたいファイルを選択します。
あとは普段のDeveloper Toolsと同じです。

## Content Scriptでできないこと

サンプルのように、Content ScriptはDOMにアクセスできますが、
一方でサイトが本来持っている**JavaScriptには全くアクセスできません**。
より正確には、サイト側のJavaScript実行環境とContent Script側のJavaScript実行環境は、**完全に別世界**だと考えれば良いと思います。
実際にアプリケーションを作る際はこの制約がだいぶ障害になるでしょう。

例えばサイト側で `window` オブジェクトに変数や関数を設定していても、Content Scriptからはアクセスできません。
サイト上にあるオブジェクトにEventListenerを**追加することはできます**が、
サイト側で設定されたEventListenerを**削除することはできません**。
Content Scriptで追加したEventLitenerで、**イベントの伝搬を阻止**[^2] しようとしても、別世界なので止めることはできません。
既存のボタンの動作を変更したり、キーバインドを変更したりするのは難しいということです。

ひとつだけ例外があり、DOMに対してイベントを発行すると、サイト側のJavaScriptで登録されたイベントリスナも反応してくれます。
直接関数が呼べず回りくどいですが、上手く利用すれば様々なことが実現できます。[^3]

[^2]: stopPropagationとかpreventDefaultとかの例のお約束
[^3]: fuck-catalogのaddCheckとremoveCheckを参照
