# 第4章：データの永続化

> 執筆者：関根 拓斗
> 
> 最終更新：2026-06-24

## この章で学ぶこと

この章では、SwiftUIにおけるデータの永続化（データ保存）の手法として、SwiftDataによる構造化データの管理と、AppStorageによる簡易的な設定情報の保存を学びます。これらの技術を組み合わせることで、ユーザーが入力した内容や設定をアプリ終了後も保持できる、実用的なメモアプリを作成します。

## 模範コードの全体像

```swift
// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

```

**このアプリは何をするものか：**

このアプリは、ユーザーがタイトルと内容を入力してメモを作成・保存できるアプリ。作成したメモは一覧で表示されて、詳細画面から編集やお気に入りにすることができる。また、設定画面からユーザー名の入力や、お気に入りメモをリストの上部に表示するかどうかの切り替えを行うことができる。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**

@Modelマクロを使って、永続化の対象となるデータ構造（Memoクラス）を定義している。

**なぜこう書くのか：**

@Modelを付けるだけで、SwiftDataが自動的にデータベースに保存・管理可能な形に変換してくれるため。

**もしこう書かなかったら：**

SwiftDataはこのクラスをデータベースに保存すべき対象として認識できず、データの永続化機能を利用できなくなる。

---

### データの追加・削除（modelContext）

```swift
//追加
modelContext.insert(memo)

//削除
func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        let memo = displayedMemos[index]
        modelContext.delete(memo)
    }
}
```

**何をしているか：**

modelContextを介して、データの追加と削除を行っている。

**なぜこう書くのか：**

modelContextはデータベースの変更を追跡し反映する役割を持つため。

**もしこう書かなかったら：**

データの追加や削除を行ってもデータベースに反映されず、アプリを再起動するとデータが元に戻ってしまう。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
```

**何をしているか：**

データベースに保存されてるすべてのMemoデータを取得し、memos配列に格納している。sort引数で作成日時の降順に並び替えられる。

**なぜこう書くのか：**

@Queryを使うと、データベースの内容が更新されたとき自動的にmemos配列を更新して、ビューが再描画されるため。

**もしこう書かなかったら：**

データを手動で取りに行くコードを書く必要があり、データが更新されても自動的に最新状態にならなくなる。

---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""
```

**何をしているか：**

UserDefaults への読み書きをプロパティのように手軽に行えるようにしている。

**なぜこう書くのか：**

今回のようなユーザー名やトグルスイッチの状態などの、構造化する必要のない設定を保存するのに効率的な方法だから。

**もしこう書かなかったら：**

複雑なSwiftDataモデルを作成するか、UserDefaultsを直接操作するコードを書く必要があり、コードが複雑化してしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `@Model` | SwiftDataでオブジェクトを永続化するためのマクロ | `@Model final class Memo { ... }` |
| `@Query` | データベースからデータを取得し、変更を自動で反映するプロパティラッパー | `@Query var memos: [Memo]` |
| `@AppStorage` | ユーザー名や設定のON/OFFなど、小さなデータを「UserDefaults」に自動保存・同期するためのプロパティラッパー | `@AppStorage("userName") private var userName = ""` |
| `modelContainer(for:)` | アプリ全体、あるいは特定の画面でSwiftDataを使用可能にするための初期化モディファイア。通常はAppクラスで指定する | `.modelContainer(for: Memo.self)` |
| `@Environment(\.modelContext)` | データの追加（insert）や削除（delete）をデータベースに対して実行するための「操作窓口（Context）」を取得するプロパティラッパー | `@Environment(\.modelContext) private var modelContext` |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：お気に入りを「上に表示」するだけでなく、「お気に入り以外を非表示にする（絞り込み）」処理に変更してみた。
- 結果：画面にはお気に入りにチェックが入ったメモだけが表示され、それ以外のメモはリストから消えた。
- わかったこと：データを永続化（SwiftData）していても、UIに表示する直前でフィルタリングをかけることで、ユーザーの目的に応じた柔軟な表示切り替えが可能になることがわかった。

**実験2：**
- やったこと：保存のタイミングを知るため TextField に入力している最中に、わざとアプリを強制終了（Xcodeで停止）させた。
- 結果：再起動した際、完全に入力し終えていなかった文字は保存されておらず、以前の値のままだった。
- わかったこと：AppStorage は基本的に「値が確定した時（フォーカスが外れた時など）」に保存されるため、リアルタイム保存ではないという性質があることが確認できた。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**　AppStorage と SwiftData の使い分けの基準は？

   **得られた理解：**　設定項目やフラグなど、単純なキーと値のペアなどの少量のデータは AppStorage、メモリストのような複雑なオブジェクトなどの大量のデータは SwiftData を使うのが基本ルールだとわかった。

2. **質問：**　@Bindable はなぜ MemoEditView で使われているのか？
   **得られた理解：**　Memo モデルのプロパティを直接 TextField や Toggle にバインドするためで、値を変更した瞬間に SwiftData がそれを検知し、自動的にデータベースを更新してくれるため、手動でセーブする処理が不要になる。

3. **質問：**　SwiftData モデルには @Model が必要なのか？
   **得られた理解：**　SwiftDataがそのクラスをデータベースのテーブルとして認識し、自動的に永続化対象として管理するために必須であり、これをつけるだけでSwiftのクラスがデータベースのデータへと早変わりする。

## この章のまとめ

SwiftUIにおいて、データの性質に応じて単純な設定値は AppStorage、複雑なデータは SwiftData というように保存方法を使い分けることを学びました。
また、@Model や @Bindable を活用することで、UI上の変更をデータベースへ自動同期させる効率的なデータ管理の仕組みや、なぜ @Model が必要なのか、@Bindable がなぜ使われているかを質問したことで、設定の永続化と構造化データの永続化を組み合わせた、動的なデータ表示を学ぶことができた。
