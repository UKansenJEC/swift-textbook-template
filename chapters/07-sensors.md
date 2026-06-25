# 第7章：センサーの活用

> 執筆者：ウカンセン
> 最終更新：2026-06-25

## この章で学ぶこと

この章では、iPhoneに搭載されている加速度・ジャイロなどのセンサーにアクセスして、デバイスの動きや姿勢を検出する方法を学ぶ。具体的にはCoreMotionを使った水平器アプリ、CMPedometerとCoreLocationを組み合わせた活動トラッカーを題材にして、センサーデータの読み取りと処理の実装を学ぶ。

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

# 第7章：センサーの活用・応用

## この章で学ぶこと

この章では、CoreMotionとCoreLocationを利用して、歩数や移動距離、現在の速度などを取得する方法を学ぶ。具体的には、CMPedometerによる歩数計測とCLLocationManagerによる位置情報取得を組み合わせ、活動トラッカーアプリを題材にして、センサーデータの取得・表示・計算方法を学ぶ。

また、Timerを利用した経過時間の表示や、複数のセンサーから取得したデータを一つの画面にまとめて表示する方法、Info.plistで必要な権限を設定する方法についても学ぶ。

---

## 模範コードの全体像

```swift
// ============================================
// 第7章（応用）：歩数計・移動距離トラッカー
// ============================================
// CoreMotion（歩数計）とCoreLocation（移動距離）を
// 組み合わせて、今日の活動を記録するアプリです。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSMotionUsageDescription
//     値: "歩数を計測するためにモーションセンサーを使用します"
//   - NSLocationWhenInUseUsageDescription
//     値: "移動距離を計測するために位置情報を使用します"
// ============================================

import SwiftUI
import CoreMotion
import CoreLocation
import Combine

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

    // MARK: - 計算プロパティ

    var elapsedTime: TimeInterval {
        guard let start = startTime else { return 0 }
        return Date().timeIntervalSince(start)
    }

    var distanceInKm: Double {
        distance / 1000
    }

    var speedInKmh: Double {
        currentSpeed * 3.6
    }

    var caloriesBurned: Double {
        // 簡易計算：歩数 × 0.04 kcal（目安）
        Double(stepCount) * 0.04
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var tracker = ActivityTracker()
    @State private var timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    // タイマー表示
                    timerSection

                    // メイン統計
                    statsGrid

                    // スタート/ストップボタン
                    controlButton

                    // 速度メーター
                    if tracker.isTracking {
                        SpeedMeter(speed: tracker.speedInKmh)
                    }
                }
                .padding()
            }
            .navigationTitle("活動トラッカー")
            .onReceive(timer) { _ in
                // タイマーの更新をトリガー（UI再描画のため）
                if tracker.isTracking {
                    // @Observableなので自動で更新される
                }
            }
        }
    }

    // MARK: - タイマーセクション

    private var timerSection: some View {
        VStack(spacing: 4) {
            Text(formatTime(tracker.elapsedTime))
                .font(.system(size: 48, weight: .thin, design: .monospaced))

            if tracker.isTracking {
                Text("計測中")
                    .font(.caption)
                    .foregroundStyle(.green)
            }
        }
        .padding()
    }

    // MARK: - 統計グリッド

    private var statsGrid: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible()),
        ], spacing: 16) {
            StatCard(
                icon: "figure.walk",
                value: "\(tracker.stepCount)",
                unit: "歩",
                color: .blue
            )
            StatCard(
                icon: "map",
                value: String(format: "%.2f", tracker.distanceInKm),
                unit: "km",
                color: .green
            )
            StatCard(
                icon: "flame",
                value: String(format: "%.0f", tracker.caloriesBurned),
                unit: "kcal",
                color: .orange
            )
            StatCard(
                icon: "speedometer",
                value: String(format: "%.1f", tracker.speedInKmh),
                unit: "km/h",
                color: .purple
            )
        }
    }

    // MARK: - コントロールボタン

    private var controlButton: some View {
        Button {
            if tracker.isTracking {
                tracker.stopTracking()
            } else {
                tracker.startTracking()
            }
        } label: {
            HStack {
                Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
                Text(tracker.isTracking ? "ストップ" : "スタート")
            }
            .font(.title3)
            .frame(maxWidth: .infinity)
            .padding()
            .background(tracker.isTracking ? Color.red : Color.green)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 16))
        }
    }

    // MARK: - 時間フォーマット

    func formatTime(_ interval: TimeInterval) -> String {
        let hours = Int(interval) / 3600
        let minutes = Int(interval) / 60 % 60
        let seconds = Int(interval) % 60
        return String(format: "%02d:%02d:%02d", hours, minutes, seconds)
    }
}

// MARK: - 統計カード

struct StatCard: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(color)

            Text(value)
                .font(.title)
                .bold()

            Text(unit)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(color.opacity(0.08))
        )
    }
}

// MARK: - 速度メーター

struct SpeedMeter: View {
    let speed: Double

    var body: some View {
        VStack(spacing: 8) {
            Text("現在の速度")
                .font(.caption)
                .foregroundStyle(.secondary)

            ZStack {
                Circle()
                    .trim(from: 0, to: 0.75)
                    .stroke(.gray.opacity(0.2), lineWidth: 8)
                    .rotationEffect(.degrees(135))

                Circle()
                    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
                    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                    .rotationEffect(.degrees(135))
                    .animation(.spring, value: speed)

                VStack {
                    Text(String(format: "%.1f", speed))
                        .font(.system(size: 32, weight: .bold, design: .monospaced))
                    Text("km/h")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .frame(width: 150, height: 150)
        }
        .padding()
    }

    var speedColor: Color {
        if speed < 4 { return .green }
        if speed < 8 { return .orange }
        return .red
    }
}

#Preview {
    ContentView()
}

```

