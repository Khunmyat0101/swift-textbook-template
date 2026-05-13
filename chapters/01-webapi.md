# 第1章：WebAPIの基本

> 執筆者：アウンミャッ
> 最終更新：2026年4月18日

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、インターネット上のサービス（API）からデータを取得して、アプリ内に表示する方法を学ぶ。具体的にはiTunes Search APIを使って音楽を検索し、その結果をリスト表示するアプリを題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

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

ユーザーが入力したキーワードを元にiTunesから音楽を検索しその結果を画像付きリストで表示するアプリです。

## コードの詳細解説

### データモデル（Codable構造体）

```swift
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

```

**何をしているか：**
APIから返ってくるJSONデータをSwiftで扱えるようにしています。Codableを使うことで複数なJSON データーをencode,
decodeし、簡単に管理できるようにしています。

**なぜこう書くのか：**

APIから届くデータの名前（trackNameなど）と、Swiftの変数の名前を合わせることで、プログラムが自動的に中身を読み取ってくれるからです。また、Identifiableをつけることで、リスト（List）で表示するときに、どの曲がどれかをSwiftが正確に区別できるようになります。


**もしこう書かなかったら：**

ータが読み込めない： 名前の書き方を間違えると、APIからデータを受け取ることができず、リストが空っぽのままになってしまいます。

リストが動かない： Identifiableがないと、Swiftが「どのデータが新しく増えたか」を判断できず、画面に表示する際にエラーになったり、動きが遅くなったりします。
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

インターネット（iTunes）に「この曲を探して！」とお願いして、返ってきたデータをアプリで使える形にする処理を書いています。


**なぜこう書くのか：**

インターネットとの通信には時間がかかるため、async や await を使います。これを使うことで、「データの準備ができるまで待つ」という命令になり、アプリがフリーズするのを防ぐことができます。


**もしこう書かなかったら：**

画面が固まる： データをダウンロードしている間、ボタンが押せなくなったり、画面が真っ白なまま動かなくなったりします。

データが壊れても気づけない： do-catch（エラー処理）を書かないと、ネットが繋がっていない時にアプリが突然終了（クラッシュ）してしまいます。


---

### ビューの構成

```swift
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
```

**何をしているか：**

リストの「1行分」のデザインです。左側に画像、右側に曲名と歌手名を並べるように指示しています。


**なぜこう書くのか：**

HStack（横並び）と VStack（縦並び）を組み合わせることで、複雑なレイアウトもシンプルに作れるからです。また、AsyncImage を使うことで、ネット上の画像を自動で読み込んで表示してくれます。


**もしこう書かなかったら：**

見た目が崩れる： すべての文字や画像が重なってしまったり、バラバラな場所に表示されたりして、アプリらしく見えません。

画像が表示されない： ネットから画像をダウンロードして表示するコードを何十行も自分で書かなければならず、とても大変になります。


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
- やったこと：
- urlString 内の limit=25 を limit=50 に書き換えた。
- 結果：
- 一度に表示される検索結果の数が25件から50件に増えた。
- わかったこと：
- APIのリクエストURLに含まれるパラメータを変更することで、取得するデータの量を調整できる。

**実験2：**
- やったこと：
- SongRow 内の .clipShape(RoundedRectangle(cornerRadius: 8)) を .clipShape(Circle()) に変更した。
- 結果：
- 四角いアルバムアートが丸い形（アイコン風）に変わった。
- わかったこと：
- clipShape を使うことで、複雑なコードを書かなくても簡単にUIのデザインを変更できる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
2. searchText.addingPercentEncoding が必要なの？
   **得られた理解：**
   URLに日本語（全角文字）やスペースが含まれているとエラーになるため、インターネットが理解できる形式（%形式）に変換する必要がある。

4. **質問：**
5. previewUrl に ? がついているのはなぜ？
   **得られた理解：**
   曲によっては試聴URLが存在しない場合があるため、nil（データがない状態）を許容する「Optional型」にしている。

7. **質問：**
8. isLoading はどうやって動いているの？
   **得られた理解：**
   通信開始時に true にして ProgressView を表示し、完了後に false に戻すことで、ユーザーに「今探しています」と伝える役割がある。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
この章では、インターネット上のデータ（API）をアプリに取り込む基礎を学びました。URLSession でデータを取得し、JSONDecoder で解析し、List で表示するという、現代のアプリ開発において最も重要な「データの流れ」を理解することができました。
