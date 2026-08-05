# 第7章：センサーの活用

> 執筆者：アウンミャッ
> 最終更新：2026/7/19

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）
この章では、CoreMotionを使ってiPhoneの傾きや向きを取得する方法を学びました。CMMotionManagerからpitch、roll、yawの値を取得し、そのデータを使って水平器のバブルを動かす方法を学びました。
例：この章では、iPhoneに搭載されている加速度・ジャイロなどのセンサーにアクセスして、デバイスの動きや姿勢を検出する方法を学ぶ。具体的にはCoreMotionを使った水平器アプリ、CMPedometerとCoreLocationを組み合わせた活動トラッカーを題材にして、センサーデータの読み取りと処理の実装を学ぶ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第7章（基本）：加速度センサーで動く水平器アプリ
// ============================================
// CoreMotionを使って端末の傾きをリアルタイムで取得し、
// 水平器（水準器）として表示するアプリです。
//
// 【注意】シミュレータではセンサーが使えません。
//         実機（iPhone / iPad）でテストしてください。
// ============================================

import SwiftUI
import CoreMotion

// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool

    init() {
        // 初回 body 評価時点で正しい値を返すよう、init で同期的にセット
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var motionManager = MotionManager()

    var body: some View {
        NavigationStack {
            if motionManager.isAvailable {
                VStack(spacing: 30) {
                    // 水平器の円
                    LevelIndicator(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll
                    )

                    // 数値表示
                    DataDisplay(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll,
                        yaw: motionManager.yaw
                    )
                }
                .padding()
                .navigationTitle("水平器")
            } else {
                ContentUnavailableView(
                    "センサーが利用できません",
                    systemImage: "iphone.slash",
                    description: Text("このアプリは実機（iPhone）で動作します。\nシミュレータではセンサーが使えません。")
                )
            }
        }
        .onAppear {
            motionManager.startUpdates()
        }
        .onDisappear {
            motionManager.stopUpdates()
        }
    }
}

// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）
このアプリは、iPhoneの傾きをリアルタイムで取得して表示する水平器アプリです。
iPhoneを前後や左右に傾けると、画面の円の中にあるバブルが傾いた方向へ移動します。端末がほぼ水平になるとバブルが緑色になり、「水平！」と表示されます。
また、画面の下には、前後の傾きであるpitch、左右の傾きであるroll、水平方向の回転であるyawの値が、ラジアンと角度で表示されます。

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    

    init() {
        // 初回 body 評価時点で正しい値を返すよう、init で同期的にセット
        isAvailable = motionManager.isDeviceMotionAvailable
    }


```

**何をしているか：**　CMMotionManagerを作成し、端末でモーションセンサーが利用できるか確認しています。

**なぜこう書くのか：**　センサーデータを取得する前に、端末がCoreMotionに対応しているか確認するためです。

**もしこう書かなかったら：**　　センサーが利用できない端末でも処理を開始しようとして、正しく動作しない可能性があります。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

```

**何をしているか：**　端末の前後の傾き、左右の傾き、水平方向の回転を取得しています。

**なぜこう書くのか：**　水平器のバブルを端末の傾きに合わせてリアルタイムで動かすためです。

**もしこう書かなかったら：**  pitch、roll、yawの値が更新されず、端末を傾けてもバブルが動きません。

---

### 歩数計（CMPedometer）

```swift
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?
    var endTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }
    func startTracking() {
        isTracking = true
        startTime = Date()
        endTime = nil
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        endTime = Date()
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }


```

**何をしているか：**　CMPedometerを使って歩数と移動距離を取っています。歩くと歩数が増えて、移動した距離も更新されます。

**なぜこう書くのか：**　ユーザーがどれくらい歩いたかを表示するためです。また、歩数と移動距離をリアルタイムで更新するためです。

**もしこう書かなかったら：**　 歩数や移動距離を取得できないので、画面の歩数や距離が更新されません。また、このアプリのメイン機能が動かなくなります。

---

### CoreLocationとの連携

```swift
    private let locationManager = CLLocationManager()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        locationManager.startUpdatingLocation()
    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

```

**何をしているか：**　CoreLocationを使って位置情報を取り、移動速度を更新しています。

**なぜこう書くのか：**　歩いているときの速度を表示するためです。また、位置情報を使って移動を記録できます。

**もしこう書かなかったら：**　　歩数だけでは、どのくらいの速さで移動したか分かりません。そのため、CoreLocationを使って位置情報や速度を取得しています。歩数計の情報と組み合わせることで、より正確に活動を記録できます。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| 例：`CMPedometer` | 歩数や歩行距離をカウント | `pedometer.queryPedometerData(from: startDate, to: Date())` |
| startDeviceMotionUpdates|姿勢データの取得を開始する | motionManager.startDeviceMotionUpdates(to: .main)pitch|
|pitch |端末の前後の傾きを表す | motion.attitude.pitch|
| [weak self]| クロージャとの相互保持による循環参照（メモリリーク）を防止|pedometer.startUpdates(from: Date()) { [weak self] data, error in |

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
2. DispatchQueue.mainはなんで必要？
   **得られた理解：**
   CMPedometer の処理がバックグラウンドスレッドで行われるためです。iOS のルールに従い、UI 更新をメインスレッドへ切り替えて安全に反映させるために必要だと理解しました。
   
4. **質問：**
5. なぜdeviceMotionUpdateIntervalを設定しますか。
   **得られた理解：**
   センサーデータを取得する間隔を設定し、バブルを滑らかに動かすためです。
   
7. **質問：**
8. なぜstopDeviceMotionUpdates()が必要ですか。
   **得られた理解：**
   画面を閉じた後に不要なセンサー処理を止め、バッテリー消費を減らすためです。
## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
この章では、CMMotionManagerを使ってiPhoneの姿勢データを取得する方法を学びました。pitch、roll、yawを使うことで端末の傾きや回転を調べ、取得した値をSwiftUIの画面表示やバブルの移動に利用できることが分かりました。