### このアプリは何をするものか

応用編では、CoreMotionの歩数計機能とCoreLocationの位置情報機能を組み合わせて、今日の活動を記録するアプリを作る。
スタートボタンを押すと、歩数、移動距離、消費カロリー、現在の速度の計測が始まる。計測中は経過時間も表示され、歩いている間に数値が更新される。ストップボタンを押すと、歩数計と位置情報の更新を停止する。
基本編では端末の傾きを取得して水平器を作ったが、応用編ではセンサーを使って人の移動や活動量を記録する。CoreMotionだけでなくCoreLocationも使うことで、歩数と移動速度の両方を扱えるようになる。

## コードの詳細解説

### ActivityTrackerクラスの役割

```swift
@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0
    var isPedometerAvailable: Bool = false

    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0
    var locations: [CLLocationCoordinate2D] = []

    var isTracking: Bool = false
    var startTime: Date?
}
```

**何をしているか：**
歩数、移動距離、現在速度、位置情報、計測中かどうかを `ActivityTracker` クラスでまとめて管理している。

**なぜこう書くのか：**
歩数計や位置情報の処理をViewの中に直接書くと、画面表示のコードとセンサー処理が混ざって分かりにくくなる。そのため、活動計測に関する処理を `ActivityTracker` にまとめている。`@Observable` を付けることで、歩数や速度が変化したときに画面も更新される。

**もしこう書かなかったら：**
Viewの中にすべての処理を書くことになり、コードが長く複雑になる。また、歩数・距離・速度の管理場所が分散して、どこで値が更新されているのか分かりにくくなる。

---

### 歩数計の利用可否チェックと初期設定

```swift
override init() {
    super.init()
    locationManager.delegate = self
    locationManager.desiredAccuracy = kCLLocationAccuracyBest
    locationManager.requestWhenInUseAuthorization()
    isPedometerAvailable = CMPedometer.isStepCountingAvailable()
}
```

**何をしているか：**
`ActivityTracker` が作られたときに、位置情報のdelegate設定、精度設定、使用許可の要求、歩数計が使えるかどうかの確認を行っている。

**なぜこう書くのか：**
CoreLocationでは、位置情報の更新を受け取るために `delegate` を設定する必要がある。また、ユーザーの位置情報を使うためには許可を求める必要がある。歩数計についても、すべての端末で使えるとは限らないため、`CMPedometer.isStepCountingAvailable()` で確認している。

**もしこう書かなかったら：**
`delegate` を設定しないと、位置情報が更新されても `didUpdateLocations` が呼ばれない。位置情報の許可を求めなければ、アプリが位置情報を取得できない。また、歩数計の利用可否を確認しないと、使えない端末でも処理を始めようとしてしまう。

---

### 計測開始処理

