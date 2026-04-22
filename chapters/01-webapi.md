# 第1章：WebAPIの基本

> 執筆者：ウカンセン
> 最終更新：2026-04-22

## この章で学ぶこと

この章では、インターネット上のサービス（API）からデータを取得して、アプリ内に表示する方法を学ぶ。具体的にはiTunes Search APIを使って音楽を検索し、その結果をリスト表示するアプリを題材にする。

## 模範コードの全体像

```swift
// ============================================
// 第1章（基本）：iTunes Search APIで音楽を検索するアプリ
// ============================================
// このアプリは、iTunes Search APIを使って
// 音楽（曲）を検索し、結果をリスト表示します。
// APIキーは不要で、すぐに動かすことができます。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

## コードの詳細解説

### データモデル（Codable構造体）

```swift
/struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}
```

**何をしているか：**
この部分では、APIから返ってくるJSONデータの形をSwiftの構造体で表している。
SearchResponseはAPI全体のレスポンスを表し、その中のresultsに曲データの配列が入っている。
Songは1曲分の情報を表していて、曲ID、曲名、アーティスト名、画像URLなどを持っている。

**なぜこう書くのか：**
Codableをつけることで、JSONDecoderを使ってJSONをそのままSwiftの構造体に変換できるからである。
手作業で1つずつ取り出すよりも安全で見通しがよい。
また、SongにIdentifiableをつけ、var id: Int { trackId }とすることで、Listの中で各要素を区別できるようにしている。let id: Intを別に持たせるのではなく、すでに存在するtrackIdを利用しているので、情報の重複を避けられる。

**もしこう書かなかったら：**
Codableがなければ、JSONDecoderでそのままデコードできず、データを扱うのがかなり面倒になる。
Identifiableがないと、List(songs)のような書き方ができず、別の方法でIDを指定する必要が出てくる。
実際にIdentifiableを外すと、リスト表示部分で要素の識別に関するエラーが出るので、この指定は必要だと分かった。

---

### API通信の処理

```swift
func searchMusic() async {
    guard let encodedText = searchText.addingPercentEncoding(
        withAllowedCharacters: .urlQueryAllowed
    ) else { return }

    let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

    guard let url = URL(string: urlString) else { return }

    isLoading = true

    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(SearchResponse.self, from: data)
        songs = response.results
    } catch {
        print("エラー: \(error.localizedDescription)")
        songs = []
    }

    isLoading = false
}
```

**何をしているか：**
この関数は、入力されたアーティスト名をURL用に変換し、その文字列を使ってiTunes Search APIのURLを作成している。
その後、インターネットからデータを取得し、JSONをSearchResponse型に変換して、songsに代入している。検索中はisLoading = trueにして、読み込み中であることを画面に表示している。

**なぜこう書くのか：**
addingPercentEncodingを使うのは、日本語や空白をそのままURLに入れると不正なURLになる場合があるからである。
guard letを使うことで、URL文字列が作れなかったときやURL型に変換できなかったときに、その場で早く処理を終えられる。
try await URLSession.shared.data(from:)を使うことで、非同期のネット通信を比較的読みやすく書ける。
昔のコールバック形式よりも流れが追いやすいと感じた。

**もしこう書かなかったら：**
検索文字列をエンコードしないと、日本語のアーティスト名などでURLが壊れる可能性がある。
guard letを使わずに無理に進めると、不正なURLをそのまま扱って問題が起きるかもしれない。
isLoadingの切り替えがなければ、ユーザーは今検索中なのか分からない。do-catchを外すと、通信失敗時のエラー処理ができず、原因が分かりにくくなる。

---

### ビューの構成

```swift
var body: some View {
    NavigationStack {
        VStack {
            HStack {
                TextField("アーティスト名を入力", text: $searchText)
                    .textFieldStyle(.roundedBorder)

                Button("検索") {
                    Task {
                        await searchMusic()
                    }
                }
                .buttonStyle(.borderedProminent)
                .disabled(searchText.isEmpty)
            }
            .padding(.horizontal)

            if isLoading {
                ProgressView("検索中...")
                    .padding()
                Spacer()
            } else if songs.isEmpty {
                ContentUnavailableView(
                    "曲を検索してみよう",
                    systemImage: "music.note",
                    description: Text("アーティスト名を入力して検索ボタンを押してください")
                )
            } else {
                List(songs) { song in
                    SongRow(song: song)
                }
            }
        }
        .navigationTitle("Music Search")
    }
}
```

**何をしているか：**
この部分では、アプリの画面全体を作っている。
上部に検索バーと検索ボタンがあり、下の部分は状態によって表示が変わる。
検索中はProgressView、検索結果がないときは説明メッセージ、結果があるときは曲のリストを表示する構成になっている。

**なぜこう書くのか：**
SwiftUIでは、状態に応じて表示を切り替える書き方が自然である。
songsやisLoadingの値が変わると、自動的に画面が更新されるので、画面遷移や表示切り替えのコードを細かく書かなくてもよい。
TaskでsearchMusic()を呼んでいるのは、ボタンのactionの中でasync関数を直接呼べないためである。
また、.disabled(searchText.isEmpty)によって、空のまま検索されるのを防いでいる。

**もしこう書かなかったら：**
Taskで囲まないと、await searchMusic()をそのままボタンのaction内で呼べずエラーになる。
if isLoadingなどの条件分岐がなければ、検索中なのか、まだ検索していないのか、結果があるのかが画面から分かりにくくなる。
disabledを外すと、空文字でも検索ボタンが押せてしまい、無駄な通信につながると思った。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目                       | 説明                            | 使用例                                                           |
| ------------------------ | ----------------------------- | ------------------------------------------------------------- |
| `Codable`                | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }`                                |
| `Identifiable`           | リスト表示で各要素を識別するためのプロトコル        | `struct Song: Codable, Identifiable`                          |
| computed property        | 値を保存せず、計算して返すプロパティ            | `var id: Int { trackId }`                                     |
| `@State`                 | Viewの状態を保持し、値が変わると画面を再描画する    | `@State private var songs: [Song] = []`                       |
| `guard let`              | 値が取り出せない場合に早めに処理を抜ける書き方       | `guard let url = URL(string: urlString) else { return }`      |
| `async/await`            | 非同期処理を順番に読める形で書く構文            | `let (data, _) = try await URLSession.shared.data(from: url)` |
| `URLSession.shared`      | 共通のネットワーク通信オブジェクト             | `URLSession.shared.data(from: url)`                           |
| `Task`                   | 非同期処理を開始するための単位               | `Task { await searchMusic() }`                                |
| `AsyncImage`             | URLから画像を非同期で読み込んで表示するView     | `AsyncImage(url: URL(string: song.artworkUrl100))`            |
| `ContentUnavailableView` | データがないときの案内表示を簡単に作れるView      | `ContentUnavailableView("曲を検索してみよう", ...)`                    |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：Button("検索")の中のTask { ... }を外して、await searchMusic()を直接書こうとした。
- 結果：非同期関数の呼び出しに関するエラーが出て、そのままでは動かなかった。
- わかったこと：Buttonのactionの中では、async関数を直接呼べないので、Taskで囲む必要がある。

