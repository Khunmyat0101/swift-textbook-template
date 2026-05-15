# 第3章：カメラの利用

> 執筆者：（カリム）
> 最終更新：2026/05/15

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

この章では、SwiftUIでフォトライブラリから写真を選択し、カメラで撮影した画像を画面に表示する方法を学ぶ。Photos​Picker による画像選択、async​/await を使った画像データの読み込み、UIView​Controller​Representable によるUIKit部品の組み込み、Coordinator によるイベント受け取りを扱う。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る
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

このアプリは、ユーザーが「ライブラリ」ボタンから端末内の写真を選ぶか、「カメラ」ボタンから写真を撮影し、その画像を画面中央に表示するアプリである。まだ画像が選ばれていないときは、写真アイコンと案内文が表示される。

## コードの詳細解説

### PhotosPickerによる写真選択

```swift
// 該当部分のコードを抜粋して貼る
@State private var selectedItem: PhotosPickerItem?

PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
```

**何をしているか：**
フォトライブラリから画像を選択するためのボタンを表示している。選択された写真は selected​Item に入る。

**なぜこう書くのか：**
Photos​Picker はSwiftUIで写真選択を簡単に扱える部品で、matching: .images にすることで画像だけを選べるようにしている。

**もしこう書かなかったら：**
フォトライブラリを開くUIが表示されず、ユーザーが端末内の写真を選択できない。

---

### 画像の非同期読み込み

```swift
// 該当部分のコードを抜粋して貼る
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}

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
写真が選ばれたタイミングで、画像データを非同期に読み込み、UIImage からSwiftUIの Image に変換して表示している。

**なぜこう書くのか：**
画像の読み込みには時間がかかる可能性があるため、async​/await を使って画面の動きを止めないようにしている。

**もしこう書かなかったら：**
選択した写真を実際に画面へ表示できない。また、重い画像を同期的に処理するとアプリの操作感が悪くなる可能性がある。
---

### UIViewControllerRepresentableによるカメラ連携

```swift
// 該当部分のコードを抜粋して貼る
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
}
```

**何をしているか：**
UIKitの UIImage​Picker​Controller をSwiftUIの画面から使えるようにしている。source​Type = .camera によってカメラ撮影モードにしている。
**なぜこう書くのか：**
カメラ撮影にはUIKitの UIImage​Picker​Controller を使うため、SwiftUIに組み込むには UIView​Controller​Representable が必要になる。
**もしこう書かなかったら：**
SwiftUIの画面からUIKitのカメラ画面を表示できない。
---

### Coordinatorパターン

```swift
// 該当部分のコードを抜粋して貼る
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
```

**何をしているか：**
カメラで撮影が完了したとき、またはキャンセルされたときの処理を受け取っている。撮影された画像は captured​Image に保存される。
**なぜこう書くのか：**
UIImage​Picker​Controller はデリゲートで結果を返すUIKitの仕組みなので、SwiftUI側で受け取るために Coordinator を使う。
**もしこう書かなかったら：**
撮影した画像を受け取れず、カメラ画面を閉じる処理もできない。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| Photos​Picker | フォトライブラリから画像を選択するSwiftUI部品 | Photos​Picker(selection: $selected​Item, matching: .images) |
| Photos​Picker​Item | 選択された写真を表す型 | @​State private var selected​Item: ​Photos​Picker​Item? |
| async​/await | 時間のかかる処理を非同期で実行する仕組み | try await item​.load​Transferable(type: ​Data​.self) |
| UIImage​Picker​Controller | カメラや写真選択に使うUIKitの画面部品 | picker​.source​Type = .camera |
| UIView​Controller​Representable | UIKitのViewControllerをSwiftUIで使うための仕組み | struct ​Camera​View: ​UIView​Controller​Representable |
| Coordinator | UIKitのデリゲート処理をSwiftUIに橋渡しするクラス | picker​.delegate = context​.coordinator |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
• やったこと：.images を外した。
• 結果：画像以外も選べる可能性がある。
• わかったこと：.images は画像だけに限定するために必要。

**実験2：**
• やったこと：selected​Image = ​Image(ui​Image: ui​Image) を消した。
• 結果：写真を選んでも表示されない。
• わかったこと：読み込んだ画像を Image に変換して保存する必要がある。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**PhotosPickerItem とは？
   **得られた理解：**選んだ写真を読み込むための情報。

2. **質問：** なぜ async/await を使う？
   **得られた理解：**画像読み込み中に画面を止めないため。

3. **質問：**なぜ Coordinator が必要？
   **得られた理解：**UIKitのカメラ結果をSwiftUIに渡すため。

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
