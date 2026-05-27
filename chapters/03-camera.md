# 第3章：カメラの利用

> 執筆者：ウカンセン
> 最終更新：2026-05-20

## この章で学ぶこと

この章では、PhotosPickerでフォトライブラリから写真を選択し、UIImagePickerControllerでカメラ撮影した画像を扱う方法を学ぶ。具体的には非同期で画像データを読み込み、UIViewControllerRepresentableを使ってUIKitをSwiftUIに統合し、Coordinatorパターンを使ってカメラ機能と連携するアプリを題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第3章（基本）：写真を選択・撮影して表示するアプリ
// ============================================
// PhotosPickerを使ってフォトライブラリから写真を選択し、
// 画面に表示します。シミュレータでも動作します。
// ============================================

import SwiftUI
import PhotosUI

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    // MARK: - 画像の読み込み

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

// MARK: - カメラビュー（UIKit連携）

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、フォトライブラリから写真を選択したりカメラで撮影した画像を画面に表示したりするアプリである。

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)
```

**何をしているか：**
PhotosPicker を使って、フォトライブラリから画像を選択できるようにしている。選択した画像は selectedItem に保存される。

**なぜこう書くのか：**
PhotosPicker は SwiftUI 用に用意された写真選択コンポーネントで、UIKit を使わなくても簡単にフォトライブラリへアクセスできる。また matching: .images を指定することで、画像のみ選択可能にしている。

$selectedItem のように Binding を使うことで、選択状態が変化した時に自動で状態更新される。

**もしこう書かなかったら：**
PhotosPicker を使わない場合、フォトライブラリから画像を選択できない。また matching: .images を指定しないと、画像以外のファイルも選択対象になる可能性がある。Binding を使わなかった場合、選択結果が View に反映されない。


---

### 画像の非同期読み込み

```swift
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}

func loadImage(from item: PhotosPickerItem?) async {
    guard let item = item else { return }

    do {
        if let data = try await item.loadTransferable(type: Data.self),
           let uiImage = UIImage(data: data) {
            selectedImage = Image(uiImage: uiImage)
        }
    } catch {
        print("画像の読み込みに失敗: \(error.localizedDescription)")
    }
}
```

**何をしているか：**
PhotosPicker で画像が選択された時に loadImage() を呼び出し、画像データを非同期で読み込んでいる。取得した Data を UIImage に変換し、画面表示用の Image に変換している。

**なぜこう書くのか：**
画像データの読み込みは時間がかかる場合があるため、async/await を使った非同期処理になっている。

guard let を使うことで、item が nil の場合に安全に処理を終了している。また do-catch を使って、画像読み込み失敗時のエラー処理も行っている。

**もしこう書かなかったら：**
非同期処理にしなかった場合、画像読み込み中に画面が止まる可能性がある。

guard let を使わなかった場合、nil のまま処理してクラッシュする可能性がある。

エラー処理を書かなかった場合、読み込み失敗時に原因が分からなくなる。

---

### UIViewControllerRepresentableによるカメラ連携

```swift
struct CameraView: UIViewControllerRepresentable {

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

}
```

**何をしているか：**
UIKit の UIImagePickerController を SwiftUI で使えるようにしている。カメラを起動し、撮影画面を表示する役割を持つ。

**なぜこう書くのか：**
SwiftUI には標準のカメラ撮影コンポーネントがないため、UIKit の UIImagePickerController を利用している。

UIViewControllerRepresentable を使うことで、UIKit の ViewController を SwiftUI に組み込める。

**もしこう書かなかったら：**
UIImagePickerController をそのまま SwiftUI に表示できないため、カメラ機能を実装できない。

また picker.sourceType = .camera を書かなかった場合、カメラではなくフォトライブラリが開かれる。

---

### Coordinatorパターン

```swift
class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {

    let parent: CameraView

    init(_ parent: CameraView) {
        self.parent = parent
    }

