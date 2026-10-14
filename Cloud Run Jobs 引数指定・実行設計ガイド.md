# Cloud Run Jobs 引数指定・実行設計ガイド

Cloud Run Jobs において「引数（パラメータ）を切り替えてジョブを実行する」ための設計パターン、各パターンのメリット・デメリット、および外部サービス（Cloud Scheduler / Cloud Workflows 等）との連携方法を包括的に解説します。

---

## 1. はじめに：Cloud Run Jobs の引数の仕組み

Cloud Run Jobs では、コンテナイメージの `ENTRYPOINT`（実行コマンド）に対して渡す引数（`CMD` / `args`）を定義できます。

- **ジョブ定義時の引数（デフォルト引数）**: `gcloud run jobs create` や `update` の `--args` で設定。
- **実行時の引数（オーバーライド引数）**: `gcloud run jobs execute` の `--args` で実行の都度、デフォルト引数を上書き（オーバーライド）。

これにより、以下の2つのアプローチが選択肢となります。

【パターン1: ジョブ定義分割方式】
```
 業務・引数ごとにジョブ定義を分離
 ┌───────────────────────────┐      ┌───────────────────────────┐
 │ job-report-branch1        │      │ job-report-branch2        │
 │ (args: ["report", "b1"])  │      │ (args: ["report", "b2"])  │
 └─────────────┬─────────────┘      └─────────────┬─────────────┘
               ▼ 実行                             ▼ 実行
      [ Task (branch1) ]                 [ Task (branch2) ]
```

【パターン2: 実行時オーバーライド方式】
```
 ジョブ定義は1つに集約し、起動時に引数を注入
               ┌───────────────────────────┐
               │ run-job                   │
               │ (汎用ジョブ定義)          │
               └─────────────┬─────────────┘
                             │
        ┌────────────────────┴────────────────────┐
        ▼ execute (--args="report,branch1")       ▼ execute (--args="report,branch2")
 [ Task (branch1) ]                        [ Task (branch2) ]
```

---

## 2. 2つのパターンの実行コマンドと概要

### パターン1：ジョブ定義分割方式（ジョブの定義に引数を含めて引数分のジョブを作成する場合）

各引数パターンごとに個別の Cloud Run Job リソースを作成・管理します。

```bash
# 1. ジョブの作成（引数をジョブ定義に埋め込む）
gcloud run jobs create run-job-report-branch1 \
  --image=asia-northeast1-docker.pkg.dev/my-project/my-repo/batch-app:latest \
  --region=asia-northeast1 \
  --args="report,branch1"

gcloud run jobs create run-job-report-branch2 \
  --image=asia-northeast1-docker.pkg.dev/my-project/my-repo/batch-app:latest \
  --region=asia-northeast1 \
  --args="report,branch2"

# 2. 定義の更新時（イメージ更新時は全ジョブ更新が必要）
gcloud run jobs update run-job-report-branch1 --image=...:v2 --region=asia-northeast1
gcloud run jobs update run-job-report-branch2 --image=...:v2 --region=asia-northeast1

# 3. 実行（引数の指定は不要）
gcloud run jobs execute run-job-report-branch1 --region=asia-northeast1
gcloud run jobs execute run-job-report-branch2 --region=asia-northeast1
```

### パターン2：実行時オーバーライド方式（ジョブの実行時に引数を指定する場合）

ジョブ定義は汎用的な1本とし、実行コマンド時に `--args` を指定します。

```bash
# 1. ジョブの作成（共通定義を1つだけ作成）
gcloud run jobs create run-job \
  --image=asia-northeast1-docker.pkg.dev/my-project/my-repo/batch-app:latest \
  --region=asia-northeast1

# 2. 定義の更新時（更新対象は1つのみ）
gcloud run jobs update run-job --image=...:v2 --region=asia-northeast1

# 3. 実行（引数を都度指定）
gcloud run jobs execute run-job --region=asia-northeast1 --args="report,branch1"
gcloud run jobs execute run-job --region=asia-northeast1 --args="report,branch2"
```

> [!TIP]
> **gcloud CLI での引数の区切り方について**
> `--args` にカンマ区切り（`--args="report,branch1"`）で渡すと、コンテナ内には `["report", "branch1"]` という2つの独立した引数配列として渡されます。引数自体にスペースやカンマを含めたい場合は、エスケープ（`^,^`）や個別のフラグ指定ルールに留意してください。

---

## 3. メリット・デメリットの詳細比較

