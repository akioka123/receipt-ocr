レシートOCR自動CSV化システム 外部設計書
1. 文書情報

文書名：レシートOCR自動CSV化システム 外部設計書

バージョン：v1.0

作成日：YYYY-MM-DD

作成者：XXX

対象読者：

開発担当（AIエージェント＋人間エンジニア）

運用担当

利用者（家計簿集計・分析を行うユーザ）

2. システム概要
2.1 目的

本システムは、ローカル（または将来的にS3）に蓄積されたレシート画像を対象に、
OCRおよび日本語テキスト解析によりレシート情報を自動構造化し、家計簿向けのCSVデータとして出力することを目的とする。

あわせて、AIエージェントがステップバイステップで実装・改善できるよう、
機能分割された外部仕様・インターフェースを定義する。

2.2 対象範囲

対象となるレシート：

日本国内の一般的なPOSレシート（日本語、金額は円）

モノクロ／カラーいずれも可

画角・傾きに多少のばらつきがある画像

対象外（当初スコープ外）：

手書きの領収書

PDFの領収書（後続フェーズで拡張可能とする）

多言語レシート（英語のみ等）

2.3 システム構成概要

実行環境：Linux コンテナ（Docker）

主なコンポーネント：

実装言語：Python 3系

OCRエンジン：Tesseract（日本語 jpn 利用）

画像前処理ライブラリ：OpenCV / Pillow

形態素解析：MeCab / Sudachi など（必要に応じて利用）

入出力：

入力：レシート画像ファイル（JPEG/PNG 等）

出力：CSVファイル（レシートヘッダ／明細）、処理済み画像、ログ

3. 想定利用シナリオ
3.1 日常利用フロー（ユーザ視点）

利用者はレシートをスマホ等で撮影し、PCへ取り込む。

レシート画像を「レシートディレクトリ」に保存する。

規定時刻（例：毎日 2:00）になると、OCRバッチが自動起動する。

バッチはレシートディレクトリ内の未処理画像を順次処理し、

OCR＋解析を実施し、

CSVに追記し、

画像を「処理済みディレクトリ」へ移動する。

解析不能／低精度と判定されたレシートは「要レビュー」として別ディレクトリに退避される。

利用者は出力されたCSV（家計簿ソフト等へインポート）と「要レビュー」フォルダを確認し、必要に応じて手修正を行う。

4. システム構成（論理構成）
4.1 コンポーネント一覧
コンポーネント名	区分	役割概要
バッチ起動スクリプト	バッチ	docker run batch_ocr を呼び出すラッパ
OCRバッチアプリケーション	アプリケーション	レシート画像の列挙～OCR～解析～CSV出力まで一連処理
画像前処理モジュール	ライブラリ	グレースケール化、リサイズ、二値化等の画像前処理
OCRクライアントモジュール	ライブラリ	Tesseract呼び出し、オプション設定、結果テキスト化
テキスト解析モジュール	ライブラリ	日付／合計／店舗名／明細抽出（共通ルール＋店舗別ルール）
形態素解析モジュール	ライブラリ	日本語トークン化・品詞付与（必要時利用）
店舗別ルール管理モジュール	ライブラリ	store_config.json 読み込みと店舗別パーサ割り当て
CSV出力モジュール	ライブラリ	receipt_header.csv / receipt_items.csv への追記
ログ・ステータス管理モジュール	ライブラリ	処理ログ・ステータス（OK/要レビュー/エラー）の管理
設定ファイル	設定	config.yml／store_config.json 等で挙動制御
4.2 ディレクトリ構成（論理）
project-root/
  docker/
    Dockerfile
  app/
    main.py
    ocr/
      preprocess.py
      tesseract_client.py
    parser/
      common_rules.py
      store_rules.py
      store_config_loader.py
    storage/
      writer_csv.py
      file_mover.py
    config/
      config.yml
      store_config.json
    logging/
      logger.py
  data/
    receipts/          # 未処理レシート
    processed/         # 正常処理済レシート
    review/            # 要レビュー（エラー/低信頼）
  output/
    receipt_header.csv
    receipt_items.csv
  logs/
    app.log

