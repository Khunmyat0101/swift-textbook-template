# 第2章：地図アプリの基本

> 執筆者： アウンミャッ
> 最終更新：2026年５月20日

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

この章では、MapKitフレームワークを使用してSwiftUIアプリ内に地図を表示し、特定の観光スポット（ランドマーク）にマーカーを配置する方法を学びます。具体的には、カスタム構造体を用いたデータモデルの定義、地図上に表示するアイテムの動的なフィルタリング、そしてマーカーをタップした際につなぐ詳細カードの表示ロジックを習得します。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る
import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

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
            Map(position: $cameraPosition, selection: $selectedLandmark) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                    .tag(landmark)
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

東京の主要な観光スポット（浅草寺、東京タワーなど）をSwiftUIのMapコンポーネント上に視覚的に配置するアプリです。画面下部の「カテゴリフィルター」ボタンで「寺社」「タワー」「公園」をタップして表示・非表示を切り替えられるほか、地図上のマーカーをタップするとその場所の詳しい名前と説明文が書かれた「詳細カード」が下からポップアップする仕組みになっています。
## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
// 該当部分のコードを抜粋して貼る
struct Landmark: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    static func == (lhs: Landmark, rhs: Landmark) -> Bool {
        lhs.id == rhs.id
    }

    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"
        // (省略: iconName, color)
    }
}
```

**何をしているか：**
（この部分が果たしている役割を説明する）
地図上に表示したい観光スポットの情報を1つにまとめるLandmark構造体を定義しています。それぞれの場所を区別するid、名前、説明、位置情報（緯度・経度）、そして「寺社」などのカテゴリを持たせています。また、地図上で「どれがタップされたか」を判別できるようにするために、Identifiable と Hashable プロトコルに準拠させています。

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）
SwiftUIの Map 内で ForEach を使って複数のマーカーを並べたり、selection: $selectedLandmark を使ってタップされた要素を特定したりするには、各データが一意に識別できる（Identifiable である）必要があります。さらに、selection に指定する要素や、フィルターで使用する Set の要素にするためには Hashable でなければならないため、このようなプロトコル準拠の書き方をしています。

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）
Identifiable や Hashable に準拠させない場合、Map の ForEach でループ処理をする際にエラーになります。また、マーカーをタップしたときに「どの場所が選ばれたか」をシステムが検知できなくなるため、詳細カードを連動して表示することが不可能になります。

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

// (body内)
Map(position: $cameraPosition, selection: $selectedLandmark) {
    // マーカー処理
}
.mapStyle(.standard(elevation: .realistic))
```

**何をしているか：**
アプリが起動したときに、最初に地図のどこをどれくらいの拡大率で表示するか（初期カメラ位置）を決定しています。ここでは東京駅付近（緯度35.6812, 経度139.7671）を中心に、周辺の観光地が見渡せる広さ（0.08度の範囲）を指定し、Map コンポーネントに状態をバインドしています。

**なぜこう書くのか：**
@State を使って MapCameraPosition を管理することで、ユーザーが地図をドラッグしたりピンチイン・アウトして動かしたときに、プログラム側でもその状態の変更を追跡できるようになります。また、.mapStyle(.standard(elevation: .realistic)) を指定することで、建物などが立体的に表現されるリアルな標準地図スタイルに仕上げています。

**もしこう書かなかったら：**
カメラ位置（position）を固定の定数で渡したり指定しなかったりすると、ユーザーが地図を動かせなくなったり、アプリ起動時に全く関係のない世界地図の初期位置が表示されてしまい、東京の観光スポットを見つけてもらうことが難しくなります。
---

### マーカーの表示

```swift
// 該当部分のコードを抜粋して貼る
Map(position: $cameraPosition, selection: $selectedLandmark) {
    ForEach(filteredLandmarks) { landmark in
        Marker(
            landmark.name,
            systemImage: landmark.category.iconName,
            coordinate: landmark.coordinate
        )
        .tint(landmark.category.color)
        .tag(landmark)
    }
}
```

**何をしているか：**
フィルターを通過したランドマークの配列（filteredLandmarks）をループで回し、それぞれの位置（coordinate）に Marker を設置しています。マーカーにはカテゴリに応じたSF Symbolsのアイコン（systemImage）とカラー（tint）を設定し、さらに tag(landmark) を付与しています。

**なぜこう書くのか：**
Marker に .tag(landmark) をつけることで、Map(selection: $selectedLandmark) と紐連くようになります。ユーザーが地図上の特定のマーカーをタップした瞬間に、その landmark データが自動的に $selectedLandmark に代入され、詳細カードが表示されるトリガーになります。

