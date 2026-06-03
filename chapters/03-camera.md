# 第3章：カメラの利用

> 執筆者：アウン　ミャッ
> 最終更新：2026-6-3

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）
この章では、アプリから「フォトライブラリの写真を選択する機能」と「カメラを起動して写真を撮影する機能」の2つを実装する方法を学びます。
具体的には、安全に写真を取り出す `PhotosPicker` の使い方、重いデータをスムーズに読み込む非同期処理（Swift Concurrency）、そしてSwiftUIにはないカメラ機能をUIKitから借りてきて連携させる手法（`UIViewControllerRepresentable` と `Coordinator` パターン）について解説します。
例：この章では、PhotosPickerでフォトライブラリから写真を選択し、UIImagePickerControllerでカメラ撮影した画像を扱う方法を学ぶ。具体的には非同期で画像データを読み込み、UIViewControllerRepresentableを使ってUIKitをSwiftUIに統合し、Coordinatorパターンを使ってカメラ機能と連携するアプリを題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
//
//  ContentView.swift
//  PhotoBasic
//
//  Created by cmStudent on 2026/05/13.
//

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

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）
起動すると、中央に「写真を選択または撮影してください」という案内が表示されます。下部にある「ライブラリ」ボタンを押すとアルバムから写真を選べ、「カメラ」ボタンを押すとスマホのカメラが起動してその場で写真を撮ることができます。選んだ（または撮影した）写真は、中央のグレーの枠内に綺麗に収まる形で表示されます。
## コードの詳細解説

### PhotosPickerによる写真選択

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}
```

**何をしているか：**
iOSの標準機能である写真選択用のポップアップ（PhotosPicker）を表示させています。ユーザーが写真（.images）を選択すると、その情報が変数 selectedItem に格納されます。そして、選択が完了した瞬間（.onChange）に、自動的に画像を読み込む処理（loadImage）を実行します。

**なぜこう書くのか：**
SwiftUIの PhotosPicker を使うと、アプリ側がユーザーのアルバム全体を覗き見することなく、「ユーザーが自分で選んだ写真だけ」を安全にアプリに引き渡すことができる仕組みになっているためです。プライバシーを守りつつ、最もシンプルなコードで写真選択を実装できます

**もしこう書かなかったら：**
昔ながらの古い写真選択画面（UIKitの UIImagePickerController など）を無理やりSwiftUIに組み込んで使う必要があります。その場合、ユーザーに対して「このアプリに写真へのアクセスを許可しますか？」という警告ポップアップをわざわざ表示させなければならず、許可をもらうための設定（Info.plist の記述）も増えて面倒になります。
---

### 画像の非同期読み込み

```swift
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
PhotosPicker から渡された写真データ（PhotosPickerItem）を、アプリで扱える「データのかたまり（Data）」として裏側で読み込み、それを UIImage から最終的にSwiftUIの Imageへと変換して画面にセットしています。

**なぜこう書くのか：**
最近のスマートフォンの写真は画質が良く、ファイルサイズが非常に大きいです。そのため、データを読み込むのに少し時間がかかります。ここで async や try await というキーワード（Swift Concurrency）を使うことで、「データの読み込みは裏側のスレッド（バックグラウンド）に任せて、表側の画面処理を止めない」 という賢い処理をしています。

**もしこう書かなかったら：**
もし裏側を使わず、メインの処理と同じ場所（メインスレッド）で巨大な写真を読み込もうとすると、写真を選んだ瞬間にアプリが一時的にフリーズし、ボタンを押しても反応しない「カクつき」が発生します。さらに最悪の場合、スマホのOSから「反応が遅いアプリ」と見なされ、アプリが強制終了（クラッシュ）してしまいます。
---

### UIViewControllerRepresentableによるカメラ連携

```swift
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
        // 撮影完了時やキャンセル時の動きを制御する処理
    }
}
```

