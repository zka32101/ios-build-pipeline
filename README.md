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
| `okane_kore` | `zka32101/kinnyu` | 配布証明書+プロファイルあり |
| `nihon_future_map` | `zka32101/seisaku_tohyo_map`（private） | ユーザー確認済み。`geography_puzzle_king`内`apps/nihon_future_map`は不採用とした。privateリポジトリのため`APPS_READ_PAT`が必須 |

### 要確認（`cert_status: "needs_confirmation"`。確認が取れるまで自動スキップ）

| アプリキー | 内容 | 必要な確認 |
|---|---|---|
| `kokugo-kore` | provisioning profileのみ、.p12が無い | `geography_puzzle_king` の配布証明書を共用して良いか。両アプリとも Team ID `6UWJGP52W5` で一致しており、`kokugo-kore` の `ExportOptions.plist` は証明書を `Apple Distribution`（総称）で指定しているため技術的には共用可能な可能性が高いが、`.p12` のTeam IDは復号しないと確認できないため最終確認が必要 |
| `japan_explorer` | provisioning profileのみ、.p12が無い | 同上（Team ID `6UWJGP52W5` で一致） |
| `kotoba-e` | .p12のみ、provisioning profileが無い。さらに `ios/ExportOptions.plist` 自体がリポジトリに存在しないことを確認済み（他5アプリには全て存在） | 配布用provisioning profileの新規発行、および `ExportOptions.plist` の新規作成が必要 |

### 証明書が未発行（対象外。Apple Developer Programでの新規発行が必要）

`apps.json` の `pending_apps_no_certificate` に一覧化（リポジトリ・Bundle ID付き）。
配布用証明書・provisioning profileの発行はユーザー本人のApple Developer Programでの
操作が必須のため、発行後に `ios-certs-vault` へ追加 → `apps.json` の `apps` 配列へ昇格、
という流れになる。

| アプリキー | 内容 | 備考 |
|---|---|---|
| `sansu-kore` | 小学コレ！算数 | 証明書一式が未作成 |
| `shogaku-kore-programming` | 小学コレ！プログラミング | 証明書一式が未作成 |
| `shinshin` | 小学コレ！道徳 | ⚠️ private repo `shougaku-kore-shinshin` と pubspec name・Bundle IDが完全一致する複製が存在。**どちらを正とするか要ユーザー確認**（`nihon_future_map`と同様のパターン） |
| `shougaku-kore-shinshin` | 小学コレ！道徳（複製repo候補） | 上記と同一アプリの複製。一本化が必要 |
| `social_quiz_app` | 小学コレ！社会（推定） | Bundle ID から推定。証明書一式が未作成 |
| `kanken` | 小学コレ！漢検 | README記載で確認済み。**`ios/`ディレクトリ自体が存在せず、iOSターゲット未作成**。証明書以前にXcodeプロジェクトの作成が必要 |
| `eigo` | 英語コレ！ | 小学コレ7アプリの正式メンバーか未確認（独立ブランド名・別Bundle ID名前空間）。証明書一式が未作成 |

## 必要な GitHub Secrets（このリポジトリに設定）

| Secret名 | 内容 | 用意する人 |
|---|---|---|
| `CERT_PASSPHRASE` | `ios-certs-vault` の復号パスフレーズ | ユーザー（パスワードマネージャー等で既に保管済みのはず） |
| `CERTS_VAULT_PAT` | `zka32101/ios-certs-vault`（private）を読み取るためのPAT（`contents:read`で十分。Fine-grained PAT推奨） | ユーザー |
| `APPS_READ_PAT` | `zka32101/seisaku_tohyo_map`（`nihon_future_map`、private）を読み取るためのPAT。`CERTS_VAULT_PAT`と同様にFine-grained PAT・`contents:read`で可 | ユーザー（必須。未設定だと`nihon_future_map`のみ自動スキップされる） |
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
   - 確認済み: 既存の `apk` フォルダ（owner: `funvestment1@gmail.com`）のフォルダIDは
     `1iXxOO750AyuIwXNwGF-qTJLmElIldgwD`。このフォルダが対象で間違いなければ、
     `GDRIVE_APK_FOLDER_ID` にはこの値をそのまま登録できる。
5. `gdrive-sa.json` の内容を `GDRIVE_SA_KEY_JSON` に、フォルダIDを `GDRIVE_APK_FOLDER_ID` に
   GitHub Secretsとして登録する（このセッションの `gh` CLI・GitHub連携はSecrets書き込み権限を
   持たないため、登録はユーザー自身がGitHub上で行う必要がある）

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
