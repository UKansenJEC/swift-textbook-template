# 第2章：地図アプリの基本

> 執筆者：ウカンセン
> 最終更新：2026-04-27

## この章で学ぶこと

この章では、MapKitを使ってアプリ内に地図を表示し、複数のランドマークをマーカーとして配置する方法を学ぶ。また、SwiftUIの状態管理（@State）やIdentifiableプロトコルを利用して、カテゴリによるフィルター機能を実装する流れを理解する。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第2章（基本）：MapKitで地図を表示するアプリ
// ============================================
// 東京の観光スポットを地図上にマーカーで表示します。
// マーカーをタップすると詳細情報が表示されます。
// ============================================

import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、東京の観光スポットを地図上に表示し、カテゴリごとにフィルターできる地図アプリである。  
ユーザーは地図上のマーカーをタップすることで、その場所の名前や説明を見ることができる。また、画面下部のフィルターを操作することで、特定のカテゴリ（寺社・タワー・公園）のみを表示することが可能である。
## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category
}
```

**何をしているか：**
地図上に表示する観光スポット（ランドマーク）の情報をまとめたデータ構造を定義している。

**なぜこう書くのか：**
Identifiableをつけることで、それぞれのデータに一意のIDが割り当てられ、ForEachやMapで正しく識別できるようになる。また、HashableをつけることでSetなどで扱えるようになる。

**もしこう書かなかったら：**
Identifiableがないと、ForEachでエラーが出たり、UIの更新時にどの要素が変わったか判断できなくなる。

---

### 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .region(
    MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
        span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
    )
)

Map(position: $cameraPosition, selection: $selectedLandmark) {
}
```

**何をしているか：**
地図の初期表示位置（東京駅付近）と表示範囲を設定し、Mapビューを表示している。

**なぜこう書くのか：**
@Stateを使うことで、カメラ位置を動的に変更できるようになる。MapにBindingで渡すことで、地図の操作と状態が同期される。

**もしこう書かなかったら：**
@Stateを使わないと、地図の位置が固定され、ユーザー操作に応じた更新が反映されない可能性がある。

---

### マーカーの表示

```swift
ForEach(filteredLandmarks) { landmark in
    Marker(
        landmark.name,
        systemImage: landmark.category.iconName,
        coordinate: landmark.coordinate
    )
    .tint(landmark.category.color)
    .tag(landmark)
}
```

**何をしているか：**
フィルターされたランドマークのリストをもとに、地図上にマーカーを表示している。

**なぜこう書くのか：**
ForEachを使うことで、配列のデータをまとめてUIに反映できる。Markerを使うことで、簡単に地図上にピンを表示できる。

**もしこう書かなかったら：**
ForEachを使わない場合、マーカーを1つずつ手動で書く必要があり、データ数が増えると管理が難しくなる。

---

### フィルター機能

```swift
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}
```

**何をしているか：**
選択されたカテゴリに応じて、表示するランドマークを絞り込んでいる。

**なぜこう書くのか：**
Setを使うことで、複数カテゴリを効率よく管理できる。また、computed propertyとして書くことで、状態が変わるたびに自動的に再計算される。

**もしこう書かなかったら：**
@Stateを使わないと、ボタンを押してもUIが更新されない。また、filterを使わないとカテゴリごとの表示切り替えができない。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目             | 説明                 | 使用例                                    |
| -------------- | ------------------ | -------------------------------------- |
| `Map`          | SwiftUIで地図を表示するビュー | `Map(position: $cameraPosition)`       |
| `Marker`       | 地図上にマーカーを表示する      | `Marker("名前", coordinate: coordinate)` |
| `@State`       | 状態を管理し、UI更新を行う     | `@State private var selectedCategory`  |
| `Identifiable` | 一意のIDを持つデータとして扱う   | `struct Item: Identifiable`            |
| `filter`       | 条件に合う要素だけ取り出す      | `array.filter { 条件 }`                  |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：マーカーの座標を変更してみた
- 結果：地図上の位置が変わった
- わかったこと：coordinateの値によって表示位置が決まる

