# 第6章：ジェスチャー操作

> 執筆者： 関根 拓斗
> 最終更新：2026-07-03

## この章で学ぶこと

この章では、SwiftUIでタップ・ロングプレス・ドラッグ・ピンチ・回転などのジェスチャーを扱う方法を学ぶ。さらに、複数のジェスチャーを同時に使い、アニメーション付きで直感的に操作できるUIを作成する方法を理解する

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第6章（基本）：ジェスチャーで操作するカードアプリ
// ============================================
// タップ、ロングプレス、ドラッグ、ピンチ、回転の
// 各ジェスチャーを実際に体験しながら学びます。
// ============================================

import SwiftUI

// MARK: - メインビュー

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("タップ & ロングプレス") {
                    TapDemoView()
                }
                NavigationLink("ドラッグ") {
                    DragDemoView()
                }
                NavigationLink("ピンチ（拡大縮小）") {
                    MagnifyDemoView()
                }
                NavigationLink("回転") {
                    RotateDemoView()
                }
                NavigationLink("組み合わせ") {
                    CombinedDemoView()
                }
            }
            .navigationTitle("ジェスチャー体験")
        }
    }
}

// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}

// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}

// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}

// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                // 複数のジェスチャーを「同時に」効かせるには
                // .gesture を重ねるのではなく .simultaneousGesture を使う
                .simultaneousGesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .simultaneousGesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、さまざまなジェスチャー操作を体験できるアプリで、画面ごとにタップ、長押し、ドラッグ、拡大縮小、回転を試すことができ、最後の「組み合わせ」画面ではドラッグ・ピンチ・回転を同時に操作できる。

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
シングルタップ
RoundedRectangle(...)
.onTapGesture {
tapCount += 1
backgroundColor = Color(...)
}
ロングプレス
Circle()
.onLongPressGesture(minimumDuration: 1.0) {
isPressed = true
}
```

**何をしているか：**

タップ回数を増やしたり、長押しされたことを検知して状態を変更している

**なぜこう書くのか：**

ユーザーのタップや長押しを検出するため

**もしこう書かなかったら：**

タップや長押しをしても反応しなくなる

---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero
@State private var lastOffset: CGSize = .zero
DragGesture()
.onChanged { value in
offset = CGSize(...)
}
.onEnded { _ in
lastOffset = offset
}
```

**何をしているか：**

カードを指で動かせるようにしている

**なぜこう書くのか：**

ドラッグ中の移動量を保持して、位置を更新するため

**もしこう書かなかったら：**

カードが移動しなかったり、指を離した後に位置が戻ってしまう

---

### 拡大縮小と回転

```swift
拡大縮小
MagnifyGesture()
.onChanged { value in
scale = lastScale * value.magnification
}
回転
RotateGesture()
.onChanged { value in
angle = lastAngle + value.rotation
}
```

**何をしているか：**

ピンチで拡大縮小し、2本指で回転できるようにしている

**なぜこう書くのか：**

ジェスチャーの変化量を使って倍率や角度を更新するため

**もしこう書かなかったら：**

拡大縮小や回転の操作ができなくなる

---

### ジェスチャーの組み合わせとアニメーション

```swift
.animation(.spring(duration: 0.3), value: isPressed)

.gesture(
    DragGesture()
        .onChanged { value in
            offset = CGSize(
                width: lastOffset.width + value.translation.width,
                height: lastOffset.height + value.translation.height
            )
        }
        .onEnded { _ in
            lastOffset = offset
        }
)

.simultaneousGesture(
    MagnifyGesture()
        .onChanged { value in
            scale = lastScale * value.magnification
        }
        .onEnded { _ in
            lastScale = scale
        }
)

.simultaneousGesture(
    RotateGesture()
        .onChanged { value in
            angle = lastAngle + value.rotation
        }
        .onEnded { _ in
            lastAngle = angle
        }
)
```

**何をしているか：**

ドラッグ・拡大縮小・回転を同時に使い、動きをアニメーションで滑らかにしている

**なぜこう書くのか：**

.simultaneousGesture を使うと複数のジェスチャーを同時に認識できるため

**もしこう書かなかったら：**

一部のジェスチャーしか反応しなかったり、動きが不自然になる

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `DragGesture` | ドラッグジェスチャーを認識するジェスチャーレコグナイザー | `.gesture(DragGesture().onChanged { ... })` |
| `MagnificationGesture` | ピンチジェスチャーで拡大・縮小を認識 | `.gesture(MagnificationGesture().onChanged { scale in ... })` |
| `onTapGesture` | 要素がタップされたときに特定の処理を実行する最も手軽なモディファイア | `.onTapGesture { tapCount += 1 }` |
| `onLongPressGesture` | 要素が一定時間（minimumDuration）長押しされたときに処理を実行するモディファイア | `.onLongPressGesture(minimumDuration: 1.0) { ... }` |
| `RotateGesture` | 2本指でひねって回転させる操作を検出するジェスチャー | `RotateGesture().onChanged { value in ... }` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：DragDemoView で .onEnded の時の lastOffset = offset をコメントアウトして動かしてみた。
- 結果：ドラッグを離すと元の位置に巻き戻ってしまい、連続してカードを移動させることができなくなった。
- わかったこと：操作中の移動とは別に、直前の最終位置を保持しておかないと位置がリセットされてしまうことがわかった。

**実験2：**
- やったこと：CombinedDemoView の .simultaneousGesture を通常の .gesture に書き換えて動かしてみた。
- 結果：ドラッグしながらピンチで拡大しようとしても、片方の操作しか反応しなくなった。
- わかったこと：SwiftUIで複数のジェスチャーをユーザーに同時に行わせたい時は、.simultaneousGesture を使う必要がある。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   ドラッグ中のコードにある value.translation と lastOffset はどう違うの？
    
   **得られた理解：**
   value.translation は「今回指を離さずに動かした距離（相対値）」であり、lastOffset は「前回までに移動を終えた確定位置（絶対値）」。両方を足し合わせることでスムーズな連続移動が実現できる。
   
2. **質問：**
   onLongPressGesture の minimumDuration: 1.0 は何を指定しているの？
   
   **得られた理解：**
   長押しと判定されるまでに必要な秒数。これを調整することで、誤作動を防ぎつつユーザーにしっかり長押しさせるUIを作ることができる。

3. **質問：**
   リセットボタンを押した時の withAnimation(.spring) は何をしているの？
   
   **得られた理解：**
   値がゼロに戻る瞬間にバネ（スプリング）のような自然でヌルヌルとした動きをつけて、カードがピタッと元の位置に吸い付くような気持ちいいアニメーションを再現している。

## この章のまとめ

ユーザーに不満を与えないために工夫されていること

・正確な計算： 現在の移動量（translation）と過去の位置（lastOffset）を足し合わせてスムーズに連続移動させる。

・誤作動防止： minimumDuration で長押しの秒数を調整し、意図しない操作を防ぐ。

・心地よい演出： withAnimation(.spring) を使い、バネのような自然な動きで操作の心地よさを高める。
