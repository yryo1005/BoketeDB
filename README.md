# BoketeDBv3: データストレージ構造定義

大喜利投稿サイト「ボケて（[bokete.jp](https://bokete.jp/))」から収集した大規模データセットです．
本プロジェクトでは，お題（Odai），ボケ（Boke），評価（Rating），およびお題画像を効率的に管理・処理するため，Apache Parquet形式による列指向メタデータ保存と，バッチ分割された画像ストレージ構造を採用しています．

---

# 1. データの読み込み方法 (Usage Example)

データセットはPolars等のDataFrameライブラリを用いて，ディレクトリ単位で高速に一括読み込みが可能です．

```python
import polars as pl

# お題データ（全シャード）の一括読み込み
odai_df = pl.read_parquet("BoketeDBv3/odai/")

# ボケデータ（全シャード）の一括読み込み
boke_df = pl.read_parquet("BoketeDBv3/boke/")
```

---

# 2. ディレクトリ構造 (Directory Structure)

データはお題のID（`ODAI_ID`）を基準に **10,000件ずつのバッチ（シャード）** に分割され，以下の階層で保存されています．

```text
BoketeDBv3/
├── odai/                         # お題メタデータ (Parquet形式)
│   ├── odai_1.parquet            # ODAI_ID: 0 〜 9999 のデータ
│   ├── odai_2.parquet            # ODAI_ID: 10000 〜 19999 のデータ
│   └── ...
├── boke/                         # ボケメタデータ (Parquet形式)
│   ├── boke_1.parquet            # ODAI_ID: 0 〜 9999 に紐づくボケデータ
│   ├── boke_2.parquet            # ODAI_ID: 10000 〜 19999 に紐づくボケデータ
│   └── ...
└── image/                        # お題画像ファイル (.jpg)
    ├── 0/                        # ODAI_ID: 0 〜 9999 の画像
    │   ├── 0.jpg
    │   ├── 1.jpg
    │   └── ...
    ├── 10000/                    # ODAI_ID: 10000 〜 19999 の画像
    │   ├── 10000.jpg
    │   └── ...
    └── ...
```

---

# 3. データスキーマ定義 (Data Schemas)

## 3.1 お題データ (`odai/odai_*.parquet`)

お題の基本情報に加え，マルチモーダルAIモデル等により抽出された画像メタデータを格納しています．

| フィールド名 | 型 | 説明 |
| --- | --- | --- |
| `id` | `int64` | お題の固有ID (`ODAI_ID`) |
| `date` | `timestamp('s')` | お題の投稿日時 |
| `bokeCount` | `int32` | このお題に対する総ボケ数 |
| `category` | `string` | お題のカテゴリ |
| `largeURL` | `string` | お題画像のソースURL |
| `imageOCR` | `list<ocr_schema>` | 画像から抽出されたOCRテキスト情報の可変長配列 |
| `imageCaption` | `string` | 画像の内容を説明するテキスト |
| `imageType` | `map<string, float32>` | 画像スタイルのカテゴリ確率分布（Map構造） |

### メタデータフィールドの仕様

#### **`imageCaption` (画像キャプション)**
`Qwen/Qwen3-VL-2B-Instruct-FP8` を使用し，お題画像の内容をコンテキストとして言語化．

#### **`imageOCR` (`ocr_schema`)**
`Yomitoku` を使用し，画像内に含まれるテキスト情報とその位置を構造化．
* `content` (`string`): 認識された文字列
* `det_score` (`float32`): テキスト領域の検出スコア
* `direction` (`string`): 文字の書字方向（縦書き/横書きなど）
* `points` (`list<list<int32>>`): 四角形のバウンディングボックス座標群
* `rec_score` (`float32`): 文字認識の確信度スコア

#### **`imageType` (画像スタイルの確率分布)**
`openai/clip-vit-base-patch32` を使用し，以下の4つのテキストプロンプトと画像との類似度を算出し，ソフトマックス関数により**総和が $1.0$ の確率分布**に変換．
* `a real life photograph`: 現実の写真
* `an anime screenshot`: アニメのスクリーンショット
* `a manga drawing with speech bubbles`: フキダシ付きの漫画の一コマ
* `a traditional artwork`: 伝統的な絵画・アートワーク

---

## 3.2 ボケデータ (`boke/boke_*.parquet`)

各お題に対してユーザーが投稿したボケの内容と，そのボケに対する評価履歴を格納しています．

| フィールド名 | 型 | 説明 |
| --- | --- | --- |
| `id` | `int64` | ボケの固有ID |
| `odaiId` | `int64` | 紐づくお題のID (`ODAI_ID`) |
| `date` | `timestamp('s')` | ボケの投稿日時 |
| `text` | `string` | ボケのテキスト（大喜利の回答内容） |
| `isNG` | `bool` | ボケのテキストにNGワードを含んでいるか |
| `isUniqueNoun` | `bool` | ボケのテキストに固有名詞を含んでいるか |
| `rateSum` | `float32` | 獲得スコアの合計値 |
| `rateCount` | `int32` | 評価を投じた総ユーザー数 |
| `lastRateDate` | `timestamp('s')` | 最終評価が観測された日時 |
| `category` | `string` | ボケのカテゴリ |
| `rates` | `list<rate_schema>` | 評価履歴の時系列可変長配列 |

### メタデータフィールドの仕様

#### **`isNG` (ボケのテキストにNGワードを含んでいるか)**
`https://ja.wikipedia.org/wiki/性風俗用語一覧` 内の単語を抽出し，
その単語のうちいずれかを含んでいる場合にTrue，いずれも含んでいない場合にFalseになります．

#### **`isUniqueNoun` (ボケのテキストに固有名詞を含んでいるか)**
`MeCab-NEologd` 辞書を用いてテキストを分かち書きし，
品詞が固有名詞の単語を含んでいる場合にTrue，含んでいない場合にFalseになります．

#### `rate_schema` (評価履歴構造体)

ボケに対して蓄積された星（評価）のログ．

* `date` (`timestamp('s')`): 評価された日時
* `rate` (`float32`): 付与された評価スコア