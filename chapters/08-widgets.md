# 第8章：ウィジェット

> 執筆者：ウカンセン  
> 最終更新：2026-07-26

## この章で学ぶこと

この章では、WidgetKitを使ってホーム画面に表示できるウィジェットを実装する方法を学ぶ。具体的には、毎日異なる名言を表示するアプリとウィジェットを題材にして、TimelineProviderの仕組み、TimelineEntryの役割、ウィジェットビューの構成、複数サイズへの対応、そしてメインアプリとウィジェットでコードを共有する方法を学ぶ。

通常のSwiftUIアプリでは、アプリを開いて画面を表示する。しかし、ウィジェットを使うと、アプリを開かなくてもホーム画面から情報を確認できる。今回のアプリでは、今日の名言を小サイズまたは中サイズのウィジェットとして表示する。

## 模範コードの全体像

今回のプログラムは、メインアプリ側のコード、共有データのコード、ウィジェット側のコードの3つに分けて作成する。

### QuoteStore.swift

```swift
import Foundation

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(
            of: .day,
            in: .year,
            for: Date()
        ) ?? 0

        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}
```

### ContentView.swift

```swift
import SwiftUI

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)

                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}
```

### QuoteWidget.swift

```swift
import WidgetKit
import SwiftUI

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(
                id: 0,
                text: "読み込み中...",
                author: ""
            )
        )
    }

    func getSnapshot(
        in context: Context,
        completion: @escaping (QuoteEntry) -> Void
    ) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )

        completion(entry)
    }

    func getTimeline(
        in context: Context,
        completion: @escaping (Timeline<QuoteEntry>) -> Void
    ) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()

        let entry = QuoteEntry(
            date: currentDate,
            quote: quote
        )

        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(
                byAdding: .day,
                value: 1,
                to: currentDate
            )!
        )

        let timeline = Timeline(
            entries: [entry],
            policy: .after(tomorrow)
        )

        completion(timeline)
    }
}

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry

    @Environment(\.widgetFamily)
    var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget

        case .systemMedium:
            mediumWidget

        default:
            mediumWidget
        }
    }

    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(
            kind: kind,
            provider: QuoteProvider()
        ) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(
                    .fill.tertiary,
                    for: .widget
                )
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([
            .systemSmall,
            .systemMedium
        ])
    }
}

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(
        date: .now,
        quote: QuoteStore.todaysQuote()
    )
}
```

**このアプリは何をするものか：**

このアプリは、毎日1つの名言を表示するアプリである。メインアプリでは、今日の名言と、登録されているすべての名言を一覧で確認できる。ホーム画面のウィジェットでは、アプリを開かなくても今日の名言を見ることができる。

表示する名言は、現在の日付を使ってQuoteStoreの配列から選ばれる。そのため、日付が変わると別の名言が表示される。ウィジェットは小サイズと中サイズの2種類に対応している。

## コードの詳細解説

### TimelineProviderの仕組み

```swift
struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(
                id: 0,
                text: "読み込み中...",
                author: ""
            )
        )
    }

    func getSnapshot(
        in context: Context,
        completion: @escaping (QuoteEntry) -> Void
    ) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )

        completion(entry)
    }

    func getTimeline(
        in context: Context,
        completion: @escaping (Timeline<QuoteEntry>) -> Void
    ) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()

        let entry = QuoteEntry(
            date: currentDate,
            quote: quote
        )

        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(
                byAdding: .day,
                value: 1,
                to: currentDate
            )!
        )

        let timeline = Timeline(
            entries: [entry],
            policy: .after(tomorrow)
        )

        completion(timeline)
    }
}
```

**何をしているか：**

`TimelineProvider`は、ウィジェットに表示するデータと、ウィジェットを更新するタイミングをシステムに渡す役割を持つ。今回の`QuoteProvider`には、`placeholder`、`getSnapshot`、`getTimeline`の3つのメソッドがある。

`placeholder`は、ウィジェットのデータを読み込んでいる途中などに、一時的な表示を返す。今回のコードでは、「読み込み中...」という名言を表示する。