    func imagePickerController(
        _ picker: UIImagePickerController,
        didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
    ) {
        if let image = info[.originalImage] as? UIImage {
            parent.capturedImage = image
        }
        parent.dismiss()
    }

}
```

**何をしているか：**
UIImagePickerController の delegate を受け取り、撮影した画像データを SwiftUI 側へ渡している。

**なぜこう書くのか：**
UIImagePickerController は delegate パターンで動作するため、Coordinator クラスが必要になる。

SwiftUI の View 構造体だけでは delegate メソッドを書けないため、NSObject を継承した Coordinator を用意している。

**もしこう書かなかったら：**
撮影完了時に画像を取得できず、カメラ機能が正常に動作しない。

また parent.dismiss() を書かなかった場合、撮影後もカメラ画面が閉じなくなる。

---


## 新しく学んだSwiftの文法・API

| 項目                              | 説明                       | 使用例                                                |
| ------------------------------- | ------------------------ | -------------------------------------------------- |
| `PhotosPicker`                  | フォトライブラリから画像を選択するコンポーネント | `PhotosPicker(selection: $selectedItem)`           |
| `UIImagePickerController`       | カメラ機能を利用するUIKitコンポーネント   | `picker.sourceType = .camera`                      |
| `UIViewControllerRepresentable` | UIKit を SwiftUI に組み込む仕組み | `struct CameraView: UIViewControllerRepresentable` |
| `Coordinator`                   | delegate を受け取るためのクラス     | `makeCoordinator()`                                |
| `async/await`                   | 非同期処理を書くための構文            | `await loadImage(from:)`                           |
| `loadTransferable()`            | PhotosPicker からデータを取得する  | `item.loadTransferable(type: Data.self)`           |


## 自分の実験メモ


**実験1：**
- やったこと：matching: .images を削除した
- 結果：画像以外も選択対象になった
- わかったこと：matching で選択対象を制限できる

**実験2：**
- やったこと：parent.dismiss() を削除した
- 結果：撮影後もカメラ画面が閉じなかった
- わかったこと：dismiss は画面を閉じるために必要

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：UIViewControllerRepresentable はなぜ必要なのか？**
 
   **得られた理解：**
   SwiftUI だけでは UIKit の ViewController を直接扱えないため、橋渡しとして必要だと分かった。

2. **質問：Coordinator パターンは何のためにあるのか？**
   
   **得られた理解：**
   delegate を受け取るための仕組みで、UIImagePickerController と SwiftUI を接続していると理解できた。

3. **質問：なぜ async/await を使うのか？**
   
   **得られた理解：**
   画像読み込みには時間がかかる場合があるため、非同期処理にして画面が止まらないようにしていると分かった。

## この章のまとめ

この章では、PhotosPicker を使ったフォトライブラリ連携と、UIImagePickerController を使ったカメラ撮影について学んだ。また、SwiftUI と UIKit を連携するために UIViewControllerRepresentable や Coordinator パターンが必要になることを理解した。
さらに async/await を使った非同期処理や、@State を使った状態管理によって、画像選択時に自動で画面更新される仕組みも学ぶことができた。

# 第3章：カメラの利用　応用

> 執筆者：ウカンセン
> 最終更新：2026-05-27

## この章で学ぶこと

この章では、PhotosPickerで選択した写真にCoreImageのフィルターを適用し、加工した画像をフォトライブラリに保存する方法を学ぶ。具体的には、写真をUIImageとして読み込み、CIImageに変換してCIFilterを適用し、最後にUIImageとして表示・保存する流れを確認する。

また、フィルターの種類をenumで管理する方法や、CIContextを使ってCoreImageの画像処理結果を画面表示用の画像に変換する方法も学ぶ。さらに、元画像と表示用画像を分けて管理する理由、フィルター選択時に画面を自動更新する仕組みについても理解する。


## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第3章（応用）：写真にフィルターをかけて保存するアプリ
// ============================================
// 選択した写真にCoreImageフィルターを適用し、
// フォトライブラリに保存する機能を追加します。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSPhotoLibraryAddUsageDescription
//     値: "加工した写真を保存するためにフォトライブラリを使用します"
// ============================================

import SwiftUI
import PhotosUI
import CoreImage
import CoreImage.CIFilterBuiltins

// MARK: - フィルター定義

enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
    case chrome = "クローム"
    case fade = "フェード"
    case bloom = "ブルーム"

    var id: String { rawValue }

    func apply(to inputImage: CIImage, context: CIContext) -> CIImage? {
        switch self {
        case .original:
            return inputImage
        case .sepia:
            let filter = CIFilter.sepiaTone()
            filter.inputImage = inputImage
            filter.intensity = 0.8
            return filter.outputImage
        case .mono:
            let filter = CIFilter.photoEffectMono()
            filter.inputImage = inputImage
            return filter.outputImage
        case .chrome:
            let filter = CIFilter.photoEffectChrome()
            filter.inputImage = inputImage
            return filter.outputImage
        case .fade:
            let filter = CIFilter.photoEffectFade()
            filter.inputImage = inputImage
            return filter.outputImage
        case .bloom:
            let filter = CIFilter.bloom()
            filter.inputImage = inputImage
            filter.radius = 10
            filter.intensity = 0.8
            return filter.outputImage
        }
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var originalUIImage: UIImage?
    @State private var displayImage: Image?
    @State private var currentFilter: PhotoFilter = .original
    @State private var isSaving = false
    @State private var showSaveAlert = false
    @State private var saveMessage = ""

    private let context = CIContext()

    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                // 画像表示
                if let image = displayImage {
                    image
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .frame(maxHeight: 350)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                        .padding(.horizontal)
                } else {
                    placeholderView
                }

                // フィルター選択
                if originalUIImage != nil {
                    filterSelector
                }

                // ボタン群
                HStack(spacing: 16) {
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選ぶ", systemImage: "photo")
                    }
                    .buttonStyle(.bordered)

                    if displayImage != nil {
                        Button {
                            saveFilteredImage()
                        } label: {
                            Label("保存", systemImage: "square.and.arrow.down")
                        }
                        .buttonStyle(.borderedProminent)
                        .disabled(isSaving)
                    }
                }
                .padding()

                Spacer()
            }
            .navigationTitle("フォトフィルター")
            .onChange(of: selectedItem) { _, newItem in
                Task { await loadOriginalImage(from: newItem) }
            }
            .onChange(of: currentFilter) { _, _ in
                applyFilter()
            }
            .alert("保存結果", isPresented: $showSaveAlert) {
                Button("OK") {}
            } message: {
                Text(saveMessage)
            }
        }
    }

    // MARK: - プレースホルダー

    private var placeholderView: some View {
        RoundedRectangle(cornerRadius: 12)
            .fill(.gray.opacity(0.1))
            .frame(height: 300)
            .overlay {
                VStack(spacing: 8) {
                    Image(systemName: "camera.filters")
                        .font(.system(size: 48))
                        .foregroundStyle(.gray)
                    Text("写真を選んでフィルターを試そう")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .padding(.horizontal)
    }

    // MARK: - フィルター選択UI

    private var filterSelector: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack(spacing: 12) {
                ForEach(PhotoFilter.allCases) { filter in
                    VStack(spacing: 4) {
                        // フィルタープレビュー（サムネイル）
                        if let thumbnail = createThumbnail(filter: filter) {
                            Image(uiImage: thumbnail)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 60, height: 60)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                                .overlay(
                                    RoundedRectangle(cornerRadius: 8)
                                        .stroke(
                                            currentFilter == filter ? Color.blue : Color.clear,
                                            lineWidth: 3
                                        )
                                )
                        }

                        Text(filter.rawValue)
                            .font(.caption2)
                            .foregroundStyle(
                                currentFilter == filter ? .blue : .secondary
                            )
                    }
                    .onTapGesture {
                        currentFilter = filter
                    }
                }
            }
            .padding(.horizontal)
        }
    }

    // MARK: - 画像処理

    func loadOriginalImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                originalUIImage = uiImage
                currentFilter = .original
                displayImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像読み込みエラー: \(error)")
        }
    }

    func applyFilter() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return }

        guard let outputImage = currentFilter.apply(to: ciImage, context: context) else { return }

        if let cgImage = context.createCGImage(outputImage, from: ciImage.extent) {
            displayImage = Image(uiImage: UIImage(cgImage: cgImage))
        }
    }

    func createThumbnail(filter: PhotoFilter) -> UIImage? {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage) else { return nil }

        guard let output = filter.apply(to: ciImage, context: context) else { return nil }

        if let cgImage = context.createCGImage(output, from: ciImage.extent) {
            return UIImage(cgImage: cgImage)
        }
        return nil
    }

    func saveFilteredImage() {
        guard let uiImage = originalUIImage,
              let ciImage = CIImage(image: uiImage),
              let output = currentFilter.apply(to: ciImage, context: context),
              let cgImage = context.createCGImage(output, from: ciImage.extent) else { return }

        let finalImage = UIImage(cgImage: cgImage)
        UIImageWriteToSavedPhotosAlbum(finalImage, nil, nil, nil)

        saveMessage = "写真を保存しました"
        showSaveAlert = true
    }
}

#Preview {
    ContentView()
}
```


