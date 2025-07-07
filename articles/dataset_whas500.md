---
title: "WHAS (Worcester Heart Attack Study) データセット"
emoji: "📊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["R", "Python", "データセット"]
published: true
---

データサイエンスの演習で用いられる、急性心筋梗塞に関する地域コホート研究に由来する WHAS500 (Worcester Heart Attack Study) データセットについて。

## データセットについて

Google Gemini によると、

> Worcester Heart Attack Study (WHAS) は、アメリカ合衆国マサチューセッツ州ウースター都市圏の住民を対象とした、急性心筋梗塞（AMI、いわゆる心臓発作）の発生率、治療法、および転帰の経時的な変化を追跡調査した、地域ベースの大規模な疫学研究です。\
> （中略）\
> この研究は、1975年から始まり、数十年にわたって継続的にデータが収集されてきました。これにより、長期間にわたるAMIの疫学的変化、医療進歩の影響、そして公衆衛生戦略の有効性を評価するための貴重なデータを提供しています。\
> WHASデータセットは、患者がAMIで入院した後の生存期間（日数）を主要なアウトカム変数としており、データサイエンスや生存時間解析の演習で頻繁に用いられます。このデータは、単に個々の患者の予後を予測するだけでなく、地域レベルでの心臓病対策の改善にも貢献する情報源となっています。

WHAS500 データセットはマサチューセッツ大学アマースト校 UMass Amherst のサイトで 2017 年頃まで配布されていました [^2] が、現在は Fobbiden になっており直接入手することはできません（大本は Wiley の書籍 [^1] 関連ファイルとして同社 ftp サーバーで配布されていたもののようです）。

筆者は過去に学習用にダウンロードしたファイルを所持していますが、これから練習用に使いたい方へ紹介できる方法を検討しました。現在オンラインで入手可能な方法としては、

- R パッケージ `smoothHR`
    - `whas500` としてデータセットが添付されている
    - 22変数、500症例で UMass Amherst のサイトで配布されていたものと最も近い
    - ライセンスは GPL-3 であり、出典を示して改変・再配布は可能
- Python パッケージ `scikit-survival`
    - `sksurv.datasets.load_whas500()` 関数で利用可能
    - 目的変数として 2変数、説明変数として 14変数。ともに 500症例
    - ライセンスは GPL-3 であり、出典を示して改変・再配布は可能