```swift
func startTracking() {
    isTracking = true
    startTime = Date()
    stepCount = 0
    distance = 0
    locations = []

    if isPedometerAvailable {
        pedometer.startUpdates(from: Date()) { [weak self] data, error in
            guard let self = self, let data = data else { return }

            DispatchQueue.main.async {
                self.stepCount = data.numberOfSteps.intValue
                if let dist = data.distance {
                    self.distance = dist.doubleValue
                }
            }
        }
    }

    locationManager.startUpdatingLocation()
}
```

**何をしているか：**
計測を開始し、開始時刻・歩数・距離・位置履歴を初期化している。その後、歩数計と位置情報の更新を開始している。

**なぜこう書くのか：**
新しく計測を始めるときには、前回の歩数や距離が残っていると正しく記録できない。そのため、`stepCount`、`distance`、`locations` を最初にリセットしている。また、歩数計の更新はバックグラウンドの処理として返ってくるため、画面に反映する部分は `DispatchQueue.main.async` の中で行っている。

**もしこう書かなかったら：**
前回の計測結果が残ったままになり、今回の活動量が分かりにくくなる。また、メインスレッドで値を更新しないと、画面更新のタイミングが不安定になる可能性がある。

---

### 計測停止処理

```swift
func stopTracking() {
    isTracking = false
    pedometer.stopUpdates()
    locationManager.stopUpdatingLocation()
}
```

**何をしているか：**
計測中の状態を解除し、歩数計と位置情報の更新を停止している。

**なぜこう書くのか：**
ユーザーがストップを押した後もセンサーを動かし続ける必要はない。歩数計や位置情報を止めることで、不要な処理やバッテリー消費を防ぐことができる。

**もしこう書かなかったら：**
画面上では止まったように見えても、内部ではセンサー取得が続いてしまう可能性がある。特に位置情報はバッテリー消費が大きいため、不要になったら停止することが重要である。

---

### CoreLocationで速度と位置を取得する

```swift
func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
    guard let location = newLocations.last else { return }
    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}
```

**何をしているか：**
新しい位置情報が取得されたときに、最新の位置を取り出し、現在速度と位置履歴を更新している。

**なぜこう書くのか：**
`didUpdateLocations` には複数の位置情報が渡されることがあるため、最新の位置である `newLocations.last` を使っている。`location.speed` はm/sで返されるが、状況によっては負の値になることがあるため、`max(0, location.speed)` で0未満にならないようにしている。

**もしこう書かなかったら：**
最新ではない位置情報を使ってしまうと、現在の速度や移動経路がずれる可能性がある。また、負の速度をそのまま表示すると、ユーザーにとって意味の分からない数値になってしまう。

---

### 計算プロパティ

```swift
var elapsedTime: TimeInterval {
    guard let start = startTime else { return 0 }
    return Date().timeIntervalSince(start)
}

var distanceInKm: Double {
    distance / 1000
}

var speedInKmh: Double {
    currentSpeed * 3.6
}

var caloriesBurned: Double {
    Double(stepCount) * 0.04
}
```

**何をしているか：**
経過時間、km単位の距離、km/h単位の速度、消費カロリーを計算している。

**なぜこう書くのか：**
センサーから得られる値は、そのままだと画面表示に使いにくい場合がある。距離はメートルからkmに変換し、速度はm/sからkm/hに変換している。消費カロリーは、歩数から簡単な目安として計算している。

**もしこう書かなかったら：**
画面表示のたびに同じ計算を書くことになり、コードが読みにくくなる。また、m/sやメートルのまま表示すると、日常的な活動記録としては少し分かりにくい。

---

### タイマー表示とCombine

```swift
@State private var timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()
```

```swift
.onReceive(timer) { _ in
    if tracker.isTracking {
        // @Observableなので自動で更新される
    }
}
```

**何をしているか：**
1秒ごとにタイマーを動かし、経過時間の表示を更新するきっかけを作っている。

**なぜこう書くのか：**
`elapsedTime` は現在時刻から開始時刻を引いて計算しているため、画面を定期的に更新しないと時間表示が変わらない。`Timer.publish` と `.onReceive` を使うことで、1秒ごとに画面更新のタイミングを作っている。

**もしこう書かなかったら：**
計測自体は始まっていても、画面上の時間表示が自動で進まない可能性がある。また、`autoconnect()` を使うためには `Combine` をimportする必要がある。`import Combine` がないと、コンパイルエラーになる。

---

### 統計グリッド

