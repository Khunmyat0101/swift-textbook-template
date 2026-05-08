# 第2章：地図アプリの基本

> 執筆者：（カリム）
> 最終更新：8/05/2026

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、MapKitを使ってアプリ内に地図を表示し、特定の位置にマーカーを配置する方法を学ぶ。具体的にはランドマークデータを構造体で定義し、地図上にマーカーを表示して、カテゴリでフィルターするアプリを題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る
import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
// 該当部分のコードを抜粋して貼る
struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"
    }
}
```
何をしているか：
地図に表示する場所の情報を1つの構造体にまとめている。
名前、説明、座標、カテゴリを持たせることで、1つのランドマークをわかりやすく管理できる。

なぜこう書くのか：
地図に出す情報を1つの型にまとめると、データの追加や表示がしやすい。
Identifiable を付けることで、For​Each で一覧表示するときに使いやすくなる。

もしこう書かなかったら：
場所ごとの情報がバラバラになり、管理しにくくなる。
For​Each で地図に表示するときにエラーになったり、コードが読みにくくなったりする。

---

### 地図の表示とカメラ制御

```swift
// 該当部分のコードを抜粋して貼る
@State private var cameraPosition: MapCameraPosition = .region(
    MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
        span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
    )
)

Map(position: $cameraPosition) {
    ForEach(filteredLandmarks) { landmark in
        Marker(
            landmark.name,
            systemImage: landmark.category.iconName,
            coordinate: landmark.coordinate
        )
        .tint(landmark.category.color)
    }
}
```
何をしているか：
地図の最初の表示位置を東京駅付近に設定している。
また、Map(position:) を使って地図の表示範囲を状態として管理している。

なぜこう書くのか：
@​State を使うことで、地図の位置や範囲をアプリの状態として持てる。
最初から東京の見やすい範囲を表示できるので、アプリを開いたときに分かりやすい。

もしこう書かなかったら：
地図がどこを表示するか分かりにくくなる。
初期位置が適切でないと、ユーザーが目的の場所を見つけにくくなる。

---

### マーカーの表示

```swift
// 該当部分のコードを抜粋して貼る
ForEach(filteredLandmarks) { landmark in
    Marker(
        landmark.name,
        systemImage: landmark.category.iconName,
        coordinate: landmark.coordinate
    )
    .tint(landmark.category.color)
}
```

何をしているか：
絞り込み後のランドマークを1つずつ地図に表示している。
カテゴリごとにアイコンと色を変えて、見分けやすくしている。

なぜこう書くのか：
For​Each を使うと、配列のデータをまとめて表示できる。
Marker を使うと、地図上に簡単に目印を置ける。

もしこう書かなかったら：
地図には場所が表示されず、ただの地図になる。
カテゴリごとの違いも分からず、使いにくいアプリになる。

---

### フィルター機能

```swift
// 該当部分のコードを抜粋して貼る
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}
ForEach(Landmark.Category.allCases, id: \.self) { category in
    Button {
        if selectedCategories.contains(category) {
            selectedCategories.remove(category)
        } else {
            selectedCategories.insert(category)
        }
    } label: {
        HStack(spacing: 4) {
            Image(systemName: category.iconName)
            Text(category.rawValue)
        }
    }
}
```

何をしているか：
どのカテゴリを表示するかを Set で管理している。
ボタンを押すと、そのカテゴリを表示したり非表示にしたりできる。

なぜこう書くのか：
Set は重複しないデータを管理しやすく、選択状態の切り替えに向いている。
all​Cases を使うと、カテゴリが増えてもボタンを自動で作れる。

もしこう書かなかったら：
カテゴリの絞り込みができず、すべての場所が常に表示される。
場所が増えたときに見づらくなり、目的の場所を探しにくくなる。

### ランドマーク詳細カード

```swift
struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}
```
何をしているか：
選択したランドマークの名前や説明をカードの形で表示するビューである。
見た目を整えて、情報を読みやすくしている。

なぜこう書くのか：
詳細表示を別ビューに分けると、Content​View が見やすくなる。
再利用もしやすく、コードの役割がはっきりする。

もしこう書かなかったら：
Content​View に全部の表示処理を書くことになり、コードが長くなる。
詳細情報も見づらくなり、画面の整理がしにくい。

---

（必要に応じてセクションを増やす）

新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| Map | SwiftUIで地図を表示するビュー | Map(position: $camera​Position) |
| Marker | 地図上にマーカーを表示する | Marker("浅草寺", coordinate: coordinate) |
| @​State | 画面の状態を保存し、変更時に再描画する | @​State private var selected​Landmark: ​Landmark? |
| For​Each | 配列のデータを繰り返して表示する | For​Each(filtered​Landmarks) { landmark in ... } |
| Set | 重複しない値をまとめて管理する | Set(​Landmark​.​Category​.all​Cases) |
| Case​Iterable | enum の全ケースを取り出せるようにする | Landmark​.​Category​.all​Cases |
| Identifiable | 各データを一意に識別できるようにする | struct ​Landmark: ​Identifiable |
| MKCoordinate​Region | 地図の中心座標と表示範囲を表す | MKCoordinate​Region(center: ..., span: ...) |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

実験1：
• やったこと：latitude​Delta と longitude​Delta の値を小さくしてみた。
• 結果：地図がより拡大されて、狭い範囲が表示された。
• わかったこと：span の値を変えると、地図のズームレベルを調整できる。

実験2：
• やったこと：selected​Categories の初期値を Set([.temple]) に変えてみた。
• 結果：最初は寺社だけが地図に表示された。
• わかったこと：初期状態を変えることで、最初に見せる情報を調整できる。

AIに聞いて特に理解が深まった質問 TOP3

1. 質問：@State はなぜ必要なのか？
得られた理解： 値が変わったときに画面を自動で更新するために必要だと分かった。

2. 質問：なぜカテゴリの選択に Set を使うのか？
得られた理解： 重複を防ぎながら、追加・削除を簡単にできるので、選択状態の管理に向いていると分かった。

3. 質問：Identifiable は何のために付けるのか？
得られた理解： For​Each が各データを区別して正しく表示するために必要だと分かった。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