`getSnapshot`は、ウィジェットギャラリーやプレビューなどで、表示例が必要なときに使われる。ここでは、現在の日付に対応する名言を1つ取得して返している。

`getTimeline`は、実際にホーム画面へ表示するデータと、次に更新する時刻を設定する。今回のコードでは、現在の名言を表示し、次の日の0時以降にウィジェットを更新するようにしている。

**なぜこう書くのか：**

ウィジェットは通常のアプリのように、常に起動して毎秒画面を更新することができない。そのため、WidgetKitに対して、現在表示する内容と次に更新するタイミングをあらかじめ渡す必要がある。

また、読み込み中、プレビュー、実際の表示では、それぞれ必要なデータや処理時間が異なる。そのため、`placeholder`、`getSnapshot`、`getTimeline`の3つに処理が分けられている。

**もしこう書かなかったら：**

`getTimeline`を正しく実装しなかった場合、ウィジェットに表示するデータや更新時刻をシステムへ渡せなくなる。その結果、ウィジェットが正常に表示されなかったり、日付が変わっても名言が更新されなかったりする可能性がある。

また、`completion`を呼び出さなければ、作成したエントリーやタイムラインがシステムへ返されないため、ウィジェットの処理が完了しない。

---

### TimelineEntryとウィジェットビュー

```swift
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}
```

```swift
struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry

    @Environment(\.widgetFamily)
    var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget

        case .systemMedium:
            mediumWidget

        default:
            mediumWidget
        }
    }
}
```

**何をしているか：**

`QuoteEntry`は、ある時刻にウィジェットへ表示するデータをまとめる構造体である。`date`には表示する時刻、`quote`には表示する名言が入る。

`QuoteWidgetEntryView`は、TimelineProviderから渡された`entry`を使って、実際のウィジェット画面を作る。`entry.quote.text`を使うと名言の本文を表示でき、`entry.quote.author`を使うと作者名を表示できる。

**なぜこう書くのか：**

ウィジェットでは、表示するデータと画面のレイアウトを分けて管理する必要がある。`TimelineEntry`にデータをまとめることで、どの時刻にどの内容を表示するかが分かりやすくなる。

また、ビュー側では、渡された`entry`を表示することだけに集中できる。そのため、データ取得の処理と画面表示の処理を分離でき、コードが読みやすくなる。

**もしこう書かなかったら：**

`TimelineEntry`に表示用データを入れなければ、ウィジェットビューはどの名言を表示すればよいか分からなくなる。

ビューの中で直接日付計算やデータ取得を行うこともできるが、処理の役割が混ざり、コードが複雑になる。また、WidgetKitが管理するタイムラインの仕組みも利用しにくくなる。

---

### ウィジェットサイズごとのレイアウト

```swift
@Environment(\.widgetFamily)
var family

var body: some View {
    switch family {
    case .systemSmall:
        smallWidget

    case .systemMedium:
        mediumWidget

    default:
        mediumWidget
    }
}
```

```swift
var smallWidget: some View {
    VStack(spacing: 4) {
        Image(systemName: "quote.opening")
            .font(.caption)
            .foregroundStyle(.blue)

        Text(entry.quote.text)
            .font(.caption)
            .bold()
            .multilineTextAlignment(.center)
            .lineLimit(3)

        Text(entry.quote.author)
            .font(.caption2)
            .foregroundStyle(.secondary)
    }
    .padding(12)
}
```

```swift
var mediumWidget: some View {
    HStack(spacing: 16) {
        Image(systemName: "quote.opening")
            .font(.title)
            .foregroundStyle(.blue)

        VStack(alignment: .leading, spacing: 4) {
            Text("今日の名言")
                .font(.caption2)
                .foregroundStyle(.secondary)

            Text(entry.quote.text)
                .font(.subheadline)
                .bold()

            Text("— \(entry.quote.author)")
                .font(.caption)
                .foregroundStyle(.secondary)
        }

        Spacer()
    }
    .padding()
}
```

**何をしているか：**

`@Environment(\.widgetFamily)`を使って、現在表示されているウィジェットのサイズを取得している。

小サイズの場合は`smallWidget`を表示し、中サイズの場合は`mediumWidget`を表示する。小サイズでは縦方向の`VStack`を使用し、中サイズでは横方向の`HStack`を使用している。