```swift
private var statsGrid: some View {
    LazyVGrid(columns: [
        GridItem(.flexible()),
        GridItem(.flexible()),
    ], spacing: 16) {
        StatCard(
            icon: "figure.walk",
            value: "\(tracker.stepCount)",
            unit: "歩",
            color: .blue
        )
        StatCard(
            icon: "map",
            value: String(format: "%.2f", tracker.distanceInKm),
            unit: "km",
            color: .green
        )
        StatCard(
            icon: "flame",
            value: String(format: "%.0f", tracker.caloriesBurned),
            unit: "kcal",
            color: .orange
        )
        StatCard(
            icon: "speedometer",
            value: String(format: "%.1f", tracker.speedInKmh),
            unit: "km/h",
            color: .purple
        )
    }
}
```

**何をしているか：**
歩数、距離、消費カロリー、速度をカード形式で表示している。

**なぜこう書くのか：**
活動量の情報は複数あるため、縦に文章で並べるよりも、カード形式で整理した方が見やすい。`LazyVGrid` を使うことで、2列のグリッドとして表示できる。また、同じデザインを `StatCard` として部品化することで、表示項目を増やしやすくしている。

**もしこう書かなかったら：**
すべての表示を一つずつ直接書くことになり、同じようなUIコードが繰り返される。後から項目を増やしたりデザインを変更したりするときに手間が増える。

---

### スタート・ストップボタン

```swift
Button {
    if tracker.isTracking {
        tracker.stopTracking()
    } else {
        tracker.startTracking()
    }
} label: {
    HStack {
        Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
        Text(tracker.isTracking ? "ストップ" : "スタート")
    }
    .background(tracker.isTracking ? Color.red : Color.green)
}
```

**何をしているか：**
現在計測中かどうかによって、ボタンの動作と表示を切り替えている。計測していないときはスタート、計測中はストップとして動作する。

**なぜこう書くのか：**
スタートボタンとストップボタンを別々に置くより、一つのボタンで状態に応じて切り替えた方が分かりやすい。`isTracking` を使うことで、アプリの状態とボタン表示を一致させている。

**もしこう書かなかったら：**
計測中なのにスタートボタンが表示されたり、停止中なのにストップボタンが表示されたりして、ユーザーが混乱する。状態とUIを連動させることが重要である。

---

### 速度メーター

```swift
Circle()
    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
    .rotationEffect(.degrees(135))
    .animation(.spring, value: speed)
```

```swift
var speedColor: Color {
    if speed < 4 { return .green }
    if speed < 8 { return .orange }
    return .red
}
```

**何をしているか：**
現在の速度を円形メーターとして表示している。速度が上がると円の表示部分が伸び、速度に応じて色も変わる。

**なぜこう書くのか：**
速度を数字だけで表示するより、メーターとして表示した方が現在の速さを直感的に理解しやすい。`trim` を使うことで円の一部だけを表示し、速度に応じた進み具合を表現している。`speedColor` によって、低速・中速・高速を色で区別している。

**もしこう書かなかったら：**
速度は数字としては表示できるが、視覚的に分かりにくくなる。`animation` がなければ、速度の変化が急に切り替わって見える。

---

### Info.plistの権限設定

```text
NSMotionUsageDescription
歩数を計測するためにモーションセンサーを使用します

NSLocationWhenInUseUsageDescription
移動距離を計測するために位置情報を使用します
```

**何をしているか：**
モーションセンサーと位置情報を使う理由を、Info.plistに記述している。

**なぜこう書くのか：**
iOSでは、歩数計や位置情報のようなプライバシーに関係する機能を使うとき、ユーザーに理由を説明する必要がある。Info.plistに説明文を書いておくことで、権限確認のダイアログに表示される。

**もしこう書かなかったら：**
アプリがモーションセンサーや位置情報を使おうとしたときに、正しく許可を求められない。場合によってはアプリがクラッシュしたり、機能が動かなかったりする。

---

## 応用編で新しく学んだSwiftの文法・API