**もしこう書かなかったら：**
.tag(landmark) を書き忘れると、いくら地図上のマーカーをタップしても selectedLandmark の中身が更新されないため、画面下部に詳細情報カードが浮き上がってくる機能が全く動かなくなってしまいます。
---

### フィルター機能

```swift
// 該当部分のコードを抜粋して貼る
    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }
Button {
    if selectedCategories.contains(category) {
        selectedCategories.remove(category)
    } else {
        selectedCategories.insert(category)
    }
} label: { ... }
```

**何をしているか：**
ユーザーが画面下部のカテゴリボタン（寺社・タワー・公園）をタップした際に、選択されたカテゴリの状態（Set 形式の selectedCategories）を更新し、そのカテゴリに一致するランドマークだけを配列からリアルタイムに絞り込んで（filter）地図に渡しています。

**なぜこう書くのか：**
SwiftUIは状態（@State や @Binding）が変更されると、ビューを自動的に再描画する特性を持っています。フィルターの選択状態を Set（集合）で管理し、その要素の有無に応じて計算プロパティ filteredLandmarks が動くようにすることで、複雑な描画更新の処理を手動で書くことなく、安全かつ高速に地図上のマーカーを同期できるからです。

**もしこう書かなかったら：**
ボタンがタップされたときに、毎回手動で配列を走査して地図のマーカーを消去・追加するコードを書く必要があり、コードが非常に複雑になります。また、状態の同期がズレて「ボタンは押されているのに地図上のマーカーが消えない」といったバグの原因になります。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: .constant(.region(region)))` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
|　MapCameraPosition | 地図の表示領域やカメラの追従状態を管理する構造体　|　@State private var cameraPosition: MapCameraPosition = .region(...) |
| Set　|重複した値を持たない、要素の順序がないコレクション（集合） | var selectedCategories: Set<Landmark.Category>|
| .mapStyle　|地図の見た目（標準、航空写真、3Dなど）を変更するモディファイア | .mapStyle(.standard(elevation: .realistic))|

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：`Map` のスタイルを `.standard` から `.hybrid(elevation: .realistic)` に変更してみた。
- 結果：道路や都市名などのラベルが維持されたまま、背景がリアルな航空写真（サテライトビュー）に切り替わった。
- わかったこと：アプリのテーマや目的に応じて、一行書き換えるだけで地図の見た目を簡単に切り替えられる。

**実験2：**
- やったこと：34.9858, 経度: 135.7588）に変更し、`span` の値を `0.01` に小さくした。
- 結果：アプリ起動時に、京都駅周辺の建物や道路がはっきりと見える詳細な地図が拡大表示されるようになった。
- わかったこと：`latitudeDelta` や `longitudeDelta` の数値を小さく指定するほど、地図がズームイン（拡大）されてピンポイントな場所を表示できることが理解できた。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
2. なぜ Landmark 構造体に Identifiable だけでなく Hashable も必要なのですか？
   **得られた理解：**
Map の selection（タップされたマーカーの検知）や、フィルター機能で使う Set の中に要素を入れるためには、要素同士が同一かどうかを高速に判定できる Hashable の仕組みが必須だから。

4. **質問：**
5. MKCoordinateRegion の span（latitudeDelta など）の数値を大きくするとどうなりますか？
6.  **得られた理解：**
数値を大きくするほどカメラが引き（広域表示）、小さくするほど寄り（詳細表示）になる。東京全体を見せたい時は 0.08 前後がちょうどいい。

8. **質問：**
9. 詳細カードの出現時にアニメーションを滑らかにするにはどうすればいいですか？
   **得られた理解：**
   if let landmark = selectedLandmark の周辺や、selectedLandmark にデータが入る処理を withAnimation で囲む、もしくは .animation() モディファイアを組み合わせると綺麗にポップアップする。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
OS 17以降の新しいMapKitを使用すると、SwiftUIの宣言的UIのなかに驚くほど簡単に地図を組み込めることが分かった。特に、地図上のピンの選択状態（selection）とSwiftUIの @State（selectedLandmark）をバインドさせることで、「マーカーをタップしたら詳細ビューを浮かび上がらせる」 という本格的なUIロジックが、UIKit時代よりも遥かに少ないコード量で実装できる。データモデルの定義段階で Identifiable や Hashable を意識することが、SwiftUIのUIコンポーネントと連動させる上での鍵となる。
