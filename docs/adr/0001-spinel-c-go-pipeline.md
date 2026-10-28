# ADR 0001: Spinel（Ruby → C）と ccgo（C → Go）を連結して Ruby のメソッドを Go に移植する

- Status: Proposed（PoC で検証してから Accepted にする）
- Date: 2026-10-29

## Context

s245 は、Ruby のおもしろいメソッド（`String#squeeze`、`String#succ`、`Integer#digits` など）を Go から使えるようにするライブラリです。手で移植すると Ruby の挙動とずれやすいので、変換ツールを連結して機械的に移植できるかを検討しました。

### Spinel について（matz/spinel @ ec0ad13e, 2026-10-29 時点）

- Ruby の AOT コンパイラです。全体の型推論をしたうえで C を 1 ファイル出力します（`spinel app.rb -c`）。
- Ruby の組み込みメソッドの多くは、ランタイムの手書き C 関数として実装されています。
  - 例: `lib/sp_str.c` の `sp_str_squeeze` / `sp_str_tr` / `sp_str_center` / `sp_str_casecmp`、`succ` と `digits` は `lib/sp_cold.c` など。
  - そのため、Go に変換されるのは **CRuby の実装ではなく、Spinel が C で書き直した実装** です。
- ブロックを取るメソッド（`each_slice` / `each_cons` など）はランタイムの関数ではありません。コンパイル時に、型ごとのループとして生成コードの中に展開されます（`src/analyze_arm.c`）。
- 生成された C は `lib/libspinel_rt.a`（`lib/*.c` で約 5 万行）と `spinel_rt.h`（約 2.3 万行）に依存します。
- 1 つのメソッドでもランタイムの一部が付いてきます。`sp_str_squeeze` は次に依存しています。
  - GC のルート登録（`SP_GC_ROOT_STR`）
  - 長さヘッダ付きの文字列確保（`sp_str_alloc_raw`）
  - エンコーディングの判定（`sp_str_is_binary`）
  - 例外（`sp_raise_cls`）
- 出力は `main()` を持つ実行ファイルです。Ruby のメソッドは `static` な C 関数になり、外から呼べるライブラリとしては出力されません。`--emit-symbol-map` で C のシンボル名と Ruby の名前の対応は取れます。
- 移植性のための仕組みがあります。
  - `SP_PORTABLE`: `__builtin_*` などのコンパイラ拡張を使わず、すべて素の C の実装にします。
  - `SP_EXC_ABORT`: setjmp/longjmp を使いません。例外は rescue できず、発生したら trap します。
- GC のルートは shadow stack（`SP_GC_ROOT`）で明示的に管理します。C スタックを保守的にスキャンしないので、別の言語に変換されてもルートが壊れません。
- 変換の妨げになりそうな箇所:
  - Fiber 用のインライン asm / ucontext
  - pthread と TLS（`_Thread_local` / `__thread`）
  - Darwin でだけ使う `__builtin_frame_address`
- Spinel 自身は CRuby を正解（oracle）にしたスナップショットテストで互換性を確かめています（`spin test`）。

### C → Go 変換器

| ツール | 状況 | Spinel ランタイムとの相性 |
|---|---|---|
| ccgo v4（modernc.org） | 現在もメンテされている（v4.38.0, 2026-10）。SQLite などの大規模な C を変換した実績がある | modernc.org/libc に setjmp/longjmp がある（ネストしたケースは最近の libc で修正済み）。ポインタは `uintptr` になる |
| cxgo | setjmp は実行時に panic する。TLS は未実装 | ランタイムが setjmp と TLS に依存しているので不適 |
| c2go（elliotchance） | 開発停止 | 不適 |

## Decision

以下のパイプラインを採用し、**PoC で動くことを確かめてから Accepted にします**。

```
移植したいメソッドを呼ぶだけの Ruby（例: def squeeze(s) = s.squeeze）
  │ inline RBS で型を固定（#: (String) -> String）
  ▼
  │ ① spinel -c  （-DSP_PORTABLE, 必要なら -DSP_EXC_ABORT）
  ▼
C（生成された TU）+ Spinel ランタイムの C ソース（必要な分だけ）
  │ ② ccgo v4 で Go に変換
  ▼
Go パッケージ（内部用・unsafe/uintptr ベース）
  │ ③ s245 が生成する薄いラッパーで Go らしい API にする
  ▼
s245.Squeeze(s string) string
  │ ④ CRuby との差分テスト
  ▼
同じ入力で CRuby と Go の結果が一致することを CI で保証する
```