| 評価軸 | パターン1: ジョブ定義分割方式 | パターン2: 実行時オーバーライド方式 |
| :--- | :--- | :--- |
| **リソース管理・数** | ❌ **引数パターン数分ジョブが増加**<br>数が多いと一覧が埋まる | ⭕ **ジョブ定義は1つのみ**<br>リソース構成が極めてシンプル |
| **コンテナ更新・保守性** | ❌ **変更の手間が大きい**<br>全ジョブに `update` を実行する必要がある | ⭕ **メンテナンスが容易**<br>1つのジョブを更新するだけで全引数に反映 |
| **監視・ログの可視性** | ⭕ **可視性が非常に高い**<br>ジョブ名単位でダッシュボード、メトリクス、アラートが分離可能 | ⚠️ **ログ集約型**<br>同じジョブの履歴に並ぶため、ログ内や実行詳細で引数を確認する必要がある |
| **コンピュート資源の最適化** | ⭕ **引数ごとに最適化可能**<br>branch1は4GiB/2CPU、branch2は512MiB/1CPUのように設定可能 | ❌ **一律のリソース設定**<br>最大負荷の引数にCPU/メモリ/タイムアウトを合わせる必要がある |
| **動的パラメータの対応** | ❌ **対応不可**<br>「処理日（20261009）」「動的なID」等は事前定義できない | ⭕ **完全対応**<br>実行時に任意の日付や動的変数を渡せる |
| **Cloud Scheduler 連携** | ⭕ **設定が簡単・安全**<br>「ジョブ名を選択してキック」するだけでミスが起きない | ⚠️ **HTTP POST 設定が必要**<br>overrides JSON の構築が必要（後述） |
| **IAM 権限管理** | ⭕ **きめ細かい権限分離が可能**<br>「branch1ジョブのみ実行可能」なサービスアカウントを定義可 | ❌ **個別制御は困難**<br>ジョブ自体の実行権限ですべての引数が実行可能になる |
| **引数ミスのリスク** | ⭕ **ミスが発生しない**<br>ジョブ定義に引数が固定されているため安全 | ⚠️ **呼出元のタイポリスク**<br>スケジューラやスクリプト側で引数指定を誤る可能性がある |

---

## 4. どちらを採用すべきかの判断基準

### パターン1（ジョブ定義分割）が推奨されるケース
1. **対象パターンが少数（例: 2〜5個程度）で固定されている場合**
   - 例: 「日次集計（daily）」と「月次集計（monthly）」
   - 例: 「本番環境同期（prod）」と「ステージング同期（stg）」
2. **引数ごとに要求スペックや所要時間が極端に異なる場合**
   - branch1 はデータ量が膨大でメモリ 8GiB・タイムアウト 1時間が必要だが、branch2 は数分・512MiB で終わる場合。リソースを最適化してコストを削減できます。
3. **Cloud Monitoring で独立したアラート・ダッシュボードを構築したい場合**
   - 「branch1 のジョブが失敗した時のみ担当チームAにSlack通知する」といった運用が標準機能で容易に組めます。
4. **Cloud Scheduler の標準GUIから最小限の設定で定期実行したい場合**

### パターン2（実行時オーバーライド）が推奨されるケース
1. **引数の種類が多い、または動的に変化する場合**
   - 例: 全国50箇所の支店コード（branch01〜branch50）、顧客ID、ファイルパスなど。
   - 例: 処理対象日付（`--args="--target-date=2026-10-09"`）など、日次で変わる引数。
2. **CI/CD パイプラインでのデプロイコストを抑えたい場合**
   - イメージ更新時に1つのジョブのみを更新すればよいため、デプロイ時間が短縮され、デプロイ漏れのリスクがなくなります。
3. **外部ワークフロー（Cloud Workflows, Airflow / Cloud Composer, Eventarc）と連携する場合**
   - 前段の処理結果（ファイル名やレコード数）を実行時パラメータとしてジョブに引き渡すアーキテクチャに最適です。

---

## 5. 実践：外部オーケストレーションからの引数指定実行

### A. Cloud Scheduler からの実行

#### パターン1の場合（定義済みジョブのキック）
Cloud Scheduler のターゲットタイプに「Cloud Run」を選択し、ジョブ名を指定するだけで設定できます。
```bash
# gcloud での登録例
gcloud scheduler jobs create http schedule-branch1 \
  --location=asia-northeast1 \
  --schedule="0 2 * * *" \
  --uri="https://asia-northeast1-run.googleapis.com/v2/projects/PROJECT_ID/locations/asia-northeast1/jobs/run-job-report-branch1:run" \
  --http-method=POST \
  --oauth-service-account-email=scheduler-sa@PROJECT_ID.iam.gserviceaccount.com
```

