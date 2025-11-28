---
title: "パイプ演算子の挙動"
author: "@mokztk"
date-format: iso
date: "2025-11-28"
format:
  html: 
    fontsize: normal
    page-layout: full
    toc: true
    toc-depth: 3
    html-math-method: katex
    fig-format: svg
    fig-height: 5
    fig-width: 9
    self-contained: true
    number-sections: false
    code-fold: false
    code-block-border-left: true
    code-line-numbers: false
    code-overflow: wrap
    highlight-style: atom-one
    df-print: kable
categories:
  - R
  - mJOHNSNOW
---

R のパイプ演算子（`|>`, `%>%`）の挙動をあらためて確認します。

本記事は、mJOHNSNOW R学習ピア勉強会 vol.4 の発表資料から抜粋・加筆したものです。

## パイプ演算子の基本

現在、R では主に R 4.2 以降の base pipe (`|>`) と tidyverse 環境で使われる magrittr pipe (`%>%`) の 2種類のパイプ演算子が使用されています。パイプ演算子はどちらも **パイプの左側にあるものを右側の関数の1つ目のパラメーター（引数）に挿入し、元々あった引数を1つずつ後ろにずらす** のが基本動作となります。

R の関数に渡される引数は、

1. 引数名を指定した場合は、指定した引数に値が割り当てられる
2. 引数名を書かないものは、関数定義の順に当てはめられる

のルールで決まりますが、**パイプ演算子は原則として「引数名を書かないもの」の一番最初**に挿入されます。

この挙動により、パイプを受け取る側の関数で指定していた引数が想定したものとズレてしまいエラーになることがあります。できるだけ、パイプを使用した記載のときは[引数名を省略せずに記載]{style="text-decoration:underline;"}するようにした方が安全です。

## 動作の確認

### 確認のための準備

関数がどのように呼び出されたかを表示する `match.call()` を使います。


::: {.cell}

```{.r .cell-code}
# 基本
f <- function(x, y, z, ...) match.call()

# g() は y が数値でないとエラーになる関数として作成
g <- function(x, y = 1, z, ...) {
  if (!is.numeric(y)) message("ERROR: 'y' must be numeric.")
  match.call()
}

# テスト用のデータフレーム
data <- data.frame(name = c("A", "B"), value = c(12, 34)) 
```
:::


### Base pipe の場合


::: {.cell}

```{.r .cell-code}
# パラメーター（引数）名を書かないと関数定義の順に当てはめられる
f("p1", "p2", "p3")
## f(x = "p1", y = "p2", z = "p3")

# 引数名を指定すると、記載順にかかわらず指定した引数に割り当てられる
f(z = "p1", y = "p2", "p3")
## f(x = "p3", y = "p2", z = "p1")

# 引数名を指定せずパイプで受け渡す場合
# > ひとつめの引数に挿入され、元々あったものはひとつずつずれる
f("param", "this is 'y'", "this is 'z'")
## f(x = "param", y = "this is 'y'", z = "this is 'z'")

data |> f("param", "is this 'y'?", "is this 'z'?")
## f(x = data, y = "param", z = "is this 'y'?", "is this 'z'?")

# 多段階のパイプライン
# > 左から順に f() に data、g() には f() の結果が挿入される
data |> f("p1") |> g("p2", 123)
## ERROR: 'y' must be numeric.
## g(x = f(data, "p1"), y = "p2", z = 123)

# ズレないためには引数名を指定する
# > パイプの左側は指定されていない一番最初の引数に挿入される
data |> f(x = "this must be 'x'")
## f(x = "this must be 'x'", y = data)

# 関数定義にない可変変数（...）の部分には、名前つきのものがあっても可変変数の1番目に入る
data |> f(x = "X-ray", y = "Yankee", z = "Zulu", a = "Alpha", "Bravo")
## f(x = "X-ray", y = "Yankee", z = "Zulu", data, a = "Alpha", "Bravo")
```
:::


Base pipe はパイプ演算子の左側を、右側の関数の1つ目（特に指定しない場合）の引数に**そのまま**渡しています。

記載を簡単にして、実際の実行時には本来の文法に変換される所謂「**糖衣構文 Syntax suger**」として機能していることがわかります。

### Magrittr pipe の場合


::: {.cell}

```{.r .cell-code}
#library(tidyverse)
library(magrittr)

# 引数名を指定せずパイプで受け渡す場合
# > ひとつめの引数に . として挿入され、元々あったものはひとつずつずれる
data %>% f("param", "is this 'y'?", "is this 'z'?")
## f(x = ., y = "param", z = "is this 'y'?", "is this 'z'?")

# ズレないためには引数名を指定する
# > パイプの左側は指定されていない一番最初の引数に挿入される
data %>% f(x = "this must be 'x'")
## f(x = "this must be 'x'", y = .)

# 関数定義にない可変変数の部分には、名前つきのものがあっても可変変数の1番目に入る
data %>% f(x = "X-ray", y = "Yankee", z = "Zulu", a = "Alpha", "Bravo")
## f(x = "X-ray", y = "Yankee", z = "Zulu", ., a = "Alpha", "Bravo")

## ここまでは base pipe と同じ

# 多段階のパイプライン
# > 左から順に処理され、g() には . にまとめられた結果が挿入される
data %>% f("p1") %>% g("p2", 123)
## ERROR: 'y' must be numeric.
## g(x = ., y = "p2", z = 123)
```
:::


Magrittr pipe はパイプ演算子の左側を、**`.` という特別なオブジェクトとして**右側の関数の1つ目（特に指定しない場合）の引数に渡しています。

