# configuration cache 失敗の原因調査レポート

## 結論
setup-gradle を使った構成で **ジョブ間の configuration cache 再利用に成功**しました。
鍵の形式、キャッシュの保存先、cleanup の挙動がポイントでした。

## 事象（ログ根拠）
- 失敗時: 2ジョブ目が `Configuration cache entry stored.` となり再利用できていない
- 失敗時: `Not saving configuration-cache state, as no encryption key was provided`
- 失敗時: `Invalid AES key length: 48 bytes`
- 失敗時: `.../caches/jars-9/.../dsl has been removed` により再利用不可
- 成功時: 2ジョブ目で `Reusing configuration cache.` / `Configuration cache entry reused.` が出る

## 原因
1) **暗号鍵未設定**
   - setup-gradle の configuration cache 保存がスキップされる
2) **暗号鍵形式ミス**
   - 16進 64文字を渡すと Base64 として解釈され 48 bytes になり失敗
3) **cleanup で参照 jar が削除**
   - `cache-cleanup: on-success` のままでは configuration cache が参照する jar が消える場合がある

## 対応方針（setup-gradle を使う前提）
- **暗号鍵を Base64 で用意して設定**する
  - 32 bytes を Base64 にした値を `GRADLE_ENCRYPTION_KEY` へ登録
- **cleanup を無効化**して参照 jar の削除を防ぐ
  - `cache-cleanup: never`
- **Gradle User Home を actions/cache で触らない**
  - setup-gradle に任せる方針を採用
- **判定は両方許容**
  - `Reusing configuration cache` / `Configuration cache entry reused` のどちらでもOK

## 実施した変更（最終状態）
- setup-gradle に `cache-encryption-key` を設定
- setup-gradle に `cache-cleanup: never` を設定
- actions/cache で `.gradle/configuration-cache` を触るステップを削除
- 2ジョブ目の判定を「reused のどちらの文言でも OK」に変更

## 最終確認結果
- 2ジョブ目で `Reusing configuration cache.` と `Configuration cache entry reused.` を確認
- setup-gradle のみでジョブ間再利用が成立
- doc 変更のみの push でも 1ジョブ目/2ジョブ目ともに再利用を確認

## メモ
- もし再び `.../dsl has been removed` が出る場合は cleanup の設定を最初に疑う

## デメリット・制約（setup-gradle + --configuration-cache）
- **キャッシュ整合性に敏感**: 依存 jar や DSL キャッシュの差異で再利用不可になりやすい
- **暗号鍵に依存**: 鍵の形式/変更でキャッシュ無効化（鍵変更は実質全無効）
- **cleanup 設定のトレードオフ**: `cache-cleanup: never` は安定するがキャッシュ肥大の懸念
- **非対応プラグイン/スクリプト**: configuration cache 非対応だと再利用不可
- **環境差異に弱い**: OS/Java/Gradle の違いで再利用不可になりやすい
- **初回は遅い**: 1回目は保存、効果は2回目以降
- **ログ判定が紛らわしい**: `Reusing configuration cache.` と `Configuration cache entry reused.` の両方が出る

## ワークフローの詳細解説（各ジョブの中身）
### config-cache-store
1) **actions/checkout@v4**
   - リポジトリを取得し、ジョブの作業ディレクトリを用意
2) **actions/setup-java@v4**
   - JDK 17（temurin）をセットアップ
3) **gradle/actions/setup-gradle@v4**
   - Gradle User Home のキャッシュ復元/保存を実施
   - `cache-encryption-key` で configuration cache の暗号化保存を有効化
   - `cache-cleanup: never` で cleanup による参照 jar 削除を防止
4) **./gradlew --version**
   - ランナー上の Gradle/Java 環境の確認
5) **./gradlew --configuration-cache help**
   - configuration cache を使用して `help` タスクを実行
   - ログから `Reusing configuration cache.` または `Configuration cache entry stored.` を判定
6) **post-action**
   - setup-gradle が Gradle User Home をキャッシュ保存（必要な場合）

### config-cache-reuse
1) **actions/checkout@v4**
   - リポジトリを取得
2) **actions/setup-java@v4**
   - JDK 17（temurin）をセットアップ
3) **gradle/actions/setup-gradle@v4**
   - Gradle User Home と configuration cache の復元
   - 鍵と cleanup 設定は store と同様
4) **./gradlew --configuration-cache help**
   - configuration cache の再利用を確認
   - ログの `Reusing configuration cache.` または `Configuration cache entry reused.` を許容
5) **post-action**
   - setup-gradle が Gradle User Home をキャッシュ保存（必要な場合）