#### パターン2の場合（実行時引数のオーバーライド）
Cloud Run API v2 の `runJob` エンドポイントに対し、リクエストボディで `overrides` を指定します。
```bash
# overrides JSON を含めてスケジューラを登録
gcloud scheduler jobs create http schedule-branch1-override \
  --location=asia-northeast1 \
  --schedule="0 2 * * *" \
  --uri="https://asia-northeast1-run.googleapis.com/v2/projects/PROJECT_ID/locations/asia-northeast1/jobs/run-job:run" \
  --http-method=POST \
  --message-body='{"overrides": {"containerOverrides": [{"args": ["report", "branch1"]}]}}' \
  --headers="Content-Type=application/json" \
  --oauth-service-account-email=scheduler-sa@PROJECT_ID.iam.gserviceaccount.com
```

### B. Cloud Workflows からの引数指定実行（パターン2向け）

Cloud Workflows を利用すると、引数を動的に計算してジョブを実行できます。

```yaml
main:
  params: [input]
  steps:
    - init:
        assign:
          - branchName: ${input.branch} # 引数として受け取った値
          - targetDate: ${time.format(sys.now(), "2006-01-02")}
    - runJob:
        call: googleapis.run.v2.projects.locations.jobs.run
        args:
          name: ${"projects/" + sys.get_env("GOOGLE_CLOUD_PROJECT_NUMBER") + "/locations/asia-northeast1/jobs/run-job"}
          body:
            overrides:
              containerOverrides:
                - args:
                    - "report"
                    - ${branchName}
                    - ${"--date=" + targetDate}
        result: jobExecution
    - returnResult:
        return: ${jobExecution}
```

---

## 6. 発展：環境変数オーバーライドと並列タスク

引数を切り替える手段は `--args` だけではありません。用途に応じて以下の方式も検討できます。

### 1. 環境変数によるオーバーライド（`--update-env-vars`）
引数（args）の順序依存を避け、キー＆バリューで安全にパラメータを渡したい場合に適しています。
```bash
gcloud run jobs execute run-job \
  --region=asia-northeast1 \
  --update-env-vars="COMMAND=report,TARGET_BRANCH=branch1"
```

### 2. 並列タスク（Tasks）と `CLOUD_RUN_TASK_INDEX`
支店（branch1〜branch10）のレポート作成を**同時に並列分散処理**したい場合、ジョブを10回呼ぶのではなく、1回のジョブ実行で並行タスク数を指定するのが最も効率的です。

```bash
# 1回のジョブ実行で10個のタスク（インデックス 0〜9）を並列起動
gcloud run jobs execute run-job-batch --region=asia-northeast1 --tasks=10
```

コンテナアプリケーション内部では、Cloud Run Jobs が自動注入する環境変数を参照します：
- `CLOUD_RUN_TASK_INDEX`: 0 から始まるタスク番号（0〜9）
- `CLOUD_RUN_TASK_COUNT`: 総タスク数（10）

```python
# アプリケーションコード例 (Python)
import os

task_index = int(os.environ.get("CLOUD_RUN_TASK_INDEX", 0))
branches = ["branch1", "branch2", "branch3", ..., "branch10"]

target_branch = branches[task_index]
print(f"Processing report for {target_branch}")
```

---

## 7. まとめと推奨構成

| 状況 | 推奨パターン | 理由 |
| :--- | :--- | :--- |
| **定例バッチ・引数が数個で固定** | **パターン1（定義分割）** | 設定がシンプルで、Cloud Scheduler やコンソール監視が容易 |
| **支店・顧客などパラメータが多数** | **パターン2（実行時指定）** | ジョブ定義の爆発を防ぎ、イメージ更新等の保守性を最大化 |
| **日付など動的な値を渡したい** | **パターン2（実行時指定）** | 事前定義が不可能なため、Workflows 等から動的注入 |
| **全支店を一括で同時並行処理したい** | **並列タスク（--tasks）** | 1回のジョブ実行で `CLOUD_RUN_TASK_INDEX` を使って分散処理可能 |

---

## 8. JP1/AJS3 (Manager / Agent) 連携時の実行設計

オンプレミスまたはクラウド環境に導入された **JP1/AJS3 (Manager / Agent)** から Cloud Run Jobs を起動・監視する場合、エンタープライズ特有の要件（完了待ち合わせ、終了コード判定、共通スクリプト化）を考慮した設計が必要です。