**なぜこう書くのか：**

ウィジェットはサイズによって表示できる領域が大きく異なる。小サイズに中サイズと同じレイアウトを使うと、文字が途中で切れたり、情報が画面内に収まらなかったりする。

そのため、`widgetFamily`を確認して、サイズごとに適したレイアウトを用意する必要がある。小サイズでは情報量を少なくして中央に表示し、中サイズではアイコンと名言を横に並べることで、空間を有効に使っている。

**もしこう書かなかったら：**

すべてのサイズで同じレイアウトを使用すると、小さいウィジェットでは文字が読みづらくなる可能性がある。

また、`lineLimit(3)`を設定しなければ、長い名言が多くの行を使用して、作者名が表示されなくなる場合がある。

---

### メインアプリとの連携

```swift
struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]
}
```

**何をしているか：**

`Quote`と`QuoteStore`を共有ファイルに移動し、Target MembershipでメインアプリとWidget Extensionの両方に所属させている。

これにより、メインアプリ側の`ContentView`と、ウィジェット側の`QuoteProvider`の両方から、同じ`Quote`と`QuoteStore`の定義を利用できる。

**なぜこう書くのか：**

メインアプリとウィジェットは、Xcode上では別のTargetとして動作する。そのため、メインアプリのファイルにだけ書かれた構造体は、Widget Extensionから直接利用できない。

共有したいコードを両方のTarget Membershipに追加すると、同じSwiftファイルがそれぞれのTargetでコンパイルされる。これにより、同じコードを2つのファイルへ重複して書かなくてもよくなる。

**もしこう書かなかったら：**

`QuoteStore.swift`をWidget ExtensionのTarget Membershipに追加しなければ、ウィジェット側で次のようなエラーが発生する。

```text
Cannot find type 'Quote' in scope
```

また、`QuoteStore`だけを移動して`Quote`構造体を移動しなかった場合も、ウィジェット側は`Quote`型を認識できない。`Quote`と`QuoteStore`は両方を同じ共有ファイルに入れる必要がある。

---

### 今日の名言を選ぶ仕組み

```swift
static func todaysQuote() -> Quote {
    let dayOfYear = Calendar.current.ordinality(
        of: .day,
        in: .year,
        for: Date()
    ) ?? 0

    let index = dayOfYear % quotes.count
    return quotes[index]
}
```

**何をしているか：**

`Calendar.current.ordinality`を使って、今日が1年の中で何日目かを取得している。

その数を名言の数で割った余りを配列のインデックスとして使い、今日表示する名言を決めている。

**なぜこう書くのか：**

日付を使って名言を選ぶことで、同じ日には同じ名言が表示され、次の日になると別の名言へ切り替わる。

また、余りを求める`%`を使うことで、日数が名言の数より大きくなっても、配列の範囲内のインデックスを作ることができる。

**もしこう書かなかったら：**

`dayOfYear`をそのまま配列のインデックスに使うと、配列の要素数を超えてしまい、実行時エラーになる。

また、毎回ランダムに名言を選ぶ方法では、同じ日にアプリとウィジェットで異なる名言が表示される可能性がある。日付を使えば、同じ日は同じ結果になる。

---

### Widgetの定義

```swift
@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(
            kind: kind,
            provider: QuoteProvider()
        ) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(
                    .fill.tertiary,
                    for: .widget
                )
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([
            .systemSmall,
            .systemMedium
        ])
    }
}
```

**何をしているか：**

`QuoteWidget`は、ウィジェット全体の設定を定義している。

`StaticConfiguration`では、ウィジェットを識別する`kind`、データを提供する`QuoteProvider`、表示に使う`QuoteWidgetEntryView`を指定している。

`configurationDisplayName`はウィジェットの名前、`description`はウィジェット選択画面に表示される説明である。`supportedFamilies`では、使用できるサイズを小サイズと中サイズに限定している。

**なぜこう書くのか：**

WidgetKitでは、どのProviderを使い、どのビューを表示するかをWidgetConfigurationとしてまとめる必要がある。