**実験2：**
- やったこと：SongからIdentifiableを外してみた。
- 結果：List(songs)の部分で要素の識別に関するエラーが出た。
- わかったこと：Listでデータを並べるときには、各要素を区別するためのIDが必要だと分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   Codableとは何か。JSONとSwiftの構造体はどう対応するのか。
   **得られた理解：**
   Codableをつけることで、JSONのキーとSwiftのプロパティ名が対応していれば、自動的に構造体へ変換できる。APIのデータを扱うときにとても重要だと分かった。

2. **質問：**
   guard letとif letはどちらもオプショナルを安全に取り出す書き方だと思うが、どう使い分けるのか。
   **得られた理解：**
   条件を満たさない場合にその場で処理を終わらせたいときはguard letの方が読みやすい。今回のようにURLが作れないなら先へ進めない処理では、guard letが向いていると感じた。

3. **質問：**
   Task { await searchMusic() }と書く理由は何か。
   **得られた理解：**
   searchMusic()はasync関数なので、普通のボタン処理の中からそのまま呼べない。Taskを使うことで、非同期処理を開始できると分かった。

## この章のまとめ
この章で最も重要だと思ったのは、WebAPIを使うアプリでは「データの形を構造体で表すこと」「非同期で通信すること」「取得した結果を状態として画面に反映すること」の3つが基本になるという点である。特にSwiftUIでは、@Stateの値が変わると画面も変わるので、データの流れを意識するとコードが理解しやすくなると感じた。また、分からない部分を生成AIに質問しながら読むことで、ただ写すだけよりも理解が深まりやすかった。