### 1. JP1 連携における最重要ポイント：同期待ち (`--wait`) と終了コード

JP1 のジョブは、**コマンドの終了コード（Exit Code: 0 = 正常、非0 = 異常）を検知して後続ジョブの実行やアラートを制御**します。

- **デフォルトの挙動（非同期）**:
  `gcloud run jobs execute` はジョブの起動リクエストを送信した直後に終了コード `0` で即座にコマンドが復帰します。そのため、JP1 側は「コンテナがまだ動いている（または異常終了した）」にもかかわらず「正常終了」と判定して後続ジョブを実行してしまいます。
- **必須パラメータ：`--wait`**:
  `--wait` フラグを付与することで、**ジョブの完了まで gcloud コマンドが待機し、コンテナの処理結果に応じた終了コードを返却**します。

```bash
# JP1 Agent から実行する基本形（--wait が必須）
gcloud run jobs execute run-job \
  --region=asia-northeast1 \
  --wait \
  --args="report,branch1"
```

| 実行結果 | Exit Code | JP1 の挙動 |
| :--- | :--- | :--- |
| **すべてのタスクが正常終了** | `0` | JP1: **正常終了**（後続ジョブへ進行） |
| **コンテナが異常終了 / タイムアウト** | `1`（非0） | JP1: **異常終了**（リカバリジョブ起動 / アラート発報） |

---

### 2. JP1 利用時における「パターン1 vs パターン2」の比較

JP1/AJS3 を導入する場合、**「パターン2（実行時引数指定）」のメリットが飛躍的に大きくなります**。

| 観点 | パターン1: ジョブ定義分割 | パターン2: 実行時引数指定（推奨） |
| :--- | :--- | :--- |
| **JP1 ジョブネットの可視性** | ⭕ **ジョブ名で識別可能**<br>JP1/AJS3 View の定義名やコマンド名で対象ブランチが一目でわかる | ⭕ **引数・ジョブ名で識別可能**<br>JP1 のジョブ名や引数指定欄に `branch1` と記載すれば視認性を維持可能 |
| **JP1 資産の共通化** | ❌ **個別コマンド指定**<br>ジョブごとに Cloud Run ジョブ名を書き分ける必要がある | ⭕ **共通ラッパースクリプトが使える**<br>Agent 上に共通スクリプトを1本作れば、JP1側は引数を変えるだけで全ジョブ共通化可能 |
| **JP1 マクロ変数・日付連携** | ❌ **連携不可**<br>JP1 の処理日や基準日を渡せない | ⭕ **柔軟に連携可能**<br>JP1 のマクロ変数（`?AJS2ENV?`）や実行日変数を `--args` に注入可能 |
| **GCP 側の運用保守** | ❌ **運用負荷大**<br>コンテナ更新時に全ジョブの更新が必要 | ⭕ **最小限の運用負荷**<br>ジョブは1本のみ。イメージ更新も1回で完了 |

---

### 3. 推奨アーキテクチャ：JP1 Agent 上の共通ラッパースクリプト

JP1 のジョブ定義に直接 `gcloud run jobs execute ...` を書き込むのではなく、JP1 Agent サーバー上に**共通ラッパースクリプト（Bash / PowerShell）**を配置して呼び出す構成がベストプラクティスです。

```
[ JP1/AJS3 Manager ]
         │ (実行指示・スケジュール)
         ▼
[ JP1/AJS3 Agent ] (Linux / Windows)
         │ 実行: /opt/scripts/run_cloud_run.sh run-job "report,branch1"
         ▼
 ┌──────────────────────────────────────────────┐
 │ 共通ラッパースクリプト (run_cloud_run.sh)    │
 │  1. サービスアカウント認証チェック           │
 │  2. 開始ログ & 引数出力                      │
 │  3. gcloud run jobs execute --wait 実行      │
 │  4. 実行ログ URL 出力                        │
 │  5. 終了ステータス判定 & exit code 返却      │
 └──────────────────────┬───────────────────────┘
                        │ HTTPS (API)
                        ▼
            [ Google Cloud Run Jobs ]
```

#### 共通ラッパースクリプト実装例 (`/opt/scripts/run_cloud_run.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

# 引数受け取り
JOB_NAME="${1:?Error: Cloud Run Job name is required}"
JOB_ARGS="${2:-}"
REGION="asia-northeast1"