また、`@main`を付けることで、この構造体がWidget Extensionの開始地点になる。

**もしこう書かなかったら：**

Widget Extension内に`@main`が存在しなければ、ウィジェットのエントリーポイントがなくなるため、ビルドできない。

反対に、同じTarget内に`@main`が2つ存在すると、次のエラーが発生する。

```text
'main' attribute can only apply to one type in a module
```

そのため、今回の構成では`QuoteWidget.swift`だけに`@main`を残し、`QuoteWidgetBundle.swift`などの不要なファイルは削除する必要がある。

## セットアップでつまずいた点

今回のセットアップでは、最初にWidget Extensionを追加した後、多数のエラーが発生した。

最初に発生した問題は、共有ファイルのTarget Membershipが正しく設定されていなかったことである。Widget Extension側から`Quote`型を参照できず、次のエラーが表示された。

```text
Cannot find type 'Quote' in scope
```

この問題は、`Quote`と`QuoteStore`を同じ共有ファイルに入れ、メインアプリとWidget Extensionの両方のTarget Membershipを有効にすることで解決した。

次に、同じ名前のWidget Extensionが重複して作成されていたため、次のエラーが発生した。

```text
Multiple commands produce QuoteWidgetExtension.appex
```

これは、複数のTargetが同じ出力ファイルを作成しようとしていたことが原因だった。重複していたWidget Extensionを削除し、1つだけ残すことで解決した。

さらに、メインアプリの`_8App.swift`がWidget Extensionにも含まれていたため、Widget Extension内に`@main`が2つ存在する状態になった。

```text
'main' attribute can only apply to one type in a module
```

この問題は、Target Membershipを確認して、`_8App.swift`をメインアプリのTargetだけに所属させることで解決した。

最終的なTarget Membershipは次のように設定した。

| ファイル | メインアプリ | Widget Extension |
|---|---|---|
| `_8App.swift` | 対象 | 対象外 |
| `ContentView.swift` | 対象 | 対象外 |
| `QuoteStore.swift` | 対象 | 対象 |
| `QuoteWidget.swift` | 対象外 | 対象 |

今回の作業から、ファイルが左側のどのフォルダに表示されているかだけではなく、Target Membershipを確認することが重要だと分かった。

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `TimelineProvider` | ウィジェットに表示するデータと更新タイミングを提供するプロトコル | `struct QuoteProvider: TimelineProvider { ... }` |
| `TimelineEntry` | 特定の時刻に表示するデータをまとめるためのプロトコル | `struct QuoteEntry: TimelineEntry { ... }` |
| `Timeline` | ウィジェットの表示内容と更新予定をまとめる | `Timeline(entries: [entry], policy: .after(tomorrow))` |
| `placeholder` | ウィジェットの読み込み中などに仮の表示を返す | `func placeholder(in context: Context) -> QuoteEntry` |
| `getSnapshot` | ウィジェットギャラリーやプレビュー用のデータを返す | `func getSnapshot(in:completion:)` |
| `getTimeline` | 実際の表示データと次回更新時刻を設定する | `func getTimeline(in:completion:)` |
| `Widget` | ウィジェット全体の設定を定義するプロトコル | `struct QuoteWidget: Widget` |
| `StaticConfiguration` | ユーザーによる設定項目を持たないウィジェットを定義する | `StaticConfiguration(kind:provider:content:)` |
| `@main` | アプリやウィジェットのエントリーポイントを指定する | `@main struct QuoteWidget: Widget` |
| `@Environment(\.widgetFamily)` | 現在のウィジェットサイズを取得する | `@Environment(\.widgetFamily) var family` |
| `WidgetFamily` | ウィジェットのサイズを表す | `.systemSmall`、`.systemMedium` |
| `supportedFamilies` | 使用できるウィジェットサイズを指定する | `.supportedFamilies([.systemSmall, .systemMedium])` |
| `Calendar.current` | 現在のカレンダー情報を取得する | `Calendar.current.ordinality(...)` |
| `%` | 割り算の余りを求める | `dayOfYear % quotes.count` |
| Target Membership | 1つのSwiftファイルを複数のTargetで使用できるようにする | `QuoteStore.swift`を両方のTargetに追加 |

