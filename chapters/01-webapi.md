# 第1章：WebAPIの基本

> 執筆者：関根 拓斗
> 
> 最終更新：2026-04-17

## この章で学ぶこと

この章では、iTunes Search APIを利用して、インターネット上の音楽情報をアプリに読み込む仕組みを学ぶ。具体的には、ユーザーが入力したキーワードをURLに変換して送信し、返ってきたデータをSwiftの構造体として解析して、見やすいリストの形で表示する。

## 模範コードの全体像


```swift
// ============================================
// 第1章（基本）：iTunes Search APIで音楽を検索するアプリ
// ============================================
// このアプリは、iTunes Search APIを使って
// 音楽（曲）を検索し、結果をリスト表示します。
// APIキーは不要で、すぐに動かすことができます。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var songs: [Song] = []
    @State private var searchText: String = ""
    @State private var isLoading: Bool = false

    var body: some View {
        NavigationStack {
            VStack {
                // 検索バー
                HStack {
                    TextField("アーティスト名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)

                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(searchText.isEmpty)
                }
                .padding(.horizontal)

                // 検索結果リスト
                if isLoading {
                    ProgressView("検索中...")
                        .padding()
                    Spacer()
                } else if songs.isEmpty {
                    ContentUnavailableView(
                        "曲を検索してみよう",
                        systemImage: "music.note",
                        description: Text("アーティスト名を入力して検索ボタンを押してください")
                    )
                } else {
                    List(songs) { song in
                        SongRow(song: song)
                    }
                }
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - API通信

    func searchMusic() async {
        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else { return }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else { return }

        isLoading = true

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)
            songs = response.results
        } catch {
            print("エラー: \(error.localizedDescription)")
            songs = []
        }

        isLoading = false
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                Color.gray.opacity(0.3)
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)

                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }
        }
        .padding(.vertical, 4)
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

1.入力： 検索バーに気になるアーティスト名やキーワードを入力する。

2.通信： 検索ボタンを押すと、インターネット経由でiTunesのデータベースに問い合わせる。

3.表示： 取得したデータから「曲名」「アーティスト名」「ジャケット写真」をリスト形式で一覧表示する。

4.補助機能： 検索中は読み込み中であることを示すインジケーターが表示される。また、結果が0件の場合は専用のメッセージが表示される。


## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct SearchResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let artworkUrl100: String
    let previewUrl: String?

    var id: Int { trackId }
}
```

**何をしているか：**
APIから送られてくるJSON形式のデータを、Swiftのプログラム内で扱える「型」として定義している。Codableを適用することで、複雑なJSONデータを一瞬で構造体に変換できるようにしている。

**なぜこう書くのか：**
APIのデータ構造と、構造体のプロパティ名を一致させる必要があるから

**もしこう書かなかったら：**
JSONデータを辞書型として一つずつ手動で取り出す必要があってミスが起きやすくなる。


---

### API通信の処理

```swift
func searchMusic() async {
    guard let encodedText = searchText.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) else { return }
    let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"
    guard let url = URL(string: urlString) else { return }

    isLoading = true
    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(SearchResponse.self, from: data)
        songs = response.results
    } catch {
        print("エラー: \(error.localizedDescription)")
        songs = []
    }
    isLoading = false
}
```

**何をしているか：**
入力されたキーワードをURLに含め、インターネット経由でiTunesのサーバーにリクエストを送り、結果を受け取ってsongs配列に保存しています。

**なぜこう書くのか：**
async/awaitを使うことで、時間のかかる通信処理を「待ち」つつも、アプリの画面がフリーズしないように制御しています。また、日本語（全角文字）を検索できるようにaddingPercentEncodingでURLとして正しい形式に変換しています。

**もしこう書かなかったら：**
通信中に画面が全く動かなくなったり（フリーズ）、日本語で検索した瞬間にアプリがクラッシュしたりします。

---

### ビューの構成

```swift
List(songs) { song in
    SongRow(song: song)
}
```

**何をしているか：**
取得した音楽データの配列（songs）をループ処理し、一行ずつSongRowというカスタムビューを使ってリスト表示しています。

**なぜこう書くのか：**
Listを使うことで、iPhoneらしい標準的なリスト表示を簡単に実装でき、データの数に応じたスクロールも自動で行われるからです。

**もしこう書かなかったら：**
ScrollViewとForEachを組み合わせて自作する必要がありますが、データの数が多い場合にメモリ消費が激しくなり、パフォーマンスが低下する可能性があります。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| addingPercentEncoding | 文字列をURLとして安全に送信できる形式に変換するメソッド | searchText.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) |
| ContentUnavailableView | 検索結果が空のときなど、表示するコンテンツがない状態を美しく伝えるための専用View | ContentUnavailableView("曲を検索してみよう", systemImage: "music.note") |
| ProgressView | データの読み込み中を表示するためのView | ProgressView("検索中...") |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：limit=25 を limit=5 や limit=50 に変更したり、country=jp を他の国コード（例：us, gb）に変更して検索しました。
- 結果：取得できる曲の数が変わった。日本国外のランキング結果が返ってくるようになった。
- わかったこと：APIへのリクエストURLを組み立てる際のパラメータ（クエリパラメータ）によって、サーバー側から返ってくるデータの内容や件数をコントロールできる。

**実験2：**
- やったこと：APIのURL文字列を意図的に間違えて通信失敗を起こしました。
- 結果：catch ブロックが実行され、コンソールにエラーメッセージが表示された。また、songs が空配列になり、画面が初期状態に戻った。
- わかったこと：do-catch 文を使うことで、通信が失敗してもアプリがクラッシュすることなく、適切にエラー時の処理（画面表示の切り替えなど）ができる。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**　Codable とは何ですか？またこれをつけるだけでJSONが読み込めるのはなぜ？

   **得られた理解：**
   データの「型（Swiftの構造体）」と「外部のデータ形式（JSON）」を自動的に変換するための仕組み。エンコード（変換）とデコード（復元）を自動化してくれるプロトコルである。

2. **質問：**　async / await とは何ですか？なぜ通信処理に必要なのですか？
   
   **得られた理解：**
   通信は時間がかかる処理なので、メイン画面の動きを止めない（フリーズさせない）ために「非同期処理」が必要。完了を待つことを await で明示することで、可読性が高く安全に書ける。

3. **質問：**　@State とは何ですか？なぜ変数を直接書き換えるだけではダメなのですか？
   
   **得られた理解：**
   SwiftUIのビューがデータの変化を監視するための箱。@State をつけると、その変数が変わった瞬間に「画面を再描画せよ」という命令が走り、UIが最新の状態に更新される。

## この章のまとめ

API通信は、「URLで要求 → JSONで返却 → Codableで型変換 → @StateでUI更新」という一連のデータの流れで成り立っている。
通信処理には時間がかかるため、async/awaitを使ってメインスレッドをブロックしないことが重要であると学んだ。
