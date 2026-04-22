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

这部分是在用 Swift 的结构体来表示从 API 返回的 JSON 数据的形式。
SearchResponse 表示整个 API 的响应，其中的 results 里装着歌曲数据的数组。
Song 表示一首歌曲的信息，里面包含歌曲 ID、歌曲名、艺人名、图片 URL 等内容。

**なぜこう書くのか：**
Codableをつけることで、JSONDecoderを使ってJSONをそのままSwiftの構造体に変換できるからである。
手作業で1つずつ取り出すよりも安全で見通しがよい。
また、SongにIdentifiableをつけ、var id: Int { trackId }とすることで、Listの中で各要素を区別できるようにしている。let id: Intを別に持たせるのではなく、すでに存在するtrackIdを利用しているので、情報の重複を避けられる。

加上 Codable，就可以使用 JSONDecoder 直接把 JSON 转换成 Swift 的结构体。
和手动一个一个把值取出来相比，这样写更安全，也更容易看懂整体结构。
另外，给 Song 加上 Identifiable，并写成 var id: Int { trackId }，是为了让 List 能够区分其中的每一个元素。
这里不是另外再写一个 let id: Int，而是直接利用本来就已经存在的 trackId，这样可以避免信息重复。

**もしこう書かなかったら：**
Codableがなければ、JSONDecoderでそのままデコードできず、データを扱うのがかなり面倒になる。
Identifiableがないと、List(songs)のような書き方ができず、別の方法でIDを指定する必要が出てくる。
実際にIdentifiableを外すと、リスト表示部分で要素の識別に関するエラーが出るので、この指定は必要だと分かった。

如果没有 Codable，就不能直接用 JSONDecoder 来解码 JSON，处理数据会变得相当麻烦。
如果没有 Identifiable，就不能写成 List(songs) 这样的形式，而必须用别的方法另外指定 ID。
实际上，把 Identifiable 去掉之后，在列表显示的部分就会出现和元素识别有关的错误，所以我明白了这个指定是有必要的。


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

这个函数的作用是，把输入的艺人名称转换成适合放入 URL 的格式，并用这个字符串来生成 iTunes Search API 的请求地址。
之后，从互联网获取数据，将返回的 JSON 转换成 SearchResponse 类型，并赋值给 songs。
在搜索过程中，通过把 isLoading = true，让界面显示当前正在加载。

**なぜこう書くのか：**
addingPercentEncodingを使うのは、日本語や空白をそのままURLに入れると不正なURLになる場合があるからである。
guard letを使うことで、URL文字列が作れなかったときやURL型に変換できなかったときに、その場で早く処理を終えられる。
try await URLSession.shared.data(from:)を使うことで、非同期のネット通信を比較的読みやすく書ける。
昔のコールバック形式よりも流れが追いやすいと感じた。

使用 addingPercentEncoding，是因为如果直接把日文或空格放进 URL 中，可能会变成非法的 URL。
使用 guard let，可以在 URL 字符串生成失败或者无法转换为 URL 类型时，立即结束处理流程。
通过 try await URLSession.shared.data(from:)，可以用比较直观的方式来写异步的网络请求。
相比以前的回调（callback）写法，这种方式更容易看懂程序的执行流程。

**もしこう書かなかったら：**
検索文字列をエンコードしないと、日本語のアーティスト名などでURLが壊れる可能性がある。
guard letを使わずに無理に進めると、不正なURLをそのまま扱って問題が起きるかもしれない。
isLoadingの切り替えがなければ、ユーザーは今検索中なのか分からない。do-catchを外すと、通信失敗時のエラー処理ができず、原因が分かりにくくなる。

如果不对搜索字符串进行编码，在输入日文艺人名等情况下，URL 可能会损坏，从而导致请求失败。
如果不使用 guard let，而是继续执行的话，可能会在使用错误 URL 的情况下引发问题。
如果不切换 isLoading，用户就无法判断当前是否正在进行搜索。
如果去掉 do-catch，在通信失败时就无法进行错误处理，也更难找到问题原因。

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

这一部分是在构建整个应用的界面。
上面是搜索栏和搜索按钮，下面的内容会根据当前状态发生变化。
正在搜索时显示 ProgressView，没有搜索结果时显示说明信息，有结果时显示歌曲列表。
整体是一个根据状态切换显示内容的结构。

**なぜこう書くのか：**
SwiftUIでは、状態に応じて表示を切り替える書き方が自然である。
songsやisLoadingの値が変わると、自動的に画面が更新されるので、画面遷移や表示切り替えのコードを細かく書かなくてもよい。
TaskでsearchMusic()を呼んでいるのは、ボタンのactionの中でasync関数を直接呼べないためである。
また、.disabled(searchText.isEmpty)によって、空のまま検索されるのを防いでいる。

在 SwiftUI 中，根据“状态”来切换界面是一种很自然的写法。
当 songs 或 isLoading 的值发生变化时，界面会自动更新，因此不需要手动写很多界面切换的代码。
在按钮的 action 中用 Task 来调用 searchMusic()，是因为在这个位置不能直接调用 async 函数。
另外，通过 .disabled(searchText.isEmpty)，可以防止在输入为空的情况下执行搜索。

**もしこう書かなかったら：**
Taskで囲まないと、await searchMusic()をそのままボタンのaction内で呼べずエラーになる。
if isLoadingなどの条件分岐がなければ、検索中なのか、まだ検索していないのか、結果があるのかが画面から分かりにくくなる。
disabledを外すと、空文字でも検索ボタンが押せてしまい、無駄な通信につながると思った。

如果不使用 Task 包裹，就不能在按钮的 action 中直接调用 await searchMusic()，会产生错误。
如果没有像 if isLoading 这样的条件分支，用户就很难从界面判断当前是正在搜索、还没有搜索，还是已经有结果。
如果去掉 disabled，即使输入为空也可以点击搜索按钮，会导致不必要的网络请求。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| | | |
| | | |
| | | |

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