## 自分の実験メモ

**実験1：Target Membershipを変更する**

- やったこと：`QuoteStore.swift`をメインアプリのTargetだけに所属させた。
- 結果：Widget Extension側で`Quote`と`QuoteStore`を認識できず、`Cannot find type 'Quote' in scope`というエラーが発生した。
- わかったこと：メインアプリとウィジェットの両方で使う型やデータは、両方のTarget Membershipに追加する必要がある。

**実験2：メインアプリのファイルをWidget Extensionにも追加する**

- やったこと：`_8App.swift`がメインアプリとWidget Extensionの両方に所属した状態でビルドした。
- 結果：Widget Extension内に`@main`が2つ存在する状態になり、`'main' attribute can only apply to one type in a module`というエラーが発生した。
- わかったこと：メインアプリとWidget Extensionは別のTargetであり、それぞれに1つだけエントリーポイントが必要である。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：`placeholder`、`getSnapshot`、`getTimeline`は、それぞれどのような場面で使われるのか。**  
   **得られた理解：**  
   `placeholder`は読み込み中の仮表示、`getSnapshot`はウィジェットギャラリーやプレビュー用、`getTimeline`は実際のホーム画面表示と更新スケジュールを作るために使われる。それぞれ用途が異なるため、3つのメソッドに分けられている。

2. **質問：ウィジェットは、なぜ普通のアプリのように毎秒画面を更新できないのか。**  
   **得られた理解：**  
   ウィジェットが常に動き続けると、バッテリーやCPUを多く消費する。そのため、WidgetKitではTimelineを使い、表示内容と次の更新時刻をあらかじめシステムへ渡す仕組みになっている。

3. **質問：Target Membership方式とApp Group方式には、どのような違いがあるのか。**  
   **得られた理解：**  
   Target Membershipは、同じSwiftファイルのコードを複数のTargetで使用するための仕組みである。一方、App Groupは、メインアプリとウィジェットがUserDefaultsやファイルなどの保存データを共有するために使う。今回のプログラムは固定された名言データを使うため、Target Membershipだけでも動作する。

## この章のまとめ　発表メモ：

この章では、WidgetKitを使って、ホーム画面に今日の名言を表示するウィジェットを作成した。

ウィジェットは通常のアプリとは異なり、常に動き続けるのではなく、TimelineProviderが表示内容と更新時刻をシステムへ渡す。`placeholder`、`getSnapshot`、`getTimeline`にはそれぞれ異なる役割があり、実際の更新処理は`getTimeline`で設定する。

また、TimelineEntryは、特定の時刻に表示するデータをまとめる役割を持つ。ウィジェットビューは、TimelineEntryから渡されたデータを使って画面を表示する。

メインアプリとウィジェットは別々のTargetであるため、共有する`Quote`と`QuoteStore`は両方のTarget Membershipに追加する必要がある。一方、メインアプリの`_8App.swift`やウィジェットの`QuoteWidget.swift`は、それぞれ対応するTargetだけに所属させる必要がある。

今回の作業では、コードの内容だけでなく、Widget Extensionの追加、不要なファイルの削除、Target Membership、`@main`の重複など、プロジェクト設定も重要であることを学んだ。エラーが大量に表示されても、最初の原因はTargetの重複やファイルの所属設定など、少数の問題である場合が多い。今後Widgetを作成するときは、コードだけでなくTargetの構成も確認したい。

# 発表メモ：

## ① 开始（约30秒）

【操作】

打开 08-widgets.md，不用往下滚，就停在最上面的标题。

【话す】

みなさん、こんにちは。私は第8章「ウィジェット」について発表します。この章を選んだ理由は、この章が一番時間をかけて学習した章であり、一番理解が深まった章だからです。今回作成したアプリは、毎日一つの名言を表示するアプリです。また、ホーム画面のWidgetにも今日の名言を表示できるようにしました。今日は、TimelineProvider、AIへ質問して理解した内容、この二つを中心に説明します。

---

## ② アプリの説明（约30秒）

【操作】

慢慢往下滚，找到「このアプリは何をするものか」，让这一整段正文都显示出来，不用指代码。

