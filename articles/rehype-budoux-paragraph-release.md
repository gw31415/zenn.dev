---
title: BudouXで分かち書きするrehypeプラグインを作った
emoji: 🇯🇵
type: tech # tech: 技術記事 / idea: アイデア
topics:
  - budoux
published: true
---

BudouXで分かち書きをするrehypeのプラグインを作りました。

https://github.com/gw31415/rehype-budoux-paragraph

# なぜ作ったか

[Astro.js](https://astro.build) で[個人的に試験勉強に用いているサイト](https://github.com/gw31415/mdblank)を作成しているのですが、長文を多く掲載するため分かち書きが必要になりました。 **せっかくSSGで読み込み時に呼び出される処理を減らしているのに、ブラウザサイドで `<wbr/>` を挿入する操作を行うのはもったいない** と思いました。そこでAstro.jsでStatic Siteを生成する時に分かち書き操作を行なえれば便利だなと思って作りました。

:::message
[`word-break: auto-phrase`](https://caniuse.com/mdn-css_properties_word-break_auto-phrase) が一般的になったとしても、これによってブラウザサイドで `<wbr/>` を挿入する操作を行うのに変わりはないので、この先も使えると思います。
:::

# 仕組み

仕組みはシンプルで、`<p>` タグのノードを検出した際に`children`の中身のテキストをBudouXプロセッサにかけ、ノードの`children`を置きかえるだけです。

```ts
visit(tree, "element", (node: any) => {
  if (node.tagName === "p") {
    // <p> タグの時
    // >>>>>>>>>>>>>>> 前処理：ノードの中身のテキストを連結する
    const processor = unified().use(rehypeStringify);
    let nodeText = "";
    for (const child of node.children) {
      nodeText += processor.stringify(child);
    }
    // <<<<<<<<<<<<<<<

    let value: string;
    switch (
      options.lang // BudouXで処理する
    ) {
      case "ja":
        value = jaParser.translateHTMLString(nodeText);
        break;
      case "zh-TW":
        value = zhtParser.translateHTMLString(nodeText);
        break;
      case "zh-CN":
        value = zhcParser.translateHTMLString(nodeText);
        break;
    }
    node.children = [
      // 中身を入れ換える
      {
        type: "raw",
        value: value,
      },
    ];
  }
});
```

## 苦労したところ

まず、`node`の`children`が単純なテキストで取得できなかったことです。rehypeのプラグインを初めて作成する人はひっかかるかもしれません。上に挙げたコードでは「前処理」と書いた部分に記載しています。また、`node`の`children`の中身を生のHTMLにする時もハマりました。

どちらもrehypeプラグインに作り馴れていないため苦労したところです。初めて作る方は参考にしていただけると嬉しいです。作り馴れている人はあたたかい目で見守ってください。
