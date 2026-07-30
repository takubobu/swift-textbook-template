# 第5章：機能統合の実践

> 執筆者： 関根 拓斗
> 
> 最終更新：2026-06-19

## この章で学ぶこと

この章では、SwiftData・CoreLocation・MapKit・PhotosUIを組み合わせ、写真・位置情報・地図を連携したアプリの開発方法を学ぶ。具体的には、写真の選択や現在地の取得、地図への表示、SwiftDataを利用したデータの保存・読み込みを行い、複数のフレームワークを活用した実践的なアプリを作成する。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第5章：写真 + 地図 + データ保存の統合アプリ
// ============================================
// 写真を選択し、選択時の現在地を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**

SwiftDataを使って、タイトル、メモ、位置情報、写真データを端末に永続保存するためのデータ構造を定義している。

**なぜこう書くのか：**

@Model を付けるだけで自動保存の対象になり、MapKitなどで扱いづらい写真や位置情報の型を基本型に変換して安全に保持するため。

**もしこう書かなかったら：**

アプリを終了するたびに保存したデータがすべて消えてしまい、表示する際も毎回複雑なデータ変換処理が必要になってしまう。

---

### タブ構成の設計

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}
```

**何をしているか：**

画面下部に「マップ」と「一覧」の2つのタブを配置し、画面を切り替えるための土台を作っている。

**なぜこう書くのか：**

地図で視覚的に探したい時と、リストで文字として整理・削除したい時に、ユーザーの目的が異なるため画面を綺麗に分離させるためq。

**もしこう書かなかったら：**

1つの画面に地図とリストが無理やり詰め込まれて非常に見づらくなってしまう。

---

### カメラと位置情報の連携

```swift
// LocationManagerの一部と、AddRecordView内での利用部分
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// AddRecordViewでの保存処理
func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
```

**何をしているか：**

現在地を常時取得するマネージャーを作成し、写真保存ボタンが押されたときの緯度・経度をデータに紐づけている。

**なぜこう書くのか：**

位置情報の取得に時間がかかるため、確実に取得できている場合のみ保存ボタンを押せるようにしてエラーを防ぐため。

**もしこう書かなかったら：**

位置情報が空のまま保存処理が行われてアプリが強制終了してしまう。

---

### SwiftDataでの画像保存

```swift
let record = PhotoRecord(..., imageData: selectedImageData)
modelContext.insert(record)
```

**何をしているか：**

選択された画像を Data 形式でデータベースに挿入している。

**なぜこう書くのか：**

画像型（Image）のままではデータベースに直接保存できないため、Data 型に変換して安全に書き込むため。

**もしこう書かなかったら：**

写真データをデータベースに保存できず、アプリを落とした瞬間に写真のデータが消失してしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `TabView` | 複数のビューをタブで切り替えるコンポーネント | `TabView { ... }.tabViewStyle(.page)` |
| `CLLocationManager` | GPS位置情報を取得するAPIManager | `let location = manager.location?.coordinate` |
| `CLLocationManager`| CoreLocationフレームワークで端末のGPS位置情報を取得・管理する | `private let manager = CLLocationManager()` |
| `UserAnnotation` | MapKitの地図上に「ユーザーの現在地を示す青い丸」を自動表示するコンポーネント | `Map { UserAnnotation() }` |
| `loadTransferable` | PhotosPicker で選択されたアイテムから、画像データ（Data）などを非同期（await）で読み込むメソッド | `newItem?.loadTransferable(type: Data.self)` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：@Queryの並び替え条件を変更してみた
- 結果：一覧画面の表示順序が作成日時の昇順やタイトル順に切り替わった。
- わかったこと：@Query の sort や order 引数を変えるだけで、データベースの複雑なソート処理を簡潔に書くことができる。

**実験2：**
- やったこと：AddRecordView の「保存」ボタンの .disabled 条件に selectedImageData == nil を追加してみた。
- 結果：タイトルと位置情報だけでなく、写真を選択しないと保存ボタンが押せなくなった。
- わかったこと：.disabled() の論理条件を書き換えることで、必須入力項目のバリデーション（入力チェック）を柔軟に制御できる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
    PhotosPickerItem から画像を取り出す loadTransferable とは何をしているの？

   **得られた理解：**
   フォトライブラリ内の安全な写真データを、アプリ内で扱える Data 型（バイナリデータ）へ非同期（await）で安全に転送・変換するための処理であること。

2. **質問：**
    SwiftDataで保存した画像データ（Data）を地図上のアイコンとして表示する仕組みは？
    
   **得られた理解：**
   @Model 内の計算プロパティ uiImage で UIImage(data:) を使って復元し、Map内の Annotation クロージャの中で Image(uiImage:) を使ってカスタムUI（丸型画像アイコン）として描画している。

3. **質問：**
    Map の中に UserAnnotation() と ForEach(records) を並べて書くだけで、なぜ複数のピンが表示されるの？
    
   **得られた理解：**
   iOS 17以降のMapKitはSwiftUIの List や VStack と同じように「ViewBuilder」の構造になっており、クロージャの中にピンの要素を並べるだけで自動的に地図上に描画してくれる仕組みになっているから。

## この章のまとめ

3つの重要技術

・データの集約： 写真データ・位置情報（緯度経度）・テキストを PhotoRecord モデル1つにまとめてSwiftDataで保存する流れを習得した。

・UIとデータの同期： 保存したデータを @Query で読み出し、地図（ピン表示）と一覧（リスト表示）の2つの画面に即座に反映させる仕組みを理解した。

・権限と安全対策： 位置情報や写真を使う際は Info.plist での使用目的の明記や、オプショナル型を用いたエラー回避が必須であることを学んだ。