# 応用編

## 模範コードの全体像
```swift
// ============================================
// 第1章（応用）：エラーハンドリングとMVVM構成
// ============================================
// 基本編のコードをMVVMパターンで書き直し、
// エラーハンドリングとローディング状態を
// より適切に管理するバージョンです。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let resultCount: Int
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }

    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}

// MARK: - ViewModel

@Observable
class MusicSearchViewModel {
    var songs: [Song] = []
    var searchText: String = ""
    var isLoading: Bool = false
    var errorMessage: String?

    enum SearchError: LocalizedError {
        case invalidURL
        case networkError(Error)
        case decodingError(Error)
        case noResults

        var errorDescription: String? {
            switch self {
            case .invalidURL:
                return "検索URLの作成に失敗しました"
            case .networkError(let error):
                return "通信エラー: \(error.localizedDescription)"
            case .decodingError:
                return "データの読み取りに失敗しました"
            case .noResults:
                return "検索結果が見つかりませんでした"
            }
        }
    }

    func searchMusic() async {
        guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }

        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        isLoading = true
        errorMessage = nil

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)

            if response.results.isEmpty {
                errorMessage = SearchError.noResults.errorDescription
                songs = []
            } else {
                songs = response.results
            }
        } catch let error as DecodingError {
            errorMessage = SearchError.decodingError(error).errorDescription
            songs = []
        } catch {
            errorMessage = SearchError.networkError(error).errorDescription
            songs = []
        }

        isLoading = false
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - 検索バー

    private var searchBar: some View {
        HStack {
            TextField("アーティスト名を入力", text: $viewModel.searchText)
                .textFieldStyle(.roundedBorder)
                .onSubmit {
                    Task { await viewModel.searchMusic() }
                }

            Button("検索") {
                Task { await viewModel.searchMusic() }
            }
            .buttonStyle(.borderedProminent)
            .disabled(viewModel.searchText.isEmpty || viewModel.isLoading)
        }
        .padding()
    }

    // MARK: - コンテンツエリア

    @ViewBuilder
    private var contentArea: some View {
        if viewModel.isLoading {
            Spacer()
            ProgressView("検索中...")
            Spacer()
        } else if viewModel.songs.isEmpty {
            ContentUnavailableView(
                "曲を検索してみよう",
                systemImage: "music.note",
                description: Text("アーティスト名を入力して検索ボタンを押してください")
            )
        } else {
            List(viewModel.songs) { song in
                NavigationLink(destination: SongDetailView(song: song)) {
                    SongRow(song: song)
                }
            }
        }
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                RoundedRectangle(cornerRadius: 8)
                    .fill(.gray.opacity(0.2))
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)
                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            Spacer()

            Text(song.priceText)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding(.vertical, 4)
    }
}

// MARK: - 詳細ビュー

struct SongDetailView: View {
    let song: Song

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                    image.resizable().aspectRatio(contentMode: .fit)
                } placeholder: {
                    ProgressView()
                }
                .frame(width: 200, height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 8)

                Text(song.trackName)
                    .font(.title2)
                    .bold()
                    .multilineTextAlignment(.center)

                Text(song.artistName)
                    .font(.title3)
                    .foregroundStyle(.secondary)

                if let albumName = song.collectionName {
                    Text(albumName)
                        .font(.subheadline)
                        .foregroundStyle(.tertiary)
                }

                Text(song.priceText)
                    .font(.headline)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 8)
                    .background(.blue.opacity(0.1))
                    .clipShape(Capsule())
            }
            .padding()
        }
        .navigationTitle("曲の詳細")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - エラーバナー

struct ErrorBanner: View {
    let message: String

    var body: some View {
        HStack {
            Image(systemName: "exclamationmark.triangle.fill")
                .foregroundStyle(.yellow)
            Text(message)
                .font(.caption)
        }
        .padding(10)
        .frame(maxWidth: .infinity)
        .background(.red.opacity(0.1))
    }
}

#Preview {
    ContentView()
}
```

