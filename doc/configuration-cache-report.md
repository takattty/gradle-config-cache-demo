# configuration cache 失敗の原因調査レポート

## 結論
setup-gradle を使った現在の構成では、**configuration cache の保存先が Gradle User Home ではなくプロジェクト配下**にあるため、
ジョブ間でキャッシュが引き継がれていません。さらにログ上、**configuration-cache 状態の保存が暗号鍵不足でスキップ**されています。
その結果、2ジョブ目で `Reusing configuration cache.` が出ず失敗しています。

## 事象（ログ根拠）
- 2ジョブ目で `Configuration cache entry stored.` が出ており、再利用ではなく**新規作成**になっている
- setup-gradle の後処理で **`Not saving configuration-cache state, as no encryption key was provided`** が出ている

## 原因
1) **configuration cache の保存場所の問題**
   - configuration cache はプロジェクト配下の `.gradle/configuration-cache` に保存されるため、
     setup-gradle がキャッシュ対象にする **Gradle User Home（`~/.gradle`）だけでは引き継げない**
2) **暗号鍵が未設定**
   - setup-gradle のログに「暗号鍵がないので configuration-cache を保存しない」と出ている

## 対応方針（setup-gradle を使う前提）
- **暗号鍵を設定して configuration cache の保存を有効化**する
  - 例: GitHub Secrets に暗号鍵を用意し、workflow に渡す
  - setup-gradle 側の「configuration-cache の保存」を有効にする（アクションの設定に従う）
- **configuration cache の保存先が Gradle User Home ではない点を補う**
  - setup-gradle が configuration-cache の保存をサポートする場合はその機能を使用
  - サポートしない場合は、プロジェクト配下 `.gradle/configuration-cache` をジョブ間で引き継ぐ仕組みが必要

## 次のアクション案
1) `GRADLE_ENCRYPTION_KEY` を GitHub Secrets に追加
2) workflow の setup-gradle に `cache-encryption-key` を指定
3) 再度2ジョブ実行して `Reusing configuration cache.` が出ることを確認
