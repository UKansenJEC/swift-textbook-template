# 第5章：機能統合の実践

> 執筆者：ウカンセン
> 最終更新：2026-06-10

## この章で学ぶこと

この章では、これまでに学んだカメラ・地図・データ保存の各機能を組み合わせて、「フォトマップ」アプリを実装する方法を学ぶ。具体的には撮影した写真をGPS位置情報と一緒に保存し、地図上に表示し、永続化したデータを検索・編集するアプリを題材にする。複数機能を統合するためのアーキテクチャ設計が重要になる。

## 模範コードの全体像


```swift
// ============================================
// 第5章：写真 + 地図 + データ保存の統合アプリ
// ============================================
// 写真を選択し、選択時の現在地を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}
```

**このアプリは何をするものか：**

このアプリは、写真と位置情報を一緒に記録できるフォトマップアプリである。

ユーザーは写真を選択し、その時点の現在地を取得して保存することができる。保存された記録は地図上のマーカーとして表示されるほか、一覧画面からも確認できる。また、詳細画面では写真やメモ、撮影場所を確認することができる。

第2章の地図機能、第3章の写真機能、第4章のデータ保存機能を統合したアプリである。

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date
}
```

**何をしているか：**
写真のタイトル、メモ、位置情報、画像データ、作成日時を保存するためのデータモデルを定義している。

**なぜこう書くのか：**
SwiftDataでデータを保存するためには、保存したい情報をまとめたモデルが必要である。@Modelを付けることでSwiftDataが自動的にデータを管理できるようになる。

**もしこう書かなかったら：**
写真や位置情報を保存できなくなり、アプリを再起動したときに記録したデータが失われてしまう。

---

### タブ構成の設計

```swift
TabView {
    MapTab()
    ListTab()
}
```

**何をしているか：**
地図画面と一覧画面をタブで切り替えられるようにしている。

**なぜこう書くのか：**
地図で場所を確認したい場合と、一覧で記録を確認したい場合があるため、それぞれの機能を分けて使いやすくしている。

**もしこう書かなかったら：**
すべての機能を一つの画面に表示する必要があり、画面が複雑になってしまう。

---

### カメラと位置情報の連携

```swift
let location = locationManager.currentLocation
```

```swift
guard let location = locationManager.currentLocation else { return }
```

**何をしているか：**
現在地を取得し、写真と一緒に保存している。

**なぜこう書くのか：**
フォトマップアプリでは、写真だけでなく撮影場所も重要な情報になるためである。

**もしこう書かなかったら：**
写真は保存できても地図上に表示できなくなり、フォトマップとしての機能が失われる。

---

### SwiftDataでの画像保存

```swift
var imageData: Data?
```

**何をしているか：**
選択した画像をData形式で保存している。

**なぜこう書くのか：**
UIImageはそのままSwiftDataに保存できないため、Data形式に変換して保存する必要がある。

**もしこう書かなかったら：**
アプリを再起動したときに画像が表示されなくなる。

---


## 新しく学んだSwiftの文法・API

| 項目                | 説明                    | 使用例                               |
| ----------------- | --------------------- | --------------------------------- |
| TabView           | タブで画面を切り替える           | `TabView { ... }`                 |
| @Model            | SwiftDataのデータモデルを定義する | `@Model class PhotoRecord`        |
| @Query            | 保存されたデータを取得する         | `@Query private var records`      |
| PhotosPicker      | 写真を選択する               | `PhotosPicker(selection:)`        |
| CLLocationManager | 現在地を取得する              | `manager.startUpdatingLocation()` |
| Map               | 地図を表示する               | `Map { ... }`                     |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：TabViewを削除した。
- 結果：一覧画面へ移動できなくなった。
- わかったこと：TabViewは複数画面を切り替えるために必要であることが分かった。


**実験2：**
- やったこと：manager.startUpdatingLocation() を削除した
- 結果：現在地が取得できなくなった
- わかったこと：権限取得と位置情報更新は別処理である

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：なぜ画像をUIImageのまま保存せず、Data型に変換して保存するのか？UIImageを直接保存する方法と比べて何が違うのか？**
   **得られた理解：**
   UIImageはメモリ上のオブジェクトであり、そのまま永続化できない。Data型に変換することでSwiftDataに保存できる。アプリを再起動した後も画像を復元できるようになる。

2. **質問：なぜMapTabとListTabに画面を分けるのか？一つの画面ですべて表示した場合と比較してどんな利点があるのか？**
   **得られた理解：**
   機能ごとに責務を分離することで画面構成が分かりやすくなる。一つの画面に全ての機能を置くとUIが複雑になり保守性も低下する。
   
3. **質問：なぜ PhotoRecord は struct ではなく @Model class として定義するのか？**
   **得られた理解：**
   SwiftDataは参照型であるclassを前提としてデータを管理する。structにすると変更追跡や永続化が正しく行えない。@Modelを付けたclassを使うことでSwiftDataが自動的に保存・更新を管理できる。

## この章のまとめ

この章では、地図、写真、データ保存という複数の機能を一つのアプリに統合する方法を学んだ。また、複数の機能を連携させる際には、それぞれの役割を分けて設計することが重要だと感じた。