## このアプリは何をするものか

応用編のアプリは、基本編と同じくiTunes Search APIを使って音楽を検索し、その結果を一覧表示するアプリである。  
ただし、内部の設計が変わっており、検索中の表示、エラーメッセージの表示、曲の詳細画面への遷移などが追加されている。  

基本編では、通信に失敗してもエラーが画面上に分かりやすく表示されなかったが、応用編ではエラーバナーによって利用者にも状況が伝わるようになっている。  
また、検索結果の曲をタップすると詳細画面に移動できるので、基本編よりも実際のアプリに近い作りになっていると感じた。

---

## 基本編との違い

### 動作の違い
- 検索中に「検索中...」と表示される  
- 通信エラーや検索結果なしの場合にエラーメッセージが表示される  
- 曲をタップすると詳細画面へ遷移する  
- 価格情報が表示される  

### コードの違い
- View と ViewModel を分離している  
- エラーを enum で定義し、画面に表示する  
- 状態（songs / isLoading / errorMessage）を ViewModel で管理  
- NavigationLink による画面遷移を追加  

---

## MVVMとは何か

MVVMとは、画面表示（View）とデータ処理（ViewModel）を分離する設計パターンである。  

今回のコードでは、ContentView や SongRow が View にあたり、MusicSearchViewModel が ViewModel にあたる。  

Viewは表示を担当し、ViewModelは状態管理や処理を担当することで、コードの見通しが良くなると感じた。

---

## コードの詳細解説

### ViewModel

```swift
@Observable
class MusicSearchViewModel {
    var songs: [Song] = []
    var searchText: String = ""
    var isLoading: Bool = false
    var errorMessage: String?
```

**何をしているか：**  
検索結果、検索文字列、ローディング状態、エラーメッセージなどをまとめて管理している。  

**なぜこう書くのか：**  
状態と処理を分離することで、Viewのコードをシンプルに保つため。  

**もしこう書かなかったら：**  
Viewにすべての処理が混ざり、コードが読みづらくなる。

---

### エラーハンドリング

```swift
enum SearchError: LocalizedError {
    case invalidURL
    case networkError(Error)
    case decodingError(Error)
    case noResults

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "検索URLの作成に失敗しました"
        case .networkError(let error):
            return "通信エラー: \(error.localizedDescription)"
        case .decodingError:
            return "データの読み取りに失敗しました"
        case .noResults:
            return "検索結果が見つかりませんでした"
        }
    }
}
```

**何をしているか：**  
検索時に発生するエラーを enum で定義し、種類ごとに処理している。  

**なぜこう書くのか：**  
エラーの種類を明確にし、適切なメッセージを表示するため。  

**もしこう書かなかったら：**  
エラーの原因が分かりにくくなり、ユーザーにも状況が伝わらない。

---

### API通信の処理