| 項目                                | 説明                | 使用例                                                          |
| --------------------------------- | ----------------- | ------------------------------------------------------------ |
| `CMPedometer`                     | 歩数や移動距離を取得する      | `private let pedometer = CMPedometer()`                      |
| `isStepCountingAvailable()`       | 歩数計が利用できるか確認する    | `CMPedometer.isStepCountingAvailable()`                      |
| `startUpdates(from:)`             | 指定した時刻から歩数計測を開始する | `pedometer.startUpdates(from: Date())`                       |
| `stopUpdates()`                   | 歩数計測を停止する         | `pedometer.stopUpdates()`                                    |
| `CLLocationManager`               | 位置情報を取得する         | `private let locationManager = CLLocationManager()`          |
| `CLLocationManagerDelegate`       | 位置情報の更新を受け取る      | `class ActivityTracker: NSObject, CLLocationManagerDelegate` |
| `requestWhenInUseAuthorization()` | 使用中のみ位置情報の許可を求める  | `locationManager.requestWhenInUseAuthorization()`            |
| `startUpdatingLocation()`         | 位置情報の更新を開始する      | `locationManager.startUpdatingLocation()`                    |
| `Timer.publish`                   | 一定間隔でイベントを発生させる   | `Timer.publish(every: 1, on: .main, in: .common)`            |
| `.onReceive`                      | Publisherから値を受け取る | `.onReceive(timer) { _ in ... }`                             |
| `LazyVGrid`                       | グリッド形式でViewを並べる   | `LazyVGrid(columns: [...]) { ... }`                          |
| `.trim`                           | 図形の一部だけを表示する      | `Circle().trim(from: 0, to: 0.75)`                           |

## 応用編の自分の実験メモ

**実験1：**

* やったこと：`caloriesBurned` の計算式を `Double(stepCount) * 0.04` から `Double(stepCount) * 0.05` に変更した。
* 結果：同じ歩数でも表示される消費カロリーが大きくなった。
* わかったこと：消費カロリーはセンサーから直接取得している値ではなく、歩数をもとにした目安の計算であることが分かった。

**実験2：**

* やったこと：`speedInKmh` の `currentSpeed * 3.6` を `currentSpeed` のみに変更した。
* 結果：速度がm/sの値のまま表示され、km/hとして見ると分かりにくくなった。
* わかったこと：CoreLocationの速度はm/sで取得されるため、日常的に分かりやすいkm/hに変換する必要がある。

**実験3：**

* やったこと：`SpeedMeter` の `min(speed / 15.0, 1.0)` から `min` を外した。
* 結果：速度が大きくなったときに、メーターの表示範囲を超えてしまう可能性がある。
* わかったこと：UI表示では、センサー値をそのまま使うのではなく、表示範囲に収める処理が必要である。

## 応用編でAIに聞いて特に理解が深まった質問 TOP3

1. **質問：なぜ歩数計にはCoreMotionのCMPedometerを使い、速度にはCoreLocationを使うのか？**
   **得られた理解：**
   CMPedometerは歩数や歩行距離の取得に向いているが、現在の移動速度や位置の履歴はCoreLocationから取得する必要がある。目的に応じて複数のフレームワークを組み合わせている。

2. **質問：なぜ `DispatchQueue.main.async` の中で `stepCount` や `distance` を更新しているのか？**
   **得られた理解：**
   歩数計の更新処理はメインスレッド以外で呼ばれることがある。画面に関係する値を安全に更新するために、メインスレッドへ戻してから値を変更している。

3. **質問：なぜ Info.plist に `NSMotionUsageDescription` と `NSLocationWhenInUseUsageDescription` を追加する必要があるのか？**
   **得られた理解：**
   歩数や位置情報はユーザーのプライバシーに関係する情報である。iOSでは、そのような機能を使う理由をアプリ側が説明しないと、許可を求めたり機能を使ったりできない。

## 応用編のまとめ

応用編では、CoreMotionの `CMPedometer` と CoreLocationの `CLLocationManager` を組み合わせて、活動トラッカーを作る方法を学んだ。歩数、移動距離、速度、消費カロリーなど、複数のデータを一つの画面に整理して表示することで、実用的なアプリに近い形になった。

特に、センサーの値は取得するだけでなく、ユーザーに分かりやすい単位やUIに変換する必要があると分かった。メートルをkmに変換したり、m/sをkm/hに変換したり、速度をメーターで表示したりすることで、数字の意味が理解しやすくなる。

また、モーションセンサーや位置情報を使う場合は、Info.plistで利用目的を説明し、必要な権限を設定することが重要である。センサーアプリでは、技術的な実装だけでなく、ユーザーへの説明や許可の扱いも大切だと分かった。