【話す】

まず、このアプリについて説明します。このアプリは、毎日一つの名言を表示するアプリです。メインアプリでは、今日の名言と登録されているすべての名言を一覧で表示します。また、ホーム画面では、Widgetを使って、アプリを開かなくても今日の名言を見ることができます。表示する名言は現在の日付から計算しているため、日付が変わると表示される名言も変わります。

---

## ③ TimelineProvider（约1分40秒）

【操作】

继续往下滚，找到「TimelineProviderの仕組み」。让整个代码块尽量显示完整，然后把鼠标放在 `struct QuoteProvider: TimelineProvider` 这一行。

【話す】

次に、TimelineProviderについて説明します。TimelineProviderは、Widgetへ表示するデータと、いつ更新するかという時刻をシステムへ渡す役割があります。このコードには、placeholder、getSnapshot、getTimelineという三つのメソッドがあります。最初は、この三つとも同じような処理だと思っていました。しかし、AIへ質問したことで、それぞれ役割が違うことを理解しました。

【操作】

鼠标往下移动到 placeholder，不用滚页面。

【話す】

まず、placeholderです。placeholderは、データがまだ読み込まれていないときに表示する仮のデータです。今回のプログラムでは、「読み込み中...」という文字を表示しています。

【操作】

鼠标移动到 getSnapshot。

【話す】

次はgetSnapshotです。これはWidgetギャラリーやプレビューで表示するデータを返します。ここでは、今日の名言を取得して表示しています。

【操作】

鼠标移动到 getTimeline，然后把鼠标停在 Timeline(...) 那几行。

【話す】

最後はgetTimelineです。このメソッドが一番重要です。ここでは今日表示する名言を作成し、次に更新する時刻も設定しています。今回のプログラムでは、次の日の0時になったら新しい名言へ更新するようにしています。そのため、毎日自動で違う名言が表示されます。

## ④ Target Membership（约1分钟20秒）

【操作】

切回 08-widgets.md，往下滚到「メインアプリとの連携」。让 Quote 和 QuoteStore 那段代码显示出来，然后再往下滚一点，让「Target Membership」那一段说明和表格都能看到。

【話す】

次に、メインアプリとWidgetの連携について説明します。今回、メインアプリとWidgetの両方で同じデータを使うために、QuoteとQuoteStoreを共有しました。最初は、同じプロジェクトなので自動的に使えると思っていました。しかし、メインアプリとWidget Extensionは別々のTargetとしてコンパイルされるため、そのままでは共有できません。そこで、QuoteStore.swiftをメインアプリとWidget Extensionの両方のTarget Membershipへ追加しました。これによって、同じコードを両方から利用できるようになりました。
実際にTarget Membershipを変更して実験したところ、Widget側で「Cannot find type 'Quote' in scope」というエラーが発生しました。この実験を通して、共有するファイルは両方のTargetに追加する必要があることを理解しました。

---

## ⑤ 今日の名言を選ぶ仕組み（约40秒）

【操作】

继续往下滚，找到「今日の名言を選ぶ仕組み」。把 todaysQuote() 那段代码完整显示出来，把鼠标放在 `%` 这一行。

【話す】

次に、今日の名言をどのように選んでいるかを説明します。このプログラムでは、現在の日付から今年の何日目かを取得しています。そして、その数を名言の数で割った余りを配列の番号として使っています。そのため、同じ日には同じ名言が表示され、日付が変わると別の名言へ切り替わります。この方法なら、アプリとWidgetでも同じ日に同じ名言が表示されます。

---

## ⑥ まとめ（约40秒）


最後にまとめです。

この章では、WidgetKitを使ってホーム画面に今日の名言を表示するWidgetを作成しました。また、TimelineProviderの役割や、WidgetはTimelineによって更新される仕組みについて学びました。さらに、Target Membershipを使ってメインアプリとWidgetでコードを共有する方法や、Targetの設定が重要であることも理解しました。今回の章では、コードだけではなく、Widget Extensionの設定やTargetの構成も重要であることを学ぶことができました。
以上で発表を終わります。ありがとうございました。
