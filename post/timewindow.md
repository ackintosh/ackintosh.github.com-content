---
title: "TimeWindowの種類"
date: 2018-01-25T10:14:04+09:00
draft: false
description: TimeWindowが処理対象とする枠を決めるためのロジックにはいくつか種類があるのでまとめる。できるだけ一般的と思われるものをまとめたが、個別のサービスやプロダクトによって違いがあるかもしれないのでご了承いただきたい。
---

## TimeWindowとは

ググってみた感じだと(ソフトウェア開発以外も含めた)文脈によっていくつか微妙に異なる意味がありそうなのだが、当記事では "ある測定の対象となる時間枠" の意味で扱う。例えば、システムで発生したイベントを集めて加工を行うようなストリーム処理において、どこからどこまでのイベントを対象とするかを決定するのがTimeWindowである。

<!--more-->

TimeWindowが処理対象とする枠を決めるためのロジックにはいくつか種類があるので以下にまとめる。できるだけ一般的と思われるものをまとめたが、個別のサービスやプロダクトによって違いがあるかもしれないのでご了承いただきたい。  
([Stream Analytics ウィンドウ関数の概要 | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/stream-analytics/stream-analytics-window-functions#tumbling-window)にある画像がわかりやすかったので引用させていただいている)

## Tumbling Window

最もシンプルなロジックで、時間軸を任意の間隔( = ウィンドウサイズ )で区切る。1つのイベントは必ず1つのTimeWindowに属する。

![Tumbling Window](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/timewindow/tumbling-window.png)

引用元: [Stream Analytics ウィンドウ関数の概要 | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/stream-analytics/stream-analytics-window-functions#tumbling-window)


## Hopping Window

Tumbling Windowと似ているが、ウィンドウサイズの他にホップサイズという変数が存在する。ホップサイズは、枠と枠の間隔を表す。(文章で説明するのが難しい。)1つのイベントは1つ以上のTimeWindowに属する。

![Hopping Window](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/timewindow/hopping-window.png)

引用元: [Stream Analytics ウィンドウ関数の概要 | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/stream-analytics/stream-analytics-window-functions#hopping-window)

## Sliding Window

名称からしてイメージしやすいかもしれないが、"直近○○のイベント" を処理対象とする場合に使いやすい。
Sliding Windowは `Eviction policy` と `Trigger policy` という2種類のポリシー(特性)を持つ。開発者はこれらを組み合わせて柔軟なウィンドウ処理を行う。

![Sliding Window](https://s3-ap-northeast-1.amazonaws.com/ackintosh.github.io/timewindow/sliding-window.png)

引用元: [Stream Analytics ウィンドウ関数の概要 | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/stream-analytics/stream-analytics-window-functions#sliding-window)

### Trigger Policy (トリガーポリシー)

"直近"という枠を扱うには何らかのトリガーが必要になる。Trigger Policyで、何を軸にしてウィンドウ処理を起動するかを決める。

- Time Trigger Policy
    - N秒間経過するごとに起動する、といったように時間を軸にして起動する
- Count Trigger Policy
    - N個イベントが発生するごとに起動する、といったように数を軸にして起動する

### Eviction Policy (排他ポリシー)

何を軸にしてイベントを集めるか( = 範囲を外れたイベントを排除するか)を決めるポリシー。

- Time Eviction Policy
    - 直近N秒間、といったように時間を軸に集める
- Count Eviction Policy
    - 直近N個、といったように数を軸にして集める

### ポリシーの組み合わせの例

##### Time Trigger Policy x Time Eviction Policy

60秒ごと(Trigger Policy)に、直近10秒間(Eviction Policy)のイベントを集めて行うウィンドウ処理を起動する。

##### Time Trigger Policy x Count Eviction Policy

60秒ごと(Trigger Policy)に、直近30個(Eviction Policy)のイベントを集めて行うウィンドウ処理を起動する。

---

#### 参考

- [Stream Analytics ウィンドウ関数の概要 | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/stream-analytics/stream-analytics-window-functions#tumbling-window)
- [SPL Sliding Windows Explained - Streamsdev](https://developer.ibm.com/streamsdev/2014/08/22/spl-sliding-windows-explained/)