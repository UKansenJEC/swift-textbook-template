# 第7章：センサーの活用

> 執筆者：ウカンセン
> 最終更新：2026-06-24

## この章で学ぶこと

例：この章では、iPhoneに搭載されている加速度・ジャイロなどのセンサーにアクセスして、デバイスの動きや姿勢を検出する方法を学ぶ。具体的にはCoreMotionを使った水平器アプリ、CMPedometerとCoreLocationを組み合わせた活動トラッカーを題材にして、センサーデータの読み取りと処理の実装を学ぶ。

## 模範コードの全体像

```swift
// ============================================
// 第7章（基本）：加速度センサーで動く水平器アプリ
// ============================================
// CoreMotionを使って端末の傾きをリアルタイムで取得し、
// 水平器（水準器）として表示するアプリです。
//
// 【注意】シミュレータではセンサーが使えません。
//         実機（iPhone / iPad）でテストしてください。
// ============================================

import SwiftUI
import CoreMotion

// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool = false

    func startUpdates() {
        guard motionManager.isDeviceMotionAvailable else {
            isAvailable = false
            return
        }

        isAvailable = true
        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var motionManager = MotionManager()

    var body: some View {
        NavigationStack {
            if motionManager.isAvailable {
                VStack(spacing: 30) {
                    // 水平器の円
                    LevelIndicator(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll
                    )

                    // 数値表示
                    DataDisplay(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll,
                        yaw: motionManager.yaw
                    )
                }
                .padding()
                .navigationTitle("水平器")
            } else {
                ContentUnavailableView(
                    "センサーが利用できません",
                    systemImage: "iphone.slash",
                    description: Text("このアプリは実機（iPhone）で動作します。\nシミュレータではセンサーが使えません。")
                )
            }
        }
        .onAppear {
            motionManager.startUpdates()
        }
        .onDisappear {
            motionManager.stopUpdates()
        }
    }
}

// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、iPhoneのセンサーを使って端末の傾きを取得し、水平器として表示するアプリである。

端末を前後・左右に傾けると、画面中央のバブルがその傾きに合わせて移動する。端末がほぼ水平になると、バブルの色が赤から緑に変わり、「水平!」と表示される。

また、画面下には `pitch`、`roll`、`yaw` の数値が表示され、端末の姿勢をラジアンと度の両方で確認できる。シミュレータではセンサーが使えないため、実機で動かす必要がある。

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
import CoreMotion
```

```swift
@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0
    var roll: Double = 0
    var yaw: Double = 0
    var isAvailable: Bool = false
}
```

**何をしているか：**
CoreMotionを使うために `CoreMotion` を import し、センサー情報を管理する `MotionManager` クラスを作っている。`CMMotionManager` は、端末の動きや姿勢を取得するためのクラスである。

**なぜこう書くのか：**
センサーの取得処理をViewの中に直接書くと、画面表示のコードとセンサー処理が混ざって分かりにくくなる。そのため、`MotionManager` という別のクラスにまとめている。また、`@Observable` を付けることで、`pitch` や `roll` の値が変わったときにSwiftUIの画面も更新される。

**もしこう書かなかったら：**
センサー処理をViewの中に全部書くと、コードが長くなり、どこでセンサーを開始・停止しているのか分かりにくくなる。また、`@Observable` がないと、センサーの値が変わっても画面に反映されにくくなる。

---

### センサーの利用可否チェック

```swift
func startUpdates() {
    guard motionManager.isDeviceMotionAvailable else {
        isAvailable = false
        return
    }

    isAvailable = true
    motionManager.deviceMotionUpdateInterval = 1.0 / 60.0
```

**何をしているか：**
端末でDeviceMotionが利用できるかを確認している。利用できる場合は `isAvailable` を `true` にし、センサー値の更新間隔を設定している。

**なぜこう書くのか：**
センサーはすべての環境で使えるわけではない。特にシミュレータでは実際の加速度やジャイロが使えないため、事前にチェックする必要がある。`isAvailable` を使うことで、センサーが使える場合は水平器画面を表示し、使えない場合はエラー画面を表示できる。

**もしこう書かなかったら：**
シミュレータなどセンサーが使えない環境でも無理に処理を始めようとしてしまう。結果として、画面が正しく動かなかったり、ユーザーに原因が分かりにくい表示になったりする。

---

