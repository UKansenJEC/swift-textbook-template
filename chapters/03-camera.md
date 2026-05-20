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
