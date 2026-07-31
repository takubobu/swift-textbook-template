# 第7章：センサーの活用

> 執筆者：関根 拓斗
> 最終更新：2026-06-26

## この章で学ぶこと

この章では、iPhoneに搭載されている加速度・ジャイロなどのセンサーに「CoreMotion」フレームワークを通じてアクセスし、取得したデータからデバイスの前後の傾き（Pitch）や左右の傾き（Roll）を計算してデバイスの傾きや姿勢をリアルタイムで検出する方法を学びます。

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

iPhone内蔵のセンサーから取得した傾きデータを基に、画面中央の円をリアルタイムに動かして端末の姿勢を視覚的に伝える水平器アプリであり、ほぼ水平になると円が赤から緑に変化して直感的に知らせるほか、画面下部には前後・左右の傾きや水平回転の数値をラジアンと度数の両方で正確に表示するもの。

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
@Observable
class MotionManager {
    private let motionManager = CMMotionManager()
    var isAvailable: Bool

    init() {
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }
        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0
        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            // アップデート処理
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}
```

**何をしているか：**

iOSのセンサー管理クラスである CMMotionManager をインスタンス化し、Device Motion が利用可能かチェックした上で、1秒間に60回の頻度でメインスレッドに向けてセンサーデータを定期的にアップデートする処理を行っています。

**なぜこう書くのか：**

センサーの起動や停止、更新頻度の設定は CMMotionManager を通して行う必要があるため。また、データの通知先をメインスレッドに指定することで、取得したセンサーデータを使ってSwiftUIの画面表示を安全に書き換えられるようにしている。

**もしこう書かなかったら：**

センサーが起動しないため画面の水平器が動かなくなり、deviceMotionUpdateInterval を設定し忘れると、既定値の遅い周期のままになり、傾きに対する画面の反応がカクついたり遅れたりする。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
    guard let self = self, let motion = motion else { return }

    self.pitch = motion.attitude.pitch
    self.roll = motion.attitude.roll
    self.yaw = motion.attitude.yaw
}
```

**何をしているか：**

定期的に通知されるセンサーデータ（CMDeviceMotion）の中から、デバイスの現在の姿勢（attitude）を表す3つの軸の回転角 pitch、roll、yaw をラジアン単位で取得し、クラスのプロパティに代入して画面側に共有している。

**なぜこう書くのか：**

加速度センサーだけの生データでは端末の「重力方向に対する正確な傾き」を計算するのが難しいため、内部でジャイロセンサーなどと motion.attitude（姿勢データ）を利用していて、ノイズの少ない滑らかな傾き情報を直接取得できる。

**もしこう書かなかったら：**

画面のインジケーター（LevelIndicator）を動かすための座標計算（xOffset / yOffset）ができなくなる。結果として、いくら端末を傾けても画面上の円は中心から動かず、水平かどうかの判定も一切行われない。

---

### 歩数計（CMPedometer）

```swift
private let pedometer = CMPedometer()

isPedometerAvailable = CMPedometer.isStepCountingAvailable()

pedometer.startUpdates(from: Date()) { [weak self] data, error in
    guard let self = self, let data = data else { return }

    DispatchQueue.main.async {
        self.stepCount = data.numberOfSteps.intValue
        if let dist = data.distance {
            self.distance = dist.doubleValue
        }
    }
}
```

**何をしているか：**

iPhoneの歩数センサーを利用して、歩数と移動距離をリアルタイムで取得している。

**なぜこう書くのか：**

CMPedometerは歩数や歩行距離を取得するためのクラスであり、startUpdates()を使うことで歩数が増えるたびに自動で最新の値を受け取れるため。

**もしこう書かなかったら：**

歩数や移動距離を取得できず、アプリに現在の歩数や距離が表示されなくなる。

---

### CoreLocationとの連携