**何をしているか：**
SwiftUIには、「標準のカメラ撮影画面」が用意されていません。そのため、一世代前の画面フレームワーク（UIKit）に用意されている UIImagePickerController（カメラ画面）をSwiftUIでも使えるように変換（ラップ）しています。
また、カメラ画面で「シャッターが押された」とか「キャンセルされた」という報告を受け取るために、メッセンジャー役（Coordinator）を用意して、撮影された写真をSwiftUI側の変数 capturedImage に届けています。

**なぜこう書くのか：**
SwiftUIとUIKitという異なるシステムの間で、画面を橋渡しするための公式ルールが UIViewControllerRepresentable プロトコルだからです。これを使うことで、既存の信頼性の高いiOS標準のカメラ画面をそのまま自分のアプリに埋め込むことができます。

**もしこう書かなかったら：**
SwiftUIだけでカメラ機能をゼロから作ろうとすると、レンズの制御、光の自動調節、画面へのプレビュー投影など、スマホのハードウェアに近い部分（AVFoundation という難解なライブラリ）のコードを何百行も自作しなければならなくなり、初心者向けアプリとしては実装不可能に近い難易度になってしまいます。
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

    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        parent.dismiss()
    }
}
```

**何をしているか：**
UIKitのカメラ画面（UIImagePickerController）からの報告をキャッチする「メッセンジャー」の役割を定義しています。
ユーザーがカメラで写真を撮影し終えたとき（didFinishPickingMediaWithInfo）に、撮影された生データからオリジナルの画像（.originalImage）を取り出してSwiftUI側の変数（capturedImage）に引き渡し、最後にカメラ画面を閉じます（dismiss）。また、キャンセルされたとき（imagePickerControllerDidCancel）も安全に画面を閉じる処理を行います。

**なぜこう書くのか：**
SwiftUIのビュー（構造体）は、UIKitのカメラ画面からの「写真が撮れたよ」「キャンセルされたよ」というリアルタイムの通知（デリゲートイベント）を直接受け取ることができない仕様だからです。
そのため、通知を受け取る専門のクラスとして Coordinator を内部に用意し、SwiftUIとUIKitの間を取り持たせる必要があります。これがiOS開発で標準的に使われる「Coordinatorパターン」の記述ルールです。

**もしこう書かなかったら：**
カメラ画面（UIImagePickerController）自体は起動してシャッターを切ることはできます。しかし、シャッターを押した後に「アプリ側の画面に写真を持ち帰る」という処理や、「撮影後にカメラ画面を自動で閉じる」という連動が一切行われなくなり、カメラ画面が開いたままフリーズしたような状態になってしまいます。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `PhotosPicker` | アプリに写真の全アクセス権を与えず、ユーザーが選んだ写真だけを安全に取り出すためのSwiftUI標準のボタン。 | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| `loadTransferable` | `PhotosPicker` で選んだ写真から、中身のデータ（`Data` 型など）を裏側で引き出すための非同期命令。 | `try await item.loadTransferable(type: Data.self)` |
| `async / await` | 画像の読み込みなど、時間がかかる処理を裏側（バックグラウンド）で実行し、画面がフリーズするのを防ぐ仕組み（非同期処理）。 | `func loadImage(...) async` / `await loadImage(...)` |
| `UIViewControllerRepresentable` | SwiftUIにはまだ用意されていない、一世代前の機能（UIKitのカメラ画面など）をSwiftUIで使えるように橋渡しする枠組み。 | `struct CameraView: UIViewControllerRepresentable` |
| `UIImagePickerController` | iOSが昔から用意している、カメラを起動したり写真を撮影したりするための標準の画面コンポーネント。 | `let picker = UIImagePickerController()` / `picker.sourceType = .camera` |
| `Coordinator` | UIKitのカメラ画面から「撮影が終わったよ」「キャンセルされたよ」という報告を受け取り、SwiftUI側にデータを届けるメッセンジャー役のクラス。 | `func makeCoordinator() -> Coordinator` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：パソコンのシミュレータで「カメラ」ボタンを押してみた**
- **やったこと：** 実機（iPhone）を繋がずに、Mac上のシミュレータでアプリを起動し、「カメラ」ボタンを押して画面を動かそうとした。
- **結果：** カメラ画面が起動せず、画面が真っ黒になるか、アプリが強制終了（クラッシュ）してしまった。
- **わかったこと：** シミュレータには本物のレンズ（物理カメラ）がついていないため、`UIImagePickerController` の `.camera` は正常に動作しない。カメラ機能をテストするときは、必ず本物のiPhone（実機）を接続してテストする必要がある。

**実験2：画像の読み込み処理（Task や await）を消してみた**
- **やったこと：** `.onChange` の中にある `Task` や、`loadImage` を呼び出す際の `await` キーワード、および関数定義の `async` をすべて削除して、普通の処理として画像を読み込もうとした。
- **結果：** Xcodeから「非同期関数（async）は、非同期環境（Task）の中で await を使って呼び出さなければならない」という内容のエラー（コンパイルエラー）が発生し、アプリをビルドすることすらできなかった。
- **わかったこと：** 写真のような大きなデータを安全に読み込むための命令（`loadTransferable`）は、最初から「非同期（async）で動かすルール」としてAppleが作っている。そのため、SwiftUIでは時間の進み方が違う処理を扱うとき、必ず `Task` と `await` をセットで書かなければいけないという文法ルールが厳格に決まっている。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `PhotosPicker` を使うとき、なぜ「写真へのアクセスを許可しますか？」という警告（プライバシーポップアップ）が出ないのですか？
   **得られた理解：** 従来のアプリは「アルバム全体」にアクセスしていたため許可が必要だったが、`PhotosPicker` はシステム側が安全に管理された画面を貸し出しているだけだから。ユーザーが「これをアプリに渡す」とタップして選んだ写真だけがアプリに届くため、事前の全権限の許可が不要になり、安全かつスマートに実装できる。

2. **質問：** なぜ画像の読み込みに `async` や `await` を使わなければいけないのですか？
   **得られた理解：** 最近の写真は高画質なためデータサイズが非常に大きい。もしこれを画面を描画するメインの処理（メインスレッド）と同じ場所で読み込もうとすると、処理が終わるまでアプリの画面が完全にフリーズ（カクつき）してしまう。最悪の場合、OSに「固まったアプリ」と判定されて強制終了させられるのを防ぐため、裏側（バックグラウンド）でロードする非同期処理が必須。

3. **質問：** `UIViewControllerRepresentable` と `Coordinator` はそれぞれ何のためにあるのですか？
   **得られた理解：** SwiftUIにはまだ標準のカメラ画面を作る機能がないため、一世代前の「UIKit」のカメラ機能を借りる必要がある。その際の「画面の橋渡し役」が `UIViewControllerRepresentable` であり、カメラ画面から「撮影が終わった」「キャンセルされた」というイベント（報告）をSwiftUI側に持ち帰る「メッセンジャー役」が `Coordinator` である。この2つがセットで動くことで、古い資産をSwiftUIに統合できる。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
- **データの重さと非同期処理の重要性：** 写真や動画などの重いリソースを扱うときは、アプリをフリーズさせないために必ず `async/await`（Swift Concurrency）による非同期処理を意識して書くこと。
- **SwiftUIとUIKitの共存：** SwiftUIに足りない機能（カメラや一部の高度なUI）があっても諦めず、`UIViewControllerRepresentable` を使えばUIKitの強力な標準機能をそのまま安全に再利用できる。
- **実機テストの鉄則：** ハードウェア（カメラや各種センサーなど）が絡むアプリは、シミュレータではエラーになるか動かないケースが多いため、必ず最初から実機（iPhone）を繋いで開発・テストを行うこと。