echo "=========================================================="
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Starting Cloud Run Job: ${JOB_NAME}"
echo "Region: ${REGION}"
echo "Args:   ${JOB_ARGS}"
echo "=========================================================="

# gcloud コマンド引数の組み立て
CMD_ARRAY=(gcloud run jobs execute "${JOB_NAME}" --region="${REGION}" --wait)

if [[ -n "${JOB_ARGS}" ]]; then
  CMD_ARRAY+=(--args="${JOB_ARGS}")
fi

# ジョブの実行と完了待機
set +e
"${CMD_ARRAY[@]}"
EXIT_CODE=$?
set -e

echo "=========================================================="
if [[ ${EXIT_CODE} -eq 0 ]]; then
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] Cloud Run Job completed successfully."
else
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] Cloud Run Job failed with exit code: ${EXIT_CODE}"
fi
echo "=========================================================="

# JP1 Agent に終了コードをそのまま返却
exit ${EXIT_CODE}
```

---

### 4. JP1/AJS3 でのジョブ定義設定例

#### (1) JP1/AJS3 View のジョブ定義（PC ジョブ / UNIX ジョブ）
- **ジョブ名**: `JOB_REPORT_BRANCH1`
- **実行ファイル名**: `/opt/scripts/run_cloud_run.sh`
- **パラメータ**: `run-job "report,branch1"`
- **打ち切り時間**: 必要に応じて設定（例: 60分）

#### (2) JP1 のシステム日時・マクロ変数を動的に渡す例
JP1 が管理する「スケジュール実行日（処理基準日）」を引数として渡す場合：

- **パラメータ**: `run-job "report,branch1,$AJS2SYS_DATE"`
  （または JP1 マクロ変数 `?AJS2ENV?` やシェル内で環境変数を展開）

このように構成することで、**GCP 側のジョブ定義を1つに保ちながら、JP1 側の堅牢なジョブスケジュール・先行後続制御・カレンダー管理・障害検知を完全に両立**できます。

---

## 9. JP1 Agent (Linux) における OS ユーザ作成と GCP サービスアカウント設定手順

JP1/AJS3 Agent が動作する Linux サーバから安全に Google Cloud Run Jobs を実行できるようにするための、OS ユーザ作成、秘密鍵の配置・パーミッション設定、gcloud 認証、および JP1/Base のマッピング設定の完全手順です。

```
【セキュリティと権限分離の全体像】
 ┌────────────────────────────────────────────────────────┐
 │ JP1/AJS3 Agent (Linux Server)                          │
 │                                                        │
 │  OS ユーザ: batchuser (ジョブ実行専用ユーザ)           │
 │  ホームディレクトリ: /home/batchuser (chmod 700)       │
 │    └─ .gcp/ (chmod 700)                                │
 │        └─ sa-key.json (chmod 400: 他者閲覧不可)        │
 │                                                        │
 │  共通スクリプト: /opt/scripts/run_cloud_run.sh         │
 └──────────────────────────┬─────────────────────────────┘
                            │ HTTPS (TLS)
                            ▼ サービスアカウント認証
 ┌────────────────────────────────────────────────────────┐
 │ Google Cloud (IAM & Cloud Run)                         │
 │  サービスアカウント: sa-cloud-run-job-invoker         │
 │  ロール: roles/run.developer (必要最小限の権限)        │
 └────────────────────────────────────────────────────────┘
```

---

### Step 1: Google Cloud 側でのサービスアカウント作成とキー発行

管理端末（または Cloud Shell）にて、JP1 専用のサービスアカウントを作成し、必要最小限の権限を付与してキーファイルを発行します。

```bash
# 1. 環境変数の設定
export PROJECT_ID="your-gcp-project-id"
export SA_NAME="sa-cloud-run-invoker"
export SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# 2. サービスアカウントの作成
gcloud iam service-accounts create "${SA_NAME}" \
  --project="${PROJECT_ID}" \
  --display-name="JP1 Cloud Run Jobs Invoker Service Account"

# 3. Cloud Run Jobs 実行に必要なロールを付与
# 【検証環境・簡易設定】roles/run.developer を付与する場合
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/run.developer"

# 4. サービスアカウントの秘密鍵 (JSON) を生成・出力
gcloud iam service-accounts keys create ./sa-cloud-run-invoker.json \
  --iam-account="${SA_EMAIL}" \
  --project="${PROJECT_ID}"
