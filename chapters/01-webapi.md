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