**実験2：**
- やったこと：selectedCategoriesの初期値を空にした
- 結果：マーカーが表示されなくなった
- わかったこと：フィルター条件に一致しないと表示されない

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：Mapの新しい書き方（iOS17）と従来のcoordinateRegionを使う方法の違いは何か？**  
   **得られた理解：**  
   iOS17以降では、MapCameraPositionを使って地図の表示位置を管理する新しい書き方が導入されている。従来のcoordinateRegionを直接渡す方法に比べて、カメラの移動やズームなどの制御が柔軟になり、状態管理とも連携しやすくなっている。

2. **質問：CaseIterableをつけると何ができるのか？ForEach(Category.allCases)はどのように動作しているのか？**  
   **得られた理解：**  
   CaseIterableをつけることで、enumのすべてのケースを配列として取得できるallCasesプロパティが自動生成される。これにより、ForEachで全カテゴリをループ処理し、ボタンやUIを動的に生成できるようになる。

3. **質問：MarkerとAnnotationの違いは何か？それぞれの使いどころは？**  
   **得られた理解：**  
   Markerは簡単に地図上にマーカーを表示するためのコンポーネントであり、標準的な見た目で素早く実装できる。一方、Annotationはクロージャ内で自由にViewを定義できるため、よりカスタマイズされた表示が可能である。用途に応じて使い分ける必要がある。
   
## この章のまとめ

この章では、MapKitを使った地図表示の基本と、SwiftUIにおける状態管理の仕組みを学んだ。

# 第2章：地図アプリの応用

> 執筆者：ウカンセン
> 最終更新：2026-05-08

## この章で学ぶこと

この章では、MapKitを使った地図アプリに、現在地の取得と周辺検索の機能を追加する方法を学ぶ。  
基本編では、あらかじめコード内に用意された東京の観光スポットを地図上に表示したが、応用編ではユーザー自身の現在地を取得し、その周辺にあるコンビニ、カフェ、レストラン、駅などを検索して表示する。
また、位置情報を利用するために必要な `CLLocationManager`、`Info.plist` の設定、`UserAnnotation`、`Marker`、`MKLocalSearch` などの使い方について理解する。


## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第2章（応用）：現在地を表示し、周辺検索する地図アプリ
// ============================================
// ユーザーの現在地を取得して地図上に表示し、
// 周辺のコンビニやカフェなどを検索する機能を追加します。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//     値: "現在地を地図に表示するために位置情報を使用します"
// ============================================

import SwiftUI
import MapKit

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    var userLocation: CLLocationCoordinate2D?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func requestPermission() {
        manager.requestWhenInUseAuthorization()
    }

    func startUpdating() {
        manager.startUpdatingLocation()
    }

    func stopUpdating() {
        manager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        userLocation = locations.last?.coordinate
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        authorizationStatus = manager.authorizationStatus

        switch authorizationStatus {
        case .authorizedWhenInUse, .authorizedAlways:
            startUpdating()
        default:
            break
        }
    }
}

// MARK: - 検索結果モデル

struct NearbyPlace: Identifiable {
    let id = UUID()
    let name: String
    let coordinate: CLLocationCoordinate2D
    let category: String
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var searchResults: [MKMapItem] = []
    @State private var selectedCategory: String = "コンビニ"