```

> [!IMPORTANT]
> **【プロの視点：本番環境での IAM 最小権限の原則（Least Privilege）】**
> `roles/run.developer` は手軽ですが「ジョブの作成・更新・削除・コンテナイメージ変更」まで可能な過剰な権限（Over-privileged）です。本番環境では、JP1 Agent に「既存ジョブの実行と状態確認のみ」を許可する**カスタムロール**を作成してバインドすることを強く推奨します。
> ```bash
> # ジョブ実行専用カスタムロールの作成例
> gcloud iam roles create CloudRunJobInvokerCustom \
>   --project="${PROJECT_ID}" \
>   --title="Cloud Run Job Invoker Custom Role" \
>   --description="Allows executing and monitoring Cloud Run Jobs only" \
>   --permissions="run.jobs.run,run.jobs.get,run.executions.get,run.executions.list,run.operations.get" \
>   --stage="GA"
>
> # 作成したカスタムロールを付与
> gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
>   --member="serviceAccount:${SA_EMAIL}" \
>   --role="projects/${PROJECT_ID}/roles/CloudRunJobInvokerCustom"
> ```

> [!WARNING]
> **サービスアカウントキーの取り扱いとセキュリティ**
> - 生成された `sa-cloud-run-invoker.json` は強力な永続的クレデンシャルです。Git リポジトリには絶対にコミットせず、安全な経路（SCP 等）で JP1 Agent サーバに転送してください。
> - **ローテーション運用**: キーには定期的な更新期限（例: 90日）を定め、定期的なローテーション計画を策定してください。
> - **モダンな推奨構成**: Google Cloud の組織ポリシーでキー発行が禁止されている場合や、さらなる高セキュリティが求められる環境では、キーファイルを廃止し **Workload Identity 連携（Federation）** または GCE インスタンスにアタッチされたサービスアカウント（インスタンスメタデータ）の利用を推奨します。

---

### Step 2: JP1 Agent (Linux) での専用 OS ユーザ作成

JP1 からルート（root）権限で GCP コマンドを実行させるのはセキュリティ上極めて危険です。Cloud Run 実行専用の OS ユーザ（例: `batchuser`）を作成します。

```bash
# root 権限で実行
sudo -i

# 1. 専用 OS ユーザを作成 (ホームディレクトリ作成、ログインシェル /bin/bash)
useradd -m -s /bin/bash batchuser

# 2. パスワードを設定 (必要に応じて)
passwd batchuser
```

---

### Step 3: サービスアカウントキーの安全な配置とパーミッション設定

転送した秘密鍵ファイルを専用ユーザのホームディレクトリ下に配置し、**他の一般ユーザから一切読み取れないようにパーミッションを厳格に制限**します。

```bash
# root 権限で実行

# 1. 鍵配置用のディレクトリを作成
mkdir -p /home/batchuser/.gcp

# 2. 秘密鍵ファイルを配置 (例: /tmp から移動)
mv /tmp/sa-cloud-run-invoker.json /home/batchuser/.gcp/sa-cloud-run-invoker.json

# 3. 所有者を batchuser に変更
chown -R batchuser:batchuser /home/batchuser/.gcp

# 4. パーミッションを厳格化 (重要)
# ディレクトリ: 所有者のみアクセス可 (rwx------)
chmod 700 /home/batchuser/.gcp
# 秘密鍵ファイル: 所有者のみ読み取り可 (r--------)
chmod 400 /home/batchuser/.gcp/sa-cloud-run-invoker.json

# 5. ホームディレクトリ自体のアクセス制限を確認
chmod 700 /home/batchuser
```

確認コマンド：
```bash
ls -ld /home/batchuser/.gcp
# 実行結果例: drwx------ 2 batchuser batchuser 4096 Oct  9 19:30 /home/batchuser/.gcp

ls -l /home/batchuser/.gcp/sa-cloud-run-invoker.json
# 実行結果例: -r-------- 1 batchuser batchuser 2345 Oct  9 19:30 /home/batchuser/.gcp/sa-cloud-run-invoker.json
```

---

### Step 4: gcloud CLI の初期化と認証動作確認

作成した `batchuser` に切り替え、`gcloud` CLI の初期認証を実行して保存します。

```bash
# 1. batchuser にスイッチ (ログインシェル環境)
sudo -u batchuser -i

# 2. gcloud CLI のパスが通っているか確認
which gcloud
# 例: /usr/bin/gcloud または /usr/local/google-cloud-sdk/bin/gcloud

# 3. サービスアカウントキーを使って恒久的にアクティベート
gcloud auth activate-service-account \
  --key-file=/home/batchuser/.gcp/sa-cloud-run-invoker.json