`.` は呼び出された右側の関数内では展開されて元のパイプ演算子の左側の処理の結果として取り扱われます。


::: {.cell}

```{.r .cell-code}
# 関数内での引数の取り扱いを確認するための関数 h() を用意
h <- function(x, y, ...) {
  # 引数の内容
  args <- list(
    "x"   = environment()$x,
    "y"   = environment()$y
  )
  # 可変変数があれば取得して追加
  args <- append(args, list(...))
  
  list(
    "Call" = match.call(),
    "Args" = args
  )
}

# 入れ子構造の確認のため、別名コピーを作成
h2 <- h
```
:::



::: {.cell}

```{.r .cell-code}
# h() の結果は階層のある list なので、str() で整理して表示する

data %>% h() %>% str()
## List of 2
##  $ Call: language h(x = .)
##  $ Args:List of 2
##   ..$ x:'data.frame':	2 obs. of  2 variables:
##   .. ..$ name : chr [1:2] "A" "B"
##   .. ..$ value: num [1:2] 12 34
##   ..$ y: symbol

# 関数定義にある引数 x, y が指定されている場合は、可変変数の最初 = 3番目に挿入される
data %>% h(x = 123, y = "this is 'y'", "unnamed_1") %>% str()
## List of 2
##  $ Call: language h(x = 123, y = "this is 'y'", ., "unnamed_1")
##  $ Args:List of 4
##   ..$ x: num 123
##   ..$ y: chr "this is 'y'"
##   ..$  :'data.frame':	2 obs. of  2 variables:
##   .. ..$ name : chr [1:2] "A" "B"
##   .. ..$ value: num [1:2] 12 34
##   ..$  : chr "unnamed_1"

# 多段階のパイプライン
data %>% h(123) %>% h2() %>% str()
## List of 2
##  $ Call: language h2(x = .)
##  $ Args:List of 2
##   ..$ x:List of 2
##   .. ..$ Call: language h(x = ., y = 123)
##   .. ..$ Args:List of 2
##   .. .. ..$ x:'data.frame':	2 obs. of  2 variables:
##   .. .. .. ..$ name : chr [1:2] "A" "B"
##   .. .. .. ..$ value: num [1:2] 12 34
##   .. .. ..$ y: num 123
##   ..$ y: symbol
```
:::


3例目の多段階のパイプラインでは、

- `h2(x = .)` の . は `h(x = ., y = 123)`
- `h(x = ., y = 123)` の . は データフレーム `data` 

にそれぞれ展開されています。


::: {.cell}

```{.r .cell-code}
data |> h(123) |> h2() |> str()
## List of 2
##  $ Call: language h2(x = h(data, 123))
##  $ Args:List of 2
##   ..$ x:List of 2
##   .. ..$ Call: language h(x = data, y = 123)
##   .. ..$ Args:List of 2
##   .. .. ..$ x:'data.frame':	2 obs. of  2 variables:
##   .. .. .. ..$ name : chr [1:2] "A" "B"
##   .. .. .. ..$ value: num [1:2] 12 34
##   .. .. ..$ y: num 123
##   ..$ y: symbol
```
:::


Base pipe の場合も関数内では同じように展開されているので、どちらを採用しても同じように使うことができます。

## 2つのパイプの違い

1番目の引数以外にパイプの左側の結果を受け渡したい場合は、特殊な記号 (placeholder) を使います。

Base pipe では `.` をそのまま、Magrittr pipe では `_` を使いますが、細かい違いがあります。

### Base pipe の場合


::: {.cell}

```{.r .cell-code}
# パイプの左側を反映する引数を指定する場合は placeholder を使う
data |> f("param", z = _)
## f(x = "param", z = data)

# base pipe の _ は名前付きの引数のみ
data |> f("param", _)
## Error:
## ! 
## Failed to parse input: Error evaluating 'f("param", "_")': pipe placeholder can only be used as a named argument (<input>:1:9)

# base pipe の _ は 1箇所だけ
data |> f(x = _, y = "param", z = _)
## Error:
## ! 
## Failed to parse input: Error evaluating 'f(x = "_", y = "param", z = "_")': pipe placeholder may only appear once (<input>:1:9)
```
:::


### Magrittr pipe の場合


::: {.cell}

```{.r .cell-code}
# 基本の動作は同じ
data %>% f("param", z = .)
## f(x = "param", z = .)

# magrittr pipe の . は引数名を指定しなくてもよい
data %>% f("param", .)
## f(x = "param", y = .)
data %>% f(x = 123, y = "this is 'y'", z = "Z", .)
## f(x = 123, y = "this is 'y'", z = "Z", .)

# magrittr pipe の . は複数可能
data %>% f(x = ., y = "param", z = .)
## f(x = ., y = "param", z = .)

# magrittr pipe の右側は関数でなくても良い（. というオブジェクトで送られるため）
data %>% h(x = ., y = "param", z = .) %>% .$Args
## $x
##   name value
## 1    A    12
## 2    B    34
## 
## $y
## [1] "param"
## 
## $z
##   name value
## 1    A    12
## 2    B    34
```
:::


### return()

また、base pipe は右側に `return()` を使用することができません。


::: {.cell}

```{.r .cell-code}
data |> return()
## Error:
## ! 
## Failed to parse input: Error evaluating 'return()': function 'return' not supported in RHS call of a pipe (<input>:1:9)

data %>% return()
##   name value
## 1    A    12
## 2    B    34
```
:::

