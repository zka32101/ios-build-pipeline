# ios-build-pipeline

Petit Works Apps 群の iOS ビルドを一箇所に集約する GitHub Actions パイプライン。

`workflow_dispatch`（手動実行）で起動し、`apps.json` に登録されたアプリを
**1つずつ順番に**（並行実行なし）iOS ビルドし、できた `.ipa` だけを
Google Drive の `apk` フォルダへアップロードする。

- ワークフロー本体: [`.github/workflows/build-ios-apps.yml`](.github/workflows/build-ios-apps.yml)
- アプリ一覧・証明書情報: [`apps.json`](apps.json)

macOS ランナーは GitHub Actions 上で10倍課金になるため、このワークフローは
`workflow_dispatch` 以外では絶対に自動起動しない。

## 現在のステータス（2026-10-01時点）

### ビルド対象（証明書確認済み・`cert_status: "ready"`）

| アプリキー | リポジトリ | 備考 |
|---|---|---|
| `geography_puzzle_king` | `zka32101/geography_puzzle_king`（モノレポ `apps/geography_puzzle_king`） | 配布証明書+プロファイルあり |
| `nihon_future_map` | `zka32101/geography_puzzle_king`（モノレポ `apps/nihon_future_map`） | ⚠️要確認（下記） |
| `okane_kore` | `zka32101/kinnyu` | 配布証明書+プロファイルあり |

### 要確認（`cert_status: "needs_confirmation"`。確認が取れるまで自動スキップ）

| アプリキー | 内容 | 必要な確認 |
|---|---|---|
| `kokugo-kore` | provisioning profileのみ、.p12が無い | `geography_puzzle_king` の配布証明書を共用して良いか |
| `japan_explorer` | provisioning profileのみ、.p12が無い | 同上 |
| `kotoba-e` | .p12のみ、provisioning profileが無い | 配布用provisioning profileの新規発行が必要 |
| `nihon_future_map` | リポジトリ候補が2つ存在 | `zka32101/geography_puzzle_king` 内 `apps/nihon_future_map`（公開・CI実績あり）を採用した。`zka32101/seisaku_tohyo_map`（非公開）が正しい場合は `apps.json` の `repo`/`app_subdir`/`private` を修正する必要あり |

### 証明書が未発行（対象外。Apple Developer Programでの新規発行が必要）

`sansu-kore`, `eigo`, `social_quiz_app`, `shogaku-kore-programming`,
`shinshin` / `shougaku-kore-shinshin`, `kanken` ほか小学コレ・Flutterアプリ全般。
配布用証明書・provisioning profileの発行はユーザー本人のApple Developer Programでの
操作が必須のため、発行後に `ios-certs-vault` へ追加 → `apps.json` に登録、という流れになる。

## 必要な GitHub Secrets（このリポジトリに設定）

| Secret名 | 内容 | 用意する人 |
|---|---|---|
| `CERT_PASSPHRASE` | `ios-certs-vault` の復号パスフレーズ | ユーザー（パスワードマネージャー等で既に保管済みのはず） |
| `CERTS_VAULT_PAT` | `zka32101/ios-certs-vault`（private）を読み取るためのPAT（`contents:read`で十分。Fine-grained PAT推奨） | ユーザー |
| `APPS_READ_PAT` | 対象アプリに private リポジトリがある場合の読み取り用PAT（例: `seisaku_tohyo_map`を使う場合） | ユーザー（全アプリが public ならスキップ可） |
| `IOS_P12_PASSWORD` | `.p12`（配布証明書）自体のPKCS12パスワード | ユーザー（vaultのREADMEには含まれていないため別途確認が必要） |
| `GDRIVE_SA_KEY_JSON` | Google Drive用サービスアカウントキー(JSON)の内容そのもの | ユーザー（下記手順） |
| `GDRIVE_APK_FOLDER_ID` | アップロード先 `apk` フォルダの Google Drive フォルダID | ユーザー |

### Google Drive サービスアカウントの準備手順（ユーザー操作が必要）

既存の `shared_core/infrastructure/scripts/add-new-app.sh` はGCP/Firebase用の
サービスアカウント自動化であり、Google Driveへの共有権限付与は含まれていない
（確認済み）。そのため以下は別途手動セットアップが必要:

1. GCPプロジェクトで Google Drive API を有効化する
2. サービスアカウントを新規作成し、JSONキーを発行する
   ```
   gcloud iam service-accounts create ios-build-drive-uploader \
     --display-name="iOS Build Pipeline - Drive Uploader"
   gcloud iam service-accounts keys create gdrive-sa.json \
     --iam-account=ios-build-drive-uploader@<PROJECT_ID>.iam.gserviceaccount.com
   ```
3. Google Drive側で、アップロード先の `apk` フォルダ（既存のWindowsローカルビルド運用で
   使っている `マイドライブ\apk\` と同じフォルダ）を、上記サービスアカウントの
   メールアドレス（`...@<PROJECT_ID>.iam.gserviceaccount.com`）に**編集者**権限で共有する
4. そのフォルダのURLからフォルダIDを取得する（`https://drive.google.com/drive/folders/<ここがフォルダID>`）
5. `gdrive-sa.json` の内容を `GDRIVE_SA_KEY_JSON` に、フォルダIDを `GDRIVE_APK_FOLDER_ID` に
   GitHub Secretsとして登録する

## 使い方

1. GitHub の Actions タブ → `Build iOS Apps (sequential)` → `Run workflow`
2. `apps` 入力に対象アプリキー（`apps.json`の`key`）をカンマ区切りで指定、
   または `all`（デフォルト）で `cert_status: "ready"` の全アプリを対象にする
3. 実行結果を確認し、`.ipa` が Google Drive の `apk` フォルダに追加されたことを確認する

実際にiOSビルドが通るかはmacOS/Xcodeが必要なためこの環境では検証できていない。
初回実行では、証明書・パスワード以外にも各アプリ固有の設定ファイル
（`GoogleService-Info.plist` 等、Gitで管理されていない機密ファイル）が
不足してビルドが失敗する可能性がある。失敗した場合はActionsのログを元に
アプリごとに追加のSecret/ステップを追加していく想定。