```swift
func searchMusic() async {
    guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }

    guard let encodedText = searchText.addingPercentEncoding(
        withAllowedCharacters: .urlQueryAllowed
    ) else {
        errorMessage = SearchError.invalidURL.errorDescription
        return
    }

    let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

    guard let url = URL(string: urlString) else {
        errorMessage = SearchError.invalidURL.errorDescription
        return
    }

    isLoading = true
    errorMessage = nil

    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(SearchResponse.self, from: data)

        if response.results.isEmpty {
            errorMessage = SearchError.noResults.errorDescription
            songs = []
        } else {
            songs = response.results
        }
    } catch let error as DecodingError {
        errorMessage = SearchError.decodingError(error).errorDescription
        songs = []
    } catch {
        errorMessage = SearchError.networkError(error).errorDescription
        songs = []
    }

    isLoading = false
}
```

**何をしているか：**  
検索文字列からURLを作成し、API通信を行い、結果を songs に代入している。  

**なぜこう書くのか：**  
URLの安全性を保ちつつ、非同期処理でデータを取得するため。  

**もしこう書かなかったら：**  
URLエラーや通信エラーに適切に対応できなくなる。

---

### メインビュー

```swift
struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }
}
```

**何をしているか：**  
検索バー、エラー表示、検索結果を組み合わせて画面を構成している。  

**なぜこう書くのか：**  
ViewModelの状態に応じて表示を切り替えるため。  

**もしこう書かなかったら：**  
画面の状態管理が複雑になり、コードが読みづらくなる。

---

### コンテンツエリア

```swift
@ViewBuilder
private var contentArea: some View {
    if viewModel.isLoading {
        Spacer()
        ProgressView("検索中...")
        Spacer()
    } else if viewModel.songs.isEmpty {
        ContentUnavailableView(
            "曲を検索してみよう",
            systemImage: "music.note",
            description: Text("アーティスト名を入力して検索ボタンを押してください")
        )
    } else {
        List(viewModel.songs) { song in
            NavigationLink(destination: SongDetailView(song: song)) {
                SongRow(song: song)
            }
        }
    }
}
```

**何をしているか：**  
状態に応じて表示内容を切り替えている。  

**なぜこう書くのか：**  
SwiftUIでは状態に応じたUI変更が基本のため。  

**もしこう書かなかったら：**  
検索中・空状態・結果表示の区別ができなくなる。

---

### 詳細ビュー

**何をしているか：**  
選択した曲の詳細情報を表示する画面。  

**なぜこう書くのか：**  
一覧画面と詳細画面を分けることで情報を整理するため。  

**もしこう書かなかったら：**  
詳細情報を表示する場所がなくなる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 |
|------|------|
| @Observable | 状態の変化を監視してUI更新する |
| class | 参照型として状態を共有する |
| enum | 状態や種類をまとめる |
| LocalizedError | エラーメッセージを持たせる |
| @ViewBuilder | 条件でViewを切り替える |
| NavigationLink | 画面遷移を行う |
| ContentUnavailableView | 空状態表示 |
| onSubmit | 入力確定時の処理 |
| trimmingCharacters | 文字列の空白除去 |

---

## 自分の実験メモ

**実験1：**
- やったこと：@Observable を外した  
- 結果：画面が更新されなくなった  
- わかったこと：状態監視が必要  

**実験2：**
- やったこと：errorMessage を初期化しなかった  
- 結果：エラー表示が残った  
- わかったこと：状態リセットが必要  

**実験3：**
- やったこと：NavigationLink を削除した  
- 結果：詳細画面に遷移できなくなった  
- わかったこと：画面遷移には必要  

---

## この章のまとめ（応用編）

応用編では、基本編と同じ機能でも設計を変えることでコードの見通しや拡張性が大きく変わることを学んだ。  

特に、MVVMによって役割を分けることで、コードが整理されると感じた。  
また、エラーハンドリングによって、ユーザーにも状況を伝えることの重要性を理解した。
