# 第4章：データの永続化

> 執筆者：（カリム）
> 最終更新：2026-05-22

## この章で学ぶこと

（この章では、SwiftData と AppStorage を使ってアプリのデータを端末に保存する方法を学ぶ。SwiftData ではメモのタイトル・内容・作成日・お気に入り状態を保存し、AppStorage ではユーザー名や表示設定を保存する。）

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る
//
//  ContentView.swift
//  DataPersistence
//
//  Created by cmStudent on 2026/05/22.
//

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
このアプリは、メモを作成・表示・編集・削除できるメモ帳アプリです。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
// 該当部分のコードを抜粋して貼る
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

何をしているか：
Memo というデータモデルを定義している。タイトル、本文、作成日、お気に入り状態を保存する。

なぜこう書くのか：
@​Model を付けることで SwiftData がこのクラスを保存対象として扱えるようになる。

もしこう書かなかったら：
SwiftData に保存できるモデルとして認識されず、@​Query や model​Context​.insert() で扱えない。

---

### データの追加・削除（modelContext）

```swift
// 該当部分のコードを抜粋して貼る
@Environment(\.modelContext) private var modelContext

let memo = Memo(title: title, content: content)
modelContext.insert(memo)

modelContext.delete(memo)
```
何をしているか：
model​Context を使って、新しいメモを保存したり、既存のメモを削除したりしている。

なぜこう書くのか：
SwiftData では、データベースへの追加や削除を model​Context を通して行うため。

もしこう書かなかったら：
画面上でメモを作っても永続化されず、一覧にも正しく反映されない。

---

### @Queryによるデータ取得

```swift
// 該当部分のコードを抜粋して貼る
@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
```

何をしているか：
保存されている Memo データを取得し、作成日の新しい順に並べている。

なぜこう書くのか：
@​Query を使うと、SwiftData の内容が変わったときに画面も自動で更新される。

もしこう書かなかったら：
保存済みのメモを自動で取得できず、追加や削除後に一覧を自分で更新する必要がある。

---

### @AppStorageによる設定保存

```swift
// 該当部分のコードを抜粋して貼る
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""
```

何をしているか：
ユーザー名とお気に入り表示設定を保存している。

なぜこう書くのか：
@​App​Storage は簡単な設定値を User​Defaults に保存でき、アプリを閉じても値が残るため。

もしこう書かなかったら：
アプリを再起動したときに、ユーザー名や表示設定が初期値に戻ってしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| @​Model | SwiftDataで保存するデータモデルを定義する | @​Model class ​Memo { ... } |
| @​Query | SwiftDataからデータを取得し、変更を自動反映する | @​Query var memos: [​Memo] |
| model​Context | データの追加・削除を行うための環境値 | model​Context​.insert(memo) |
| @​App​Storage | 簡単な設定値を保存する | @​App​Storage("user​Name") var user​Name |
| @​Bindable | SwiftDataモデルを編集画面で直接変更できるようにする | @​Bindable var memo: ​Memo |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

実験1：
• やったこと：設定画面でユーザー名を入力した。
• 結果：ナビゲーションタイトルが「〇〇のメモ帳」に変わった。
• わかったこと：@​App​Storage の値を変えると画面にもすぐ反映される。

実験2：
• やったこと：メモをお気に入りにして、「お気に入りを上に表示」をオンにした。
• 結果：お気に入りのメモが一覧の上に表示された。
• わかったこと：保存されたデータを並び替えて表示できる。

## AIに聞いて特に理解が深まった質問 TOP3

1. 質問： @​Model は何のために使うのか？
得られた理解： SwiftDataに保存するためのデータ型として登録するために使う。

2. 質問： @​Query と普通の配列の違いは何か？
得られた理解： @​Query は保存データの変更を自動で画面に反映してくれる。

3. 質問： @​App​Storage と SwiftData の違いは何か？
得られた理解： @​App​Storage は設定などの小さな値、SwiftData はメモのようなアプリの主要データに向いている。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