### センサー値の取得と pitch / roll / yaw

```swift
motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
    guard let self = self, let motion = motion else { return }

    self.pitch = motion.attitude.pitch
    self.roll = motion.attitude.roll
    self.yaw = motion.attitude.yaw
}
```

**何をしているか：**
DeviceMotionの更新を開始し、端末の姿勢情報を取得している。取得した `motion.attitude` から `pitch`、`roll`、`yaw` を取り出して、それぞれの変数に代入している。

**なぜこう書くのか：**
水平器では、端末がどの方向に傾いているかをリアルタイムで知る必要がある。`pitch` は前後の傾き、`roll` は左右の傾き、`yaw` は水平方向の回転を表す。今回は水平器なので、主に `pitch` と `roll` を使ってバブルの位置を動かしている。

**もしこう書かなかったら：**
`startDeviceMotionUpdates` を呼ばなければ、センサー値が更新されない。そのため、端末を傾けても `pitch` や `roll` が変わらず、水平器として動かなくなる。

---

### 更新間隔の設定

```swift
motionManager.deviceMotionUpdateInterval = 1.0 / 60.0
```

**何をしているか：**
センサー値を1秒間に約60回更新するように設定している。

**なぜこう書くのか：**
水平器は端末の傾きに合わせてバブルが滑らかに動く必要がある。更新回数が少なすぎると、バブルの動きがカクカクして見える。`1.0 / 60.0` にすることで、画面の更新に近い感覚でセンサー値を取得できる。

**もしこう書かなかったら：**
更新間隔が長いと、端末を傾けてから画面が反応するまで遅く感じる。例えば `1.0 / 10.0` にすると、1秒に10回しか更新されないため、動きがなめらかではなくなる。

---

### Viewでの開始と停止

```swift
.onAppear {
    motionManager.startUpdates()
}
.onDisappear {
    motionManager.stopUpdates()
}
```

```swift
func stopUpdates() {
    motionManager.stopDeviceMotionUpdates()
}
```

**何をしているか：**
画面が表示されたときにセンサーの取得を開始し、画面が消えたときに取得を停止している。

**なぜこう書くのか：**
センサーは常に動かしておく必要はない。画面を見ていない間もセンサーを動かし続けると、無駄に処理が続き、バッテリー消費にもつながる。そのため、必要なときだけ開始し、不要になったら停止している。

**もしこう書かなかったら：**
`startUpdates()` を呼ばなければ、センサー値が取得されず水平器が動かない。逆に `stopUpdates()` を呼ばないと、画面を閉じた後もセンサー更新が続いてしまう可能性がある。

---

### 水平器インジケーター

```swift
private var xOffset: CGFloat {
    CGFloat(roll) * maxOffset
}

private var yOffset: CGFloat {
    CGFloat(pitch) * maxOffset
}

private var isLevel: Bool {
    abs(pitch) < 0.03 && abs(roll) < 0.03
}
```

```swift
Circle()
    .fill(isLevel ? .green : .red)
    .frame(width: 40, height: 40)
    .opacity(0.8)
    .shadow(color: isLevel ? .green : .red, radius: 8)
    .offset(
        x: max(-maxOffset, min(maxOffset, xOffset)),
        y: max(-maxOffset, min(maxOffset, yOffset))
    )
```

**何をしているか：**
`roll` を横方向、`pitch` を縦方向の移動量に変換し、バブルを動かしている。端末がほぼ水平な場合は `isLevel` が `true` になり、バブルが緑色になる。

**なぜこう書くのか：**
センサー値はそのまま画面上の位置として使うには小さいため、`maxOffset` を掛けて見やすい移動量に変換している。また、`max` と `min` を使って、バブルが円の外に大きく飛び出しすぎないようにしている。

**もしこう書かなかったら：**
`pitch` や `roll` を画面表示に反映しなければ、端末を傾けてもバブルが動かない。`max` と `min` で制限しないと、大きく傾けたときにバブルが画面外へ出てしまう可能性がある。

---

### 数値データ表示

```swift
Text(String(format: "%.3f rad", value))
```

```swift
Text(String(format: "(%.1f°)", value * 180 / .pi))
```

**何をしているか：**
`pitch`、`roll`、`yaw` の値を、ラジアンと度の両方で表示している。`String(format:)` を使って、小数点以下の桁数を整えている。