4.3 処理フロー（概念図）
flowchart TD
  A[レシートディレクトリ<br>未処理画像] --> B[画像前処理]
  B --> C[OCR(Tesseract)]
  C --> D[テキスト行分割・正規化]
  D --> E[共通ルール解析<br>日付/合計/店舗名候補]
  E --> F[店舗判定<br>店舗別ルール]
  F --> G[ヘッダ情報抽出]
  F --> H[明細情報抽出<br>共通 or 店舗別]
  G --> I[ヘッダCSV出力<br>receipt_header.csv]
  H --> J[明細CSV出力<br>receipt_items.csv]
  I --> K[ステータス判定]
  J --> K
  K -->|OK| L[処理済みディレクトリへ移動]
  K -->|REVIEW/ERROR| M[要レビューディレクトリへ移動]
  K --> N[ログ出力]

5. 実行方式とインターフェース
5.1 実行方式（バッチ）

実行単位：コンテナ 1回起動ごとに1バッチ実行

実行コマンド例（ホスト側想定）：

docker run --rm \
  -v /host/path/receipts:/app/data/receipts \
  -v /host/path/processed:/app/data/processed \
  -v /host/path/review:/app/data/review \
  -v /host/path/output:/app/output \
  -v /host/path/logs:/app/logs \
  batch_ocr


実行タイミング：cron 等でスケジューリング（例：毎日 2:00）

5.2 入力インターフェース（ディレクトリ／ファイル）

入力ディレクトリ：/app/data/receipts/

対象拡張子：

.jpg, .jpeg, .png, .bmp

ファイル命名規則：

特に制約なし（端末デフォルト命名可）

ただし、同名ファイル再投入による重複処理は利用者側で回避することが望ましい。

5.3 出力インターフェース（CSV）
5.3.1 レシートヘッダ CSV

出力ファイル：/app/output/receipt_header.csv

文字コード：UTF-8

改行コード：LF

区切り文字：,（カンマ）

ヘッダ行：あり

カラム名	型	必須	説明
receipt_id	文字列(36)	○	レシートID（UUIDなど）
image_path	文字列	○	元画像の相対パス（処理後は processed/review 側）
store_name	文字列	△	店舗名（判定できない場合は空）
store_code	文字列	△	店舗別ルールID（未判定時は空）
purchased_at	日時文字列	△	購入日時（YYYY-MM-DD HH:MM）
total_amount	数値	△	合計金額（税込）。不明時は空
subtotal_amount	数値	×	小計（取得不可なら空）
tax_amount	数値	×	税額（取得不可なら空）
payment_method	文字列	×	支払方法（CASH, CREDIT, IC, QR, OTHER 等）
ocr_raw_text	文字列（長）	○	OCR生テキスト（検証・デバッグ用）
status	文字列	○	OK / REVIEW / ERROR
5.3.2 レシート明細 CSV

出力ファイル：/app/output/receipt_items.csv

文字コード：UTF-8

改行コード：LF

区切り文字：,（カンマ）

ヘッダ行：あり

カラム名	型	必須	説明
receipt_id	文字列(36)	○	レシートID（ヘッダと紐付け）
line_no	数値	○	明細行番号（1から採番）
item_name	文字列	○	商品名（行左側）
unit_price	数値	×	単価（取得不可なら空、後続で計算可）
quantity	数値	×	数量（取得不可なら1をデフォルトなど）
line_total	数値	△	行金額。不明時は空
category	文字列	×	商品カテゴリ（後続処理で付与想定、初期は空）
5.4 画像ディレクトリ

処理済みディレクトリ：/app/data/processed/

正常に解析できた画像を移動。

要レビューディレクトリ：/app/data/review/

解析不能／低信頼度の画像を移動。

ファイル名は元を維持。

6. 設定ファイル仕様
6.1 共通設定ファイル（config.yml）

場所：/app/config/config.yml

ocr:
  tesseract_cmd: "/usr/bin/tesseract"
  lang: "jpn"
  psm: 4      # 単一列テキストを想定
  oem: 1      # LSTMエンジン

preprocess:
  resize_max_width: 1600
  binarization_method: "otsu"   # "otsu" / "adaptive"
  denoise: true

paths:
  input_dir: "/app/data/receipts"
  processed_dir: "/app/data/processed"
  review_dir: "/app/data/review"
  output_header_csv: "/app/output/receipt_header.csv"
  output_items_csv: "/app/output/receipt_items.csv"
  log_file: "/app/logs/app.log"

