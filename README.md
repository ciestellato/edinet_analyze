# EDINET Data Pipeline & Dash Dashboard

## 使用技術一覧

本プロジェクトはPythonを中心としたデータ収集、処理、および可視化のためのツールキットです。

| 言語・フレームワーク | 役割 |
| :--- | :--- |
| **Python 3.x** | メイン言語 |
| **Dash / Flask** | Webダッシュボードおよび認証サーバー |
| **SQLAlchemy** | ORMおよびデータベース接続 |
| **SQLite** | 財務データ永続化データベース (`edinet_data.db`) |
| **Pandas** | データ処理、サマリー管理、CSV解析 |
| **Flask-Login** | セッションベースのユーザー認証 |
| **Plotly Express** | 財務データのグラフ可視化 |

## 目次

1. [プロジェクトについて](#プロジェクトについて)
2. [環境](#環境)
3. [ディレクトリ構成](#ディレクトリ構成)
4. [開発環境構築](#開発環境構築)
5. [トラブルシューティング](#トラブルシューティング)

## プロジェクト名

EDINET Data Pipeline & Dash Dashboard

## プロジェクトについて

本プロジェクトは、金融庁のEDINET APIを利用し、上場企業の有価証券報告書や四半期報告書などの提出書類のデータパイプラインを構築し、結果を可視化するWebダッシュボードを提供するものです。

**データパイプライン (`edinet_main.py`):**

*   **サマリー取得:** EDINET APIから書類一覧を取得し、メタデータをローカルファイルおよびデータベース（`edinet_document_summaries`）に保管します。
*   **堅牢なダウンロード:** CSV提供フラグが'1'で証券コードが存在する書類を対象にZIPファイルをダウンロードし、提出日時 に基づきフォルダに整理します。ダウンロード失敗ファイルはログに記録され、再試行されます。
*   **CSV解析と永続化:** ダウンロードされたZIPからCSVを抽出し（`edinet_extracted_csv_details`）、エンコーディングや区切り文字の多様性に対応した堅牢な解析ロジックを使用して、主要な勘定科目（例: 売上高、資産合計）の財務数値を抽出します。
*   **DB格納:** 抽出された財務数値はSQLiteデータベース（`edinet_financial_data`）に保管されます。

**Webダッシュボード (`app.py`):**

*   **認証:** Flask-Loginに基づくセッション認証を提供します（デフォルト認証情報: `admin`/`adminpass`, `user`/`userpass`）。
*   **管理者機能 (Admin):** Web UIからAPI設定（APIキー、取得年数、対象書類タイプコードなど）の一時的な変更および、データパイプラインのステップ（全処理、サマリー更新、ダウンロード・解析）の実行をトリガーできます。実行中のリアルタイムログが表示されます。
*   **データ可視化 (User/Admin):** 会社名または証券コードによるあいまい検索、および会計期間終了日の指定により、データベース内の財務データを検索し、テーブル表示およびPlotlyグラフで可視化します。

## 環境

| 言語・フレームワーク | バージョン |
| :--- | :--- |
| Python | 3.x (推奨) |
| Flask / Dash | (依存パッケージを参照) |

その他のパッケージのバージョンはプロジェクト内の依存関係ファイルを参照してください。

## ディレクトリ構成

```
/edinet_analyze
├── .env                    # 環境変数ファイル（EDINET APIキーなど）
├── directory_overview.md   # ★このファイル★ ディレクトリ構造メモ
├── app.py                  # Dashアプリケーションのエントリーポイント
├── edinet_api.py           # EDINETとの通信
├── edinet_config.py        # ① 全体の設定ファイル (Configクラスを定義)
├── edinet_main.py          # ② メインのデータパイプラインエントリーポイント
├── logging_setup.py        # ロギング設定とバッファ定義
├── requirements.txt        # 必要なライブラリの一覧（pip install -r 用）

├── /edinet_pipeline/     # ③ ソースコード本体 (Pythonパッケージ)
│   ├── __init__.py       # パッケージとして機能させるためのファイル
│   ├── base.py           # SQLAlchemy ORMのベース定義
│   ├── models.py         # ORMモデル (edinet_document_summariesなど)
│   ├── database.py       # DB接続、Engine/Session、初期化ロジック
│   ├── edinet_steps.py   # 高レベルの処理フロー制御 (step1〜step7)
│   ├── storage_repo.py   # DB永続化 (データ保存、読み出し) 処理
│   ├── file_processor.py # ファイルI/O、CSV抽出、解析ロジック
│   └── zip_utils.py      # ZIP操作など汎用的なユーティリティ

├── /data/                # ④ 実行時に生成されるデータ (永続化データ)
│   ├── edinet_data.db    # SQLiteデータベースファイル (Config.DB_PATH)
│   ├── /downloaded_zips/ # EDINET APIからダウンロードしたZIPファイルの保存先
│   └── /extracted_csvs/  # ZIPから抽出されたCSVデータの一時/永続保存先

├── /webapp/              # Webアプリケーション（Dash）関連のモジュール
│   ├── __init__.py             # パッケージとして機能させるためのファイル
│   ├── auth.py                 # Flask-Loginの認証設定とユーザーモデル
│   ├── callbacks_auth.py       # Dashのコールバック処理(WEBアプリログイン認証)
│   ├── callbacks_config.py     # Dashのコールバック処理(設定関連)
│   ├── callbacks_financial.py  # Dashのコールバック処理(財務関連)
│   ├── callbacks_processing.py # Dashのコールバック処理(処理実行)
│   └── layout.py               # Dashのレイアウト定義

├── /assets/              # Webアプリケーション用静的ファイル置き場
│   ├── style.css         # 外観用スタイルシート
│   └── script.js         # JavaScript

└── /logs/                # ⑤ 実行ログファイル
    └── application.log   # ロギング出力先
```

データはデフォルトで以下のパスに保存されます:
*   `edinet_data.db`: SQLiteデータベースファイル。
*   `01_zip_files/`: ダウンロードされたZIPファイル保存先。
*   `failed_downloads.csv`: ダウンロード失敗ログ。
*   `EDINET_Summary_v3.csv`: メタデータサマリーファイル。

## 開発環境構築

### 1. 環境設定ファイル (.env) の作成

プロジェクトのルートディレクトリに `.env` ファイルを作成し、EDINET APIキーを設定します。

```env
EDINET_API_KEY="あなたのEDINET APIキー"

# (オプション) ベースディレクトリ。未設定の場合、デフォルトパスが使用されます。
# EDINET_BASE_DIR="C:/path/to/your/data/"
```

> **注意**: `EDINET_API_KEY` が設定されていない場合、処理は実行されません。

### 2. データベースの初期化

`edinet_main.py` または `app.py` を初めて実行する際に、データベース (`edinet_data.db`) が自動的に初期化され、必要なテーブルが作成されます。

### 3. データパイプラインの実行 (CLI)

過去データの取得や更新をバッチ処理で行います。

```bash
python edinet_main.py
```

### 4. Webダッシュボードの実行

データ処理の管理と結果の可視化を行います。

```bash
python app.py
```

アプリケーションはデフォルトで `http://127.0.0.1:8050/` で起動します。

### 5. 認証情報

| ユーザー名 | パスワード | ロール |
| :--- | :--- | :--- |
| `admin` | `adminpass` | 管理者 |
| `user` | `userpass` | 一般ユーザー |

### コマンド一覧

| コマンド | 実行する処理 | 備考 |
| :--- | :--- | :--- |
| `python edinet_main.py` | 全データパイプライン処理を実行 | サマリー更新、ダウンロード、CSV解析、DB保管。 |
| `python app.py` | Webダッシュボードの起動 | ポート8050で実行されます。 |

## トラブルシューティング

### `EDINET_API_KEY が .env に設定されていません。`

環境変数 `EDINET_API_KEY` がプロジェクトルートの `.env` ファイルに正しく設定されているか確認してください。設定されていない場合、処理は実行できません。

### `docker daemon is not running`

*(コンテナ環境を使用している場合)* Docker Desktopが起動しているか確認してください。

### `Ports are not available: address already in use`

Dash/Flaskアプリケーションが使用するポート8050が、すでに他のプロセスで使用されています。他のアプリケーションを停止するか、ポート番号を変更してください。

### SQLite データベースエラー (e.g., table not found)

`edinet_data.db` ファイルが存在しないか、テーブルが初期化されていません。`edinet_main.py` または `app.py` を実行し、データベースの初期化が行われたか確認してください。
