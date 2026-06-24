# 第6章：ジェスチャー操作

> 執筆者：ウカンセン
> 最終更新：2026-06-24

## この章で学ぶこと

この章では、ユーザーの指の動きを検出するジェスチャー認識の方法を学ぶ。タップ・ロングプレス・ドラッグ・拡大縮小・回転など、各ジェスチャーの実装方法を学び、最終的にTinder風のスワイプUIで複数のジェスチャーを組み合わせた実装を題材にする。

## 模範コードの全体像

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

# 第6章：ジェスチャー操作・応用

## この章で学ぶこと

応用編では、ドラッグジェスチャーを使って、Tinder風のスワイプカードUIを作る。カードを左右に動かし、一定以上スワイプされたらLIKEまたはNOPEとして判定する仕組みを学ぶ。
また、ジェスチャー操作では、現在の位置・倍率・角度だけでなく、前回の操作結果を保存しておくことが重要である。offset と lastOffset、scale と lastScale、angle と lastAngle のように、操作中の値と確定した値を分けて管理する考え方を理解する。


## 模範コードの全体像

```swift
// ============================================
// 第6章（応用）：Tinder風スワイプカードUI
// ============================================
// ドラッグジェスチャーとアニメーションを組み合わせて、
// カードを左右にスワイプして仕分けるUIを作ります。
// ============================================

import SwiftUI

// MARK: - データモデル

struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}

extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink),
        Animal(name: "ペンギン", emoji: "🐧", description: "南極のタキシード紳士", color: .cyan),
        Animal(name: "パンダ", emoji: "🐼", description: "笹が大好きなのんびり屋", color: .green),
        Animal(name: "フクロウ", emoji: "🦉", description: "夜型の知恵者", color: .purple),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？")
                .font(.title2)
                .bold()

            // スコア表示
            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            // カードスタック
            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！")
                            .font(.largeTitle)

                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    ForEach(animals.reversed()) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            // 手動ボタン
            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.last {
                            removeCard(animal: top, direction: .left)
                        }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }

                    Button {
                        if let top = animals.last {
                            removeCard(animal: top, direction: .right)
                        }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }

        switch direction {
        case .left:
            dislikedAnimals.append(animal)
        case .right:
            likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            // カード背景
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            // カード内容
            VStack(spacing: 16) {
                Text(animal.emoji)
                    .font(.system(size: 80))

                Text(animal.name)
                    .font(.title)
                    .bold()

                Text(animal.description)
                    .font(.body)
                    .foregroundStyle(.secondary)
            }

            // いいね / NG オーバーレイ
            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .position(x: 80, y: 60)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        // 右スワイプ → LIKE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        // 左スワイプ → NOPE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        // 元に戻す
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}

#Preview {
    ContentView()
}
```

### このアプリは何をするものか

この応用編では、動物カードを左右にスワイプして、「好き」と「好きではない」に仕分けるアプリを作る。

カードを右にスワイプすると `LIKE` として記録され、左にスワイプすると `NOPE` として記録される。また、画面下のボタンから手動で「いいね」や「NG」を選ぶこともできる。すべてのカードを仕分けると「完了！」と表示され、もう一度最初から試すことができる。

基本編で学んだドラッグジェスチャーを、実際のアプリらしいUIに応用した例である。

---

## コードの詳細解説

### データモデルとサンプルデータ

```swift
struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}
```

```swift
extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink)
    ]
}
```

**何をしているか：**
動物カードに表示する情報を `Animal` という構造体でまとめている。名前、絵文字、説明文、カードの色を一つのデータとして扱っている。

**なぜこう書くのか：**
カードに表示する内容をバラバラの変数で管理すると、どの名前とどの説明が対応しているのか分かりにくくなる。`Animal` として一つにまとめることで、カード一枚分の情報を扱いやすくしている。また、`Identifiable` に準拠しているため、`ForEach` でカード一覧を表示するときに、それぞれのカードを区別できる。

**もしこう書かなかったら：**
`Identifiable` がないと、`ForEach` で複数のカードを表示するときに、SwiftUIがどのカードがどのデータなのか判断しにくくなる。また、名前や説明文を別々の配列で管理すると、データの対応関係が崩れやすくなる。

---

### カードの状態管理

```swift
@State private var animals: [Animal] = Animal.sampleData
@State private var likedAnimals: [Animal] = []
@State private var dislikedAnimals: [Animal] = []
```

