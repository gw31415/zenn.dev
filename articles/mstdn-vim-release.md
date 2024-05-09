---
title: "MastodonのVimプラグインを作成した際の設計・実装で工夫したこと"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Vim", "Mastodon", "Deno", "Denops"]
published: true
published_at: "2024-05-14 18:00"
---

<!-- この記事は [MastodonのVimプラグインを作成した際の設計・実装で工夫したこと](https://blog.shinonome.io/mstdn-vim-release/) のクロスポストです。 -->

今回はVimからMastodonにアクセスするプラグインを作ったので、紹介を兼ねて設計・実装で工夫したことについて書きます。

https://github.com/gw31415/mstdn.vim

![mstdn-editor-rec](/images/mstdn-vim-release/mstdn-editor-rec.gif)

# 設計

## バックエンド

MastodonはWeb系のAPIを叩く必要がありますが、Vim標準の機能では実装が大変です。具体的には次のような課題があります。

- `curl` などのコマンドを使う必要がある
- 非同期処理が標準機能ではローレベルすぎる
- レスポンスのパースが大変

こういったプラグインを作成する際には [denops.vim](https://github.com/vim-denops/denops.vim) が便利です。denopsはVimプラグインの作成に[Deno](https://deno.land)を用いることができるシステムで、外部APIを叩くような処理を簡単に行える他、V8による高速なデータ処理、TypeScriptの強い表現力を利用して開発することができます。

## フロントエンド

使用に関わる部分は、なるべくVimの標準機能に沿ったもの、コマンドを増やさないものにしたいと考えました。具体的には次のような設計にしました。

- `mstdn://ama@example.com/home` のようなURL的スキームを用いて、Mastodonのタイムラインにアクセスできるようにする
  - `:e mstdn://ama@example.com/home` でタイムラインを開く
- 投稿に関してはコマンドを増やさず、 `:call` で呼び出せる関数を提供する
  - 投稿編集画面を開くなどの機能は別途プラグインで提供する

# 実装

## denops.vimの簡単な使い方

denops.vimの使い方については本題ではないので、ここでは簡単な使い方を紹介します。

denops.vimのプラグインは以下のようなディレクトリ構成にします。

```
.
└── denops
    └── hoge
        └── main.ts
```

こうするとdenopsのプラグインとして認識され、Vimから `hoge` をキーとして `main.ts` を呼び出すことができます。

`main.ts` の中身は `async main(Denops) -> Promise<void>` の関数をエクスポートするとそこがエントリーポイントになります。
main関数の引数には `Denops` が渡され、これを使ってVimとのやり取りを行います。

```ts
import { Denops } from "https://deno.land/x/denops_std@v5.1.0/mod.ts";
export async function main(denops: Denops): Promise<void> {
  // ここに処理を書く
}
```

### dispatcherについて

初期化処理は main 関数の中で行いますが、このままだと標準の設定ファイルで設定できるようなことしかできません。
TypeScript関数を適宜呼び出す際は、Denopsオブジェクトのdispatcherを使ってクロージャを登録しておき、Vimから呼び出すという形にします。

```ts
import { Denops } from "https://deno.land/x/denops_std@v5.1.0/mod.ts";
export async function main(denops: Denops): Promise<void> {
  denops.dispatcher = {
    async echo(denops: Denops, message: unknown): Promise<void> {
      await denops.cmd(`echo "${message}"`);
    },
  };
}
```

このようにすることで、Vimから `:call denops#request('hoge', 'echo', ['Hello, World!'])` というコマンドを実行して `Hello, World!` というメッセージを表示することができます(`denops/hoge/main.ts`内で登録した `echo` の引数を前から順に渡している)。

## Mastodon APIをプラグインのために抽象化する

最初の実装に基けば、目標のプラグインはバッファとタイムラインが一対一に対応する形になります。つまり、対応するタイムラインについては初期化処理、再接続時に同期的にフェッチする一方、更新は非同期で取得する必要があります。前者は通常のTimeline APIを使い、後者はStreaming APIを使うことになります。Streaming APIにはWebSocketとPollingの2つの方法がありますが、今回はWebSocketを使うことにしました。

ただし、MastodonのAPIはStreaming APIが提供されているタイムラインが限られていたり、通常のTimeline APIとアクセス方法に若干一貫性がありません。

| タイムライン           | Timelineのエンドポイント                           | Streamingで最初にPOSTするデータ                     |
| ---------------------- | -------------------------------------------------- | --------------------------------------------------- |
| ホーム                 | `/api/v1/timelines/home`                           | `{ "stream": "user" }`                              |
| 連合                   | `/api/v1/timelines/public?local=false`             | `{ "stream": "public" }`                            |
| リモート               | `/api/v1/timelines/public?local=false&remote=true` | `{ "stream": "public:remote" }`                     |
| ローカル               | `/api/v1/timelines/public?local=true`              | `{ "stream": "public:local" }`                      |
| ハッシュタグ(連合)     | `/api/v1/timelines/tag/${タグ名}?local=false`      | `{ "stream": "hashtag", "tag": "${タグ名}" }`       |
| ハッシュタグ(ローカル) | `/api/v1/timelines/tag/${タグ名}?local=true`       | `{ "stream": "hashtag:local", "tag": "${タグ名}" }` |

- 参考
  - [timelines API methods](https://docs.joinmastodon.org/methods/timelines/)
  - [streaming API methods](https://docs.joinmastodon.org/methods/streaming/)

そのため、これらの違いを吸収するために、Mastodon APIをプラグインのために`Method`という名で抽象化しました。

```ts
/**
 * 非同期取得のストリームの種類
 */
export type StreamType =
  | "public"
  | "public:media"
  | "public:local"
  | "public:local:media"
  | "public:remote"
  | "public:remote:media"
  | "hashtag"
  | "hashtag:local"
  | "user"
  | "user:notification"
  | "list"
  | "direct";

/**
 * ストリームの種類
 */
export interface Stream<T extends StreamType = StreamType> {
  stream: T;
  list: T extends "list" ? string : undefined;
  tag: T extends "hashtag" | "hashtag:local" ? string : undefined;
}
/**
 * 通信先
 */
export interface Method {
  /**
   * WebSocket確立のためのStream
   */
  get stream(): Stream;
  /**
   * REST APIのエンドポイント
   */
  get endpoint(): string;
}
```

Methodはstreamやendpointを持つものと宣言しています。これを implements することで、非同期更新と同期的取得をまとめた構造体として扱うこととしました。

実際に抽象化されたMethodオブジェクトは以下の部分に宣言しています。

https://github.com/gw31415/mstdn.vim/blob/320c801cd921e6ba10946c74921d1b0ae172ea14/denops/mstdn/entities/methods.ts

## 非同期更新されるバッファの実装

denopsを用いるとVimと通信して処理を行えますが、それでも投稿を全体的に更新していくと大変なコストがかかります。そのため、バッファ1つにつき1つのオブジェクトを割り当て、状態をキャッシュして差分更新する設計にしました。

### TimelineRendererに持たせるメンバ変数

当然、バッファにある投稿を配列として保持する必要があります。投稿が時系列で途切れないことが保証されていればそれでいいですが、WebSocketが切断された部分が途切れていることも保持しなければなりません。そのため、投稿保持のための配列の要素は `Status | LoadMore` としました。

`Status` は投稿の構造体を表し、`LoadMore` は「さらに読み込む」を表す別の構造体です。どちらも並べ替えのための `createdAt` プロパティを持っています。

### TimelineRendererに持たせるメソッド

以下はバッファに一対一対応し更新を引き受ける構造体 `TimelineRenderer` の大まかな設計です。メソッドの中身は省略しています。

```ts
interface LoadMore {
  createdAt: string;
  id: string;
}

/** LOAD MOREもしくはStatus */
interface StatusOrLoadMore<
  T extends "Status" | "LoadMore" = "Status" | "LoadMore",
> {
  /** Status か LoadMoreの実体 */
  type: T;
  data: T extends "Status" ? Status : LoadMore;
}
/** タイムラインのレンダーを引き受ける構造体 */
export class TimelineRenderer {
  private _statuses: StatusOrLoadMore[] = [];
  private bufnr: number;
  private constructor(bufnr: number) {
    this.bufnr = bufnr;
  }
  /** バッファに紐ついているStatus一覧 */
  get statuses(): StatusOrLoadMore[] {}
  /** 現在のバッファを初期設定する */
  public static async setupCurrentBuffer(
    denops: Denops,
  ): Promise<TimelineRenderer> {}
  /** 「さらに読み込む」マークを先頭に挿入する */
  public async addLoadMore(denops: Denops) {}
  /** 指定したIDのLOAD MOREの前後のStatusを取得する */
  public loadMoreInfo(id: string): {
    /** LOAD MORE直前の投稿 */
    prev: Status | null;
    /** LOAD MORE直後の投稿 */
    next: Status | null;
  } {}
  /** 適切な位置に投稿を挿入または更新する。 */
  public async add(
    denops: Denops,
    statuses: Status[],
    opts: {
      update_only?: boolean | undefined;
    } = {},
  ) {}
  /** 再描画 */
  public async redraw(denops: Denops, view?: WinSaveView) {}
  /** 投稿やLOAD MOREを削除する */
  public async delete(denops: Denops, id: string): Promise<boolean> {}
}
```

- `statuses` はバッファに紐ついているStatus一覧を返します
- `setupCurrentBuffer` は読み込み専用にしたりなど、現在のバッファを初期設定します
- `addLoadMore` は切断された際に呼び出し、「さらに読み込む」マークを先頭に挿入します
- `loadMoreInfo` はLOAD MORE直前もしくは直後の投稿を返します(何度も読み込むことを想定)
- `add` は適切な位置に投稿を挿入または更新します

## 投稿編集バッファ

これでタイムラインの実装は終わりですが、投稿を行うためのバッファも欲しいです。投稿をシームレスに行うために、タイムラインバッファ内で特定のキーバインドを押下することで専用バッファを開く、という実装が考えられます。
実装は別のプラグインとして提供しましたが、本記事でも設計などについて紹介します。

https://github.com/gw31415/mstdn-editor.vim

投稿編集用バッファの操作感としては、保存処理 (`:w`, `:wq`, `ZZ`, ...)をした際にタイムラインバッファに投稿を追加する、というものです。これは `BufWriteCmd` の自動コマンドを用いることで実装します。

https://github.com/gw31415/mstdn-editor.vim/blob/a4533096ad75e124356f169d12724823e3192fb2/autoload/mstdn/editor.vim#L22-L37

# 最後に

要点については以上です。このように、VimでMastodonのような特定のプラットフォームに依存するようなプラグインを作成するのには Denops は本当に強いなと感じました。

結構簡単にプラグインを作成できるので、興味がある方はぜひ試してみるのをおすすめします。手軽に環境を育てられるのがVimの良いところですね。
