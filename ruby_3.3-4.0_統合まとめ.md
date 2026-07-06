# Ruby 3.3 → 3.4 → 4.0 変更点 統合まとめ

出典：
- [Ruby 3.3.0 リリース](https://www.ruby-lang.org/ja/news/2023/12/25/ruby-3-3-0-released/)（2023-12-25）
- [Ruby 3.4.0 リリース](https://www.ruby-lang.org/ja/news/2024/12/25/ruby-3-4-0-released/)（2024-12-25）
- [Ruby 4.0.0 リリース](https://www.ruby-lang.org/ja/news/2025/12/25/ruby-4-0-0-released/)（2025-12-25）

---

## 1. パーサー

### 3.3: Prism の導入
- default gem として Prism パーサーを追加。ポータブルで、エラートレラント、保守可能な再帰下降パーサー。
- 本番利用可能なレベルで、Ripper の代替として利用できる。
- `ruby --parser=prism` または `RUBYOPT="--parser=prism"` で試用可能（このリリース時点ではデバッグ用フラグ）。

### 3.3: Bison → Lrama
- Bison を Lrama LALR パーサージェネレーターに置き換え。
- パラメータ化ルール `(?, *, +)` をサポートし、CRuby の parse.y で使用。

### 3.4: デフォルトパーサーが Prism に
- parse.y 由来のパーサーから Prism がデフォルトになった（内部的な変更でユーザー影響はほぼ無し）。
- 従来パーサーは `--parser=parse.y` で利用可能。

---

## 2. JIT コンパイラ

### 3.3: RJIT の導入（MJIT を置き換え）
- Ruby で書かれた JIT コンパイラ。Unix の x86_64 のみ対応、実行時に C コンパイラ不要。
- 実験目的のみ。本番では引き続き YJIT を推奨。

### 3.3 / 3.4: YJIT の強化
**3.3のポイント**
- 引数展開・レジスタ利用・例外ハンドラのコンパイルなど大幅な性能改善。Rails の `#blank?`/`#present?` 等の単純メソッドをインライン化。
- メモリ使用量の削減（メタデータ圧縮、`--yjit-cold-threshold` 追加など）。
- コード GC がデフォルト無効に（`--yjit-exec-mem-size` はハードリミット化）。
- `RubyVM::YJIT.enable` を追加し実行時にYJIT有効化が可能に（Rails 7.2 がこれを利用してデフォルト有効化）。
- 統計・プロファイリング機能（`--yjit-perf`、`--yjit-trace-exits` 等）を追加。

**3.4のポイント**
- x86-64 / arm64 双方で多くのベンチマークが向上、メモリ使用量も削減。
- `--yjit-mem-size`（デフォルト128MiB）による統一的メモリ制限、`--yjit-log` によるコンパイルログ監視。
- コンテキスト圧縮、ローカル変数・引数のレジスタ割当。
- YJIT有効時に `Array#each`/`#select`/`#map` など Ruby 実装のコアクラスを使用。
- 空メソッド・定数返却・`self`返却・引数そのまま返却のような小さなメソッドのインライン化。
- マルチ Ractor モードでの定数共有をサポート。

**4.0のポイント**
- デフォルトビルドで `ratio_in_yjit` が利用不可に（`--enable-yjit=stats` + `--yjit-stats` が必要）。
- `invalidate_everything` 統計を追加。
- `RubyVM::YJIT.enable` に `mem_size:`・`call_threshold:` オプションを追加。

### 4.0: ZJIT の導入（RJIT は廃止）
- YJIT の次世代として開発された新しい JIT コンパイラ。より大きなコンパイル単位・SSA IR を採用し、外部コントリビューションしやすい設計を志向。
- ビルドには Rust 1.85.0 以降が必要。`--zjit` または `RubyVM::ZJIT.enable` で有効化。
- 現時点ではインタプリタより高速だが YJIT には及ばない。本番投入は非推奨、Ruby 4.1 での高速化・本番対応が目標。
- `--rjit` は削除。サードパーティ実装は [ruby/rjit](https://github.com/ruby/rjit) リポジトリへ移動。

---

## 3. GC・メモリ管理

### 3.3
- Object Shapes による `defined?(@ivar)` の最適化。
- GC に弱参照を追加。古いオブジェクトの子オブジェクトが即座にプロモートされなくなった。
- 環境変数 `RUBY_GC_HEAP_REMEMBERED_WB_UNPROTECTED_OBJECTS_LIMIT_RATIO` を追加。
- 非推奨環境変数 `RUBY_GC_HEAP_INIT_SLOTS` を削除（`RUBY_GC_HEAP_{0-4}_INIT_SLOTS` を使用）。

### 3.4: Modular GC
- 標準以外の GC 実装を動的ロードできる機能。ビルド時に `--with-modular-gc` を指定し、環境変数 `RUBY_GC_LIBRARY` でロード。
- 組み込みGCは `gc/default/default.c` に分離、`gc/gc_impl.h` のAPI経由でランタイムとやり取り。`RUBY_GC_LIBRARY=default` で有効化可能。
- [MMTk](https://www.mmtk.io/) ベースの実験的GCライブラリを提供（`RUBY_GC_LIBRARY=mmtk`、ビルドにRustが必要）。
- `GC.config` と `rgengc_allow_full_mark` パラメータを追加。

### 4.0: 実装レベルの改善
- `Class#new` の高速化（キーワード引数利用時に特に効果、YJIT/ZJIT にも反映）。
- サイズプールごとのGCヒープを独立成長させ、メモリ使用量を削減。
- 大きなオブジェクトページのGCスイープ高速化。
- "Generic ivar" オブジェクト（String, Array, TypedData等）が新しい内部 "fields" オブジェクトでインスタンス変数アクセスを高速化。
- `id2ref` テーブルを初回使用まで保持しないことで `object_id` 割当とGCスイープを高速化。
- 大きな多倍長整数も可変幅アロケーションで埋め込みのまま保持可能に。
- `Random`、`Enumerator::Product`/`Chain`、`Addrinfo`、`StringScanner`等が書き込みバリア保護され、GCオーバーヘッド削減。

---

## 4. 並行処理（スレッド／Ractor）

### 3.3: M:N スレッドスケジューラ
- M個のRubyスレッドをN個のネイティブスレッドで管理し、生成コストを削減。
- C拡張ライブラリとの互換性問題のため、メインRactorではデフォルト無効（`RUBY_MN_THREADS=1` で有効化）。メインRactor以外は常時有効。
- `RUBY_MAX_CPU=n` でネイティブスレッド数の上限を設定可能（デフォルト8）。

### 3.4: Ractor 関連の追加API
- `Ractor.main?` を追加。
- Ractor内で `require` が可能に（処理はメインRactorで実行）。`Ractor._require(feature)` を追加。
- `Ractor.[]`/`Ractor.[]=`（Ractorローカルストレージアクセス）、`Ractor.store_if_absent(key){init}` を追加。

### 4.0: Ractor の大幅刷新
- `Ractor::Port` クラスを新設し、Ractor間通信を刷新。`#receive`、`#send`（`<<`）、`#close`、`#closed?` を提供。
- これに伴い **`Ractor.yield` と `Ractor#take` は削除**。
- Ractor終了待ちのため `Ractor#join`／`Ractor#value` を追加（内部実装として `#monitor`／`#unmonitor` も追加）。
- `Ractor.select` は Ractor と Port のみ受付。
- 各Ractorがデフォルトポートを持つようになり `Ractor.send`/`Ractor.receive` で使用（`#default_port` 追加）。
- `Ractor#close_incoming`／`#close_outgoing` は削除。
- `Ractor.shareable_proc`／`Ractor.shareable_lambda` を追加し、Proc/lambdaの共有を容易に。
- **パフォーマンス**：凍結文字列・シンボルテーブルのロックフリー化、メソッドキャッシュ参照のロック回避、クラスのivarアクセス高速化、Ractorごとのカウンタによるキャッシュ競合回避など、グローバルロック競合が大幅減少。
- **安定性**：Ractor×Threadのデッドロック、require/autoloadの問題、エンコーディング変換の問題、fork時の問題などを多数修正。GC後にTracePointが動作しない不具合も修正。
- Ruby 3.0で実験的機能として導入されて以降、4.0の時点でも引き続き experimental。「来年くらいには experimental を取りたい」とのコメントあり。

---

## 5. 言語仕様の変更

### 3.3: `it` 予約の予告
- ブロック内で引数なしの `it` を呼ぶことが非推奨に（3.4から最初のブロック引数を参照する仕様に変更される予告）。

### 3.4: `it` の正式導入
```ruby
ary = ["foo", "bar", "baz"]
p ary.map { it.upcase } #=> ["FOO", "BAR", "BAZ"]
```
- `_1` とほぼ同じ動作だが、`_2`・`_3` の存在を意識させない分、認知負荷が低い。単純な一行ブロック向け。

### 3.4: その他の言語仕様変更
- `frozen_string_literal` マジックコメントがないファイルの文字列リテラルは、freeze されているかのように振る舞い、破壊的変更時に警告（`-W:deprecated` 等で表示）。無効化は `--disable-frozen-string-literal`。
- メソッド呼び出し時の `**nil` 展開をサポート（`**{}` と同様に扱われる）。
- インデックス（`[]`）にブロックを渡せなくなった。
- インデックスにキーワード引数を渡せなくなった。
- トップレベルに `::Ruby` を予約（`Warning[:deprecated]` 有効時、既存定義があれば警告）。

### 4.0: 言語仕様の変更
- `*nil` は `nil.to_a` を呼ばなくなった（`**nil` が `nil.to_hash` を呼ばないのと同様の挙動に統一）。
- 行頭の論理二項演算子（`||`、`&&`、`and`、`or`）がドット記法のように前の行を継続するようになった：
```ruby
if condition1
   && condition2
  ...
end
```

---

## 6. コアクラスの更新（主なもの）

### 3.4
- **Exception**：`#set_backtrace` が `Thread::Backtrace::Location` の配列を受付可能に（`Kernel#raise`/`Thread#raise`/`Fiber#raise` も同様）。
- **Range**：`#size` は反復不可能なオブジェクトに対し `TypeError` を送出するように。

### 4.0
- **Array**：`#rfind`（`reverse_each.find` の効率的な代替）、`#find`（`Enumerable#find` の効率的な代替）を追加。
- **Binding**：`#local_variables` 系メソッドが番号付きパラメータを扱わなくなり、代わりに `#implicit_parameters`／`#implicit_parameter_get`／`#implicit_parameter_defined?` を追加（番号付き・`it` パラメータ用）。
- **Enumerator**：`.produce` がキーワード引数 `size` を受付（整数、`Float::INFINITY`、callable、`nil`）。
- **ErrorHighlight**：`ArgumentError` 発生時に caller/callee 両方のコードスニペットを表示。
- **Fiber / Fiber::Scheduler**：`Fiber#raise(cause:)` サポート、`#fiber_interrupt`・`#yield`・`#io_close`・`#io_write` フックを追加。
- **File**：Linuxでも `statx` 経由で `File::Stat#birthtime` が利用可能に。
- **IO**：`IO.select` が `Float::INFINITY` タイムアウトを受付。`|` で始まるプロセス生成（非推奨）を削除。
- **Kernel**：`#inspect` が `#instance_variables_to_inspect` により表示するivarを制御可能に。`|` で始まる `#open` のプロセス生成を削除。
- **Math**：`.log1p`、`.expm1` を追加。
- **Pathname**：default gem からコアクラスに昇格。
- **Proc**：`#parameters` の匿名オプション引数表示を `[:opt]` に統一。
- **Range**：`#to_set` がサイズチェック、`#overlap?` が無限範囲対応、始点なし整数範囲の `#max` を修正。
- **Ruby**（新モジュール）：Ruby関連の定数を含む新トップレベルモジュールを正式定義（3.4で予約済み）。
- **Ruby::Box**（実験的機能）：後述「7. Ruby Box」参照。
- **Set**：stdlibからコアクラスに昇格。`#inspect` が `Set[1, 2, 3]` 形式に簡潔化。`#to_set`／`Enumerable#to_set` への引数指定は非推奨に。
- **Socket**：`Socket.tcp`／`TCPSocket.new` が `open_timeout` キーワード引数を受付。タイムアウト時は原則 `IO::TimeoutError` に統一。
- **String**：Unicode 17.0.0／Emoji 17.0 に更新。`#strip` 系メソッドが `*selectors` 引数を受付。
- **Thread**：`#raise(cause:)` をサポート。

---

## 7. Ruby Box（4.0の新機能・実験的）

- クラス等の定義の分離／隔離を提供する実験的機能（クラス名 `Ruby::Box`）。環境変数 `RUBY_BOX=1` で有効化。
- Box内で読み込まれた定義（モンキーパッチ、グローバル/クラス変数操作、クラス・モジュール定義、ライブラリ）はそのBox内に閉じる。
- 想定ユースケース：
  - モンキーパッチを伴うテストをBox内に隔離して実行
  - Webアプリをプロセス内・アプリケーションサーバ上でBlue-Greenデプロイ
  - 依存関係更新時に新旧を並行稼働させてレスポンス等を検証
  - 将来の「パッケージAPI」のような高レベルAPIの基盤
- 詳細：[Ruby::Box ドキュメント](https://docs.ruby-lang.org/en/master/Ruby/Box.html)

---

## 8. Socket / ネットワーク

### 3.4: Happy Eyeballs Version 2（RFC 8305）対応
- `TCPSocket.new`（`.open`）と `Socket.tcp` が対応。IPv6/IPv4の名前解決を同時実行し、250ミリ秒間隔でIPv6優先の並行接続を試行、最初に成功した接続を採用。
- デフォルトで有効。無効化は環境変数 `RUBY_TCP_NO_FAST_FALLBACK=1` または `Socket.tcp_fast_fallback=false`、メソッド単位では `fast_fallback: false`。

### 4.0
- `Socket.tcp`／`TCPSocket.new` に接続タイムアウト用 `open_timeout` キーワード引数を追加。
- タイムアウト時の例外を `IO::TimeoutError` に統一（`Socket.tcp` では状況により `Errno::ETIMEDOUT` の場合あり）。

---

## 9. IRB・開発ツール

### 3.3
- IRBとrdbgの連携でpry-byebug風のデバッグ体験。
- `ls`／`show_cmds` の出力がPager表示に対応、より詳細な情報を出力。
- 型情報を使った補完を実験的に実装。
- `Reline::Face` により補完ダイアログの色・装飾をカスタマイズ可能に。

---

## 10. 標準ライブラリの更新（主なもの）

### 3.3
- 将来 bundled gem 化予定の gem（`abbrev`、`base64`、`bigdecimal`、`csv`、`drb`、`getoptlong`、`mutex_m`、`nkf`、`observer`、`racc`、`resolv-replace`、`rinda`、`syslog`）が、Gemfile/gemspecに未記載のままrequireされた際に警告。
- `racc` が default gem から bundled gem に変更。
- `prism 0.19.0` が default gem として追加。

### 3.4
- RubyGems：`gem push --attestation` オプション追加（[sigstore](https://www.sigstore.dev/) への署名情報保存）。
- Bundler：`bundle config` に `lockfile_checksums`、`bundle lock --add-checksums` コマンドを追加。
- JSON：`JSON.parse` が3.3系比で約1.5倍高速化。
- Tempfile：`Tempfile.create(anonymous: true)` で作成後即削除される一時ファイルを作成可能に。
- `win32/sspi.rb` は [ruby/net-http-sspi](https://github.com/ruby/net-http-sspi) に分離。
- `mutex_m`、`getoptlong`、`base64`、`bigdecimal`、`observer`、`abbrev`、`resolv-replace`、`rinda`、`drb`、`nkf`、`syslog`、`csv`、`repl_type_completor` が default gem → bundled gem に変更。

### 4.0
- `ostruct`、`pstore`、`benchmark`、`logger`、`rdoc`、`win32ole`、`irb`、`reline`、`readline`、`fiddle` が bundled gem に昇格（default gemから移行）。
- 新規 default gem として `win32-registry` を追加。
- RubyGems / Bundler が **Version 4** に更新（詳細は[RubyGemsブログ](https://blog.rubygems.org/2025/12/03/upgrade-to-rubygems-bundler-4.html)を参照）。
- `openssl` が 4.0.0 に、`json` が 2.18.0 に、`minitest` が 6.0.0 になるなど、多数のgemがメジャーバージョンアップ。

---

## 11. 互換性に関する変更（非推奨化・削除など）

### 3.3
- ブロック内引数なし `it` の呼び出しが非推奨に（3.4で意味が変わる予告）。
- `ext/readline` を削除。`reline` が全環境で標準採用（`libreadline`/`libedit` 不要に）。以前の `ext/readline` が必要な場合は `readline-ext` gem を利用。

### 3.4
- バックトレース表示形式の変更（引用符がbacktickからシングルクォートに、クラス名がメソッド名の前に表示）。
- `Hash#inspect` の表示変更（Symbolキーはコロン表記、他のキーは `=>` 前後にスペース）。
- `Kernel#Float()`／`String#to_f` が小数部なしの10進表記を受付（例：`Float("1.")` が `1.0` を返すように）。
- `Refinement#refined_class` を削除。
- `DidYouMean::SPELL_CHECKERS[]=`／`.merge!` を削除。
- `Net::HTTP` の2012年から非推奨だった定数群を削除（`ProxyMod` など）。
- `Timeout.timeout` が負の値を受付なくなった。
- `URI` のデフォルトパーサーが RFC 2396 準拠から RFC 3986 準拠に変更。
- `rb_newobj`／`rb_newobj_of` 系マクロ、非推奨だった `rb_gc_force_recycle` をC APIから削除。
- ブロックを使わないメソッドにブロックを渡すとverboseモードで警告。
- JIT/インタプリタが特別最適化するメソッド（`String.freeze`、`Integer#+` 等）の再定義でperformance警告。

### 4.0
- `Ractor::Port` 追加に伴い `Ractor.yield`／`Ractor#take`／`Ractor#close_incoming`／`Ractor#close_outgoing` を削除。
- `ObjectSpace._id2ref` が非推奨に。
- `Process::Status#&`／`#>>` を削除（3.3で非推奨化されたもの）。
- 非推奨だった `rb_path_check` を削除。
- `ArgumentError` のバックトレースに受け手のクラス/モジュール名が付与されるように（例：`bar` → `Foo#bar`）。
- バックトレースに `internal` フレームが表示されなくなり、C実装メソッドもRubyソースファイル上にあるかのように表示。
- **CGI**：default gem から除外され、`cgi/escape` のエスケープ系メソッドのみ提供。
- **Set**：コアクラス化に伴い `set/sorted_set.rb` を削除、`SortedSet` は自動ロード対象外に（利用時は `sorted_set` gem を別途インストール）。
- **Net::HTTP**：POST/PUT等でContent-Type未指定時に `application/x-www-form-urlencoded` を自動設定する挙動を削除。
- **Windows**：MSVC 1900（Visual Studio 2015）未満のサポートを終了。
- **C API**：`rb_thread_fd_close` が非推奨・no-op化（`RUBY_IO_MODE_EXTERNAL`＋`rb_io_close`を推奨）。`rb_thread_call_with_gvl` がGVLの有無に関わらず動作するように。Set用C API（`rb_set_*`系）を新規追加。

---

## 12. パフォーマンス概況（コミット規模）

| リリース | 差分元 | 変更ファイル数 | 追加行 | 削除行 |
|---|---|---|---|---|
| 3.3.0 | 3.2.0 | 5,532 | 326,851 | 185,793 |
| 3.4.0 | 3.3.0 | 4,942 | 202,244 | 255,528 |
| 4.0.0 | 3.4.0 | 3,889 | 230,769 | 297,003 |

---

## 13. リリース基本情報

| バージョン | リリース日 | Posted by | 主なキャッチコピー |
|---|---|---|---|
| 3.3.0 | 2023-12-25 | naruse | Prism / Lrama / RJIT / YJIT高速化 |
| 3.4.0 | 2024-12-25 | naruse | `it` / Prismデフォルト化 / Happy Eyeballs v2 / Modular GC |
| 4.0.0 | 2025-12-25 | naruse | Ruby Box / ZJIT |