**何をしているか：**
まだ表示するカード、好きとして選んだカード、好きではないとして選んだカードを、それぞれ別の配列で管理している。

**なぜこう書くのか：**
スワイプUIでは、現在残っているカードと、すでに仕分けた結果を分けて管理する必要がある。`animals` からカードを削除し、右スワイプなら `likedAnimals`、左スワイプなら `dislikedAnimals` に追加することで、画面上のカード枚数とスコア表示を連動させている。

**もしこう書かなかったら：**
残りカードと結果を同じ配列で管理すると、どのカードがまだ未処理で、どのカードがすでに選ばれたのか分かりにくくなる。また、好き・NGの数を画面に表示することも難しくなる。

---

### カードスタックの表示

```swift
ZStack {
    if animals.isEmpty {
        VStack(spacing: 12) {
            Text("完了！")
                .font(.largeTitle)

            Button("もう一度") {
                animals = Animal.sampleData.shuffled()
                likedAnimals = []
                dislikedAnimals = []
            }
            .buttonStyle(.borderedProminent)
        }
    } else {
        ForEach(animals.reversed()) { animal in
            SwipeCardView(animal: animal) { direction in
                removeCard(animal: animal, direction: direction)
            }
        }
    }
}
.frame(height: 400)
```

**何をしているか：**
`ZStack` を使って、複数のカードを重ねて表示している。カードが残っている場合は `SwipeCardView` を表示し、すべてのカードがなくなった場合は「完了！」と「もう一度」ボタンを表示する。

**なぜこう書くのか：**
Tinder風のUIでは、カードが一枚ずつ並ぶのではなく、上に重なって表示される必要がある。そのため、縦や横に並べる `VStack` や `HStack` ではなく、重ねて表示する `ZStack` を使っている。また、カードがなくなったときの画面も同じ場所に表示することで、自然に完了画面へ切り替えられる。

**もしこう書かなかったら：**
`ZStack` を使わないと、カードが一覧のように並んでしまい、スワイプカードUIらしく見えなくなる。また、`animals.isEmpty` の判定がないと、すべてのカードを仕分けた後に何も表示されず、ユーザーが次に何をすればよいか分かりにくくなる。

---

### `animals.reversed()` と一番上のカード

```swift
ForEach(animals.reversed()) { animal in
    SwipeCardView(animal: animal) { direction in
        removeCard(animal: animal, direction: direction)
    }
}
```

```swift
if let top = animals.last {
    removeCard(animal: top, direction: .left)
}
```

**何をしているか：**
`animals.reversed()` を使って、配列の後ろにあるカードが画面の上に見えるようにしている。手動ボタンでは、画面で一番上に見えている `animals.last` のカードを削除している。

**なぜこう書くのか：**
`ZStack` では、後から描画されたビューほど前面に表示される。そのため、配列をそのまま表示すると、意図したカードの重なり順にならない場合がある。`animals.reversed()` で表示順を調整し、さらに手動ボタンでは `animals.last` を対象にすることで、画面の一番上に見えているカードと削除されるカードを一致させている。

**もしこう書かなかったら：**
表示されているカードと、実際に削除されるカードがずれてしまうことがある。たとえば、画面ではネコのカードが一番上に見えているのに、ボタンを押すと下に隠れている別のカードが消えてしまう。この場合、ユーザーの操作とアプリの動きが一致しないため、不自然なUIになる。

---

### スワイプ方向の管理

```swift
enum SwipeDirection {
    case left, right
}
```

```swift
func removeCard(animal: Animal, direction: SwipeDirection) {
    withAnimation(.spring(duration: 0.3)) {
        animals.removeAll { $0.id == animal.id }
    }

    switch direction {
    case .left:
        dislikedAnimals.append(animal)
    case .right:
        likedAnimals.append(animal)
    }
}
```

**何をしているか：**
左スワイプか右スワイプかを `SwipeDirection` で表し、方向に応じてカードを `likedAnimals` または `dislikedAnimals` に追加している。

**なぜこう書くのか：**
左と右を文字列や数字で表すと、意味が分かりにくく、間違いも起きやすい。`SwipeDirection.left`、`SwipeDirection.right` のように書くことで、どちらの方向なのかがコード上で分かりやすくなる。

**もしこう書かなかったら：**
方向を `"left"` や `"right"` のような文字列で管理すると、スペルミスをしても気づきにくい。数字で管理すると、`0` が左なのか右なのか読み返したときに分かりにくくなる。`enum` を使うことで、コードの意味がはっきりする。

---

### スワイプ中のカードの動き