- UCLA OARC（統計学応用研究センター）の [オンライン講義資料](https://stats.oarc.ucla.edu/sas/seminars/sas-survival/)
    - SAS 用データファイルが配布されている
    - 19変数、500症例
    - ライセンスは記載なく不明

があります。

## 入手元による違い

筆者はこの中で R しか使えないので、まず Python 版、SAS 版をそれぞれ R で使用できるように読み込みます。

```r
# Python 版
# Gemini に作成してもらった Python コードをもとに reticulate で R に読み込む

# Python 環境に Pandas など頻用パッケージは導入済みとして、scikit-survival を追加
pacman::p_load("reticulate")
reticulate::py_install("scikit-survival")

# WHAS500 データセットを読み込んで、目的変数と説明変数を一つのデータフレームにまとめる
pd   <- reticulate::import("pandas")
skds <- reticulate::import("sksurv.datasets")
whas <- skds$load_whas500()
y    <- pd$DataFrame(whas[[2]]$tolist(), columns = whas[[2]]$dtype$names)
whas500_py <- cbind(whas[[1]], y)

# SAS 版
# UCLA OARC のサイトから whas500.sas7bdat を working directory にダウンロードしておく
# そのままだと変数名が大文字なので R, Python 版に揃えて小文字に変換
if (!("haven" %in% .packages(all.available = TRUE))) install.packages("haven")
whas500_sas <- haven::read_sas("whas500.sas7bdat", .name_repair = tolower)

# R 版は smoothHR パッケージの物をそのまま使う
if (!("smoothHR" %in% .packages(all.available = TRUE))) install.packages("smoothHR")
whas500_r <- smoothHR::whas500
```

3つのデータセットを比較します。

```r
pacman::p_load(tidyverse)

# データの行数、変数の数
tibble(name = paste0("whas500_", c("r", "py", "sas"))) %>%
  mutate(
    rows = map_int(name, \(x) dim(get(x))[1]),
    cols = map_int(name, \(x) dim(get(x))[2])
  )
```

|name        | rows| cols|
|:-----------|----:|----:|
|whas500_r   |  500|   22|
|whas500_py  |  500|   16|
|whas500_sas |  500|   19|

```r
# 各データに含まれる変数とデータの型
tibble(variables = names(whas500_r)) %>%
  mutate(
    R      = sapply(whas500_r  , class),
    # それぞれに存在するかチェック & R 版の順番に沿って並べ替える
    Python = sapply(whas500_py , class)[match(variables, names(whas500_py))],
    SAS    = sapply(whas500_sas, class)[match(variables, names(whas500_sas))]
  ) %>%
  # 削除されている変数には x をつける
  mutate(across(-variables, \(y) if_else(is.na(y), "x", y)))
```

|variables |R       |Python  |SAS     |
|:---------|:-------|:-------|:-------|
|id        |integer |x       |numeric |
|age       |integer |numeric |numeric |
|gender    |integer |factor  |numeric |
|hr        |integer |numeric |numeric |
|sysbp     |integer |numeric |numeric |
|diasbp    |integer |numeric |numeric |
|bmi       |numeric |numeric |numeric |
|cvd       |integer |factor  |numeric |
|afb       |integer |factor  |numeric |
|sho       |integer |factor  |numeric |
|chf       |integer |factor  |numeric |
|av3       |integer |factor  |numeric |
|miord     |integer |factor  |numeric |
|mitype    |integer |factor  |numeric |
|year      |integer |x       |numeric |
|admitdate |factor  |x       |x       |
|disdate   |factor  |x       |x       |
|fdate     |factor  |x       |x       |
|los       |integer |numeric |numeric |
|dstat     |integer |x       |numeric |
|lenfol    |integer |numeric |numeric |
|fstat     |integer |logical |numeric |


R 版（あるいは元データ）と比較して、Python 版や SAS 版では生の日付情報（入院日 `admitdate`, 退院日 `disdate`, 転帰日 `fdate`）が削除されています。後述のように、ここから計算される日数情報は別にありますので、昨今の個人情報保護の流れも踏まえて削除して取り扱った方が良さそうです。

また、Python 版では他に症例番号 `id` とコホート情報 `year` が省かれていますが、これは残すこととして情報量は SAS 版と同様の作業量データを作ることにします。

## 情報を整理

UMass Amherst のサイトにあった変数の解説文書 [^3] を基に、**GPL-3 ライセンスの R (smoothHR) 版を加工して、SAS 版と同様の情報量のデータを整えます。**

| 変数名 | 説明                          | 値の意味                     |
|:-------|:------------------------------|:-----------------------------|
| id     | 症例番号                      | 1 - 500 の整数               |
| gender | 性別                          | 0: 男性、1: 女性             |
| hr     | 初回の脈拍数                  | 整数（回/分）                |
| sysbp  | 初回の収縮期血圧              | 整数（mmHg）                 |
| diasbp | 初回の拡張期血圧              | 整数（mmHg）                 |
| bmi    | Body Mass Index               | 実数（kg/m^2）               |
| cvd    | 心血管疾患の既往              | 0:なし、1:あり               |
| afb    | 心房細動の有無                | 0:なし、1:あり               |
| sho    | 心原性ショックの有無          | 0:なし、1:あり               |
| chf    | うっ血性心不全の有無          | 0:なし、1:あり               |
| av3    | 完全房室ブロックの有無        | 0:なし、1:あり               |
| miord  | 心筋梗塞は初回か否か          | 0:初回、1:再発               |
| mitype | 心筋梗塞の型（異常Q波の有無） | 0:非Q波型、1:Q波型           |
| year   | コホート開始年                | 1:1997年、2:1999年、3:2001年 |
| los    | 入院期間                      | 整数（日）                   |
| dstat  | 退院時の状態                  | 0:生存退院、1:死亡退院       |
| lenfol | 総観察期間（入院～最終観察）  | 整数（日）                   |
| fstat  | 最終観察時の状態              | 0:生存、1:死亡               |

変数を取捨選択しただけのデータと、内容に応じて型や因子水準を設定したデータを作成します。

```r
# 再掲
whas500_r <- smoothHR::whas500

# UCLA OARC の SAS 版と同じ変数のみ
whas500_plain <-
  whas500_r %>%
  select(
    id, gender, hr, sysbp, diasbp, bmi, cvd, afb, sho, chf,
    av3, miord, mitype, year, los, dstat, lenfol, fstat
  )

# 名義変数、順序変数はできるだけ因子水準を具体的なものにする
whas500_modified <-
  whas500_plain %>%
  mutate(
    # 名義変数化
    gender = if_else(gender == 0, "Male"      , "Female"),
    miord  = if_else(miord  == 0, "First"     , "Recurrent"),
    mitype = if_else(mitype == 0, "non-Q-wave", "Q-wave"),
    # コホート開始年は順序変数 (factor) にしておく
    year   = factor(year, labels = c("1997", "1999", "2001")),
    # 0 / 1 のタイプはそのまま factor 型にする
    across(c(cvd:av3, dstat, fstat), factor)
  )
```

## 編集したデータセットの保存

```r
# RDS形式で保存
saveRDS(whas500_plain   , file = "whas500_plain.RDS")
saveRDS(whas500_modified, file = "whas500_modified.RDS")

# 読み込む際には任意のオブジェクト名をつけられる
data_whas500 <- readRDS("whas500_modified.RDS")

glimpse(data_whas500, width = 60)
## Rows: 500
## Columns: 18
## $ id     <int> 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, …
## $ gender <chr> "Male", "Male", "Female", "Male", "Male", "…
## $ hr     <int> 89, 84, 83, 65, 63, 76, 73, 91, 63, 104, 95…
## $ sysbp  <int> 152, 120, 147, 123, 135, 83, 191, 147, 209,…
## $ diasbp <int> 78, 60, 88, 76, 85, 54, 116, 95, 100, 106, …
## $ bmi    <dbl> 25.54051, 24.02398, 22.14290, 26.63187, 24.…
## $ cvd    <fct> 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 0…
## $ afb    <fct> 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1, 0…
## $ sho    <fct> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0…
## $ chf    <fct> 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, 0, 1, 0, 1, 0…
## $ av3    <fct> 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0…
## $ miord  <chr> "Recurrent", "First", "First", "First", "Fi…
## $ mitype <chr> "non-Q-wave", "Q-wave", "Q-wave", "Q-wave",…
## $ year   <fct> 1997, 1997, 1997, 1997, 1997, 1997, 1997, 1…
## $ los    <int> 5, 5, 5, 10, 6, 1, 5, 4, 4, 5, 5, 10, 7, 21…
## $ dstat  <fct> 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0…
## $ lenfol <int> 2178, 2172, 2190, 297, 2131, 1, 2122, 1496,…
## $ fstat  <fct> 0, 0, 0, 1, 0, 1, 0, 1, 1, 0, 0, 1, 0, 1, 0…
```

上記ファイルのダウンロード（from GitHub）：

- 変数選択のみ：[whas500_plain.RDS](https://github.com/mokztk/memorandum/raw/refs/heads/main/data/whas500_plain.RDS)
- 因子水準設定済：[whas500_modified.RDS](https://github.com/mokztk/memorandum/raw/refs/heads/main/data/whas500_modified.RDS)

:::message
このページで配布している `whas500_plain.RDS`, `whas500_modified.RDS` は R パッケージ [smoothHR](https://cran.r-project.org/package=smoothHR) に含まれるデータセットを基にしており、[GNU GPL v3](https://www.gnu.org/licenses/gpl-3.0.html) ライセンスの下で再配布されています。
:::


[^1]: Hosmer, D. W., Lemeshow, S., & May, S. (2008). Applied Survival Analysis: Regression Modeling of Time to Event Data (2nd ed.). John Wiley & Sons Inc., New York, NY.
[^2]: WayBack Machine : https://web.archive.org/web/20170114043458/http://www.umass.edu/statdata/statdata/data/
[^3]: WayBack Machine : https://web.archive.org/web/20170517071528/http://www.umass.edu/statdata/statdata/data/whas500.txt
