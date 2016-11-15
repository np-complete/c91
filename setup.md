# プログラミング環境の準備

すぐにコードを試せる状態を作っておくのは、プログラミングの学習で一番重要なことです。
例を見て行く前に、Chrome Extensionのプロジェクトの構築方法を学びましょう。

## 必要なもの

まずもちろん **Google Chrome** が必須です。
そしてプログラミングをするので **Emacs** 。
本来は必須ではないのですが、環境構築およびプログラミングを便利にするために **Node.js** を使います。

## プロジェクトを作る

Chrome Extensionプロジェクトの雛形を作る便利なライブラリがあるのでそれを使います。

    npm install -g yo generator-chrome-extension

どうもJavaScript界隈の人は、ファイルを自動生成するという意識が弱いのか、
全く人気がなくほぼ死にかけているライブラリなのですが、ありがたく使わせていただきます。

ビルドに必要なコマンドも入れておきます

    npm install -g gulp-cli bower

適当にディレクトリを作っておもむろにジェネレータを実行します。

    mkdir my-extension
    cd my-extension
    yo chrome-extensino

いろいろオプションを聞かれますがとりあえず何も考えずにEnterを押しましょう。
いくつかファイルが作られているはずです。
`package.json` を見ると依存するライブラリが大量に書かれているのでライブラリを入れます。

    npm install

これだけで準備は完了です。
早速ビルドしてみましょう。

    gulp babel

`app/scripts.babel/` にあるファイルが `app/scripts` にトランスパイルされます。
もし、Babelによるトランスパイルが必要ない場合、直接 app/scripts にコードを書いてもいいでしょう。
Babelの提供する機能の多くは[すでにChromeに搭載されている](https://kangax.github.io/compat-table/es6/)ので、
よほどのことがない限り必要ないかもしれません[^1]。

[^1]: なんと、Chrome Extensionの実行環境はChromeだけなのです! Web開発者が待ち望んだ世界がここにあるのです!

## Chromeにインストール

さて、生成された `app/scripts` は完全なChrome Extensionとして動作します。
Chromeの `chrome://extensions` を開き、 `[Load unpackaged extension]` ボタンから `my-extension/app` を読みこんでみましょう。リストに登録されるはずです。

リストの中に見つかったら、 `Inspect views: background page` というリンクを開いてみましょう。
デバッグコンソールが開き、`previousVersion` とコンソールに出力されているはずです。

## 自動ビルドする

毎回 `gulp build` するのは面倒なので、ファイルが更新されるたびにビルドしなおしてくれる `gulp watch` という素晴らしいタスクを使いましょう。
ファイルを変更するたびにビルドログが流れるのが見えるでしょう。
ブラウザにExtensionの再読み込みも行ってくれます。
デバッグコンソールを開いた状態で、 `app/scripts.babel/background.js` の `console.log` で出力する文字列を変更し保存します。
ビルドがすべて終わったあと、デバッグコンソールをリロードしてみてください。
文字列が変化しているのがわかると思います。