**なぜこう書くのか：**
CoreMotionから取得できる角度はラジアンである。しかし、人間にとっては度数の方が直感的に分かりやすい。そのため、`value * 180 / .pi` でラジアンを度に変換して表示している。

**もしこう書かなかったら：**
ラジアンだけを表示すると、数値を見てもどのくらい傾いているのか分かりにくい。度も一緒に表示することで、端末の傾きが理解しやすくなる。

---

## 新しく学んだSwiftの文法・API

| 項目                           | 説明                        | 使用例                                                                   |
| ---------------------------- | ------------------------- | --------------------------------------------------------------------- |
| `CoreMotion`                 | 端末の動きや姿勢を扱うためのフレームワーク     | `import CoreMotion`                                                   |
| `CMMotionManager`            | 加速度・ジャイロなどを使って端末の動きを取得する  | `private let motionManager = CMMotionManager()`                       |
| `@Observable`                | クラスの値の変化をSwiftUIの画面に反映させる | `@Observable class MotionManager`                                     |
| `isDeviceMotionAvailable`    | DeviceMotionが利用できるか確認する   | `motionManager.isDeviceMotionAvailable`                               |
| `startDeviceMotionUpdates`   | DeviceMotionの更新を開始する      | `motionManager.startDeviceMotionUpdates(to: .main) { ... }`           |
| `stopDeviceMotionUpdates`    | DeviceMotionの更新を停止する      | `motionManager.stopDeviceMotionUpdates()`                             |
| `deviceMotionUpdateInterval` | センサー値の更新間隔を設定する           | `motionManager.deviceMotionUpdateInterval = 1.0 / 60.0`               |
| `pitch`                      | 端末の前後方向の傾き                | `motion.attitude.pitch`                                               |
| `roll`                       | 端末の左右方向の傾き                | `motion.attitude.roll`                                                |
| `yaw`                        | 端末の水平方向の回転                | `motion.attitude.yaw`                                                 |
| `ContentUnavailableView`     | 利用できない機能があるときの案内画面を表示する   | `ContentUnavailableView("センサーが利用できません", systemImage: "iphone.slash")` |

## 自分の実験メモ

**実験1：**

* やったこと：`deviceMotionUpdateInterval` を `1.0 / 60.0` から `1.0 / 10.0` に変更した。
* 結果：センサー値の更新回数が少なくなり、バブルの動きが少し遅く、なめらかではなくなると考えられる。
* わかったこと：水平器のようにリアルタイムで動くUIでは、更新間隔が操作感に影響することが分かった。

**実験2：**

* やったこと：`isLevel` の条件を `0.03` から `0.10` に変更した。
* 結果：少し傾いていても「水平!」と表示されやすくなる。
* わかったこと：水平と判定する基準を変えると、アプリの判定の厳しさが変わることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：なぜセンサー処理をViewの中ではなく、MotionManagerクラスにまとめるのか？**
   **得られた理解：**
   Viewは画面表示を担当し、MotionManagerはセンサー取得を担当する。役割を分けることでコードが読みやすくなり、センサー処理の開始・停止も管理しやすくなる。

2. **質問：pitch、roll、yaw はそれぞれ何を表しているのか？**
   **得られた理解：**
   pitch は前後の傾き、roll は左右の傾き、yaw は水平方向の回転を表している。水平器では主に pitch と roll を使って、バブルを上下左右に動かしている。

3. **質問：なぜ `value * 180 / .pi` で角度を変換しているのか？**
   **得られた理解：**
   CoreMotionの角度はラジアンで取得されるが、人間には度の方が分かりやすい。そこで、ラジアンを度に変換して表示している。

## この章のまとめ

この章では、CoreMotionを使ってiPhoneの傾きを取得し、水平器アプリとして画面に表示する方法を学んだ。センサー値は自動的に画面へ表示されるわけではなく、`CMMotionManager` で取得し、SwiftUIの状態として反映する必要がある。
特に、`pitch`、`roll`、`yaw` の違いを理解することが重要だった。水平器では、前後の傾きを表す `pitch` と、左右の傾きを表す `roll` を使ってバブルの位置を動かしている。
また、センサーはシミュレータでは使えないため、`isDeviceMotionAvailable` で利用可能かどうかを確認する必要がある。実機の機能を使うアプリでは、使えない環境への対応も重要だと分かった。