1. **入力は「メソッドを呼ぶだけの Ruby」にします。** Ruby の挙動そのものが仕様なので、他言語から書き直す工程はありません。
2. **正解は CRuby とします。** Go に変換されるのは Spinel 版の実装なので、Spinel と CRuby の差もそのまま Go に持ち込まれます。そのため、すべてのメソッドで CRuby との差分テストを必須にします。差があったメソッドは、Spinel に報告するか、s245 の対象から外します。
3. **型は inline RBS で固定します。** 型が untyped に広がるとボックス化された遅い経路になり、生成されるコードも大きくなります。`--warn-widen` が警告を出したらビルドを失敗させます。Spinel は型ごとに C を出すので、Go の API もジェネリクスではなく具体的な型の関数になります。
4. **エントリポイントはエクスポートした C 関数にします。** FFI の C 埋め込み機能か生成後の後処理で、`static` でない関数を出させます。`main()` は使いません。
5. **ランタイムは必要な部分だけ変換します。** Fiber・Thread・IO・正規表現などの翻訳単位を外せるかを PoC で確認します。asm と TLS を含む部分は、マクロでスタブに差し替えるか、変換の対象から外します。
6. **公開 API は Go らしくします。** `libc.Xmalloc` によるメモリのやりとりや、Ruby の例外（`ArgumentError` など）はラッパーの中に閉じ込めます。例外は Go の `error` として返し、利用者に `uintptr` を見せません。

### 対象メソッドの段階

| 段階 | 対象 | 例 |
|---|---|---|
| 1（PoC） | 文字列か整数を受け取り、文字列か整数を返す。ブロックを取らない | `String#squeeze` / `#tr` / `#tr_s` / `#succ` / `#center`、`Integer#digits` |
| 2 | ブロックを取るが、結果を配列にして返せる | `each_slice(n).to_a` / `each_cons(n).to_a` を `[][]int64` などで返す |
| 対象外 | 任意のブロック（Go の `func`）を受け取るもの | ブロックの中身がコンパイル時に展開されるため、Go から関数を渡せない |

### Ruby と Go の意味の違い

| 項目 | 方針 |
|---|---|
| 文字列のエンコーディング | Go の `string` は UTF-8 として扱う。バイナリ文字列用の API は段階 1 では用意しない |
| nil | 引数としては受け付けない。返り値が nil になりうるメソッドは `(T, bool)` で返す |
| Integer の溢れ | `--int-overflow=raise`（Spinel の既定）で変換し、溢れたら `error` を返す。Bigint が必要なメソッドは段階 1 の対象から外す |

### PoC の合格基準

- `String#squeeze` と `String#succ` について、マルチバイトを含むテストベクタで Go 版の結果が CRuby 4.0 と一致する
- `go build` が cgo なし（`CGO_ENABLED=0`）で通る
- 生成された Go のサイズと、手書きの Go 実装に対する速度比を記録する（目安: 生成コードは 10 万行以下、速度は手書き実装の 10 倍以内）

## Consequences

### 良い点

- 書き直しの工程がないので、移植は完全に機械化できる。メソッドを増やすコストは「Ruby を 1 行書き、テストベクタを足す」だけ
- CRuby を正解にした差分テストで、Ruby の挙動との一致を継続的に保証できる
- cgo に依存しない Go パッケージとして配布できる

### 悪い点・リスク

- 互換性の上限は Spinel の実装で決まる。CRuby と違う挙動は、差分テストで見つけるまで混入しうる
- Spinel ランタイムの一部が付いてくる。生成コードは数十万行規模になり、ビルド時間とバイナリサイズが大きく増える可能性がある
- ccgo の出力は `unsafe` / `uintptr` ベースで読みにくい。デバッグやレビューは Go のコードとしてはほぼできない
- ブロックを取るメソッドは、汎用的な形では提供できない
- Spinel はリリース形式が日付タグで、変更の頻度も高い。s245 は使う Spinel のリリースタグを固定する
- ランタイムの一部（asm・TLS・setjmp のネスト）が ccgo で変換できない場合は、Spinel 側にパッチが要る

## Alternatives considered

- **PHP など他言語の関数を対象にする**（例: PHP の `levenshtein()`）: Spinel は Ruby しか読めないので、まず Ruby で書き直す工程が必要になる。そこが自動化できず移植ミスの温床になるため、対象を Ruby のメソッドに絞った
- **CRuby の実装（string.c など）を直接 ccgo で変換する**: 本物の実装なので互換性は最も高い。ただし `VALUE` やオブジェクトモデル、GC など CRuby 全体への依存が大きく、切り離すのは Spinel ランタイムより難しい
- **Spinel の `--target=wasm32-wasi` で wasm にし、Go からは wazero で実行する**: C→Go 変換のリスクがなく、cgo も要らない。ただし呼び出しのオーバーヘッドと、WASI ホストの例外処理の対応状況が課題。ccgo での変換が PoC で行き詰まった場合の第一候補とする
- **手書き / LLM で Go に移植する**: 生成物は最も小さく速い。CRuby との差分テストは同じように使える。メソッドの数が少ないうちは、こちらの方が安く済む可能性がある
- **cxgo を使う**: setjmp と TLS が未対応なので採用しない

## References

- Spinel: https://github.com/matz/spinel（README, docs/limitations.md, docs/portability.md, docs/wasm.md, lib/sp_setjmp.h, lib/sp_gc.h, lib/sp_str.c, src/analyze_arm.c）
- ccgo v4: https://pkg.go.dev/modernc.org/ccgo/v4
- modernc.org/libc（setjmp/longjmp の修正）: https://gitlab.com/cznic/libc/compare/v1.74.4...v1.75.0
- cxgo: https://github.com/gotranspile/cxgo