**このアプリは何をするものか：**

このアプリは、フォトライブラリから写真を選択し、その写真にセピア、モノクロ、クローム、フェード、ブルームなどのフィルターをかけて表示するアプリである。また、加工した写真をフォトライブラリに保存することもできる。

## コードの詳細解説

### PhotoFilter enumによるフィルター管理

```swift
enum PhotoFilter: String, CaseIterable, Identifiable {
    case original = "オリジナル"
    case sepia = "セピア"
    case mono = "モノクロ"
    case chrome = "クローム"
    case fade = "フェード"
    case bloom = "ブルーム"

    var id: String { rawValue }
}
```

**何をしているか：**
使用できるフィルターの種類を enum でまとめて定義している。オリジナル、セピア、モノクロなど、決まった選択肢を1つの型として管理している。

**なぜこう書くのか：**
フィルターの種類は自由に増減するデータではなく、あらかじめ決まった選択肢である。そのため、struct や class よりも enum の方が自然である。
また、CaseIterable に準拠することで PhotoFilter.allCases が使えるようになり、すべてのフィルターを ForEach で表示できる。Identifiable に準拠することで、SwiftUI の ForEach で各項目を識別できるようになる。

**もしこう書かなかったら：**
フィルター名を文字列で直接管理すると、入力ミスが起きやすくなる。例えば "sepia" と "seipa" のような間違いがあっても、コンパイル時に気づきにくい。enum を使うことで、選択肢を安全に管理できる。$selectedItem のように Binding を使うことで、選択状態が変化した時に自動で状態更新される。