```swift
private let locationManager = CLLocationManager()

locationManager.delegate = self
locationManager.desiredAccuracy = kCLLocationAccuracyBest
locationManager.requestWhenInUseAuthorization()

locationManager.startUpdatingLocation()

func locationManager(_ manager: CLLocationManager,
                     didUpdateLocations newLocations: [CLLocation]) {
    guard let location = newLocations.last else { return }

    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}
```

**何をしているか：**

GPSを利用して現在地を取得し、移動速度や通過した位置情報を記録している。

**なぜこう書くのか：**

CLLocationManagerを使うことで位置情報を継続的に取得でき、didUpdateLocationsで最新の位置や速度を受け取れるため。

**もしこう書かなかったら：**

現在地や移動速度を取得できず、速度メーターや移動履歴を表示できなくなる。

歩数は取得できても、GPSを利用した位置情報の機能は動作しない。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| `CMPedometer` | 歩数や歩行距離をカウント | `pedometer.queryPedometerData(from: startDate, to: Date())` |
| `isDeviceMotionAvailable` | 端末でモーションセンサー（DeviceMotion）が利用可能かチェックするプロパティ | `motionManager.isDeviceMotionAvailable` |
| `startDeviceMotionUpdates` | モーションセンサーからのデータ取得を開始し、値が届くたびにクロージャ内の処理を実行するメソッド | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| `stopDeviceMotionUpdates` | センサーデータの取得を停止するメソッド | `motionManager.stopDeviceMotionUpdates()` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：motionManager.deviceMotionUpdateInterval = 1.0 / 60.0 の部分を1.0 / 10.0（秒間10回）や 1.0 / 1.0（秒間1回）に変更してみる。
- 結果：1.0 / 1.0 にするとバブルの動きや数値の更新がカクカクになり、リアルタイム感が完全になくなった。
- わかったこと：滑らかなアニメーションやリアルタイム処理には適切な更新インターバル（60fpsなど）の設定が不可欠である一方、バッテリー消費とのトレードオフも考慮する必要があると分かった。

**実験2：**
- やったこと：バブルの .offset にある max(-maxOffset, min(maxOffset, xOffset)) を外し、単純な xOffset / yOffset だけに書き換えてみる。
- 結果：iPhoneを大きく傾けたとき、赤色のバブルが外側のグレーの円（枠線）を飛び出して画面外まで行ってしまった。
- わかったこと：min と max を組み合わせることで、数値がどれだけ大きくなってもUIの枠内に要素を留める「安全策（クランプ処理）」の書き方を学んだ。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   センサーの数値を表示するときに value * 180 / .pi と計算しているのはなぜ？
   
   **得られた理解：**
   CoreMotionから取得できる角度データは「弧度法（ラジアン: rad）」なので、人間になじみのある「度数法（度: °）」に変換するため。

2. **質問：**
   motionManager.deviceMotionUpdateInterval = 1.0 / 60.0 の意味は？
   
   **得られた理解：**
   1秒間に60回（60fps）センサーデータを更新する設定。画面の描写速度に合わせてぬるぬる動かすための指定であること。

3. **質問：**
   MotionManager に @Observable をつけているのはなぜ？
   
   **得られた理解：**
   センサーの値（pitch や roll）が変化したときに、SwiftUIがそれを検知して画面（ContentView）を自動で再描画できるようにするため。

## この章のまとめ

CoreMotionを用いてiPhoneの傾き（Pitch / Roll / Yaw）を取得し、リアルタイムでUIに反映させる水平器アプリの実装方法を学んだ。

・状態の同期： @Observable と 1.0 / 60.0 秒周期の更新を組み合わせ、滑らかで遅延のない描画処理を実現できた。

・UIの数値制御： min / max（クランプ処理）による移動範囲の制限や、abs（絶対値）を用いた水平判定の閾値設定など、画面に正確かつ安全に描画する計算手法を習得した。

・エラーハンドリング： シミュレータ等のセンサー非対応環境に対して ContentUnavailableView で適切なメッセージを出すUI設計の大切さを学んだ。
