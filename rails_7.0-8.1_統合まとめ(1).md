# Ruby on Rails 7.0 → 8.1 変更点 統合まとめ

出典（Railsガイド 日本語版）：
- [Rails 7.0 リリースノート](https://railsguides.jp/7_0_release_notes.html)（2021/12）
- [Rails 7.1 リリースノート](https://railsguides.jp/7_1_release_notes.html)（2023/10）
- [Rails 7.2 リリースノート](https://railsguides.jp/7_2_release_notes.html)（2024/08）
- [Rails 8.0 リリースノート](https://railsguides.jp/8_0_release_notes.html)（2024/11）
- [Rails 8.1 リリースノート](https://railsguides.jp/8_1_release_notes.html)（2025/10）

---

## 1. 必須Rubyバージョン・全体的な注目ポイント

| Railsバージョン | 必須Rubyバージョン | 主なキャッチコピー |
|---|---|---|
| 7.0 | Ruby 2.7.0以上（3.0以上推奨） | - |
| 7.1 | - | Docker生成、複合主キー、Trilogyアダプタ、非同期クエリAPI |
| 7.2 | **Ruby 3.1以上**（マイナーバージョンでもEOL Rubyを都度切り捨てる方針に変更） | Dev Container、RuboCop omakase、Brakeman、YJITデフォルト化 |
| 8.0 | - | Kamal 2、Solid Trifecta（Cache/Queue/Cable）、Propshaft、認証ジェネレータ |
| 8.1 | - | Active Job継続機能、構造化イベントレポート、ローカルCI |

---

## 2. 新規アプリのデフォルト構成・デプロイ関連

### 7.1: Dockerfile自動生成
- 新規アプリでproduction用のDocker関連ファイルが生成されるように（開発用ではない点に注意）。
- `docker build` → `docker run` の基本フローと、マルチプラットフォームビルド（`docker buildx`）に対応。

### 7.2: 開発体験・CI/セキュリティのデフォルト強化
- **Dev Container**：`rails new myapp --devcontainer` で`.devcontainer/`一式（Redis、DB、ヘッドレスChrome、Active Storage設定）を生成。既存アプリには`rails devcontainer`コマンドを追加。
- **ブラウザバージョン保護**：`allow_browser versions: :modern` などをApplicationControllerにデフォルト設定。対象外ブラウザには406を返す。
- **PWAファイル**：マニフェスト・サービスワーカーを`app/views/pwa`からERBで動的配信するデフォルト構成を追加。
- **RuboCop omakase**：[rubocop-rails-omakase](https://github.com/rails/rubocop-rails-omakase)がデフォルトで導入。
- **GitHub CIワークフロー**：新規アプリにデフォルトで追加（Dependabot・Brakeman・RuboCopを含む）。
- **Brakeman**：新規アプリにデフォルトでインストールされ、CIで自動実行。
- **Puma**：デフォルトスレッド数を5→3に変更（GVL待ちによるレイテンシ悪化を避けるため）。
- **YJIT**：Ruby 3.3以降で実行時にデフォルト有効化（レイテンシ15〜25%改善）。無効化は`Rails.application.config.yjit = false`。
- **jemalloc**：DockerfileにデフォルトでmallocのメモリフラグメンテーションSAF対策として設定。
- `bin/setup`実行時にpuma-devの設定方法を提案するように。
- Rails Guidesのデザインを2009年以来初めて刷新。

### 8.0: Kamal 2 / Solid Trifecta / Propshaft
- **Kamal 2**：デプロイツールがRailsにプリインストール。`kamal setup`一発でLinuxマシンをアプリ/アクセサリサーバー化。従来のTraefikに代わる[Kamal Proxy](https://github.com/basecamp/kamal-proxy)を同梱。
- **Thruster**：PumaサーバーとKamal Proxyの間に配置し、X-Sendfileアクセラレーション、アセットキャッシュ・圧縮を提供する新しいプロキシ。
- **Solid Cable**：RedisなしでAction CableのWebSocketメッセージをpub/sub中継。メッセージはデフォルトで1日DB保持。
- **Solid Cache**：Redis/MemcachedをHTMLフラグメントキャッシュ用途で置き換え。
- **Solid Queue**：Redis＋Resque/Delayed Job/Sidekiqを不要にするジョブキュー。PostgreSQL 9.5〜/MySQL 8.0〜の`FOR UPDATE SKIP LOCKED`を活用。SQLiteでも動作。
- **Propshaft**：Sprocketsに代わる新しいデフォルトのアセットパイプライン。
- **認証システムジェネレータ**：セッションベース・パスワードリセット・メタデータ追跡付き認証を生成。

### 8.1: Kamalのローカルレジストリ対応
- Kamal 2.8で、シンプルなデプロイ時にDocker Hub/GHCRなどのリモートレジストリが不要に（デフォルトでローカルレジストリ使用）。大規模構成では引き続きリモートレジストリが必要。
- Kamalが`.kamal/secrets`内でRailsの暗号化credentialsからsecretsを直接取得できるように（`rails credentials:fetch`）。

---

## 3. Railties（CLI・ジェネレータ・アプリ基盤）

### 7.0
- **削除**：`dbconsole`の非推奨`config`。
- **変更**：`rails` gemが`sprockets-rails`に依存しなくなった（必要なら手動で`gem "sprockets-rails"`追加）。

### 7.1
- **削除**：`bin/rails secrets:setup`（非推奨）、デフォルトの`X-Download-Options`ヘッダー。
- **非推奨化**：`Rails.application.secrets`、`secrets:show`/`secrets:edit`（→`credentials`推奨）、英国スペルの`Testing::Behaviour`（→`Behavior`）。
- **変更**：production用Railsコンソールのデフォルトsandboxモード化オプション`sandbox_by_default`、テスト実行時の行範囲指定構文、`rails new --javascript=bun`対応、遅いテストの表示機能。

### 7.2
- **削除**：`Rails::Generators::Testing::Behaviour`、`Rails.application.secrets`、`Rails.config.enable_dependency_loading`、`find_cmd_and_exec`、`oracle`/`sqlserver`/JRuby固有DBアダプタ、`config.public_file_server.enabled`。
- **変更**：RuboCop omakase・Brakeman・GitHub CIワークフローの標準化（前掲）、`.devcontainer`生成、`assert_initializer`テストヘルパー、システムテストのヘッドレスChromeデフォルト化、`BACKTRACE`環境変数によるバックトレースクリーニング無効化（通常サーバー実行時にも対応）。

### 8.0
- **削除**：非推奨`config.read_encrypted_secrets`、`rails/console/app`・`rails/console/helpers`、`Rails::ConsoleMethods`拡張サポート。
- **非推奨化**：`"rails/console/methods"`のrequire、`STATS_DIRECTORIES`（→`Rails::CodeStatistics.register_directory`）、`bin/rake stats`（→`bin/rails stats`）。
- **変更**：`Regexp.timeout`がデフォルトで`1`に（ReDoS対策）。

### 8.1
- **削除**：`rails/console/methods.rb`、`bin/rake stats`、`STATS_DIRECTORIES`。
- 新機能は「2. 新規アプリのデフォルト構成」および「12. Active Job」節を参照（Active Jobの継続機能、構造化イベントレポート`Rails.event`、ローカルCI DSL `config/ci.rb`、Markdownレンダリング`render markdown:`）。

---

## 4. Action Pack（Action Controller / Action Dispatch / ルーティング）

### 7.0
- **削除**：非推奨の`ActionDispatch::Response.return_only_media_type_on_content_type`、`hosts_response_app`、`SystemTestCase#host!`、`fixture_file_upload`への相対パスサポート。

### 7.1
- **削除**：`Request#content_type`の非推奨挙動、`trusted_proxies`への単一値代入、`poltergeist`/`webkit`ドライバ登録。
- **非推奨化**：`return_only_request_media_type_on_content_type`、`MissingHelperError`、`IllegalStateError`、パーミッションポリシーの`speaker`/`vibrate`/`vr`、`show_exceptions`への`true`/`false`設定（→`:all`/`:rescuable`/`:none`）。
- **変更**：`ActionController::Parameters#exclude?`／`#extract_value`追加、カスタムCSRFトークン保存/取り出し、システムテストスクリーンショットヘルパーに`html`/`screenshot`引数追加。

### 7.2
- **削除**：`IllegalStateError`、`MissingHelperError`、`Parameters`と`Hash`の比較機能、`return_only_request_media_type_on_content_type`、パーミッションポリシー旧ディレクティブ、`show_exceptions`の旧設定値。
- **非推奨化**：`allow_deprecated_parameters_hash_equality`。

### 8.0
- **削除**：`allow_deprecated_parameters_hash_equality`。
- **非推奨化**：複数パスを指定するルーティング（高速化のため）。
- **主要機能**：`params.require(:table).permit(:attr)`をより安全・明示的にした**`params#expect`**を導入（`params.expect(table: [:attr])`）。

### 8.1
- **削除**：パラメータパーサーの先頭ブラケット`[]`スキップサポート、クエリ文字列のセミコロン区切りサポート、複数パスへのルーティング。
- **非推奨化**：`ignore_leading_brackets`。
- **変更**：development環境でのリダイレクトログが詳細化（`verbose_redirect_logs = true`で既存アプリにも適用可）。

---

## 5. Action View

### 7.0
- **削除**：`raise_on_missing_translations`（非推奨）。
- **変更**：`button_to`がURLビルドに使うActive RecordオブジェクトからHTTPメソッドを推論するように（例：更新系オブジェクトでPOSTがPATCHになる）。

### 7.1
- **削除**：`ActionView::Path`定数、パーシャルへのインスタンス変数のローカル変数渡し。
- **変更**：`checkbox_tag`/`radio_button_tag`が`checked:`キーワード引数対応、`picture_tag`追加、`simple_format`の`:sanitize_options`対応。

### 7.2
- **削除**：非推奨の`@rails/ujs`（Turboに置き換え）。
- **非推奨化**：`tag.br`など空要素タグビルダーへのコンテンツ渡し。

### 8.0
- **削除**：`form_with`の`model: nil`サポート、空タグ要素へのコンテンツ渡し（非推奨終了）。

---

## 6. Action Mailer

### 7.0
- **削除**：非推奨の`ActionMailer::DeliveryJob`/`Parameterized::DeliveryJob`（→`MailDeliveryJob`）。

### 7.1
- **非推奨化**：単数形`config.action_mailer.preview_path`、`assert_enqueued_email_with`への`:args`渡し（→`:params`）。
- **変更**：複数プレビューパス対応の`preview_paths`（複数形）追加、`capture_emails`テストヘルパー、`deliver_enqueued_emails`追加。

### 7.2
- **削除**：単数形`preview_path`、`assert_enqueued_email_with`の`:args`。

---

## 7. Action Cable

### 7.1
- `capture_broadcasts`テストヘルパー追加、Redis pub/subアダプタの自動再接続、`before_command`/`after_command`/`around_command`コールバック追加。

（7.0、7.2、8.0、8.1では大きな変更なし）

---

## 8. Active Record

### 7.0
- **削除**：非推奨の`connected_to`の`database`引数、`allow_unsafe_raw_sql`、`configs_for`の`:spec_name`、Rails 4.x形式YAML読み込み、コネクション仕様名`"primary"`解決、`ActiveRecord::Base`オブジェクトの直接引用符化／`type_cast`、`DatabaseConfig#config`、`db:structure:*`等の旧rakeタスク、`reorder(nil).first`の非決定論的順序、`DatabaseTasks`関連の複数メソッド、`Result#map!`/`#collect!`、`#remove_connection`。
- **非推奨化**：`schema_file_type`。
- **主要変更**：トランザクションブロックが期待より早く`return`されるとロールバックされるように。同一カラムへの条件マージが常に後者で置き換わる仕様に統一（`rewhere:true`で明示指定可）。

### 7.1
- **削除**：`legacy_connection_handling`、`Base`の非推奨設定アクセサ、`configs_for`の`:include_replicas`（→`:include_hidden`）、`partial_writes`、`schema_file_type`、PostgreSQL structure dumpの`--no-comments`。
- **非推奨化**：`#remove_connection`の`name`引数、`check_pending!`（→`check_all_pending!`）、`add_foreign_key`の`deferrable: true`（→`:immediate`）、単数形`fixture_path`（→`fixture_paths`）、`Base`→`connection_handler`委譲、`suppress_multiple_database_warning`、SQL文字列内での`ActiveSupport::Duration`式展開、`all_connection_pools`、非`:id`主キーで`read_attribute(:id)`が主キーを返す挙動、`#merge`の`rewhere`オプション、非属性の`alias_attribute`。
- **主要機能**：**複合主キー**のネイティブサポート（`query_constraints`マクロ、関連付けの`query_constraints:`）、**Trilogyアダプタ**導入、`has_secure_password`に`authenticate_by`追加、`insert_all`/`upsert_all`でエイリアス属性対応、マイグレーション`add_index`の`:include`オプション（PostgreSQL）、`#regroup`、SQLite3の自動生成カラム/カスタム主キー対応、`where`のタプル構文、自動生成インデックス名を最大62バイトに、`ActiveRecord.disconnect_all!`、PostgreSQL enumのリネーム/値追加、`#id_value`、`enum`の`validate`オプション、暗号化のデフォルトハッシュダイジェストをSHA1→SHA256に変更。

### 7.2
- **削除**：`suppress_multiple_database_warning`、存在しない属性への`alias_attribute`、`remove_connection`の`name`引数、`clear_active_connections!`系複数メソッド、`ActiveJobRequiredError`、2引数`explain`、`LogSubscriber.runtime`系、`Migration.check_pending!`、`MigrationContext`への旧クラス引数、単数形関連付け参照、`fixture_path`（単数形）、`read_attribute(:id)`のカスタムPK返却、`serialize`への第2引数コーダー、`#all_foreign_keys_valid?`、`SchemaCache.load_from`/`#data_sources`、`#all_connection_pools`、role省略時の複数コネクション管理メソッド、`#connection_klass`、`#quote_bound_value`、`ActiveSupport::Duration`の無変換式展開、`add_foreign_key`の`deferrable: true`、`#merge`の`rewhere`、トランザクションブロックの`return`/`break`/`throw`によるロールバック（非推奨終了）。
- **非推奨化**：`allow_deprecated_singular_associations_name`、`commit_transaction_on_non_local_return`。
- **変更**：`establish_connection`が`connection.active?`を`true`にしなくなった（必要なら`connection.verify!`）。

### 8.0
- **削除**：`commit_transaction_on_non_local_return`、`allow_deprecated_singular_associations_name`、未登録DB探索サポート、キーワード引数での`enum`定義、`warn_on_records_fetched_greater_than`、`sqlite3_deprecated_warning`、`ConnectionPool#connection`（非推奨版）、`cache_dump_filename`へのDB名渡し、`ENV["SCHEMA_CACHE"]`。
- **非推奨化**：SQLite3Adapterの`retries`オプション（→`timeout`）。
- **主要変更**：新規DBで`db:migrate`実行時にマイグレーション前にスキーマを読み込むように（従来の挙動は`db:migrate:reset`、DBをドロップ＆再作成する点に注意）。

### 8.1
- **削除**：SQLite3Adapterの非推奨`:retries`、MySQL用`:unsigned_float`/`:unsigned_decimal`。
- **非推奨化**：`order`未指定での順序依存finder（`#first`等）、`signed_id_verifier_secret`（→`message_verifiers`）、未永続化レコードを含む関連付けでの`insert_all`/`upsert_all`、`update_all`での`WITH`/`WITH RECURSIVE`/`DISTINCT`併用。
- **主要変更**：`schema.rb`のカラムソート順がアルファベット順に。**関連付けの非推奨化機能**（`has_many :posts, deprecated: true`）を追加、`:warn`/`:raise`/`:notify`の3モード対応。

---

## 9. Active Storage

### 7.0
特になし。

### 7.1
- **削除**：無効なデフォルトContent-Type（非推奨）、`Current#host`/`#host=`、添付コレクション代入時の追加的挙動（→置き換えに統一）、関連付けからの`purge`/`purge_later`。
- **変更**：`AudioAnalyzer`の`sample_rate`/`tags`出力対応、`preview`/`representation`での定義済みvariant利用、`preprocessed`オプション、variant削除機能追加。

### 7.2
- **削除**：`replace_on_assign_to_many`、`silence_invalid_content_types_warning`（いずれも非推奨）。

### 8.0
- **非推奨化**：Azureバックエンド。

### 8.1
- **削除**：非推奨化されていた`:azure`ストレージサービス。

---

## 10. Active Model

### 7.0
- **削除**：`Errors`のハッシュ列挙、`#to_h`/`#slice!`/`#values`/`#keys`/`#to_xml`（いずれも非推奨）、`#messages`関連の複数の非推奨機能、Rails 5.x形式のMarshal/YAML読み込みサポート。

### 7.1
- **変更**：`LengthValidator`のbeginless/endless range対応、`validates_inclusion_of`/`validates_exclusion_of`のbeginless range対応、`has_secure_password`に`password_challenge`追加、バリデータのlambdaで`record`引数省略可能に。

（7.2〜8.1では大きな変更なし）

---

## 11. Active Support

### 7.0
- **削除**：`use_sha1_digests`、`URI.parser`（非推奨）、`Range#include?`による日時range判定、`Multibyte::Unicode.default_normalization_form`。
- **非推奨化**：`Array`/`Range`/`Date`/`DateTime`/`Time`/`BigDecimal`/`Float`/`Integer`の`#to_s`へのフォーマット引数渡し（→`#to_fs`、Ruby 3.1の式展開最適化を活かすため）。

### 7.1
- **削除**：`Enumerable#sum`の非推奨オーバーライド、`PerThreadRegistry`、`#to_s`のフォーマット渡し、`TimeWithZone.name`オーバーライド、`core_ext/uri`、`core_ext/range/include_time_with_zone`、`SafeBuffer`の暗黙String変換、`Digest::UUID`の不正な名前空間IDサポート。
- **非推奨化**：`disable_to_s_conversion`、`remove_deprecated_time_with_zone_name`、`use_rfc4122_namespaced_uuids`、`SafeBuffer#clone_empty`、`ActiveSupport::Deprecation`のシングルトン利用、`MemCacheStore`への`Dalli::Client`インスタンス渡し、`Notification::Event#children`/`#parent_of?`。

### 7.2
- **削除**：`Notifications::Event#children`/`#parent_of?`、deprecatorなしでの`deprecate`系メソッド呼び出し、`Deprecation`インスタンスへの委譲、`SafeBuffer#clone_empty`、`#to_default_s`、キャッシュの`:pool_size`/`:pool_timeout`、`cache_format_version = 6.1`、`LogSubscriber::CLEAR`/`BOLD`定数、`#color`の位置引数ブーリアン、`disable_to_s_conversion`、`remove_deprecated_time_with_zone_name`、`use_rfc4122_namespaced_uuids`、`MemCacheStore`への`Dalli::Client`渡し。

### 8.0
- **削除**：`ProxyObject`（非推奨）、`attr_internal_naming_format`への`@`プレフィックス、`Deprecation#warn`への文字列配列渡し。
- **非推奨化**：`Benchmark.ms`、加算/`since`での`Time`と`TimeWithZone`混在。

### 8.1
- **削除**：`Time#since`へのTimeオブジェクト渡し、`Benchmark.ms`（→`benchmark` gem）、`Time`と`TimeWithZone`の加算、`to_time`のローカル時間保持（→常にレシーバーのタイムゾーンを保持）。
- **非推奨化**：`to_time_preserves_timezone`、`ActiveSupport::Configurable`。
- **主要機能**：**構造化イベントレポート**（`Rails.event.notify`/`.tagged`/`.set_context`、サブスクライバの`#emit`）。詳細は「2. 新規アプリのデフォルト構成」参照。

---

## 12. Active Job

### 7.0
- **削除**：`throw :abort`時の`after_enqueue`/`after_perform`非停止挙動、`:return_false_on_aborted_enqueue`。
- **非推奨化**：`skip_after_callbacks_if_terminated`。

### 7.1
- **削除**：`QueAdapter`。
- **主要機能**：**`perform_all_later`**（複数ジョブの一括エンキュー、`enqueue_all.active_job`イベント）、ジョブジェネレータの`--parent`オプション、`after_discard`コールバック、`verbose_enqueue_logs`設定。

### 7.2
- **削除**：`BigDecimal`引数の非推奨シリアライザ、`scheduled_at`への数値設定、`retry_on`の`:exponentially_longer`。
- **非推奨化**：`use_big_decimal_serializer`。
- **主要機能**：**ジョブがトランザクション内でスケジューリングされないように**（トランザクションコミット完了までエンキューを自動延期、ロールバック時は削除。`enqueue_after_transaction_commit`で制御可）。**トランザクションごとのコミット/ロールバックコールバック**（`transaction.after_commit`、`current_transaction`、`ActiveRecord.after_all_transactions_commit`）※Active Record機能だがActive Jobと密接に関連。

### 8.0
- **削除**：`use_big_decimal_serializer`。
- **非推奨化**：`enqueue_after_transaction_commit`、組み込み`SuckerPunch`アダプタ（→`sucker_punch` gem）。

### 8.1
- **削除**：`enqueue_after_transaction_commit`への`:never`/`:always`/`:default`設定、`Rails.application.config.active_job.enqueue_after_transaction_commit`、組み込み`SuckerPunch`アダプタ。
- **非推奨化**：カスタムシリアライザに`#klass`publicメソッドが必須に。
- **主要機能**：**Active Jobの継続機能**（`ActiveJob::Continuable`、`step`によるステップ分割・再開。Kamalのデプロイ時シャットダウン猶予対策として有用）。

---

## 13. Action Text / Action Mailbox

### Action Text
- 7.0〜8.1を通じて大きな変更なし。

### Action Mailbox
- **7.0**：非推奨の`credentials.action_mailbox.mailgun_api_key`、環境変数`MAILGUN_INGRESS_API_KEY`を削除。
- **7.1**：`Mail::Message#recipients`に`X-Forwarded-To`アドレス追加、`bounce_now_with`メソッド追加（メーラーキューを介さないバウンス送信）。

---

## 14. まとめ：バージョンごとの一言サマリ

| バージョン | 一言でいうと |
|---|---|
| 7.0 | 非推奨機能の大掃除＋トランザクション/マージ挙動の厳密化 |
| 7.1 | 複合主キー・Docker・MessagePack・非同期クエリなど「本格的な新機能」ラッシュ |
| 7.2 | 新規アプリの「デフォルトで安全・モダン」化（CI、Brakeman、RuboCop、YJIT） |
| 8.0 | 自前ホスティングを強力に支援する「Solid Trifecta」＋Kamal 2 |
| 8.1 | 運用・可観測性・ジョブの堅牢性を高める機能群（継続ジョブ、構造化イベント、ローカルCI） |