---

### CoreImageフィルターの適用

```swift
func apply(to inputImage: CIImage, context: CIContext) -> CIImage? {
    switch self {
    case .original:
        return inputImage
    case .sepia:
        let filter = CIFilter.sepiaTone()
        filter.inputImage = inputImage
        filter.intensity = 0.8
        return filter.outputImage
    case .bloom:
        let filter = CIFilter.bloom()
        filter.inputImage = inputImage
        filter.radius = 10
        filter.intensity = 0.8
        return filter.outputImage
    }
}
```

**何をしているか：**
選択されたフィルターに応じて、CoreImage の CIFilter を画像に適用している。セピアなら CIFilter.sepiaTone()、ブルームなら CIFilter.bloom() を使い、加工後の CIImage を返している。

**なぜこう書くのか：**
CIFilterBuiltins を import することで、CIFilter.sepiaTone() のような型安全な書き方ができる。古い書き方では CIFilter(name: "CISepiaTone") のように文字列でフィルター名を書く必要があり、名前を間違えても実行するまで分かりにくい。また、戻り値が CIImage? になっているのは、フィルター処理が失敗する可能性があるからである。例えば入力画像が正しくない場合や、フィルターの出力が作れない場合には nil が返る可能性がある。

**もしこう書かなかったら：**
フィルター処理を1か所にまとめない場合、View の中に画像処理のコードが増えてしまい、読みにくくなる。また、文字列でフィルター名を書く古い方法では、タイプミスによるバグが起きやすくなる。

---

### フィルター変更時の自動更新

```swift
.onChange(of: currentFilter) { _, _ in
    applyFilter()
}
```

**何をしているか：**
現在選択しているフィルターが変わった時に、自動で applyFilter() を呼び出している。

**なぜこう書くのか：**
ユーザーがサムネイルをタップしてフィルターを選んだ時、すぐに大きな画像にも同じフィルター結果を反映するためである。currentFilter の変化を監視することで、ボタンを別に押さなくても画面が更新される。

