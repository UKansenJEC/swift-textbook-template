# 第4章：データの永続化

> 執筆者：ウカンセン
> 最終更新：2026-05-27

## この章で学ぶこと

例：この章では、AppStorageとSwiftDataを使ってアプリのデータを端末に永続的に保存する方法を学ぶ。具体的にはSwiftDataを使ったメモアプリを題材として、@Modelでデータモデルを定義し、modelContextを使ったデータ操作、@Queryによる動的なデータ取得、そして@AppStorageによるユーザー設定の保存を実装する。

## 模範コードの全体像

```swift
// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

```

**このアプリは何をするものか：**

このアプリは、メモを追加・編集・削除できるメモ帳アプリである。
作成したメモは SwiftData を利用して端末内に保存されるため、アプリを終了しても内容が消えない。また、ユーザー名やお気に入り表示設定は AppStorage を使って保存されるため、次回起動時にも前回の設定が保持される。
お気に入り機能によって重要なメモを管理でき、設定画面では表示順序やユーザー名を変更することができる。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool
}
```

**何をしているか：**
Memo クラスを SwiftData のデータモデルとして定義している。タイトル、内容、作成日時、お気に入り状態を保存するためのクラスである。

**なぜこう書くのか：**
@Model を付けることで、SwiftData がこのクラスを永続化対象として認識できる。データベース用のコードを自分で書かなくても、自動的に保存・読み込みが行われる。

**もしこう書かなかったら：**
Memo は普通のクラスとして扱われるだけで、SwiftData に保存できなくなる。@Query でも取得できなくなる。

---

### データの追加・削除（modelContext）

```swift
let memo = Memo(title: title, content: content)
modelContext.insert(memo)
```
```swift
modelContext.delete(memo)
```

**何をしているか：**
新しいメモを作成して SwiftData に保存したり、不要なメモを削除したりしている。

**なぜこう書くのか：**
modelContext は SwiftData のデータ管理を担当するオブジェクトであり、insert() で追加、delete() で削除を行う。

**もしこう書かなかったら：**
画面上で入力してもデータベースには保存されず、アプリを再起動すると内容が消えてしまう。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse)
private var memos: [Memo]
```

**何をしているか：**
SwiftData に保存されている Memo データを取得している。作成日時の新しい順に並べて表示している。

**なぜこう書くのか：**
@Query を使うことで、データベースの内容が変わった時に画面も自動で更新される。手動で再読み込み処理を書く必要がない。

**もしこう書かなかったら：**
保存したデータを取得できない。また、追加や削除をしても画面に自動反映されなくなる。

---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite")
private var sortByFavorite: Bool = false

@AppStorage("userName")
private var userName: String = ""
```

**何をしているか：**
ユーザー名や表示設定を端末内に保存している。

**なぜこう書くのか：**
AppStorage は UserDefaults を簡単に扱うための仕組みであり、設定情報を少ないコードで保存できる。

**もしこう書かなかったら：**
アプリを終了するたびにユーザー名や表示設定が初期状態に戻ってしまう。

---

### @Bindableによる編集画面

```swift
@Bindable var memo: Memo
```

**何をしているか：**
編集画面で Memo オブジェクトを直接編集できるようにしている。

**なぜこう書くのか：**
TextField や Toggle と Memo のプロパティを Binding で接続するためである。

**もしこう書かなかったら：**
編集した内容が Memo オブジェクトへ反映されなくなる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model class Memo` |
| `@Query` | SwiftDataからデータを取得する | `@Query var memos: [Memo]` |
| `modelContext` | データの追加・削除を行う | `modelContext.insert(memo)` |
| `@AppStorage` | アプリ設定を保存する | `@AppStorage("userName")` |
| `@Bindable` | モデルを直接編集可能にする | `@Bindable var memo: Memo` |
| `.modelContainer()` | SwiftDataの保存領域を作成する | `.modelContainer(for: Memo.self)` |
| `ContentUnavailableView` | データがない状態を表示する | `ContentUnavailableView(...)` |
| `NavigationLink` | 画面遷移を行う | `NavigationLink(destination: ...)` |

## 自分の実験メモ


**実験1：**
- やったこと：@AppStorage を普通の @State に変更した
- 結果：アプリを再起動すると設定が消えた
- わかったこと：@AppStorage は設定を永続化するために必要

**実験2：**
- やったこと：@Query を削除した
- 結果：保存済みメモが表示されなくなった
- わかったこと：SwiftData のデータ取得には @Query が必要

**実験3：**
- やったこと：.modelContainer(for: Memo.self) を削除した
- 結果：データ保存が正常に動作しなかった
- わかったこと：SwiftData を使うためには modelContainer の設定が必要

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：なぜ @Model が必要なのか？**

   **得られた理解：**
   @Model を付けることで SwiftData がクラスをデータモデルとして認識し、自動的に保存や取得ができるようになると理解した。

2. **質問：@Query はなぜ必要なのか？**

   **得られた理解：**
   SwiftData に保存されているデータを取得し、変更があった時に画面へ自動反映する役割があると理解した。

3. **質問：AppStorage と SwiftData は何が違うのか？**

   **得られた理解：**
   AppStorage は設定値のような単純なデータを保存するために使い、SwiftData はメモのような構造化されたデータを保存するために使うと理解した。

## この章のまとめ

この章では、SwiftData と AppStorage を利用したデータの永続化について学んだ。
SwiftData では、@Model を使ってデータモデルを定義し、modelContext による追加・削除や、@Query によるデータ取得を行った。また、@Bindable を利用することで、保存済みデータを直接編集できることも理解した。
さらに、AppStorage を利用してユーザー名や表示設定を保存する方法も学んだ。SwiftData と AppStorage はどちらもデータを保存する仕組みだが、用途が異なることを理解できた。
今後アプリ開発を行う際には、「設定情報は AppStorage」「複数のデータを管理する場合は SwiftData」という使い分けを意識したい。