```swift
@State private var offset: CGSize = .zero
@State private var rotation: Double = 0

private let swipeThreshold: CGFloat = 100
```

```swift
.gesture(
    DragGesture()
        .onChanged { value in
            offset = value.translation
            rotation = Double(value.translation.width / 20)
        }
)
```

**何をしているか：**
ユーザーがカードをドラッグしている間、カードの位置を `offset` で動かし、横方向の移動量に応じて `rotation` で少し傾けている。また、`swipeThreshold` は、どのくらい横に動かしたらスワイプ成功と判定するかの基準になっている。

**なぜこう書くのか：**
カードが指の動きに合わせて移動することで、ユーザーは自分がカードを操作していることを直感的に理解できる。また、横に動かした量に応じてカードを少し回転させることで、実際にカードを払っているような見た目になる。`swipeThreshold` を用意することで、少し触っただけではカードが消えず、一定以上スワイプしたときだけ判定できる。

**もしこう書かなかったら：**
`offset` がなければ、指でドラッグしてもカードが動かない。`rotation` がなければ動きは分かるが、カードを払うような自然な見た目が弱くなる。また、`swipeThreshold` が小さすぎると誤操作が増え、大きすぎると強くスワイプしないと反応しなくなる。

---

### LIKE / NOPE オーバーレイ

```swift
private var swipeProgress: CGFloat {
    min(abs(offset.width) / swipeThreshold, 1.0)
}
```

```swift
if offset.width > 0 {
    Text("LIKE")
        .font(.system(size: 40, weight: .bold))
        .foregroundStyle(.green)
        .opacity(swipeProgress)
        .rotationEffect(.degrees(-20))
        .position(x: 80, y: 60)
} else if offset.width < 0 {
    Text("NOPE")
        .font(.system(size: 40, weight: .bold))
        .foregroundStyle(.red)
        .opacity(swipeProgress)
        .rotationEffect(.degrees(20))
        .position(x: 240, y: 60)
}
```

**何をしているか：**
カードを右に動かすと `LIKE`、左に動かすと `NOPE` の文字を表示している。横方向に大きく動かすほど、文字がはっきり表示される。

**なぜこう書くのか：**
スワイプ中に現在の判定方向が見えると、ユーザーは自分の操作が「好き」なのか「NG」なのか分かりやすい。`swipeProgress` を使って透明度を変えることで、少し動かしたときは薄く、大きく動かしたときは濃く表示できる。

**もしこう書かなかったら：**
LIKEやNOPEの表示がない場合、カードを動かしている途中で、どちらの判定になるのか分かりにくい。`opacity` を固定にすると、少し動かしただけでも文字が強く表示されてしまい、動きの変化が分かりにくくなる。

---

### スワイプ終了時の判定

```swift
.onEnded { value in
    if value.translation.width > swipeThreshold {
        withAnimation(.easeOut(duration: 0.3)) {
            offset = CGSize(width: 500, height: 0)
        }
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
            onSwipe(.right)
        }
    } else if value.translation.width < -swipeThreshold {
        withAnimation(.easeOut(duration: 0.3)) {
            offset = CGSize(width: -500, height: 0)
        }
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
            onSwipe(.left)
        }
    } else {
        withAnimation(.spring) {
            offset = .zero
            rotation = 0
        }
    }
}
```

**何をしているか：**
指を離したときに、カードの移動量を見て右スワイプか左スワイプかを判定している。しきい値を超えていればカードを画面外へ飛ばし、その後で `onSwipe` を呼んでカードを削除する。しきい値に届かなければ、カードを元の位置に戻している。

**なぜこう書くのか：**
スワイプ操作では、指を動かしている途中ではなく、指を離した瞬間に最終判定を行う必要がある。`swipeThreshold` を使うことで、少しだけ動かした場合はキャンセルし、十分に動かした場合だけLIKEやNOPEとして処理できる。また、すぐにカードを削除せず、先に画面外へ飛ぶアニメーションを見せることで、操作結果が自然に見える。

**もしこう書かなかったら：**
`onEnded` で判定しないと、カードをどのタイミングで削除するのか決められない。`DispatchQueue.main.asyncAfter` を使わずにすぐ削除すると、カードが画面外へ飛ぶアニメーションを見る前に消えてしまう。逆に、しきい値判定がないと、少し横に動かしただけでもLIKEやNOPEになってしまう。

---

### 手動ボタンによる操作