**もしこう書かなかったら：**
フィルターを選んでも画面の画像が変わらない。ユーザーから見ると、タップしているのに何も起きていないように見えてしまう。


---

### サムネイルの作成

```swift
func createThumbnail(filter: PhotoFilter) -> UIImage? {
    guard let uiImage = originalUIImage,
          let ciImage = CIImage(image: uiImage) else { return nil }

    guard let output = filter.apply(to: ciImage, context: context) else { return nil }

    if let cgImage = context.createCGImage(output, from: ciImage.extent) {
        return UIImage(cgImage: cgImage)
    }
    return nil
}
```

**何をしているか：**
各フィルターの小さいプレビュー画像を作っている。ユーザーはサムネイルを見ることで、どのフィルターを選ぶとどんな見た目になるか確認できる。

**なぜこう書くのか：**
フィルター名だけでは、実際にどのような加工になるか分かりにくい。サムネイルを表示することで、視覚的に選びやすくなる。

**もしこう書かなかったら：**
ユーザーは「セピア」「ブルーム」などの名前だけで選ぶことになる。特に初心者には違いが分かりにくく、使いにくいアプリになる。

---

### 加工した画像の保存

```swift
func saveFilteredImage() {
    guard let uiImage = originalUIImage,
          let ciImage = CIImage(image: uiImage),
          let output = currentFilter.apply(to: ciImage, context: context),
          let cgImage = context.createCGImage(output, from: ciImage.extent) else { return }

    let finalImage = UIImage(cgImage: cgImage)
    UIImageWriteToSavedPhotosAlbum(finalImage, nil, nil, nil)

    saveMessage = "写真を保存しました"
    showSaveAlert = true
}
```

**何をしているか：**
現在選択しているフィルターを元画像に適用し、その結果を UIImage に変換してフォトライブラリに保存している。保存後にはアラートを表示して、保存が完了したことをユーザーに知らせている。

**なぜこう書くのか：**
CoreImage の処理結果は CIImage なので、そのままではフォトライブラリに保存できない。そのため、CIContext で CGImage に変換し、さらに UIImage に変換してから保存している。また、Info.plist に NSPhotoLibraryAddUsageDescription を追加する必要がある。これは、アプリがフォトライブラリへ保存する理由をユーザーに説明するためである。

**もしこう書かなかったら：**
UIImage に変換しなければ、保存処理に渡すことができない。また、Info.plist に権限説明を書いていない場合、実機では保存時に問題が起きる可能性がある。アプリがフォトライブラリを使う理由をOSに伝えることが必要である。

---



## 新しく学んだSwiftの文法・API

| 項目                                  | 説明                        | 使用例                                                         |
| ----------------------------------- | ------------------------- | ----------------------------------------------------------- |
| `CoreImage`                         | 画像処理を行うためのフレームワーク         | `import CoreImage`                                          |
| `CIFilterBuiltins`                  | 型安全にCIFilterを使うためのモジュール   | `CIFilter.sepiaTone()`                                      |
| `CIImage`                           | CoreImageで扱う画像形式          | `CIImage(image: uiImage)`                                   |
| `CIContext`                         | CIImageをCGImageに変換するために使う | `context.createCGImage(...)`                                |
| `CIFilter`                          | 画像に効果を加えるフィルター            | `CIFilter.bloom()`                                          |
| `CaseIterable`                      | enumの全ケースを取得できるようにする      | `PhotoFilter.allCases`                                      |
| `Identifiable`                      | ForEachで要素を識別できるようにする     | `var id: String { rawValue }`                               |
| `.onChange`                         | 状態が変わった時に処理を実行する          | `.onChange(of: currentFilter)`                              |
| `UIImageWriteToSavedPhotosAlbum`    | UIImageをフォトライブラリに保存する     | `UIImageWriteToSavedPhotosAlbum(finalImage, nil, nil, nil)` |
| `NSPhotoLibraryAddUsageDescription` | 写真保存の権限説明                 | Info.plistに追加する                                             |



## 自分の実験メモ


