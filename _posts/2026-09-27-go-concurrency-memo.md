---
layout: post
title: "【Go言語】並行処理の勉強メモ：WaitGroup、ループ変数の落とし穴、Mutex"
date: 2026-09-27 22:43:00 +0900
categories: [Go, 並行処理]
tags: [go, goroutine, waitgroup, mutex]
---

最近Go言語の並行処理（ゴルーチン）について勉強しています！
手軽に並行処理が書けてめちゃくちゃ便利な反面、気をつけないといけないポイント（ポインタ渡しやデータ競合など）がいくつかあったので、今後の自分のために備忘録として残しておきます📝

---

## 1. sync.WaitGroup (ゴルーチンの待機)
メインプロセスが先に終了してしまうと、実行中のゴルーチンも道連れで強制終了されてしまいます。それを防ぐために、すべてのゴルーチンの完了を待ち合わせるのが `sync.WaitGroup` です。

**基本的な使い方**:

- `wg.Add(n)`: 待機するゴルーチンの数を設定します。ループに入る前など、起動する総数が分かっている場合はまとめて `Add` する方が効率的みたいです。
- `defer wg.Done()`: ゴルーチンの処理が終わった時に、完了を通知してカウントを1減らします。
- `wg.Wait()`: 全てのゴルーチンの完了通知が届き、カウントが0になるまでメインプログラムを一時停止（ブロック）して待機してくれます。

### ⚠️ WaitGroupを引数で渡す際の注意点（ポインタ渡し）
これ、初心者がすごくハマりやすいポイントらしいのですが…

WaitGroupを別関数に引数として渡すときは、**必ずポインタ (`*sync.WaitGroup`) として渡す**必要があります。呼び出すときは `&wg` のようにアドレスを渡します。

うっかり値渡し（コピー）してしまうと、うまくカウントが共有されずデッドロック（エラー）になってしまうので要注意です！

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // WaitGroupをポインタで受け取る関数
    hello := func(wg *sync.WaitGroup, id int) {
        defer wg.Done()
        fmt.Printf("Hello from goroutine %v\n", id)
    }

    const numGoroutines = 5
    var wg sync.WaitGroup
    wg.Add(numGoroutines) // まとめて追加

    for i := 0; i < numGoroutines; i++ {
        go hello(&wg, i) // WaitGroupのアドレス(&)とループ変数(i)を渡す
    }

    wg.Wait() // 全て終わるまで待機
    fmt.Println("All done!")
}
```

## 2. ゴルーチンとループ変数のキャプチャ（Goの落とし穴）
`for` ループの中でループ変数を直接参照すると、昔のGoバージョン（Go 1.21以前）ではすべてのゴルーチンがループの「最後の値」を出力してしまうという有名な落とし穴がありました。

これを安全に回避するには、**ループ変数をゴルーチンの引数として渡す**のがベストプラクティスです。

※現在（Go 1.22以降）はこの仕様が修正されて直っているらしいですが、古いバージョンのコードを読むときのためにも覚えておいた方が良さそうです！

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    greetings := []string{"Hello", "Hi", "Hey"}

    for _, greeting := range greetings {
        wg.Add(1)
        // 古いGoでも確実に動く安全な書き方
        go func(g string) {
            defer wg.Done()
            fmt.Println(g) // 渡された引数を使う
        }(greeting) // ここで現在のループの値を渡す
    }

    wg.Wait()
}
```

## 3. sync.Mutex (排他制御・データ競合の防止)
複数のゴルーチンが同じ変数（共有状態）に同時にアクセスして変更を加えようとすると、更新が上書きされたり計算が狂ったりする **データ競合（Data Race）** が発生してしまいます。

そこで `sync.Mutex` を使うことで、「いまこの変数を操作していいのは1つのゴルーチンだけ！」という排他制御（ロック）を行うことができます。

**基本的な使い方**:

- `lock.Lock()`: 処理を始める前にカギをかけます。
- `defer lock.Unlock()`: 処理が完了したら、関数を抜ける際に必ずカギを開けます（ロックを解除）。`defer` を使うことで、カギの開け忘れを防止できます。

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var count int
    var lock sync.Mutex
    var wg sync.WaitGroup

    increment := func() {
        defer wg.Done()
        lock.Lock()         // カギをかける
        defer lock.Unlock() // カギを開ける
        count++             // 安全に値を更新
    }

    // 10個のゴルーチンが一斉にcountを書き換えようとする
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go increment()
    }

    wg.Wait()
    fmt.Printf("Final count: %d\n", count) // データ競合なく確実に 10 になる
}
```

---
並行処理、最初は少し難しく感じますが、仕組みが分かってくるとパズルみたいで面白いですね！