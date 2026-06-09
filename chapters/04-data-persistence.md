# 第4章：データの永続化

> 執筆者：アウンミャッ
> 最終更新：2026-6-9

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、AppStorageとSwiftDataを使ってアプリのデータを端末に永続的に保存する方法を学ぶ。具体的にはSwiftDataを使ったメモアプリを題材として、@Modelでデータモデルを定義し、modelContextを使ったデータ操作、@Queryによる動的なデータ取得、そして@AppStorageによるユーザー設定の保存を実装する。
この章では、SwiftDataとAppStorageを使ってデータをアプリ内に保存する方法を学んだ。SwiftDataではメモのタイトルや内容を保存し、AppStorageではユーザー名や表示設定などの簡単な設定を保存できることを学んだ。また、データの追加・削除・取得の方法についても理解した。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

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

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）
このアプリはメモを作成、編集、削除できるメモ帳アプリである。作成したメモはSwiftDataによって端末内に保存されるため、アプリを閉じてもデータが消えない。また、ユーザー名やお気に入り順表示の設定はAppStorageによって保存される。
## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
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
```

**何をしているか：**
（この部分が果たしている役割を説明する）
Memoというデータモデルを作成している。メモのタイトル、内容、作成日時、お気に入り状態を保存する。

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）
@Modelを付けることでSwiftDataがこのクラスをデータベースのテーブルとして扱い、自動的に保存できるようになるため。

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）
SwiftDataがデータモデルとして認識できず、メモを保存できなくなる。
---

### データの追加・削除（modelContext）

```swift
modelContext.insert(memo)
modelContext.delete(memo)
```

**何をしているか：**
insertは新しいメモを保存し、deleteは既存のメモを削除する。

**なぜこう書くのか：**
SwiftDataではmodelContextを通してデータの追加や削除を行うため。

**もしこう書かなかったら：**
画面上では表示されても、実際には保存されないためアプリを再起動すると消えてしまう。
---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse)
private var memos: [Memo]
```

**何をしているか：**
保存されている全てのメモを取得し、新しい順に並べている。

**なぜこう書くのか：**
データが追加・削除・編集されたときに自動で画面を更新できるため。

**もしこう書かなかったら：**
自分で毎回データを読み直す処理を書く必要があり、コードが複雑になる。
---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false @AppStorage("userName") private var userName: String = ""
```

**何をしているか：**
ユーザー名とお気に入り表示設定を保存している。

**なぜこう書くのか：**
簡単な設定値を永続化するのに最も簡単な方法だから。

**もしこう書かなかったら：**
アプリを終了するたびに設定が初期化される。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model final class Memo { ... }` |
| 例：`@Query` | データベースからデータを取得し、変更を自動で反映するプロパティラッパー | `@Query var memos: [Memo]` |
|@AppStorage | 設定値を保存する| @AppStorage("userName")|
| @Bindable|データを双方向バインドする |@Bindable var memo |
| modelContext|データの追加・削除を行う | modelContext.insert(memo)|

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
2. @Modelとは何か？
   **得られた理解：**
   SwiftDataに保存するためのデータモデルを定義するマクロである。
4. **質問：**
5. @Queryはなぜ必要なのか？
   **得られた理解：**
   データの変更を自動的に検知し、画面を更新してくれるため。

7. **質問：**
8. AppStorageとSwiftDataの違いは何か？
   **得られた理解：**
   AppStorageは設定値の保存、SwiftDataは複雑なデータの保存に使う。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
この章ではSwiftDataとAppStorageを使ったデータの永続化について学んだ。SwiftDataはメモなどの構造化されたデータを保存するために使い、AppStorageはユーザー設定などの簡単なデータを保存するために使う。永続化を利用することで、アプリを終了してもデータを保持できることを理解した。