rules:
  date_patterns:
    - "\\d{4}[/-]\\d{1,2}[/-]\\d{1,2}"
    - "\\d{4}年\\d{1,2}月\\d{1,2}日"
  total_keywords:
    - "合計"
    - "総合計"
    - "お買上"
    - "お買い上げ"
  payment_keywords:
    cash: ["現金"]
    credit: ["クレジット", "VISA", "MASTER", "JCB", "AMEX"]
    ic: ["交通系IC", "Suica", "PASMO", "ICOCA", "nanaco", "WAON", "楽天Edy"]
    qr: ["PayPay", "LINE Pay", "楽天ペイ"]

status_thresholds:
  # 合計金額と明細合計の差がこの割合を超えたら REVIEW
  total_diff_rate_review: 0.1

store_config_path: "/app/config/store_config.json"

6.2 店舗別ルール設定ファイル（store_config.json）

場所：/app/config/store_config.json

形式：JSON 配列

[
  {
    "store_code": "super_aaa",
    "name_patterns": ["スーパーAAA", "ＡＡＡストア"],
    "phone_patterns": ["06-1234-5678"],
    "item_line_regex": "^(?P<name>.+?)\\s+(?P<price>\\d{2,5})円?$",
    "header_skip_until": "商品名",
    "footer_start_keywords": ["合計", "小計", "ポイント"],
    "notes": "明細は商品名＋税込金額のみ"
  },
  {
    "store_code": "conveni_x",
    "name_patterns": ["コンビニX"],
    "phone_patterns": [],
    "item_line_regex": "^(?P<name>.+?)\\s+(?P<qty>\\d+)個?\\s+(?P<price>\\d{2,5})円?$",
    "header_skip_until": "",
    "footer_start_keywords": ["合計", "ご請求額"],
    "notes": "数量・単価を含むレイアウト"
  }
]

7. 機能要件（機能一覧）
7.1 機能一覧表
ID	機能名	概要
F-01	レシート画像一覧取得	未処理ディレクトリから画像ファイル一覧を取得
F-02	画像前処理	グレースケール化、リサイズ、二値化など前処理
F-03	OCR実行	Tesseract を用いて画像からテキストを取得
F-04	OCRテキスト行分割・正規化	テキストを行単位に分割し、空行・余分な空白を除去
F-05	日付抽出（共通ルール）	正規表現により購入日を抽出
F-06	合計金額抽出（共通ルール）	「合計」等キーワード＋最大金額行から合計金額を抽出
F-07	店舗名候補抽出（共通ルール）	先頭数行から店舗名候補となる行を抽出
F-08	店舗判定（店舗別ルール適用）	store_config.json に基づき store_code を決定
F-09	明細行抽出（共通ロジック）	合計行の上下から明細ブロックを抽出、行末数値を金額、左側を商品名として取得
F-10	明細行抽出（店舗別ロジック）	店舗毎の正規表現・ルールで明細を抽出
F-11	形態素解析（オプション）	商品名・店舗名解析用に日本語トークン化
F-12	ヘッダCSV出力	receipt_header.csv にヘッダ情報を追記
F-13	明細CSV出力	receipt_items.csv に明細情報を追記
F-14	ステータス判定	正常／要レビュー／エラーの判定とフラグ付け
F-15	画像ファイル移動	処理済み／要レビュー ディレクトリへの移動
F-16	ログ出力	処理状況・エラー内容のログ出力
8. AIエージェント向け 開発ステップ設計
STEP 0：初期環境構築

目的

Dockerfile とベースディレクトリ構成を用意し、Python＋Tesseract が動くコンテナを作る。

タスク

Dockerfileの生成

app/ 配下の最低限ファイル作成（main.py、モジュールの空ファイル）

config/config.yml の雛形生成

STEP 1：単一画像の前処理＋OCR

目的

1枚のレシート画像を入力として、前処理＋OCRでテキストを取得できる状態にする。

タスク

F-02：画像前処理モジュール実装

F-03：OCRクライアントモジュール実装

F-04：OCRテキストの行分割・正規化

成果物

任意のレシート画像1枚を指定し、標準出力にテキストを出すスクリプト

STEP 2：共通ルールによるヘッダ抽出＋ヘッダCSV出力