```swift
Button {
    if let top = animals.last {
        removeCard(animal: top, direction: .left)
    }
} label: {
    Image(systemName: "xmark.circle.fill")
        .font(.system(size: 50))
        .foregroundStyle(.red)
}
```

```swift
Button {
    if let top = animals.last {
        removeCard(animal: top, direction: .right)
    }
} label: {
    Image(systemName: "heart.circle.fill")
        .font(.system(size: 50))
        .foregroundStyle(.green)
}
```

**何をしているか：**
カードをスワイプしなくても、下のボタンから左スワイプや右スワイプと同じ処理を実行できるようにしている。赤いボタンはNG、緑のボタンはLIKEとして扱う。

**なぜこう書くのか：**
すべてのユーザーがスワイプ操作だけで使うとは限らないため、ボタンでも同じ操作ができるようにしている。また、ボタンでも `removeCard` を使うことで、スワイプ時と同じ処理を再利用している。

**もしこう書かなかったら：**
スワイプ操作に慣れていないユーザーは使いにくくなる。また、ボタン用に別の処理を書いてしまうと、スワイプとボタンで結果の管理方法がずれる可能性がある。同じ `removeCard` を使うことで、処理を一つにまとめられる。

---

## 応用編で新しく学んだSwiftの文法・API

| 項目                              | 説明                | 使用例                                                     |
| ------------------------------- | ----------------- | ------------------------------------------------------- |
| `Identifiable`                  | データを一意に識別できるようにする | `struct Animal: Identifiable`                           |
| `UUID()`                        | 一意のIDを作る          | `let id = UUID()`                                       |
| `enum`                          | 決まった種類の値を定義する     | `enum SwipeDirection { case left, right }`              |
| `ZStack`                        | ビューを重ねて表示する       | `ZStack { ForEach(...) { ... } }`                       |
| `.reversed()`                   | 配列の表示順を逆にする       | `ForEach(animals.reversed())`                           |
| `.removeAll`                    | 条件に合う要素を配列から削除する  | `animals.removeAll { $0.id == animal.id }`              |
| `.opacity`                      | ビューの透明度を変える       | `.opacity(swipeProgress)`                               |
| `.position`                     | ビューを指定した位置に配置する   | `.position(x: 80, y: 60)`                               |
| `DispatchQueue.main.asyncAfter` | 少し遅らせて処理を実行する     | `DispatchQueue.main.asyncAfter(deadline: .now() + 0.3)` |

## 応用編の自分の実験メモ

**実験1：**

* やったこと：`swipeThreshold` を `100` から `30` に変更した。
* 結果：少し横に動かしただけでLIKEやNOPEとして判定されるようになった。
* わかったこと：しきい値が小さすぎると誤操作が増えるため、スワイプ判定には適切な基準が必要だと分かった。

**実験2：**

* やったこと：`DispatchQueue.main.asyncAfter` を使わず、すぐに `onSwipe` を呼ぶようにした。
* 結果：カードが画面外に飛ぶ前に消えてしまい、動きが急に見えた。
* わかったこと：アニメーションを見せてからデータを削除することで、操作結果が自然に伝わることが分かった。

**実験3：**

* やったこと：手動ボタンで `animals.last` ではなく `animals.first` を削除するようにした。
* 結果：画面の一番上に見えているカードではなく、下に隠れているカードが消えることがあった。
* わかったこと：`ZStack` の重なり順と、配列のどの要素を操作するかを合わせる必要があると分かった。

## 応用編でAIに聞いて特に理解が深まった質問 TOP3

1. **質問：なぜカードを表示するときに `animals.reversed()` を使うのか？**
   **得られた理解：**
   `ZStack` では後から描画されたビューが前面に表示されるため、配列の順番と見た目の重なり順を調整する必要がある。`animals.reversed()` を使うことで、意図したカードが上に見えるようにしている。

2. **質問：なぜ手動ボタンでは `animals.last` を削除するのか？**
   **得られた理解：**
   画面の一番上に見えているカードを操作対象にするためである。`animals.first` を使うと、表示上は下に隠れているカードを削除してしまう場合があり、ユーザーの見た目と処理が一致しなくなる。

3. **質問：なぜ `onSwipe` をすぐ呼ばず、`DispatchQueue.main.asyncAfter` で少し遅らせるのか？**
   **得られた理解：**
   先にカードが画面外へ飛ぶアニメーションを見せ、その後で配列から削除するためである。すぐに削除すると、アニメーションを見る前にカードが消えてしまい、スワイプした感覚が弱くなる。