    let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]

    var body: some View {
        ZStack(alignment: .top) {
            Map(position: $cameraPosition) {
                // 現在地のマーカー
                UserAnnotation()

                // 検索結果のマーカー
                ForEach(searchResults, id: \.self) { item in
                    if let name = item.name {
                        Marker(name, coordinate: item.placemark.coordinate)
                            .tint(.orange)
                    }
                }
            }
            .mapControls {
                MapUserLocationButton()
                MapCompass()
                MapScaleView()
            }

            // 検索カテゴリボタン
            VStack {
                categoryButtons
                    .padding(.top, 8)
                Spacer()
            }
        }
        .onAppear {
            locationManager.requestPermission()
        }
        .onChange(of: locationManager.userLocation) { _, newLocation in
            if let location = newLocation {
                cameraPosition = .region(
                    MKCoordinateRegion(
                        center: location,
                        span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)
                    )
                )
            }
        }
    }

    // MARK: - カテゴリボタン

    private var categoryButtons: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 8) {
                ForEach(searchCategories, id: \.self) { category in
                    Button {
                        selectedCategory = category
                        Task { await searchNearby(query: category) }
                    } label: {
                        Text(category)
                            .font(.subheadline)
                            .padding(.horizontal, 14)
                            .padding(.vertical, 8)
                            .background(
                                selectedCategory == category
                                    ? Color.blue
                                    : Color(.systemBackground)
                            )
                            .foregroundStyle(
                                selectedCategory == category
                                    ? .white
                                    : .primary
                            )
                            .clipShape(Capsule())
                            .shadow(color: .black.opacity(0.1), radius: 2)
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 周辺検索

    func searchNearby(query: String) async {
        guard let userLocation = locationManager.userLocation else { return }

        let request = MKLocalSearch.Request()
        request.naturalLanguageQuery = query
        request.region = MKCoordinateRegion(
            center: userLocation,
            span: MKCoordinateSpan(latitudeDelta: 0.02, longitudeDelta: 0.02)
        )

        do {
            let search = MKLocalSearch(request: request)
            let response = try await search.start()
            searchResults = response.mapItems
        } catch {
            print("検索エラー: \(error.localizedDescription)")
            searchResults = []
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、ユーザーの現在地を地図上に表示し、現在地周辺の施設を検索できる地図アプリである。
画面上部には「コンビニ」「カフェ」「レストラン」「駅」のボタンがあり、ボタンを押すと現在地周辺の検索結果が地図上にオレンジ色のマーカーとして表示される。
## コードの詳細解説

### 位置情報マネージャー

```swift
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    let manager = CLLocationManager()
    var userLocation: CLLocationCoordinate2D?
    var authorizationStatus: CLAuthorizationStatus = .notDetermined
}
```

**何をしているか：**
ユーザーの現在地を取得するためのクラスを定義している。CLLocationManager を使って位置情報を取得し、その結果を userLocation に保存する。

**なぜこう書くのか：**
位置情報の取得は、すぐに結果が返ってくる処理ではない。ユーザーが許可を押した後や、位置情報が更新された後に、あとから結果が返ってくる。そのため、CLLocationManagerDelegate を使って通知を受け取る必要がある。
また、@Observable をつけることで、userLocation の値が変わったときにSwiftUIの画面側へ変化を伝えることができる。

**もしこう書かなかったら：**
現在地を取得しても、画面側に変化が伝わらず、地図の表示が更新されない可能性がある。また、delegateを設定しないと、位置情報の更新結果を受け取れない。

---

### Info.plist の設定

```swift
NSLocationWhenInUseUsageDescription
```

**何をしているか：**
アプリが「使用中のみ」位置情報を使う理由を、iOSに伝えるための設定である。

**なぜこう書くのか：**
iOSでは、位置情報のような個人情報に関わる機能を使う場合、アプリ側が利用目的を明示する必要がある。この説明文は、初回起動時の許可ダイアログに表示される。

**もしこう書かなかったら：**
位置情報の許可ダイアログを正しく表示できず、アプリが位置情報を取得できない。また、場合によってはアプリがクラッシュすることもある。

---

### 位置情報の許可を求める処理

```swift
func requestPermission() {
    manager.requestWhenInUseAuthorization()
}
```

**何をしているか：**
ユーザーに対して、アプリ使用中の位置情報利用を許可するかどうかを確認している。

**なぜこう書くのか：**
位置情報は勝手に取得できない。ユーザーの許可が必要であるため、アプリ起動時にこのメソッドを呼び出して許可を求める。

**もしこう書かなかったら：**
アプリは現在地を取得できず、UserAnnotation や周辺検索が正しく動かない。

---

### delegate メソッド

```swift
func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
    userLocation = locations.last?.coordinate
}
```

**何をしているか：**
位置情報が更新されたときに呼ばれるメソッドである。取得した位置情報のうち、最新の座標を userLocation に保存している。

**なぜこう書くのか：**
位置情報は非同期で取得されるため、取得できたタイミングで処理を行う必要がある。delegateを使うことで、位置情報が更新された瞬間にアプリ側で反応できる。

**もしこう書かなかったら：**
現在地を取得しても、その値をアプリ内で使うことができない。

---

### 許可状態が変わったときの処理

```swift
func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
    authorizationStatus = manager.authorizationStatus

    switch authorizationStatus {
    case .authorizedWhenInUse, .authorizedAlways:
        startUpdating()
    default:
        break
    }
}
```

**何をしているか：**
ユーザーが位置情報の許可を選択したあと、その状態に応じて位置情報の取得を開始している。

**なぜこう書くのか：**
ユーザーが許可する前に位置情報を取得しようとしても、正しく取得できない。許可されたことを確認してから startUpdating() を呼ぶ必要がある。

**もしこう書かなかったら：**

許可されたあとでも位置情報の取得が始まらず、現在地が表示されない可能性がある。

---

### 現在地の表示

```swift
Map(position: $cameraPosition) {
    UserAnnotation()
}
```

**何をしているか：**
地図上にユーザーの現在地を表示している。

**なぜこう書くのか：**
UserAnnotation() を使うと、iOS標準の現在地表示を簡単に利用できる。自分で青い点のViewを作らなくても、標準の現在地マーカーを表示できる。

**もしこう書かなかったら：**

地図は表示されても、ユーザー自身の現在地が分からない。

---

### 検索結果のマーカー表示

```swift
ForEach(searchResults, id: \.self) { item in
    if let name = item.name {
        Marker(name, coordinate: item.placemark.coordinate)
            .tint(.orange)
    }
}
```

**何をしているか：**
MKLocalSearch で取得した検索結果を、地図上にマーカーとして表示している。

**なぜこう書くのか：**
検索結果は配列として取得されるため、ForEach を使って1件ずつ地図上に表示する。Marker を使うことで、標準的な地図マーカーを簡単に作成できる。

**もしこう書かなかったら：**

検索結果を取得しても、地図上に表示されないため、ユーザーが結果を確認できない。

---

### 地図のコントロール

```swift
.mapControls {
    MapUserLocationButton()
    MapCompass()
    MapScaleView()
}
```

**何をしているか：**
地図上に現在地ボタン、コンパス、スケールを表示している。

**なぜこう書くのか：**
これらを追加することで、ユーザーが現在地に戻ったり、方角や距離感を確認したりしやすくなる。

**もしこう書かなかったら：**

地図自体は使えるが、現在地に戻る操作や方角の確認がしにくくなる。

---

### カテゴリボタン

```swift
let searchCategories = ["コンビニ", "カフェ", "レストラン", "駅"]
```

**何をしているか：**
検索に使うカテゴリの一覧を配列として用意している。

**なぜこう書くのか：**
配列にしておくことで、ForEach を使ってボタンをまとめて作成できる。カテゴリが増えた場合も、配列に文字列を追加するだけで対応しやすい。

**もしこう書かなかったら：**

ボタンを1つずつ手書きする必要があり、コードが長くなって管理しにくくなる。

---

### 周辺検索

```swift
func searchNearby(query: String) async {
    guard let userLocation = locationManager.userLocation else { return }

    let request = MKLocalSearch.Request()
    request.naturalLanguageQuery = query
    request.region = MKCoordinateRegion(
        center: userLocation,
        span: MKCoordinateSpan(latitudeDelta: 0.02, longitudeDelta: 0.02)
    )

    do {
        let search = MKLocalSearch(request: request)
        let response = try await search.start()
        searchResults = response.mapItems
    } catch {
        print("検索エラー: \(error.localizedDescription)")
        searchResults = []
    }
}
```

**何をしているか：**
現在地を中心に、指定されたキーワードで周辺施設を検索している。たとえば「カフェ」を選ぶと、現在地周辺のカフェを検索する。

**なぜこう書くのか：**
MKLocalSearch.Request に検索キーワードと検索範囲を設定し、MKLocalSearch を使ってAppleの地図検索機能を利用している。検索処理は時間がかかる可能性があるため、async/await を使って非同期で実行している。

**もしこう書かなかったら：**

現在地周辺の情報を検索できず、固定された地点しか表示できないアプリになってしまう。

---

## 新しく学んだSwiftの文法・API

| 項目                          | 説明                      | 使用例                                     |
| --------------------------- | ----------------------- | --------------------------------------- |
| `CLLocationManager`         | 位置情報を取得するためのクラス         | `let manager = CLLocationManager()`     |
| `CLLocationManagerDelegate` | 位置情報の更新や許可状態の変化を受け取る仕組み | `locationManagerDidChangeAuthorization` |
| `@Observable`               | クラスの状態変化をSwiftUIに伝える仕組み | `@Observable class LocationManager`     |
| `UserAnnotation`            | ユーザーの現在地を地図上に表示する       | `UserAnnotation()`                      |
| `Marker`                    | 地図上に標準のマーカーを表示する        | `Marker(name, coordinate: coordinate)`  |
| `MapCameraPosition`         | 地図の表示位置を管理する            | `@State private var cameraPosition`     |
| `MKLocalSearch`             | Appleの地図検索を利用するAPI      | `MKLocalSearch(request: request)`       |
| `async/await`               | 時間のかかる処理を非同期で実行する       | `try await search.start()`              |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：：Info.plist のキーを確認した**
- やったこと：NSLocationWhenInUseUsageDescription を追加した。
- 結果：アプリ初回起動時に位置情報の許可ダイアログが表示された。
- わかったこと：位置情報を使うには、コードだけでなくInfo.plistの設定も必要である。

**実験2：検索カテゴリを変更してみた**
- やったこと：「コンビニ」「カフェ」「レストラン」「駅」のボタンを押してみた。
- 結果：押したカテゴリに応じて、地図上の検索結果が変わった。
- わかったこと：naturalLanguageQuery の値によって、検索される施設の種類が変わる。

**実験3：シミュレータの位置情報を変更した**
- やったこと：Simulator の Features から Location を変更した。
- 結果：現在地の位置が変わり、検索結果も周辺に合わせて変わった。
- わかったこと：このアプリは固定データではなく、現在地を基準に動いている。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：CLLocationManager と Info.plist はなぜ両方必要なのか？**  
   **得られた理解：**  
   CLLocationManager は実際に位置情報を取得するためのクラスであり、Info.plist はアプリが位置情報を使う理由をiOSに伝えるための設定である。コードだけでは不十分で、ユーザーに許可を求めるための説明文も必要になる。
   
2. **質問：UserAnnotation、Marker、Annotation の違いは何か？**  
   **得られた理解：**  
   UserAnnotation はユーザーの現在地を表示するためのもの、Marker は標準的な地図マーカーを簡単に表示するためのもの、Annotation は自分で自由にViewを作って表示するためのものである。標準表示でよい場合は Marker、自由にデザインしたい場合は Annotation を使う。

3. **質問：delegate パターンとは何か？位置情報の例で考えるとどう理解できるか？**  
   **得られた理解：**  
   delegate パターンは、「処理が終わったらあとで知らせてもらう」仕組みである。位置情報はすぐに取得できるとは限らないため、取得できたタイミングで didUpdateLocations が呼ばれる。このように、あとから起きる出来事を受け取るためにdelegateが使われている。
   
## この章のまとめ

この章では、MapKitを使った地図アプリに、現在地表示と周辺検索の機能を追加する方法を学んだ。
基本編では固定された観光地を表示していたが、応用編では CLLocationManager でユーザーの現在地を取得し、MKLocalSearch を使って現在地周辺の施設を検索した。

特に重要だと感じたのは、位置情報を使うにはコードだけでなく、Info.plist に利用目的を書く必要がある点である。また、delegate パターンによって、位置情報のように「あとから結果が返ってくる処理」を扱えることも理解できた。

今回のアプリは、地図表示だけでなく、権限、状態管理、非同期処理、検索APIが組み合わさっているため、基本編よりも実際のアプリに近い内容だった。
