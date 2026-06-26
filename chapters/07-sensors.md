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
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

### CoreLocationとの連携

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| 例：`CMPedometer` | 歩数や歩行距離をカウント | `pedometer.queryPedometerData(from: startDate, to: Date())` |
| | | |
| | | |
| | | |

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
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