**実験1：**
- やったこと：currentFilter = .original を削除した
- 結果：新しい写真を選んだ時に、前に選んでいたフィルターが残ったままになった
- わかったこと：新しい画像を読み込む時は、フィルター状態をリセットした方が自然だと分かった

**実験2：**
- やったこと：.onChange(of: currentFilter) を削除した
- 結果：フィルターをタップしても、大きい画像の表示が変わらなかった
- わかったこと：状態の変化に合わせて applyFilter() を呼ぶ必要があると分かった

## AIに聞いて特に理解が深まった質問

1. **質問：なぜ PhotoFilter は enum で定義するのか？ struct や class ではだめなのか？**

   **得られた理解：**  
   フィルターの種類は「オリジナル」「セピア」「モノクロ」など、あらかじめ決まった選択肢である。そのため、自由にデータを作る struct や class よりも、決まった種類を表す enum の方が分かりやすいと理解した。  
   また、CaseIterable によって `allCases` が使えるため、すべてのフィルターを一覧表示できる。Identifiable に準拠することで、SwiftUI の ForEach でも各フィルターを区別して扱える。

2. **質問：なぜ CIFilterBuiltins を import するのか？ 古い CIFilter(name:) との違いは何か？**

   **得られた理解：**  
   CIFilterBuiltins を import すると、`CIFilter.sepiaTone()` や `CIFilter.bloom()` のように、型安全な書き方ができる。  
   古い `CIFilter(name: "CISepiaTone")` の書き方では、フィルター名を文字列で指定するため、スペルを間違えると nil になる可能性がある。新しい書き方の方が、コードが読みやすく、ミスも減らせると分かった。

3. **質問：なぜ `let context = CIContext()` を1個だけ作るのか？**

   **得られた理解：**  
   CIContext は CoreImage の画像処理結果を実際の画像として取り出すために使う。作成コストが比較的大きいため、毎回 `applyFilter()` の中で新しく作るのではなく、1つだけ作って使い回す方が効率が良い。  
   このアプリでは、フィルターをタップするたびに画像処理が行われるため、毎回 CIContext を作ると動作が重くなる可能性があると分かった。

4. **質問：なぜ `apply(to:context:)` の戻り値は `CIImage?` なのか？**

   **得られた理解：**  
   フィルター処理は必ず成功するとは限らない。入力画像が正しくない場合や、フィルターの出力画像が作れない場合には nil が返る可能性がある。  
   そのため、戻り値を Optional にしておき、呼び出し側では `guard let` を使って安全に処理していると分かった。

5. **質問：なぜ `originalUIImage` と `displayImage` を別々に持つのか？**

   **得られた理解：**  
   `originalUIImage` は元画像を保存するため、`displayImage` は画面に表示するために使われている。分けて管理することで、フィルターを何度変更しても、常に元画像から加工し直せる。  
   もし1つの画像だけで管理すると、加工済み画像にさらに別のフィルターを重ねてしまい、画質が悪くなったり、意図しない見た目になったりする可能性があると理解した。

6. **質問：Info.plist に `NSPhotoLibraryAddUsageDescription` を書かなかったらどうなるのか？**

   **得られた理解：**  
   フォトライブラリに画像を保存する機能を使う場合、アプリが写真ライブラリを使う理由を Info.plist に書く必要がある。これは、ユーザーに対して「なぜ写真ライブラリを使うのか」を説明するためである。  
   この設定がないと、実機で保存機能を使う時に正しく動作しない可能性がある。シミュレータでは確認できることが限られる場合もあるため、写真保存のような権限が関係する機能は実機で確認することが重要だと分かった。


## この章のまとめ

この章では、PhotosPickerで選択した写真にCoreImageのフィルターを適用し、加工後の画像を保存する方法を学んだ。特に、UIImage、CIImage、CGImage の変換の流れや、CIFilterを使った画像加工の基本を理解することができた。また、PhotoFilterをenumで管理することで、複数のフィルターを安全に扱えることも分かった。CaseIterable や Identifiable を使うことで、SwiftUI の ForEach と組み合わせやすくなっている。さらに、originalUIImage と displayImage を分けて管理する理由や、CIContext を使い回す理由も学んだ。画像加工では、元画像を残しておき、表示用の画像だけを更新する設計が大切だと理解した。