# 4. デフォルトの GCP プロジェクトを設定
gcloud config set project your-gcp-project-id

# 5. 疎通・権限確認テスト (ジョブ一覧の取得)
gcloud run jobs list --region=asia-northeast1

# 6. テストが成功したら batchuser のシェルを抜ける
exit
```

---

### Step 5: ラッパースクリプトの配置とノンログインシェル対策

JP1/AJS3 からスクリプトが起動される際、環境変数 `PATH` に `/usr/local/bin` 等が含まれていない場合（ノンログインシェル動作）があります。
そのため、ラッパースクリプト内で確実に `PATH` を通す記述を追加します。

```bash
# root でスクリプトディレクトリを作成
mkdir -p /opt/scripts

# スクリプトを配置 (/opt/scripts/run_cloud_run.sh)
cat << 'EOF' > /opt/scripts/run_cloud_run.sh
#!/usr/bin/env bash
set -euo pipefail

# gcloud CLI への PATH を確実に通す (環境に合わせて指定)
export PATH="/usr/local/google-cloud-sdk/bin:/usr/bin:/bin:$PATH"

JOB_NAME="${1:?Error: Cloud Run Job name is required}"
JOB_ARGS="${2:-}"
REGION="asia-northeast1"

# 現在のプロジェクトIDを取得
PROJECT_ID=$(gcloud config get-value project 2>/dev/null || echo "your-gcp-project-id")

echo "=========================================================="
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Cloud Run Job 実行開始"
echo "Job Name:    ${JOB_NAME}"
echo "Region:      ${REGION}"
echo "Project:     ${PROJECT_ID}"
echo "Execute User:$(whoami)"
echo "Args:        ${JOB_ARGS}"
echo "=========================================================="

# gcloud コマンド配列の構築
# --wait: 同期待機
# --format="value(metadata.name)": 生成された Execution ID を標準出力で取得
CMD_ARRAY=(gcloud run jobs execute "${JOB_NAME}" --region="${REGION}" --wait --format="value(metadata.name)")

if [[ -n "${JOB_ARGS}" ]]; then
  CMD_ARRAY+=(--args="${JOB_ARGS}")
fi

# 実行および完了待機
set +e
EXECUTION_NAME=$("${CMD_ARRAY[@]}")
EXIT_CODE=$?
set -e

echo "=========================================================="
echo "Execution ID: ${EXECUTION_NAME}"
if [[ -n "${EXECUTION_NAME}" ]]; then
  echo "Cloud Logging URL:"
  echo "https://console.cloud.google.com/run/jobs/executions/details/${REGION}/${EXECUTION_NAME}/tasks?project=${PROJECT_ID}"
fi
echo "=========================================================="

if [[ ${EXIT_CODE} -eq 0 ]]; then
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] Cloud Run Job 正常終了"
else
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] Cloud Run Job 異常終了 (Exit Code: ${EXIT_CODE})"
fi
echo "=========================================================="

# JP1 Agent に終了コードをそのまま返却
exit ${EXIT_CODE}
EOF

