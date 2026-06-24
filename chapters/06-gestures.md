# 第6章：ジェスチャー操作

> 執筆者：ウカンセン
> 最終更新：2026-06-24

## この章で学ぶこと

この章では、ユーザーの指の動きを検出するジェスチャー認識の方法を学ぶ。タップ・ロングプレス・ドラッグ・拡大縮小・回転など、各ジェスチャーの実装方法を学び、最終的にTinder風のスワイプUIで複数のジェスチャーを組み合わせた実装を題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第6章（基本）：ジェスチャーで操作するカードアプリ
// ============================================
// タップ、ロングプレス、ドラッグ、ピンチ、回転の
// 各ジェスチャーを実際に体験しながら学びます。
// ============================================

import SwiftUI

// MARK: - メインビュー

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("タップ & ロングプレス") {
                    TapDemoView()
                }
                NavigationLink("ドラッグ") {
                    DragDemoView()
                }
                NavigationLink("ピンチ（拡大縮小）") {
                    MagnifyDemoView()
                }
                NavigationLink("回転") {
                    RotateDemoView()
                }
                NavigationLink("組み合わせ") {
                    CombinedDemoView()
                }
            }
            .navigationTitle("ジェスチャー体験")
        }
    }
}

// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}

// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}

// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}

// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、SwiftUIで使える基本的なジェスチャー操作を体験するためのアプリである。

タップ、ロングプレス、ドラッグ、ピンチによる拡大縮小、2本指による回転などを、画面上の図形やアイコンを操作しながら確認できる。また、複数のジェスチャーを組み合わせることで、画像を動かしながら拡大・回転するような操作も試すことができる。

スマートフォンアプリでは、ボタンを押すだけでなく、指で直接画面上の要素を操作する場面が多い。この章では、そのような直感的な操作を実装するための基本を学ぶ。

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
.onTapGesture {
    tapCount += 1
    backgroundColor = Color(
        hue: Double.random(in: 0...1),
        saturation: 0.7,
        brightness: 0.9
    )
}
```

```swift
.onLongPressGesture(minimumDuration: 1.0) {
    isPressed = true
    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
        isPressed = false
    }
}
```

**何をしているか：**
四角形をタップすると、タップ回数が増え、背景色がランダムに変わる。円を1秒以上長押しすると、色と大きさが変化し、「成功!」と表示される。

**なぜこう書くのか：**
タップや長押しは、アプリでよく使われる基本的な操作である。`.onTapGesture` を使うと、ビューがタップされたときの処理を簡単に書くことができる。また、`.onLongPressGesture(minimumDuration:)` を使うことで、ただ触っただけではなく、一定時間押し続けたときだけ反応させることができる。

**もしこう書かなかったら：**
`.onTapGesture` がなければ、四角形をタップしても回数や色は変わらない。`.onLongPressGesture` の `minimumDuration` を短くしすぎると、少し触っただけでも長押しとして判定されてしまい、誤操作が起きやすくなる。

---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero
@State private var lastOffset: CGSize = .zero
```

```swift
.offset(offset)
.gesture(
    DragGesture()
        .onChanged { value in
            offset = CGSize(
                width: lastOffset.width + value.translation.width,
                height: lastOffset.height + value.translation.height
            )
        }
        .onEnded { _ in
            lastOffset = offset
        }
)
```

**何をしているか：**
カードを指でドラッグすると、その動きに合わせてカードの位置が移動する。ドラッグが終わると、その位置を `lastOffset` に保存している。

**なぜこう書くのか：**
`DragGesture` の `translation` は、今のドラッグ操作でどれだけ動いたかを表している。しかし、次にもう一度ドラッグすると、translationはまた0から始まる。そのため、前回までの位置を `lastOffset` に保存し、今回の移動量と足し合わせる必要がある。

**もしこう書かなかったら：**
`lastOffset` を使わない場合、カードを何度もドラッグしたときに、前回の位置をうまく引き継げなくなる。新しくドラッグを始めたときに位置が不自然に戻ったり、動きが連続して見えなくなったりする。

---

### 拡大縮小と回転

```swift
@State private var scale: CGFloat = 1.0
@State private var lastScale: CGFloat = 1.0
```

```swift
.scaleEffect(scale)
.gesture(
    MagnifyGesture()
        .onChanged { value in
            scale = lastScale * value.magnification
        }
        .onEnded { _ in
            lastScale = scale
        }
)
```

```swift
@State private var angle: Angle = .zero
@State private var lastAngle: Angle = .zero
```

```swift
.rotationEffect(angle)
.gesture(
    RotateGesture()
        .onChanged { value in
            angle = lastAngle + value.rotation
        }
        .onEnded { _ in
            lastAngle = angle
        }
)
```

**何をしているか：**
ピンチ操作で星の大きさを変え、2本指の回転操作で矢印の向きを変えている。拡大縮小では `scale`、回転では `angle` を使って表示を変化させている。

**なぜこう書くのか：**
ドラッグと同じように、拡大縮小や回転も一回の操作ごとの差分を扱う必要がある。そのため、現在の値だけでなく、前回までの値を `lastScale` や `lastAngle` に保存している。これにより、操作を何度も繰り返しても、前の状態から続けて拡大・回転できる。