目的

日付・合計金額・店舗名候補を抽出し、ヘッダCSVのみ出力する。

タスク

F-05：日付抽出処理

F-06：合計金額抽出処理

F-07：店舗名候補抽出処理

F-12：ヘッダCSV出力処理

成果物

単一レシートから receipt_header.csv にレコードを1行追加できる状態

STEP 3：共通ロジックによる明細抽出＋明細CSV出力

目的

合計行の位置などを利用して明細候補ブロックを特定し、行末数値＝金額／左側＝商品名で切り出す。

タスク

F-09：明細行抽出（共通ロジック）

F-13：明細CSV出力処理

成果物

単一レシートからヘッダ＋明細の両CSVにレコードを追加できる状態

STEP 4：ディレクトリ単位のバッチ化＋ステータス判定

目的

指定ディレクトリ内の全レシート画像を対象に、シリアル処理を行うバッチを構築する。

タスク

F-01：レシート画像一覧取得

F-14：ステータス判定（OK/REVIEW/ERROR）

F-15：画像ファイル移動

F-16：ログ出力

main.py から上記を順次呼び出す制御フロー実装

成果物

docker run batch_ocr 1回で、複数レシートを処理・CSV出力し、画像を processed/review に振り分ける処理

STEP 5：店舗別ルール 1店舗分の導入

目的

よく利用するスーパーなど、1店舗分の店舗別パーサを導入し精度を上げる。

タスク

店舗サンプルレシートを元にレイアウト分析

store_config.json に店舗エントリ追加

F-08：店舗判定ロジック実装

F-10：店舗別明細抽出ロジック実装

共通ロジックとの切替制御実装

成果物

対象店舗のレシートでは、明細抽出の精度が向上していることを確認

STEP 6：精度評価・前処理／ルールのチューニング

目的

実サンプルでの精度を評価し、前処理やルールを改善する。

タスク

代表サンプルに対して、日付／合計／明細抽出成功率を算出

status = REVIEW となったサンプルを中心に原因分析

config.yml の閾値や正規表現、前処理パラメータの調整

必要に応じて追加の店舗別ルール実装

成果物

精度改善前後の結果比較と、その変更内容の記録

9. 非機能要件
9.1 性能

想定規模：

1日あたり 10〜100枚程度のレシート

応答時間：

100枚のレシート処理が数分以内に完了することを目安

並列化：

初期版はシリアル処理。

将来的に画像単位での並列処理へ拡張可能な構造とする。

9.2 信頼性・保守性

異常終了時：

途中まで処理済みのレシートはCSVに残る。

再実行時、receipt_idの重複チェックにより二重登録を回避できる設計を推奨。

ログ：

画像ファイル名、処理開始・終了時刻、ステータス、エラー内容をログ出力。

保守：

店舗別ルールは store_config.json により設定可能とし、コード変更なしで店舗追加・修正を可能にする。

9.3 セキュリティ・プライバシ

当初想定は「ローカルPC上の個人利用」。

レシート画像・CSVには会員番号・電話番号等の個人情報が含まれる可能性があるため、
クラウド環境で運用する場合は暗号化・アクセス制御を別途検討する。

10. 将来拡張（参考）

入力ソース拡張：

S3バケット・オブジェクトストレージからの取得

スマホアプリとの連携（直接アップロード）

出力拡張：

家計簿サービス向けフォーマット出力（MoneyForward等）

DB（SQLite/PostgreSQL）への保存と集計画面

モデル強化：

レイアウト認識モデル導入（LayoutLM 等）

生成AI APIを用いた「OCR結果 → 構造化データ」補完

UI：

Web管理画面での「要レビュー」レシート確認・修正

CSVエクスポート機能

付録：用語定義
用語	説明
OCR	Optical Character Recognition。画像から文字を認識する技術。
Tesseract	オープンソースのOCRエンジン。本システムでは日本語データ（jpn）を利用。
形態素解析	文を単語に分割し、品詞情報を付与する処理。日本語テキスト解析で利用。
レシートヘッダ	店舗名、日付、合計金額など、レシート全体に関する情報。
レシート明細	個々の商品行の情報（商品名、金額など）。
店舗別ルール	特定チェーンのレシート形式に合わせた解析ロジック。