# 実行権限を付与
chmod 755 /opt/scripts/run_cloud_run.sh
chown batchuser:batchuser /opt/scripts/run_cloud_run.sh
```

---

### Step 6: JP1/Base のユーザマッピング設定

JP1/AJS3 からジョブが送信された際、Linux サーバ上で `batchuser` として実行されるように JP1/Base のマッピングを設定します。

1. **マッピング定義ファイルの確認 / 設定**:
   JP1/Base のユーザマッピング定義（`/etc/opt/jp1base/conf/user_acl/jp1users` など）において、JP1 ユーザと OS ユーザ `batchuser` を紐づけます。
   ```
   # 例: JP1 ユーザ「jp1admin」から投入されたジョブを OS ユーザ「batchuser」で実行
   jp1admin:*=batchuser
   ```
2. **JP1/Base 設定の反映**:
   ```bash
   # 設定反映コマンドを実行
   jbssetusrs
   ```

3. **JP1/AJS3 View のジョブ定義での指定**:
   - ジョブ定義の詳細画面にて、**「実行OSユーザー名」に `batchuser` を指定**（マッピングにより自動適用される場合は省略可）。
   - これにより、JP1 Agent は `batchuser` の権限で `/opt/scripts/run_cloud_run.sh` を起動し、先ほど設定したサービスアカウント認証で Cloud Run Jobs が実行されます。

---

## 10. Google Cloud プロフェッショナル推奨：本番運用のベストプラクティス

Google Cloud Certified プロフェッショナルの観点から、エンタープライズ本番システムにおいて重大な障害や運用破綻を防ぐための**必須チェックポイント4選**をまとめます。

### ① リトライ責務の一元化（自動リトライ `--max-retries=0` の推奨）

Cloud Run Jobs はデフォルトで、タスクがエラー終了した場合に**最大3回自動リトライ（`--max-retries=3`）**します。

- **発生するリスク**:
  JP1 側でもジョブ異常時のリトライやリカバリ運用が組まれている場合、**Cloud Run 側で3回リトライ ✕ JP1 側で3回リトライ ＝ 計9回実行**という二重リトライが発生します。これにより、バッチ処理時間の想定外の肥大化や、非冪等な処理（DBへの重複INSERTや決済API呼び出しなど）によるデータ不整合が発生します。
- **推奨設計**:
  JP1 側でジョブの再実行やリトライを一元制御する場合は、**Cloud Run Jobs 側のリトライを無効化（`--max-retries=0`）**してください。
  ```bash
  gcloud run jobs update run-job --max-retries=0 --region=asia-northeast1
  ```

---

### ② タイムアウト設計と孤立実行（ゾンビジョブ）防止

JP1/AJS3 の「打ち切り時間（ジョブタイムアウト）」と Cloud Run Jobs の「タスクタイムアウト（`--task-timeout`）」の大小関係を必ず整合させてください。

```
【適正なタイムアウト設計】
 ────────────────────────────────────────────────────────► (時間経過)
 ├──────── Cloud Run Job タスクタイムアウト (例: 60分) ───────┤
 ├────────────── JP1/AJS3 打ち切り時間 (例: 90分) ──────────────────┤
 (コンテナ側が先にタイムアウトで確実に異常終了し、JP1 に exit 1 が返る)
```

- **重大リスク（逆転現象）**:
  もし「JP1 打ち切り時間（30分） < Cloud Run タイムアウト（60分）」となっていた場合、JP1 側は30分で打ち切って後続処理に進んでしまいますが、**Google Cloud 側ではコンテナが裏で動き続けます（孤立実行・ゾンビジョブ）**。これにより、後続処理と先行処理の競合や、想定外の課金が発生します。
- **原則**:
  **「JP1 打ち切り時間 ＞ Cloud Run Jobs の `--task-timeout`（デフォルト10分、最大24時間）」** を徹底してください。

---

### ③ 呼出用 SA とコンテナ内部用 Runtime SA の完全分離

セキュリティ統制上、以下の2つのサービスアカウントを絶対に同一にしてはいけません。

| サービスアカウント | 役割 | 必要な最小権限 | 保有場所 |
| :--- | :--- | :--- | :--- |
| **呼出用 SA**<br>(Caller SA) | JP1 Agent から Cloud Run ジョブをキック・状態取得する | `run.jobs.run`, `run.executions.get`（カスタムロール） | JP1 Agent サーバ<br>(秘密鍵またはWIF) |
| **実行用 SA**<br>(Runtime SA) | Cloud Run コンテナ内で動作し、DB や Cloud Storage にアクセスする | `roles/storage.objectViewer`, `roles/cloudsql.client` 等 | Cloud Run コンテナ内<br>(メタデータから自動取得) |

ジョブ作成時に `--service-account` フラグでコンテナ用の Runtime SA を指定し、JP1 Agent にはコンテナ内部のデータアクセス権限を一切与えないことで、万が一 JP1 Agent サーバが侵害された場合の情報漏洩範囲を最小化できます。

---

### ④ 複雑なパラメータにおける環境変数渡し（`--update-env-vars`）の併用

スペース、特殊文字、日本語文字列、あるいは長大な設定値を渡す場合、`--args`（カンマ区切り引数）はシェルのクォート展開やエスケープでトラブルが発生しやすくなります。

このようなケースでは、**環境変数オーバーライド（`--update-env-vars`）** を併用することを推奨します。

```bash
# キー・バリュー形式で安全に渡す
gcloud run jobs execute run-job \
  --region=asia-northeast1 \
  --wait \
  --update-env-vars="TARGET_BRANCH=branch1,PROCESS_DATE=20261009,CONFIG_PATH=/data/config.json"
```

- 引数の順番間違いによるバグを根絶できる
- コンテナ内の各言語（Python, Java, Go等）の標準ライブラリ（`os.environ` など）から直感的に取り出せる
- JP1 マクロ変数との親和性が非常に高い