**もしこう書かなかったら：**
`lastScale` を使わないと、ピンチをやり直すたびに倍率の基準がずれてしまう。`lastAngle` を使わないと、回転を何度も行ったときに前回の角度を引き継げず、自然な操作にならない。

---

### ジェスチャーの組み合わせとアニメーション

```swift
Image(systemName: "photo.artframe")
    .font(.system(size: 120))
    .foregroundStyle(.indigo)
    .scaleEffect(scale)
    .rotationEffect(angle)
    .offset(offset)
```

```swift
Button("リセット") {
    withAnimation(.spring) {
        offset = .zero
        lastOffset = .zero
        scale = 1.0
        lastScale = 1.0
        angle = .zero
        lastAngle = .zero
    }
}
```

**何をしているか：**
画像に対して、ドラッグ、ピンチ、回転の効果を同時に反映している。また、リセットボタンを押すと、位置・倍率・角度を元の状態に戻している。

**なぜこう書くのか：**
実際のアプリでは、一つの操作だけでなく、複数の操作を組み合わせることが多い。例えば写真編集アプリでは、画像を移動しながら拡大したり、向きを変えたりする必要がある。そのため、`scaleEffect`、`rotationEffect`、`offset` を組み合わせて、複数の状態を同時に画面へ反映している。

**もしこう書かなかったら：**
どれか一つの効果しか使わない場合、画像を自由に操作できない。例えば `offset` がなければ移動できず、`scaleEffect` がなければ拡大縮小できず、`rotationEffect` がなければ回転できない。また、`withAnimation` を使わないと、リセット時の変化が急に切り替わり、少し不自然に見える。

---

## 新しく学んだSwiftの文法・API

| 項目                    | 説明                   | 使用例                                                 |
| --------------------- | -------------------- | --------------------------------------------------- |
| `.onTapGesture`       | ビューがタップされたときの処理を書く   | `.onTapGesture { tapCount += 1 }`                   |
| `.onLongPressGesture` | 一定時間押し続けたときの処理を書く    | `.onLongPressGesture(minimumDuration: 1.0) { ... }` |
| `DragGesture`         | 指でドラッグした動きを認識する      | `.gesture(DragGesture().onChanged { ... })`         |
| `MagnifyGesture`      | ピンチ操作による拡大縮小を認識する    | `.gesture(MagnifyGesture().onChanged { ... })`      |
| `RotateGesture`       | 2本指の回転操作を認識する        | `.gesture(RotateGesture().onChanged { ... })`       |
| `.onChanged`          | ジェスチャー中に何度も呼ばれる処理を書く | `.onChanged { value in ... }`                       |
| `.onEnded`            | ジェスチャーが終わったときの処理を書く  | `.onEnded { _ in lastOffset = offset }`             |
| `.offset`             | ビューの表示位置をずらす         | `.offset(offset)`                                   |
| `.scaleEffect`        | ビューを拡大・縮小する          | `.scaleEffect(scale)`                               |
| `.rotationEffect`     | ビューを回転させる            | `.rotationEffect(angle)`                            |

## 自分の実験メモ

**実験1：**

* やったこと：ドラッグの `onEnded` の中にある `lastOffset = offset` を削除した。
* 結果：一回目のドラッグでは動くが、次にドラッグしたときに前回の位置をうまく引き継げなかった。
* わかったこと：`onEnded` で最後の位置を保存しておかないと、次のドラッグ操作につながらないことが分かった。

**実験2：**

* やったこと：リセットボタンの `withAnimation(.spring)` を外した。
* 結果：位置や大きさは元に戻るが、急に切り替わるように見えた。
* わかったこと：`withAnimation` を使うと、状態の変化が自然に見えることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：なぜドラッグでは `offset` だけでなく `lastOffset` も必要なのか？**
   **得られた理解：**
   `offset` は画面上の現在位置を表し、`lastOffset` は前回のドラッグが終わった位置を保存するために使う。`DragGesture` の `translation` は毎回0から始まるため、前回までの位置を足さないと自然に続けて動かすことができない。

2. **質問：`onChanged` と `onEnded` はそれぞれいつ使い分けるのか？**
   **得られた理解：**
   `onChanged` は指を動かしている最中に何度も呼ばれるため、画面をリアルタイムに更新する処理に使う。`onEnded` は指を離したときに一度だけ呼ばれるため、最後の位置や倍率、角度を保存する処理に向いている。

3. **質問：なぜ拡大縮小や回転でも `lastScale` や `lastAngle` が必要なのか？**
   **得られた理解：**
   ピンチや回転も、ドラッグと同じように一回の操作ごとの差分を扱っている。そのため、前回までの倍率や角度を保存しておかないと、操作を繰り返したときに前の状態から自然に続けることができない。

## この章のまとめ

この章では、SwiftUIでタップ、長押し、ドラッグ、拡大縮小、回転などのジェスチャーを扱う方法を学んだ。ジェスチャーでは、指の動きに合わせて画面をリアルタイムに変化させることが重要である。

特に、ドラッグや拡大縮小、回転では、現在の状態と前回までの状態を分けて管理する必要があると分かった。`offset` と `lastOffset`、`scale` と `lastScale`、`angle` と `lastAngle` のように、操作中の値と確定した値を分けることで、自然な操作感を作ることができる。
