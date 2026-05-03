# AbletonSRTBrowser 実装指示書

## 0. このドキュメントの位置付け

本ドキュメントは、既存の REAPER 用 ReaScript [`ReaSRTBrowser`](https://github.com/naari3/reasrtbrowser) を **Ableton Live 向けに Rust で再実装する**プロジェクトの仕様書です。新規セッションがこれだけを読んで作業着手できるよう自己完結させてあります。

実装は別リポジトリ (例: `AbletonSRTBrowser`) で行います。既存の ReaSRTBrowser リポジトリは参考実装としてのみ参照します。

ドキュメント中、**「移植元」と書いた箇所は ReaSRTBrowser** を指します。Lua/REAPER API 固有の実装は、そのまま読み替えるのではなく Rust + Ableton Live の流儀で再構成します。

## 1. プロジェクトのゴール

### 1.1 機能ゴール

- SRT 字幕ファイルと、それに紐付くオーディオ (主に `.wav`) を、Ableton Live 上のオーディオクリップとして利用できるようにする外部ブラウザアプリ
- 移植元と同等の以下の機能を提供:
  - SRT の閲覧 / テキスト検索 / タグフィルタ / お気に入りフィルタ / 話者ラベル隠蔽
  - 個別アイテムへのタグ付け / お気に入りトグル / Global Offset 編集
  - 音声プレビュー (Space キーで該当区間再生 / 音量 0–100)
  - ライブラリ (複数 SRT を 1 単位として扱うコレクション、フォルダ階層あり)
  - 話者タグ自動抽出 (`(話者)` パターン)
  - 多言語 (en / ja)、フォント設定、レイアウト永続化
  - キーボード操作中心の UX (↑↓ で移動、Space でプレビュー、Enter で挿入)
- Ableton Live への挿入経路を **2 系統サポート**:
  - **D&D**: アプリの GUI から Live の Arrangement に直接ドラッグ
  - **キーボード操作**: Enter / ダブルクリックで、Live のカーソル位置にプログラム挿入

### 1.2 非機能ゴール

- 配布形態: macOS / Windows / Linux のスタンドアロン GUI アプリ (シングルバイナリ + 同梱リソース)
- AbletonOSC など外部 OSS への実行時依存はゼロ。Remote Script は本プロジェクトが自前で同梱する Python パッケージのみ。
- Live Remote Script を入れない場合でも、`.alc` D&D 経路で最低限の挿入が動くこと
- Live が起動していなくても GUI は単独で起動でき、SRT の閲覧 / 編集 / 設定変更ができる

### 1.3 非ゴール

- Live 以外の DAW のサポート
- SRT 以外の字幕形式 (VTT, ASS, etc.) のサポート
- オーディオ自体の編集 (波形編集、書き出し、ノーマライズなど)。アプリは「参照を返す」だけで、音声ファイルは触らない。
- MIDI クリップ生成
- リアルタイムの字幕同期再生 (移植元にもない)

## 2. 全体アーキテクチャ

### 2.1 コンポーネント構成

```
┌─────────────────────────────────────────────────┐
│  AbletonSRTBrowser (Rust standalone GUI)        │
│  ┌────────┐  ┌──────────┐  ┌─────────────┐      │
│  │ Model  │←→│  GUI     │←→│  Inserter   │      │
│  │ (SRT/  │  │  (egui)  │  │  Trait      │      │
│  │  meta) │  └────┬─────┘  └──┬───────┬──┘      │
│  └────────┘       │            │       │         │
│  ┌────────┐       │            │       │         │
│  │ Audio  │←──────┘            │       │         │
│  │ Player │                    │       │         │
│  └────────┘                    │       │         │
└────────────────────────────────┼───────┼─────────┘
                                 │       │
                  ┌──────────────┘       └──────────┐
                  ▼                                 ▼
           ┌──────────────┐                ┌────────────────┐
           │ AlcInserter  │                │ RemoteInserter │
           │ (.alc 生成   │                │ (TCP+JSON)     │
           │  + drag-out) │                │                │
           └───────┬──────┘                └────────┬───────┘
                   │ OS-level                       │ TCP localhost:19823
                   │ file D&D                       │
                   ▼                                ▼
        ┌──────────────────┐               ┌────────────────────┐
        │ Ableton Live     │←─── LOM ──────│ Live Remote Script │
        │ (Arrangement)    │   (Python)    │ (本プロジェクト同梱)│
        └──────────────────┘               └────────────────────┘
```

### 2.2 設計原則

1. **Inserter trait による挿入の抽象化** が中心。GUI コードは挿入の実装方法を知らない。
2. **GUI スレッド = メインスレッド** とし、I/O や音声デコードはバックグラウンドへ委譲する。`std::sync::mpsc` または `crossbeam_channel` でメッセージングする。
3. **モデルとビューを分離** する。`model::*` は egui に依存しない純粋ロジック。テスト容易。
4. **永続化はモデル側の責務**、GUI は dirty フラグを立てるだけ。debounce で書き込み (移植元と同じ)。
5. **Live への依存はオプショナル**。Live なし / Remote Script なしでも GUI 単体は完全に動く。

### 2.3 並行モデル

- メインスレッド: egui 描画
- 音声プレビュー: rodio が内部でスレッドを持つ。アプリからは `Player::play(...)` / `Player::stop()` だけ呼ぶ。
- Remote Script クライアント: 接続/再接続/状態購読のためのワーカースレッド 1 本。受信メッセージを `mpsc::Sender<RemoteEvent>` でメインに流す。
- ファイル監視 (任意): `notify` クレートでバックグラウンド。

### 2.4 ディスク IO

- 設定 / メタデータの読み書きは原則メインスレッドで OK (サイズが小さいため)。
- 数百件規模の SRT 一括読み込み (ライブラリ起動時など) はバックグラウンドで読み、進捗をチャネルで返す。

## 3. リポジトリ構成

```
abletonsrtbrowser/
├── Cargo.toml
├── README.md
├── LICENSE
├── crates/
│   └── remote_script/                # Live にインストールする Python パッケージ
│       ├── README.md                 # インストール手順
│       ├── __init__.py
│       ├── manager.py                # ControlSurface 派生
│       ├── server.py                 # ノンブロッキング TCP+JSON サーバ
│       └── handlers.py               # op ごとのハンドラ群
├── src/
│   ├── main.rs                       # エントリ。tracing 初期化と eframe::run_native
│   ├── app.rs                        # AppState / eframe::App 実装
│   ├── settings.rs                   # 設定の読み書き、debounce
│   ├── i18n/
│   │   ├── mod.rs                    # I18n トレイト、ロード機構
│   │   ├── en.rs                     # 英語キー定義 (静的)
│   │   └── ja.rs                     # 日本語キー定義 (静的)
│   ├── model/
│   │   ├── mod.rs
│   │   ├── srt.rs                    # SRT パーサ + SrtItem
│   │   ├── source.rs                 # SrtSource (1 SRT + 関連メタ)
│   │   ├── library.rs                # ライブラリ + フォルダツリー
│   │   ├── metadata.rs               # 永続化用 JSON スキーマ
│   │   ├── audio_match.rs            # 自動オーディオマッチング
│   │   └── filter.rs                 # 検索 / タグ / お気に入りフィルタ
│   ├── audio/
│   │   ├── mod.rs                    # PreviewPlayer の公開 API
│   │   └── player.rs                 # rodio + symphonia 実装
│   ├── insertion/
│   │   ├── mod.rs                    # Inserter trait, Composite, 共通型
│   │   ├── alc.rs                    # AlcInserter
│   │   ├── alc_xml.rs                # XML 組み立て
│   │   ├── remote.rs                 # RemoteScriptInserter (TCP クライアント)
│   │   └── drag_out.rs               # drag クレートとの統合
│   ├── ui/
│   │   ├── mod.rs                    # ペイン分割 (TopBottomPanel + SidePanel)
│   │   ├── menu.rs                   # メニューバー
│   │   ├── source_pane.rs            # 左ペイン Sources タブ
│   │   ├── library_pane.rs           # 左ペイン Libraries タブ
│   │   ├── item_table.rs             # 中央: アイテム一覧 (egui_extras::Table)
│   │   ├── detail_pane.rs            # 下: 編集ペイン
│   │   ├── dialogs.rs                # 設定ダイアログ、テキスト入力プロンプト
│   │   ├── toasts.rs                 # ステータス通知 (移植元の status_msg 相当)
│   │   └── shortcuts.rs              # キーバインド集中管理
│   └── util/
│       ├── debounce.rs
│       └── path_hash.rs              # SRT パスのハッシュ (sha2)
├── resources/
│   ├── alc_template.xml              # .alc XML テンプレート (include_str!)
│   └── fonts/                        # 同梱フォント (任意 / NotoSansCJK 等)
├── tests/
│   ├── srt_parser.rs
│   ├── alc_xml.rs                    # alc 出力のスナップショット
│   └── filter.rs
└── docs/
    ├── REMOTE_SCRIPT_INSTALL.md      # ユーザ向けセットアップ
    └── ARCHITECTURE.md               # 開発者向け
```

### 3.1 命名規則

- モジュール名はスネークケース、型名はパスカルケース
- `*_pane.rs` は egui の 1 ペインに対応する
- `model::*` は egui / eframe / rodio に依存しないこと (純粋ロジック)
- `insertion::*` は GUI に依存しないこと (テスト時に GUI 抜きで実行できるように)

### 3.2 ワークスペース化について

`crates/remote_script/` は Python 側の成果物なので Cargo ワークスペースのメンバーには含めません。Cargo は Rust 側 (`abletonsrtbrowser`) のシングルクレート構成で十分です。Python 側は別途 zip にまとめて release に同梱します。

## 4. 依存クレート (`Cargo.toml`)

```toml
[package]
name = "abletonsrtbrowser"
version = "0.1.0"
edition = "2021"
rust-version = "1.78"

[dependencies]
# GUI
eframe = { version = "0.30", default-features = false, features = ["default_fonts", "glow", "persistence"] }
egui = "0.30"
egui_extras = { version = "0.30", features = ["all_loaders"] }
egui_dnd = "0.10"                    # 内部 D&D (ライブラリ並び替え)

# ファイルダイアログ
rfd = "0.15"

# シリアライズ
serde = { version = "1", features = ["derive"] }
serde_json = "1"
quick-xml = { version = "0.36", features = ["serialize"] }

# 圧縮 (.alc は gzip)
flate2 = "1"

# オーディオ
rodio = { version = "0.20", default-features = false, features = ["symphonia-all"] }
symphonia = { version = "0.5", features = ["all"] }
hound = "3"                          # WAV ヘッダ読み (フレーム数 / SR の高速取得)

# Drag-out
drag = "0.4"                         # tauri-apps/drag。winit / tao 対応

# その他
anyhow = "1"
thiserror = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
once_cell = "1"
regex = "1"
dirs = "5"
sha2 = "0.10"
notify = "6"                         # ファイル変更検知 (任意)
unicode-segmentation = "1"           # 話者ラベル抽出 / 検索の Unicode セーフ操作
walkdir = "2"                        # 自動オーディオ検出
chrono = { version = "0.4", default-features = false, features = ["clock"] }

[dev-dependencies]
insta = "1"                          # スナップショットテスト (alc XML)
pretty_assertions = "1"
tempfile = "3"

[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
strip = true
```

### 4.1 クレート選定の根拠

| 用途 | 採用 | 理由 |
|------|------|------|
| GUI | `egui` + `eframe` | 移植元の ReaImGui に最も近い (immediate mode)。Pure Rust。クロスプラットフォーム。 |
| 表 | `egui_extras::TableBuilder` | 列ごとの幅指定 / sticky header / sortable / virtual scroll に対応。 |
| 内部 D&D | `egui_dnd` | ライブラリ内のソース並び替え用。 |
| 外部 D&D | `drag` (tauri-apps) | アプリ外へのファイルドラッグを唯一クロスプラットフォームで提供。winit に直接統合できる。 |
| オーディオ | `rodio` + `symphonia` | 範囲再生・シーク・複数フォーマット対応。`hound` は WAV ヘッダ読みの高速版。 |
| シリアライズ | `serde_json` / `quick-xml` | JSON は設定 / メタデータ、XML は `.alc` 用。 |
| 圧縮 | `flate2` | `.alc` は gzip。 |
| ダイアログ | `rfd` | OS ネイティブのファイルピッカー。 |

### 4.2 採用しないもの (および理由)

- `imgui-rs`: docking が upstream にない / Pure Rust ではない / Windows 11 高 DPI で問題が出やすい
- `iced`: retained mode で immediate mode 中心の設計と合わない。表のスクロール表示が苦しい
- `slint`: DSL 学習コストとサイズ
- `tauri`: フロントエンドに Web を要求するため過剰

## 5. 挿入抽象 (中核設計)

### 5.1 設計思想

GUI コードは「いま挿入したい」というイベントだけを発火し、**どの経路でどう Live に届けるかを知らない**。経路ごとの実装は `Inserter` トレイトを実装するだけで差し替え可能。テスト時は `MockInserter` を差し込める。

### 5.2 共通型

```rust
// src/insertion/mod.rs

use std::path::PathBuf;
use anyhow::Result;

/// SRT エントリ 1 件分の挿入対象 (global_offset 適用済みの確定値)。
#[derive(Clone, Debug)]
pub struct InsertionItem {
    pub source_audio: PathBuf,        // 参照する元 wav の絶対パス
    pub start_sec: f64,               // ソース内の開始秒
    pub end_sec: f64,                 // ソース内の終了秒
    pub display_name: String,         // クリップ名 / Take 名
}

#[derive(Clone, Debug)]
pub struct InsertionRequest {
    pub items: Vec<InsertionItem>,
    /// Live の現在テンポ。秒↔ビート換算用。
    /// Remote 経路では Manager から最新値を取得して上書きしてよい。
    /// .alc 経路では tempo はクリップ長に影響する重要パラメータ。
    pub tempo_bpm: f64,
}

/// 何によって挿入が起動されたか。Composite が backend を選ぶのに使う。
#[derive(Copy, Clone, Debug, PartialEq, Eq)]
pub enum InsertionTrigger {
    /// Enter / ダブルクリック / メニューの "Insert"
    Keyboard,
    /// マウスドラッグの開始 (アプリ外への drag-out)
    DragStart,
}

#[derive(Debug)]
pub enum InsertionOutcome {
    /// 同期挿入が完了 (Remote Script 経路)
    Inserted { count: usize },
    /// 一時ファイルを作りドラッグペイロードとして渡す準備が整った (.alc 経路)
    DragPrepared { temp_files: Vec<PathBuf> },
}

pub trait Inserter: Send {
    fn supports(&self, trigger: InsertionTrigger) -> bool;
    fn insert(&mut self, req: InsertionRequest, trigger: InsertionTrigger)
        -> Result<InsertionOutcome>;
    /// 利用可能か (例: Remote Script の接続が生きているか)
    fn is_available(&self) -> bool { true }
    /// 状態表示用ラベル (UI のステータスバー表示など)
    fn status_label(&self) -> &'static str;
}
```

### 5.3 Composite Inserter

```rust
pub struct CompositeInserter {
    pub remote: Box<dyn Inserter>,           // Keyboard 担当
    pub alc:    Box<dyn Inserter>,           // DragStart 担当 (常時 OK)
    pub fallback_keyboard_to_alc: bool,      // Remote 不通時、Keyboard を .alc + open に
}

impl Inserter for CompositeInserter {
    fn supports(&self, _t: InsertionTrigger) -> bool { true }

    fn insert(&mut self, req: InsertionRequest, trigger: InsertionTrigger)
        -> Result<InsertionOutcome>
    {
        match trigger {
            InsertionTrigger::DragStart => self.alc.insert(req, trigger),
            InsertionTrigger::Keyboard => {
                if self.remote.is_available() {
                    self.remote.insert(req, trigger)
                } else if self.fallback_keyboard_to_alc {
                    self.alc.insert(req, trigger)
                } else {
                    anyhow::bail!("Remote Script not connected")
                }
            }
        }
    }

    fn status_label(&self) -> &'static str {
        if self.remote.is_available() { "remote+alc" } else { "alc-only" }
    }
}
```

### 5.4 GUI 側の使用例

```rust
// ui::item_table 内
fn handle_keyboard_insert(app: &mut AppState, ctx: &egui::Context) {
    let enter = ctx.input(|i| i.key_pressed(egui::Key::Enter));
    if !enter { return; }
    if app.is_text_input_focused() { return; }       // 入力中は誤発火しない

    let req = build_request(&app.selected_items, app.live_tempo());
    match app.inserter.insert(req, InsertionTrigger::Keyboard) {
        Ok(InsertionOutcome::Inserted { count }) =>
            app.toast.success(format!("Inserted {count} clip(s).")),
        Ok(InsertionOutcome::DragPrepared { .. }) =>
            app.toast.info("Saved .alc — drop into Ableton."),
        Err(e) => app.toast.error(format!("Insert failed: {e}")),
    }
}

// ダブルクリック挿入 (Selectable のレスポンスから)
if response.double_clicked() && !app.is_text_input_focused() {
    handle_keyboard_insert(app, ctx);
}

// drag-out 開始
if response.drag_started_by(egui::PointerButton::Primary) {
    let req = build_request(&app.selected_items, app.live_tempo());
    if let Ok(InsertionOutcome::DragPrepared { temp_files })
        = app.inserter.insert(req, InsertionTrigger::DragStart)
    {
        app.start_external_drag(temp_files);
    }
}
```

### 5.5 テスト容易性

```rust
pub struct MockInserter {
    pub calls: Vec<(InsertionRequest, InsertionTrigger)>,
}
impl Inserter for MockInserter {
    fn supports(&self, _t: InsertionTrigger) -> bool { true }
    fn insert(&mut self, req: InsertionRequest, t: InsertionTrigger)
        -> Result<InsertionOutcome>
    {
        self.calls.push((req.clone(), t));
        Ok(InsertionOutcome::Inserted { count: req.items.len() })
    }
    fn status_label(&self) -> &'static str { "mock" }
}
```

GUI レイヤのテストでは `Box<dyn Inserter>` に `MockInserter` を入れて、ショートカット → trait 呼び出しまでの経路を検証する。

## 6. `.alc` バックエンド

### 6.1 ファイル形式の概要

`.alc` (Ableton Live Clip) は **gzip 圧縮された XML**。中身はサンプルファイルへの参照、ループ範囲、ワープ情報を持つ Live Clip のシリアライズ。

- 元の音声を切らず**ファイル参照**だけを持つ → REAPER 版と同じく「サブレンジ参照」式
- Live バージョンによって `MajorVersion` / `MinorVersion` / `SchemaChangeCount` 属性が異なる
- 公式仕様書なし。実装は **本物の `.alc` を 1 つ作って中身を確認 → 必要要素だけテンプレ化** する

### 6.2 検証手順 (実装着手時に必須)

1. Live で空のセットを作る
2. オーディオトラックに `test.wav` を 1 クリップ配置 (任意の長さで OK)
3. Browser から右クリック → Save → `test.alc` で保存
4. `gunzip -c test.alc | xmllint --format -` で XML を読む
5. `SampleRef`, `Loop`, `WarpMode`, `IsWarped`, `CurrentStart` / `CurrentEnd`, `OutMarker`, `HiddenLoopStart` / `HiddenLoopEnd` の値を確認
6. 別のレンジで切り直したクリップでもう 1 つ `.alc` を作り、差分を取って「サブレンジ指定で変わるフィールド」を特定
7. 上記から **最小スキーマ** を確定し、テンプレート XML を作る

### 6.3 最小スキーマ (出発点)

`resources/alc_template.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Ableton MajorVersion="5" MinorVersion="11.0_11202" SchemaChangeCount="3"
         Creator="AbletonSRTBrowser" Revision="">
  <LiveSet>
    <Tracks>
      <AudioTrack Id="0">
        <Name><EffectiveName Value="{TRACK_NAME}"/></Name>
        <DeviceChain>
          <MainSequencer>
            <Sample>
              <ArrangerAutomation>
                <Events>
                  <AudioClip Time="0" Id="0">
                    <Name Value="{CLIP_NAME}"/>
                    <CurrentStart Value="0"/>
                    <CurrentEnd Value="{LENGTH_BEATS}"/>
                    <Loop>
                      <LoopStart Value="{START_BEATS}"/>
                      <LoopEnd Value="{END_BEATS}"/>
                      <StartRelative Value="0"/>
                      <LoopOn Value="false"/>
                      <OutMarker Value="{END_BEATS}"/>
                      <HiddenLoopStart Value="{START_BEATS}"/>
                      <HiddenLoopEnd Value="{END_BEATS}"/>
                    </Loop>
                    <SampleRef>
                      <FileRef>
                        <RelativePathType Value="3"/>
                        <RelativePath Value="{REL_PATH}"/>
                        <Path Value="{ABS_PATH}"/>
                        <Type Value="2"/>
                        <LivePackName Value=""/>
                        <LivePackId Value=""/>
                      </FileRef>
                      <DefaultDuration Value="{SAMPLE_FRAMES}"/>
                      <DefaultSampleRate Value="{SAMPLE_RATE}"/>
                    </SampleRef>
                    <WarpMode Value="0"/>
                    <WarpMarkers/>
                    <IsWarped Value="false"/>
                  </AudioClip>
                </Events>
              </ArrangerAutomation>
            </Sample>
          </MainSequencer>
        </DeviceChain>
      </AudioTrack>
    </Tracks>
  </LiveSet>
</Ableton>
```

属性は Live バージョンで増減します。検証手順 6.2 の結果に従って実装時にテンプレを最終化してください。

### 6.4 実装スケッチ

```rust
// src/insertion/alc.rs

use std::fs::{self, File};
use std::io::Write;
use std::path::{Path, PathBuf};
use std::time::{Duration, SystemTime};
use anyhow::{Context, Result};
use flate2::write::GzEncoder;
use flate2::Compression;

use crate::insertion::{
    Inserter, InsertionRequest, InsertionOutcome, InsertionTrigger, InsertionItem,
};

const ALC_TEMPLATE: &str = include_str!("../../resources/alc_template.xml");
const MAX_TEMP_AGE: Duration = Duration::from_secs(60 * 60 * 24); // 24h

pub struct AlcInserter {
    temp_dir: PathBuf,
}

impl AlcInserter {
    pub fn new() -> Result<Self> {
        let temp_dir = std::env::temp_dir().join("AbletonSRTBrowser");
        fs::create_dir_all(&temp_dir)
            .with_context(|| format!("create {:?}", temp_dir))?;
        let me = Self { temp_dir };
        me.gc_old_files();
        Ok(me)
    }

    fn gc_old_files(&self) {
        let Ok(entries) = fs::read_dir(&self.temp_dir) else { return };
        let now = SystemTime::now();
        for e in entries.flatten() {
            if let Ok(meta) = e.metadata() {
                if let Ok(modified) = meta.modified() {
                    if now.duration_since(modified).unwrap_or_default() > MAX_TEMP_AGE {
                        let _ = fs::remove_file(e.path());
                    }
                }
            }
        }
    }

    fn render_alc(&self, item: &InsertionItem, tempo_bpm: f64) -> Result<PathBuf> {
        let beats_per_sec = tempo_bpm / 60.0;
        let start_beats = item.start_sec * beats_per_sec;
        let end_beats   = item.end_sec   * beats_per_sec;
        let length      = end_beats - start_beats;

        let (sample_frames, sample_rate) = probe_wav(&item.source_audio)?;
        let abs = item.source_audio.canonicalize()?;
        let abs_str = abs.to_string_lossy();

        let xml = ALC_TEMPLATE
            .replace("{TRACK_NAME}", &xml_escape(&item.display_name))
            .replace("{CLIP_NAME}",  &xml_escape(&item.display_name))
            .replace("{LENGTH_BEATS}", &format_f64(length))
            .replace("{START_BEATS}",  &format_f64(start_beats))
            .replace("{END_BEATS}",    &format_f64(end_beats))
            .replace("{ABS_PATH}",     &xml_escape(&abs_str))
            .replace("{REL_PATH}",     &xml_escape(&abs_str))
            .replace("{SAMPLE_FRAMES}", &sample_frames.to_string())
            .replace("{SAMPLE_RATE}",   &sample_rate.to_string());

        let safe = sanitize_filename(&item.display_name);
        let stamp = chrono::Utc::now().timestamp_millis();
        let path = self.temp_dir.join(format!("{stamp}_{safe}.alc"));
        let f = File::create(&path)?;
        let mut gz = GzEncoder::new(f, Compression::default());
        gz.write_all(xml.as_bytes())?;
        gz.finish()?;
        Ok(path)
    }
}

impl Inserter for AlcInserter {
    fn supports(&self, _t: InsertionTrigger) -> bool { true }
    fn insert(&mut self, req: InsertionRequest, _t: InsertionTrigger)
        -> Result<InsertionOutcome>
    {
        let mut paths = Vec::with_capacity(req.items.len());
        for item in &req.items {
            paths.push(self.render_alc(item, req.tempo_bpm)?);
        }
        Ok(InsertionOutcome::DragPrepared { temp_files: paths })
    }
    fn status_label(&self) -> &'static str { "alc" }
}

fn probe_wav(path: &Path) -> Result<(u64, u32)> {
    let reader = hound::WavReader::open(path)?;
    let spec = reader.spec();
    let frames = reader.duration() as u64;
    Ok((frames, spec.sample_rate))
}

fn format_f64(v: f64) -> String { format!("{:.10}", v) }

fn xml_escape(s: &str) -> String {
    s.replace('&', "&amp;").replace('<', "&lt;").replace('>', "&gt;")
     .replace('"', "&quot;").replace('\'', "&apos;")
}

fn sanitize_filename(s: &str) -> String {
    s.chars().map(|c| match c {
        '/' | '\\' | ':' | '*' | '?' | '"' | '<' | '>' | '|' | '\0' => '_',
        c if c.is_control() => '_',
        c => c,
    }).take(80).collect()
}
```

### 6.5 OS レベル D&D との連携

```rust
// src/insertion/drag_out.rs

use std::path::PathBuf;
use anyhow::Result;
use drag::{DragItem, Image, start_drag};

pub fn start_external_drag(
    window: &impl raw_window_handle::HasWindowHandle,
    files: Vec<PathBuf>,
) -> Result<()> {
    let preview = Image::Raw(include_bytes!("../../resources/drag_icon.png").to_vec());
    start_drag(window, DragItem::Files(files), preview, |_result| {})?;
    Ok(())
}
```

`window` は `eframe::Frame` の `raw_window_handle()` 経由で取得できる。winit 直叩きの場合は `Window::raw_window_handle()`。

### 6.6 注意事項

- `temp_dir` のクリーンアップは起動時とアプリ終了時に行う。**ドロップ直後の即削除は NG** (Live はドロップ後数秒〜十数秒、ファイルを参照する可能性がある)。
- `.alc` は `tempo_bpm` でクリップ長が決まる。**Live 側のテンポと一致させる**ために、Remote Script から `Song.tempo` を購読しておく (取れない時は前回値 or 120.0)。
- `RelativePath` を絶対パスで埋めても多くの場合動くが、Live は警告を出すことがある。気になるなら `RelativePathType=0` + `RelativePath=""` に。
- Live がドロップを受け付けるのは Arrangement と Session の両方。**Session に落とすと clip slot に入る**ことに注意 (これは仕様として OK とする)。
- ファイル名にユーザ入力 (字幕テキスト) が入るので、**サニタイズ必須**。`sanitize_filename` で行頭。

## 7. Remote Script バックエンド

Remote Script は **Live のプロセス内で動く Python パッケージ**。本プロジェクトは AbletonOSC に依存せず、自前で最小実装を同梱します。

### 7.1 設置場所と動作要件

- macOS: `~/Music/Ableton/User Library/Remote Scripts/AbletonSRTBrowser/`
- Windows: `Documents\Ableton\User Library\Remote Scripts\AbletonSRTBrowser\`
- Live を再起動 → `Preferences → Link, Tempo & MIDI → Control Surface` のドロップダウンで `AbletonSRTBrowser` を選択
- 動作要件: **Live 11 以降 (Python 3)**

### 7.2 通信プロトコル

- **TCP + JSON Lines** (1 行 = 1 メッセージ)
- ループバック `127.0.0.1:19823` で listen
- 改行 `\n` で区切る
- **接続は外部 GUI 起動時に張りっぱなし** にし、用が終わるまで保持
- スレッドは使わず、`schedule_message(1, _tick)` で 100ms ごとに `select` で読む

### 7.3 オペレーション一覧

| op | 引数 | 戻り値 | 用途 |
|----|------|--------|------|
| `ping` | - | `{op:"pong"}` | 接続確認 |
| `get_state` | - | `{op:"state", tempo, song_time_beats, sample_rate, selected_track_index, audio_track_count}` | テンポ・カーソル位置 |
| `insert_clips` | `{items:[InsertClipItem], track_index?:int, advance_cursor:bool}` | `{op:"inserted", count}` または `{op:"error", message}` | 連続挿入 |
| `subscribe` | `{topics:["tempo","song_time","selected_track"]}` | プッシュで `{op:"event", topic, value}` | 状態変化通知 |
| `unsubscribe` | `{topics:[...]}` | `{op:"ok"}` | 購読解除 |

`InsertClipItem`:

```json
{
  "file_path": "/abs/path/to/source.wav",
  "position_beats": 12.0,
  "length_beats": 4.0,
  "start_marker_beats": 0.5,
  "end_marker_beats": 4.5,
  "name": "クリップ名 (字幕テキスト先頭など)"
}
```

`track_index` 省略時は `selected_track`、それも無ければ末尾に新規オーディオトラックを作成する。

### 7.4 Python 側実装

`crates/remote_script/__init__.py`:

```python
from .manager import Manager
def create_instance(c_instance):
    return Manager(c_instance)
```

`crates/remote_script/server.py`:

```python
import json, socket, logging
logger = logging.getLogger("abletonsrt")

class JsonLineServer:
    def __init__(self, host="127.0.0.1", port=19823):
        self._sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self._sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self._sock.bind((host, port))
        self._sock.listen(4)
        self._sock.setblocking(False)
        self._clients = []  # [[sock, recv_buf]]
        logger.info("listening on %s:%d", host, port)

    def poll(self, on_message):
        try:
            client, _ = self._sock.accept()
            client.setblocking(False)
            self._clients.append([client, b""])
        except BlockingIOError:
            pass
        for entry in list(self._clients):
            client, buf = entry
            try:
                chunk = client.recv(8192)
                if not chunk:
                    self._clients.remove(entry); client.close(); continue
                buf += chunk
            except BlockingIOError:
                continue
            except OSError:
                self._clients.remove(entry); continue
            while b"\n" in buf:
                line, buf = buf.split(b"\n", 1)
                line = line.strip()
                if not line: continue
                try:
                    msg = json.loads(line.decode("utf-8"))
                except ValueError as e:
                    logger.warning("bad json: %s", e); continue
                reply = on_message(client, msg)
                if reply is not None:
                    self._send(client, reply)
            entry[1] = buf

    def push(self, payload):
        for entry in list(self._clients):
            self._send(entry[0], payload)

    def _send(self, client, payload):
        try:
            client.sendall((json.dumps(payload) + "\n").encode("utf-8"))
        except OSError:
            pass

    def close(self):
        for entry in self._clients:
            try: entry[0].close()
            except OSError: pass
        self._clients = []
        try: self._sock.close()
        except OSError: pass
```

`crates/remote_script/manager.py`:

```python
import logging, traceback
from _Framework.ControlSurface import ControlSurface
from .server import JsonLineServer

logger = logging.getLogger("abletonsrt")

class Manager(ControlSurface):
    def __init__(self, c_instance):
        super().__init__(c_instance)
        self._server = None
        self._subs = {}    # client -> set(topics)
        try:
            self._server = JsonLineServer()
            self.show_message("AbletonSRTBrowser ready")
        except Exception:
            logger.error("server init failed:\n%s", traceback.format_exc())
        self._install_listeners()
        self._schedule_tick()

    def _schedule_tick(self):
        self.schedule_message(1, self._tick)

    def _tick(self):
        if self._server is not None:
            try:
                self._server.poll(self._handle)
            except Exception:
                logger.error("tick:\n%s", traceback.format_exc())
        self._schedule_tick()

    def _handle(self, client, msg):
        op = msg.get("op")
        try:
            if op == "ping":
                return {"op": "pong"}
            if op == "get_state":
                return self._snapshot()
            if op == "insert_clips":
                return self._insert(msg)
            if op == "subscribe":
                self._subs.setdefault(client, set()).update(msg.get("topics", []))
                return {"op": "ok"}
            if op == "unsubscribe":
                if client in self._subs:
                    self._subs[client].difference_update(msg.get("topics", []))
                return {"op": "ok"}
            return {"op": "error", "message": "unknown op: %s" % op}
        except Exception:
            return {"op": "error", "message": traceback.format_exc()}

    def _snapshot(self):
        song = self.song()
        sel  = song.view.selected_track
        idx  = list(song.tracks).index(sel) if sel in song.tracks else -1
        n_audio = sum(1 for t in song.tracks if t.has_audio_input)
        return {
            "op": "state",
            "tempo": song.tempo,
            "song_time_beats": song.current_song_time,
            "sample_rate": song.song_length,           # 要確認 (sample rate 取得 API)
            "selected_track_index": idx,
            "audio_track_count": n_audio,
        }

    def _insert(self, msg):
        song = self.song()
        items = msg.get("items", [])
        track_index = msg.get("track_index")
        advance = bool(msg.get("advance_cursor", True))

        # トラック決定
        if track_index is not None and 0 <= track_index < len(song.tracks):
            track = song.tracks[track_index]
        else:
            track = song.view.selected_track
            if track is None or not track.has_audio_input:
                track = self._create_audio_track()

        if not track.has_audio_input:
            return {"op": "error", "message": "selected track is not audio"}

        count = 0
        last_end = song.current_song_time
        for item in items:
            pos = float(item["position_beats"])
            length = float(item["length_beats"])
            sm = float(item["start_marker_beats"])
            em = float(item["end_marker_beats"])
            clip = track.create_audio_clip(item["file_path"], pos)
            if clip is None:
                continue
            clip.warping = False
            clip.looping = False
            clip.start_marker = sm
            clip.end_marker   = em
            clip.name = item.get("name", "")
            count += 1
            last_end = pos + length
        if advance:
            song.current_song_time = last_end
        return {"op": "inserted", "count": count}

    def _create_audio_track(self):
        # Live 11+: create_audio_track(index)
        idx = len(self.song().tracks)
        self.song().create_audio_track(idx)
        return self.song().tracks[idx]

    def _install_listeners(self):
        song = self.song()
        song.add_tempo_listener(self._on_tempo)
        song.add_current_song_time_listener(self._on_song_time)
        song.view.add_selected_track_listener(self._on_selected_track)

    def _on_tempo(self):
        self._push("tempo", self.song().tempo)
    def _on_song_time(self):
        self._push("song_time", self.song().current_song_time)
    def _on_selected_track(self):
        self._push("selected_track", self._snapshot()["selected_track_index"])

    def _push(self, topic, value):
        if self._server is None: return
        msg = {"op": "event", "topic": topic, "value": value}
        for client, topics in list(self._subs.items()):
            if topic in topics:
                self._server._send(client, msg)

    def disconnect(self):
        if self._server is not None:
            self._server.close()
            self._server = None
        super().disconnect()
```

### 7.5 Rust 側クライアント

```rust
// src/insertion/remote.rs

use std::io::{BufRead, BufReader, Write};
use std::net::{SocketAddr, TcpStream};
use std::sync::{Arc, Mutex};
use std::time::Duration;
use serde::{Deserialize, Serialize};
use anyhow::{Context, Result};
use crate::insertion::{Inserter, InsertionRequest, InsertionTrigger, InsertionOutcome};

#[derive(Serialize)]
#[serde(tag = "op")]
enum OutMsg<'a> {
    #[serde(rename = "ping")]
    Ping,
    #[serde(rename = "get_state")]
    GetState,
    #[serde(rename = "insert_clips")]
    InsertClips {
        items: Vec<InsertItem<'a>>,
        track_index: Option<u32>,
        advance_cursor: bool,
    },
    #[serde(rename = "subscribe")]
    Subscribe { topics: &'a [&'a str] },
}

#[derive(Serialize)]
struct InsertItem<'a> {
    file_path: &'a str,
    position_beats: f64,
    length_beats: f64,
    start_marker_beats: f64,
    end_marker_beats: f64,
    name: &'a str,
}

#[derive(Deserialize, Debug)]
#[serde(tag = "op")]
enum InMsg {
    #[serde(rename = "pong")]    Pong,
    #[serde(rename = "ok")]      Ok,
    #[serde(rename = "state")]   State {
        tempo: f64,
        song_time_beats: f64,
        selected_track_index: i32,
        audio_track_count: u32,
    },
    #[serde(rename = "inserted")] Inserted { count: usize },
    #[serde(rename = "event")]    Event { topic: String, value: serde_json::Value },
    #[serde(rename = "error")]    Error { message: String },
}

pub struct RemoteScriptInserter {
    inner: Arc<Mutex<RemoteInner>>,
    addr: SocketAddr,
}

struct RemoteInner {
    stream: Option<TcpStream>,
    last_tempo: f64,
    last_cursor_beats: f64,
    available: bool,
}

impl RemoteScriptInserter {
    pub fn new(addr: SocketAddr) -> Self {
        Self {
            inner: Arc::new(Mutex::new(RemoteInner {
                stream: None,
                last_tempo: 120.0,
                last_cursor_beats: 0.0,
                available: false,
            })),
            addr,
        }
    }

    /// 起動直後に呼んで接続を試みる。失敗しても OK (バックグラウンドで再試行)。
    pub fn connect(&self) -> Result<()> {
        let mut g = self.inner.lock().unwrap();
        let s = TcpStream::connect_timeout(&self.addr, Duration::from_millis(500))?;
        s.set_read_timeout(Some(Duration::from_millis(2000)))?;
        s.set_write_timeout(Some(Duration::from_millis(2000)))?;
        g.stream = Some(s);
        g.available = true;
        drop(g);
        // subscribe しておく
        let _ = self.send::<()>(&OutMsg::Subscribe {
            topics: &["tempo", "song_time", "selected_track"],
        });
        Ok(())
    }

    /// メインループから周期的に (例: 200ms 毎) 呼んで、event をドレインしつつ
    /// last_tempo / last_cursor_beats を更新する。
    pub fn pump(&self) {
        let mut g = self.inner.lock().unwrap();
        let Some(stream) = g.stream.as_mut() else { return };
        let mut reader = BufReader::new(stream.try_clone().unwrap());
        // ノンブロッキングで 1 行読む試み (read_timeout 短く)
        stream.set_read_timeout(Some(Duration::from_millis(1))).ok();
        let mut buf = String::new();
        match reader.read_line(&mut buf) {
            Ok(0) => { g.stream = None; g.available = false; }
            Ok(_) => {
                if let Ok(msg) = serde_json::from_str::<InMsg>(buf.trim()) {
                    match msg {
                        InMsg::Event { topic, value } => match topic.as_str() {
                            "tempo" => if let Some(v) = value.as_f64() { g.last_tempo = v; },
                            "song_time" => if let Some(v) = value.as_f64() { g.last_cursor_beats = v; },
                            _ => {}
                        },
                        _ => {}
                    }
                }
            }
            Err(_) => {} // タイムアウトは正常
        }
        stream.set_read_timeout(Some(Duration::from_millis(2000))).ok();
    }

    pub fn last_tempo(&self) -> f64 { self.inner.lock().unwrap().last_tempo }
    pub fn last_cursor_beats(&self) -> f64 { self.inner.lock().unwrap().last_cursor_beats }

    fn send<R: for<'de> Deserialize<'de>>(&self, msg: &OutMsg) -> Result<R> {
        let mut g = self.inner.lock().unwrap();
        let stream = g.stream.as_mut().context("not connected")?;
        let line = serde_json::to_string(msg)? + "\n";
        stream.write_all(line.as_bytes())?;
        stream.flush()?;
        let mut reader = BufReader::new(stream.try_clone()?);
        loop {
            let mut buf = String::new();
            reader.read_line(&mut buf)?;
            let v: serde_json::Value = serde_json::from_str(buf.trim())?;
            // event はスキップして次へ
            if v.get("op").and_then(|o| o.as_str()) == Some("event") {
                continue;
            }
            return Ok(serde_json::from_value(v)?);
        }
    }
}

impl Inserter for RemoteScriptInserter {
    fn supports(&self, t: InsertionTrigger) -> bool {
        matches!(t, InsertionTrigger::Keyboard)
    }
    fn is_available(&self) -> bool {
        self.inner.lock().unwrap().available
    }
    fn status_label(&self) -> &'static str {
        if self.is_available() { "remote (connected)" } else { "remote (disconnected)" }
    }

    fn insert(&mut self, req: InsertionRequest, _t: InsertionTrigger)
        -> Result<InsertionOutcome>
    {
        let cursor = self.last_cursor_beats();
        let beats_per_sec = req.tempo_bpm / 60.0;
        let mut pos = cursor;
        let mut items_out = Vec::with_capacity(req.items.len());
        let abs_paths: Vec<String> = req.items.iter()
            .map(|i| i.source_audio.canonicalize().unwrap_or_else(|_| i.source_audio.clone())
                       .to_string_lossy().to_string())
            .collect();
        for (i, item) in req.items.iter().enumerate() {
            let length_b = (item.end_sec - item.start_sec) * beats_per_sec;
            let sm = item.start_sec * beats_per_sec;
            let em = item.end_sec   * beats_per_sec;
            items_out.push(InsertItem {
                file_path: &abs_paths[i],
                position_beats: pos,
                length_beats: length_b,
                start_marker_beats: sm,
                end_marker_beats: em,
                name: &item.display_name,
            });
            pos += length_b;
        }
        let msg = OutMsg::InsertClips {
            items: items_out,
            track_index: None,
            advance_cursor: true,
        };
        let reply: InMsg = self.send(&msg)?;
        match reply {
            InMsg::Inserted { count } => Ok(InsertionOutcome::Inserted { count }),
            InMsg::Error { message } => anyhow::bail!(message),
            other => anyhow::bail!("unexpected reply: {:?}", other),
        }
    }
}
```

### 7.6 接続管理

- 起動時に `connect()` を試行し、失敗してもアプリは普通に立ち上がる
- バックグラウンドで 5 秒ごとに再接続を試みる (`AppState::tick` から呼ぶ)
- ステータスバーに「Live: connected / disconnected」を表示
- 再接続成功時に `subscribe` を再送

### 7.7 セキュリティ・運用上の注意

- ループバックのみ listen (外部からの接続は拒否)
- ポートは固定 (19823)。衝突する環境では設定で変更可能にする
- メッセージサイズ上限を Rust 側でチェック (例: 1MB)。XML や巨大ペイロードを送ることは想定しない

## 8. データモデル

### 8.1 SRT パース

SRT は以下の形式 (BOM 許容、CRLF/LF 両方許容):

```
1
00:00:00,000 --> 00:00:02,500
（話者A）こんにちは

2
00:00:02,500 --> 00:00:05,000
こんばんは
```

```rust
// src/model/srt.rs

#[derive(Clone, Debug)]
pub struct SrtItem {
    pub index: u32,
    pub start_ms: i64,
    pub end_ms: i64,
    pub text: String,            // 改行を含む生テキスト
    pub speaker: Option<String>, // 抽出された話者ラベル
    pub display_text: String,    // 話者ラベルを除去したもの (移植元の hide_speaker_labels 用)
}

pub fn parse_srt(input: &str) -> anyhow::Result<Vec<SrtItem>> { /* ... */ }
```

実装ルール:

- BOM (`\u{FEFF}`) を除去
- 数値→ `-->` → テキスト → 空行で 1 ブロック
- タイムスタンプ: `HH:MM:SS,mmm` (カンマ or ピリオド両対応)
- インデックスがファイル中で連続していなくても OK (順序のみ保つ)
- 不正フォーマットの行はスキップしてエラーを蓄積、ベストエフォートで返す

### 8.2 話者ラベル抽出

移植元 `ReaSRTBrowser.lua:2572-2588` のロジックを Rust に移植する:

```rust
// src/model/speaker.rs
use regex::Regex;
use once_cell::sync::Lazy;

static SPEAKER: Lazy<Regex> = Lazy::new(|| {
    // 全角括弧 () または半角括弧 () で始まり、本文との間に空白許容
    Regex::new(r"^[\s　]*[(\u{FF08}]([^)\u{FF09}\n]+)[)\u{FF09}][\s　]*").unwrap()
});

pub fn split_speaker(text: &str) -> (Option<String>, String) {
    if let Some(c) = SPEAKER.captures(text) {
        let speaker = c[1].trim().to_string();
        let rest = SPEAKER.replace(text, "").to_string();
        return (Some(speaker), rest);
    }
    (None, text.to_string())
}
```

`SrtItem::speaker` / `display_text` はパース時にこれを呼んで埋める。

### 8.3 自動オーディオマッチング

移植元 `ReaSRTBrowser.lua:2925-2979` のロジック:

```rust
// src/model/audio_match.rs

use std::path::{Path, PathBuf};

pub fn auto_match_audio(srt_path: &Path) -> Option<PathBuf> {
    let dir = srt_path.parent()?;
    let stem = srt_path.file_stem()?.to_string_lossy().to_lowercase();
    let mut exact: Option<PathBuf> = None;
    let mut partial: Vec<PathBuf> = Vec::new();
    for entry in std::fs::read_dir(dir).ok()?.flatten() {
        let p = entry.path();
        if p.extension().and_then(|e| e.to_str()).map(|s| s.eq_ignore_ascii_case("wav")) != Some(true) {
            continue;
        }
        let s = p.file_stem()?.to_string_lossy().to_lowercase();
        if s == stem {
            exact = Some(p); break;
        }
        if s.contains(&stem) || stem.contains(&s) {
            partial.push(p);
        }
    }
    if let Some(e) = exact { return Some(e); }
    // 最短ファイル名 → 大文字小文字を区別しない比較で安定ソート
    partial.sort_by_key(|p| (p.file_name().unwrap().len(),
                             p.to_string_lossy().to_lowercase()));
    partial.into_iter().next()
}
```

### 8.4 SrtSource (1 SRT 単位)

```rust
// src/model/source.rs

#[derive(Clone, Debug)]
pub struct SrtSource {
    pub srt_path: PathBuf,
    pub items: Vec<SrtItem>,
    pub audio: Option<PathBuf>,
    pub global_offset_ms: i64,
    pub item_meta: Vec<ItemMeta>,    // items とインデックス対応
    pub metadata_path: PathBuf,      // メタデータ JSON のパス
    pub dirty: bool,
}

#[derive(Clone, Debug, Default)]
pub struct ItemMeta {
    pub favorite: bool,
    pub tags: Vec<String>,
    pub note: String,
}

impl SrtSource {
    pub fn load(srt_path: &Path, settings_dir: &Path) -> anyhow::Result<Self> {
        let raw = std::fs::read_to_string(srt_path)?;
        let items = crate::model::srt::parse_srt(&raw)?;
        let audio = auto_match_audio(srt_path);
        let metadata_path = settings_dir.join("metadata")
            .join(format!("{}.json", path_hash(srt_path)));
        let stored = SrtMetadata::load_or_default(&metadata_path);
        let mut item_meta = vec![ItemMeta::default(); items.len()];
        // メタデータをアイテムにマージ (key=(index,start,end,text))
        merge_meta(&items, &stored, &mut item_meta);
        Ok(Self {
            srt_path: srt_path.to_path_buf(),
            items,
            audio: stored.primary_audio().or(audio),
            global_offset_ms: stored.global_offset_ms,
            item_meta,
            metadata_path,
            dirty: false,
        })
    }

    pub fn effective_bounds_ms(&self, idx: usize) -> (i64, i64) {
        let it = &self.items[idx];
        let off = self.global_offset_ms;
        ((it.start_ms + off).max(0), (it.end_ms + off).max(0))
    }
}
```

`path_hash` は移植元の DJB2 でも良いが、Rust では SHA-256 の prefix 16 文字を採用 (衝突安全):

```rust
pub fn path_hash(p: &Path) -> String {
    use sha2::{Digest, Sha256};
    let mut hasher = Sha256::new();
    hasher.update(p.to_string_lossy().as_bytes());
    let h = hasher.finalize();
    let s = h.iter().take(8).map(|b| format!("{b:02x}")).collect::<String>();
    s
}
```

### 8.5 メタデータ JSON スキーマ

`<config_dir>/AbletonSRTBrowser/metadata/<hash>.json`:

```json
{
  "version": 1,
  "source": {
    "srt_path": "/abs/path/to/file.srt",
    "srt_filename": "file.srt"
  },
  "audio_files": [
    {
      "path": "/abs/path/to/file.wav",
      "label": "primary",
      "is_primary": true
    }
  ],
  "global_offset_ms": 0,
  "items": [
    {
      "key": {
        "srt_index": 1,
        "start_ms": 0,
        "end_ms": 2500,
        "text": "（話者A）こんにちは"
      },
      "favorite": false,
      "tags": ["greeting", "A"],
      "note": ""
    }
  ]
}
```

`key` は `(srt_index, start_ms, end_ms, text)` の 4 値で、SRT を再パースしても同じアイテムを特定できるようにする (移植元と同じ方針)。

### 8.6 ライブラリ

```rust
// src/model/library.rs

#[derive(Clone, Debug)]
pub struct Library {
    pub id: String,                  // "lib_<hash>"
    pub name: String,
    pub source_paths: Vec<PathBuf>,  // SRT パスの一覧
    pub folders: Vec<LibFolder>,     // ライブラリ内のフォルダ階層
    pub source_folder: HashMap<PathBuf, String>, // SRT → folder_id
}

#[derive(Clone, Debug)]
pub struct LibFolder {
    pub id: String,                  // "fld_<hash>"
    pub name: String,
    pub parent_id: Option<String>,   // None = ルート
}
```

ライブラリ JSON: `<config_dir>/AbletonSRTBrowser/libraries.json`:

```json
{
  "version": 1,
  "libraries": [
    {
      "id": "lib_8a3f12",
      "name": "Project A",
      "sources": ["/path/file1.srt", "/path/file2.srt"],
      "folders": [
        {"id": "fld_001", "name": "Scene 1", "parent_id": null}
      ],
      "source_folder": {
        "/path/file1.srt": "fld_001"
      }
    }
  ]
}
```

### 8.7 設定 JSON

`<config_dir>/AbletonSRTBrowser/settings.json`:

```json
{
  "version": 1,
  "language": "ja",
  "last_opened_srt_path": "/path/last.srt",
  "last_srt_browse_dir": "/path",
  "last_audio_browse_dir": "/path",
  "hide_speaker_labels": false,
  "preview_volume": 90,
  "font_path": null,
  "font_size": 14,
  "show_detail_pane": true,
  "startup_content_mode": "source",
  "startup_library_id": null,
  "left_panel_tab": "sources",
  "left_pane_width": 400.0,
  "detail_pane_height": 130.0,
  "item_table_column_widths": {
    "source":  {"text": 500.0},
    "library": {"srt": 200.0}
  },
  "recent_sources": ["/path/recent1.srt"],
  "remote_script": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 19823
  },
  "fallback_keyboard_to_alc": true
}
```

### 8.8 フィルター仕様

```rust
// src/model/filter.rs

#[derive(Clone, Default, Debug)]
pub struct Filter {
    pub query: String,           // テキスト検索 (Unicode 正規化 + lowercase で部分一致)
    pub include_tags: Vec<String>,
    pub exclude_tags: Vec<String>,
    pub favorites_only: bool,
}

impl Filter {
    pub fn parse(input: &str) -> Self { /* スペース区切り、`-` 接頭辞は除外 */ }
    pub fn matches(&self, item: &SrtItem, meta: &ItemMeta, source_name: &str) -> bool {
        if self.favorites_only && !meta.favorite { return false; }
        for t in &self.include_tags {
            if !meta.tags.iter().any(|x| x.eq_ignore_ascii_case(t)) { return false; }
        }
        for t in &self.exclude_tags {
            if meta.tags.iter().any(|x| x.eq_ignore_ascii_case(t)) { return false; }
        }
        if !self.query.is_empty() {
            let blob = format!("{} {} {} {} {}",
                source_name, item.text, meta.note,
                meta.tags.join(" "),
                if meta.favorite { "favorite" } else { "" });
            if !contains_ci(&blob, &self.query) { return false; }
        }
        true
    }
}
```

`contains_ci` は Unicode セーフな大文字小文字無視部分一致。`unicode-segmentation` で正規化する。

## 9. GUI 機能仕様

移植元の機能を **網羅的に** 再現します。各項目は移植元のどの動作に対応するかを明記。

### 9.1 ウィンドウレイアウト

```
┌──────────────────────────────────────────────────────┐
│ Menu Bar (File / Edit / View / Settings)             │
├──────────────────────────────────────────────────────┤
│ Top toolbar (status / "Open Audio..." 等)             │
├────────────┬─────────────────────────────────────────┤
│ Left Pane  │ Item Table (中央: アイテム一覧)         │
│ ┌────────┐ │  ┌────────┬────────────┬─────┬─────┬...│
│ │Sources │ │  │ Index  │ Text       │Start│ End │   │
│ │Library │ │  │   1    │ こんにちは │00:00│00:02│   │
│ └────────┘ │  │   2    │ こんばんは │00:02│00:05│   │
│ <list>     │  └────────┴────────────┴─────┴─────┴───┤
│            ├─────────────────────────────────────────┤
│            │ Detail Pane (下: 編集)                  │
│            │ [☆ favorite] tags: ___ note: ___        │
│            │ Global Offset: [____ ms] [Apply][Reset] │
└────────────┴─────────────────────────────────────────┘
```

#### サイズ既定値 (移植元 `ui.lua:21-30`)

| 項目 | 値 | 調整 |
|------|------|------|
| 左ペイン幅 | 400px | スプリッタでドラッグ可、永続化 |
| 詳細ペイン高さ | 130px | スプリッタでドラッグ可、永続化 |
| 詳細パネル最大高さ | 160px | 固定 |
| スプリッタ幅 | 6px | 固定 |
| 左ペイン最小幅 | 120px | 固定 |
| メインペイン最小幅 | 420px | 固定 |
| アイテムリスト最小高さ | 180px | 固定 |
| 詳細ペイン最小高さ | 100px | 固定 |

### 9.2 メニューバー

#### File メニュー

| 項目 | 動作 | 有効条件 |
|------|------|---------|
| Add SRT... | ファイルピッカー (`.srt`、複数選択可)。選択した SRT を Sources / 開いているライブラリに追加 | 常時 |
| New Folder | ライブラリ用の新規フォルダ作成 (名前入力ダイアログ) | Library モード |
| New Library | 新規ライブラリ作成 (名前入力ダイアログ) | 常時 |
| Open Audio... | 現在の SRT に紐付ける音声ファイルピッカー (`.wav`) | SRT 読込済 |
| Reload Library | ライブラリ再読込 (ディスクから再構築) | Library モード |
| --- | (separator) | |
| Save Metadata | メタデータを即時 flush | Source モード |
| Clear SRT | 現在の SRT をクローズ | 常時 |

#### Edit メニュー

| 項目 | 動作 | 有効条件 |
|------|------|---------|
| Insert Selected Item(s) on Timeline | 選択アイテムを Live のカーソル位置に挿入 (= Enter と同じ) | アイテム選択中 |
| Preview Selected Item(s) | 選択アイテムをプレビュー (= Space と同じ) | アイテム選択中 |
| Mark Selected Item as Favorite | お気に入りトグル | アイテム選択中 |
| Edit Selected Item Tags... | タグ編集ダイアログ | アイテム選択中 |
| --- | | |
| Add Speaker Tags | 話者タグを自動抽出してすべてのアイテムに付与 | Source モード |
| --- | | |
| Apply Offset | オフセット入力欄の値を確定 | Source モード |
| Reset Offset | オフセットを 0 に戻す | Source モード |

#### View メニュー

| 項目 | 動作 |
|------|------|
| Show/Hide Edit Panel | 詳細ペインの表示/非表示トグル |

#### Settings メニュー

| 項目 | 動作 |
|------|------|
| Preview Volume... | 数値入力ダイアログ (0–100) |
| Font Size... | 数値入力ダイアログ (1–255) |
| Font File... | フォントファイルパス入力 |
| Language ▶ English / Japanese | UI 言語切替 |
| Remote Script... | 接続設定ダイアログ (host/port、enable トグル、接続テスト) |

### 9.3 左ペイン: Sources タブ

- 現在登録済みの SRT ファイルを一覧表示 (移植元 `source_pane.lua`)
- 各エントリ: ファイル名 + 親フォルダの省略表示 + お気に入り数 + tag 数 等
- ダブルクリックで開く (= 現在の SRT を切り替え)
- 右クリックメニュー:
  - Open
  - Move to Folder ▶
  - Remove from List
  - Reveal in File Manager
- ドラッグでの並び替え (`egui_dnd` 利用)
- フォルダはツリー (展開/折りたたみ)

### 9.4 左ペイン: Library タブ

- ライブラリの一覧表示
- ダブルクリックでライブラリを開く (中央ペインがライブラリ横断ビューに切り替わる)
- 右クリックメニュー:
  - Open
  - Rename
  - Delete
  - New Subfolder
  - Add Selected SRTs Here
  - Expand All / Collapse All
- ライブラリ内のソースもドラッグで並び替え / フォルダ移動

### 9.5 中央: アイテムテーブル

`egui_extras::TableBuilder` を使用。**virtual scroll を必ず有効化**(行数が万単位になりうる)。

#### Source モードの列 (移植元 `ui.lua:101-120`)

| 列 ID | ラベル | 幅 | ストレッチ | 並び替え |
|-------|--------|-----|-----------|---------|
| `index` | # | 60px | × | ○ |
| `text` | Text | 50% | ○ | ○ (text 字面順) |
| `start` | Start | 90px | × | ○ |
| `end` | End | 90px | × | ○ |
| `favorite` | ★ | 45px | × | ○ |
| `tags` | Tags | 残り | ○ | ○ |

#### Library モードの列

| 列 ID | ラベル | 幅 |
|-------|--------|-----|
| `srt` | SRT | 160px |
| `index` | # | 60px |
| `text` | Text | 50% |
| `start` | Start | 90px |
| `end` | End | 90px |
| `favorite` | ★ | 45px |
| `tags` | Tags | 残り |

#### 表示・操作仕様

- ヘッダクリック: 昇順 / 降順トグル (現在ソート列なら方向反転)
- セレクト: 行クリックで単選択、Shift+クリックで範囲、Ctrl/Cmd+クリックで個別追加
- ↑↓: 単選択を 1 つ移動 (Shift 付きで範囲拡張)
- ダブルクリック → 挿入 (Inserter::insert with Keyboard)
- 右クリックメニュー: Insert / Preview / Toggle Favorite / Edit Tags / Reveal Source File
- 行を Primary ボタンでドラッグ開始 → drag-out (Inserter::insert with DragStart)
- 列幅はドラッグでリサイズ。永続化
- 「話者ラベルを隠す」ON 時は `display_text` を表示

#### 検索ボックス

- ヘッダ上部に `egui::TextEdit::singleline` (placeholder: "Filter text...")
- **debounce なし** (移植元と同じ、即時反映)
- 隣に「お気に入りのみ」「話者ラベルを隠す」トグルボタン

### 9.6 下: 詳細ペイン (Edit Panel)

選択中アイテムが 1 件のときに編集 UI、複数のときは「N items selected」のサマリ + 一括操作ボタン:

- ★ お気に入りトグル
- Tags: テキストフィールド (`InputTextWithHint` 相当)
  - カンマ区切り入力
  - 入力 → トリム → 連続スペース 1 つに → 重複除去 → ItemMeta.tags に保存
  - 単選択時はその場で反映、複数選択時は「Apply to all」ボタン
- Note: 複数行テキストエリア (単選択時のみ)
- Global Offset (Source モード):
  - 数値 (ms 単位、整数)
  - [Apply] / [Reset]
  - 値変更でプレビュー停止、display_start/end 再計算、フィルター無効化、メタデータ dirty
- Audio info: 現在の音声ファイル名、Missing 表示

### 9.7 話者タグ自動抽出 (Edit メニュー)

ロジック:

1. SRT 全アイテムをスキャン
2. `(話者)` または `（話者）` パターンを検出
3. 既存タグに同名 (大文字小文字無視) があればスキップ
4. なければ tags に追加 + dirty + フィルタキャッシュ無効化
5. 結果を toast:
   - 成功: "Added speaker tags to N items."
   - 既に全部入ってる: "Speaker tags were already present."
   - パターン無し: "No speaker markers were found at the start of subtitle text."

### 9.8 ステータストースト

移植元の `status_msg` 相当。画面下端に短時間表示。

- info / success / warning / error の 4 レベル
- 同時表示は最大 3 件、自動消滅 (3 秒)
- クリックで即消去

### 9.9 ファイルダイアログ

- すべて `rfd::FileDialog` 使用 (OS ネイティブ、複数選択対応)
- フィルタは `.srt` / `.wav` を指定
- 直近のディレクトリを設定に保存 (`last_srt_browse_dir`, `last_audio_browse_dir`)

### 9.10 ファイルドロップ受け入れ (アプリへの drop)

`egui::Context::input(|i| i.raw.dropped_files.clone())` を毎フレーム監視:

- `.srt` ドロップ → Sources / 現在のライブラリに追加
- `.wav` ドロップ → 現在の SRT に音声を関連付け
- フォルダドロップ → 中の `.srt` を再帰追加

## 10. オーディオプレビュー

### 10.1 要件

- Space キー / メニューでプレビュー再生・停止 (移植元 `ReaSRTBrowser.lua:2234-2299`)
- 音源ファイルの **指定範囲** のみ再生 (start_sec, end_sec)
- 音量 0–100 (内部は 0.0–1.0 へ変換)
- 再生開始・終了の **5ms フェード**(クリック軽減)
- アイテム切り替え時は前の再生を即停止
- 範囲が複数ある場合 (複数選択) は順次連続再生

### 10.2 設計

```rust
// src/audio/player.rs

use std::path::Path;
use std::sync::{Arc, atomic::{AtomicBool, Ordering}};
use std::time::Duration;
use anyhow::Result;
use rodio::{Decoder, OutputStream, Sink, Source};

pub struct PreviewPlayer {
    _stream: OutputStream,
    sink: Sink,
    volume: f32,
    stop_flag: Arc<AtomicBool>,
}

impl PreviewPlayer {
    pub fn new(volume_pct: u8) -> Result<Self> {
        let (stream, handle) = OutputStream::try_default()?;
        let sink = Sink::try_new(&handle)?;
        let volume = (volume_pct.min(100) as f32) / 100.0;
        sink.set_volume(volume);
        Ok(Self { _stream: stream, sink, volume, stop_flag: Arc::new(AtomicBool::new(false)) })
    }

    pub fn set_volume(&mut self, pct: u8) {
        self.volume = (pct.min(100) as f32) / 100.0;
        self.sink.set_volume(self.volume);
    }

    pub fn stop(&self) {
        self.stop_flag.store(true, Ordering::SeqCst);
        self.sink.clear();
    }

    pub fn play_range(&self, path: &Path, start: Duration, end: Duration) -> Result<()> {
        self.stop();
        self.stop_flag.store(false, Ordering::SeqCst);
        let file = std::fs::File::open(path)?;
        let dec = Decoder::new(std::io::BufReader::new(file))?;
        let length = end.saturating_sub(start);
        let fade = Duration::from_millis(5);
        let src = dec
            .skip_duration(start)
            .take_duration(length)
            .fade_in(fade);
        // rodio に fade_out が直接ないので、`take_duration` の終端で
        // 別途フェードアウトを混ぜるなら manual に Source を実装する。
        // 簡略化: fade_in だけ、終端は take_duration が即停止 → 必要なら自作 source。
        self.sink.append(src);
        self.sink.play();
        Ok(())
    }

    pub fn play_ranges(&self, ranges: Vec<(std::path::PathBuf, Duration, Duration)>) -> Result<()> {
        self.stop();
        self.stop_flag.store(false, Ordering::SeqCst);
        for (p, s, e) in ranges {
            let file = std::fs::File::open(&p)?;
            let dec = Decoder::new(std::io::BufReader::new(file))?;
            let length = e.saturating_sub(s);
            let src = dec
                .skip_duration(s)
                .take_duration(length)
                .fade_in(Duration::from_millis(5));
            self.sink.append(src);
        }
        self.sink.play();
        Ok(())
    }
}
```

### 10.3 完全なフェードアウトを実現したい場合

`take_duration` だけでは末尾でフェードしないので、必要に応じて以下のアダプタを自作する:

```rust
// 末尾 N ms で線形にゲインを 1.0 → 0.0 にする source ラッパ
pub struct FadeOut<S> {
    inner: S,
    total: Duration,
    fade: Duration,
    elapsed: Duration,
}
// Source impl はサンプルごとに elapsed を更新し、
// (total - elapsed) < fade の領域でゲインを掛ける。
```

最初の MVP では `fade_in` のみで十分。フェードアウトは Phase 2 で。

### 10.4 GUI からの呼び出し

```rust
// src/ui/shortcuts.rs
fn handle_space(app: &mut AppState, ctx: &egui::Context) {
    let space = ctx.input(|i| i.key_pressed(egui::Key::Space));
    if !space { return; }
    if app.is_text_input_focused() { return; }
    let ranges = app.build_preview_ranges();   // 選択順に (path, start, end)
    if ranges.is_empty() {
        app.toast.info("No items to preview.");
        return;
    }
    if let Err(e) = app.player.play_ranges(ranges) {
        app.toast.error(format!("Preview failed: {e}"));
    }
}
```

### 10.5 注意点

- `OutputStream` を毎回作り直すと OS によっては数百 ms のオーバーヘッドあり。アプリ寿命と一致する `PreviewPlayer` を `AppState` 内で 1 つ保持する。
- 巨大 wav (>1GB) の `BufReader<File>` でも `skip_duration` が効くが、ファイル先頭からのサンプル数で seek するため O(N)。**頭出しの待ち時間が気になるなら `symphonia` を直接使って seek_table を引く**。MVP では `rodio` 標準でよい。
- `.mp3` 等にも対応するなら `rodio` の features に `mp3` を入れる (本仕様では `.wav` 前提)。

## 11. 永続化

### 11.1 保存先 (`dirs` クレート)

| OS | パス |
|----|------|
| macOS | `~/Library/Application Support/AbletonSRTBrowser/` |
| Windows | `%APPDATA%\AbletonSRTBrowser\` |
| Linux | `~/.config/AbletonSRTBrowser/` |

ディレクトリ構造:

```
AbletonSRTBrowser/
├── settings.json
├── libraries.json
└── metadata/
    ├── 8a3f12b9.json
    ├── d4e1c0a7.json
    └── ...
```

### 11.2 書き込みタイミング (debounce)

移植元と同じ debounce 方式を採用:

| 対象 | debounce | フラッシュ契機 |
|------|---------|---------------|
| `settings.json` | 500ms | 値変更後 / アプリ終了 |
| `metadata/*.json` (タグ・お気に入り・ノート) | 1500ms | 編集後 / SRT クローズ / アプリ終了 |
| `metadata/*.json` (Global Offset) | 0ms | Apply ボタン押下時に即時 |
| `libraries.json` | 500ms | 変更後 / アプリ終了 |

```rust
// src/util/debounce.rs

use std::time::{Duration, Instant};

pub struct Debounced {
    delay: Duration,
    last_dirty_at: Option<Instant>,
}

impl Debounced {
    pub fn new(delay: Duration) -> Self { Self { delay, last_dirty_at: None } }
    pub fn mark_dirty(&mut self) { self.last_dirty_at = Some(Instant::now()); }
    pub fn should_flush(&mut self) -> bool {
        if let Some(t) = self.last_dirty_at {
            if t.elapsed() >= self.delay { self.last_dirty_at = None; return true; }
        }
        false
    }
    pub fn force(&mut self) -> bool {
        if self.last_dirty_at.is_some() { self.last_dirty_at = None; true } else { false }
    }
}
```

`AppState::tick` で毎フレーム `should_flush()` をチェックして書き出す。

### 11.3 アトミック書き込み

JSON ファイルは「同名 `.tmp` に書いて rename」で常にアトミックに置き換える:

```rust
pub fn atomic_write_json<T: serde::Serialize>(path: &Path, data: &T) -> Result<()> {
    let tmp = path.with_extension("json.tmp");
    let mut f = std::fs::File::create(&tmp)?;
    serde_json::to_writer_pretty(&mut f, data)?;
    f.sync_all()?;
    std::fs::rename(tmp, path)?;
    Ok(())
}
```

書き込み中にプロセス kill されても破損ファイルが残らない。

### 11.4 マイグレーション

スキーマには `version: 1` を必ず入れる。読み込み時:

```rust
#[derive(Deserialize)]
struct Versioned { version: u32 }

let header: Versioned = serde_json::from_str(&raw)?;
match header.version {
    1 => { /* 現行 */ }
    n => anyhow::bail!("unsupported version: {n}"),
}
```

将来スキーマを変えるとき、v1 → v2 の変換関数を `migrate_v1_to_v2()` として用意する。

### 11.5 ファイル変更検知 (任意)

`notify` クレートで `metadata/` を監視し、外部編集 (Git pull など) を反映する。MVP では不要。

## 12. 国際化 (i18n)

### 12.1 設計

- 言語データは Rust の **コンパイル時定数** で持つ (`HashMap<&'static str, &'static str>`)
- 動的ロードはしない (バイナリ単体で完結)
- 設定の `language` フィールドで切替

```rust
// src/i18n/mod.rs

use std::collections::HashMap;
use once_cell::sync::Lazy;

#[derive(Copy, Clone, Debug, PartialEq, Eq)]
pub enum Lang { En, Ja }

pub struct Catalog {
    pub strings: HashMap<&'static str, &'static str>,
    pub fallback: Option<&'static Catalog>,
}

static EN: Lazy<Catalog> = Lazy::new(|| Catalog {
    strings: crate::i18n::en::STRINGS.iter().cloned().collect(),
    fallback: None,
});
static JA: Lazy<Catalog> = Lazy::new(|| Catalog {
    strings: crate::i18n::ja::STRINGS.iter().cloned().collect(),
    fallback: Some(&EN),
});

pub fn t(lang: Lang, key: &str) -> String {
    let cat = match lang { Lang::En => &*EN, Lang::Ja => &*JA };
    let mut cur = Some(cat);
    while let Some(c) = cur {
        if let Some(v) = c.strings.get(key) { return (*v).to_string(); }
        cur = c.fallback;
    }
    key.to_string()  // フォールバック: キー名そのまま
}
```

`t` はマクロ化して `t!("menu.file.add_srt")` のように使うとシンプル:

```rust
#[macro_export]
macro_rules! t {
    ($key:literal) => { crate::i18n::t(crate::i18n::current_lang(), $key) };
}
```

### 12.2 キー命名規則 (移植元の構造を継承)

| プレフィクス | 用途 | 例 |
|--------------|------|----|
| `error.*` | エラーメッセージ | `error.audio_missing` |
| `status.*` | トーストメッセージ | `status.tags_added` |
| `menu.*` | メニュー項目ラベル | `menu.file.add_srt` |
| `button.*` | ボタンラベル | `button.apply` |
| `tab.*` | タブ名 | `tab.sources` |
| `pane.*` | ペインタイトル | `pane.detail` |
| `label.*` | UI ラベル | `label.global_offset` |
| `hint.*` | プレースホルダ | `hint.filter_text` |
| `toggle.*` | トグルラベル | `toggle.favorites_only` |
| `empty.*` | 空状態メッセージ | `empty.no_items` |
| `item.column.*` | テーブル列見出し | `item.column.text` |
| `prompt.*` | ダイアログタイトル / フィールド名 | `prompt.set_volume.title` |

### 12.3 必要キーの最小一覧 (抜粋)

`src/i18n/en.rs`:

```rust
pub const STRINGS: &[(&str, &str)] = &[
    // window
    ("window.title", "AbletonSRTBrowser"),

    // menu
    ("menu.file",                     "File"),
    ("menu.file.add_srt",             "Add SRT..."),
    ("menu.file.new_folder",          "New Folder"),
    ("menu.file.new_library",         "New Library"),
    ("menu.file.open_audio",          "Open Audio..."),
    ("menu.file.reload_library",      "Reload Library"),
    ("menu.file.save_metadata",       "Save Metadata"),
    ("menu.file.clear_srt",           "Clear SRT"),

    ("menu.edit",                     "Edit"),
    ("menu.edit.insert_selected",     "Insert Selected Item(s) on Timeline"),
    ("menu.edit.preview_selected",    "Preview Selected Item(s)"),
    ("menu.edit.favorite_selected",   "Mark Selected Item as Favorite"),
    ("menu.edit.edit_selected_tags",  "Edit Selected Item Tags..."),
    ("menu.edit.add_speaker_tags",    "Add Speaker Tags"),
    ("menu.edit.apply_offset",        "Apply Offset"),
    ("menu.edit.reset_offset",        "Reset Offset"),

    ("menu.view",                     "View"),
    ("menu.view.toggle_edit_panel",   "Show/Hide Edit Panel"),

    ("menu.settings",                 "Settings"),
    ("menu.settings.preview_volume",  "Preview Volume..."),
    ("menu.settings.font_size",       "Font Size..."),
    ("menu.settings.font_path",       "Font File..."),
    ("menu.settings.language",        "Language"),
    ("menu.settings.language.en",     "English"),
    ("menu.settings.language.ja",     "Japanese"),
    ("menu.settings.remote_script",   "Remote Script..."),

    // tab
    ("tab.sources",   "Sources"),
    ("tab.libraries", "Libraries"),

    // table
    ("item.column.index",            "#"),
    ("item.column.text",             "Text"),
    ("item.column.start",            "Start"),
    ("item.column.end",              "End"),
    ("item.column.favorite_short",   "★"),
    ("item.column.tags",             "Tags"),
    ("item.column.srt",              "SRT"),

    // hints
    ("hint.filter_text",             "Filter text..."),
    ("hint.tags",                    "Edit Item Tags..."),

    // toggle
    ("toggle.favorites_only",        "Favorites only"),
    ("toggle.hide_speaker_labels",   "Hide speaker labels"),

    // status
    ("status.inserted_n",            "Inserted {n} clip(s)."),
    ("status.tags_added",            "Added speaker tags to {n} items."),
    ("status.tags_already",          "Speaker tags were already present."),
    ("status.no_speakers",           "No speaker markers were found at the start of subtitle text."),
    ("status.alc_saved",             "Saved .alc — drop into Ableton."),

    // error
    ("error.audio_missing",          "The audio file bound to the selected subtitle could not be found."),
    ("error.remote_disconnected",    "Remote Script not connected."),
    ("error.insert_failed",          "Insert failed: {msg}"),

    // labels
    ("label.global_offset",          "Global Offset (ms)"),
    ("label.note",                   "Note"),
    ("label.audio_file",             "Audio"),
    ("label.audio_missing",          "missing"),

    // buttons
    ("button.apply",                 "Apply"),
    ("button.reset",                 "Reset"),
    ("button.test_connection",       "Test connection"),
];
```

`src/i18n/ja.rs` は同じキーで日本語訳。en にあって ja に無いキーは自動的に en にフォールバックする。

### 12.4 フォーマット引数

`{n}` `{msg}` のような placeholder は `String::replace` で愚直に処理 (依存最小化のため)。`format!` マクロ的なものが要るなら `runtime-fmt` か自作の小さなパーサで。

```rust
pub fn tf(lang: Lang, key: &str, args: &[(&str, &str)]) -> String {
    let mut s = t(lang, key);
    for (k, v) in args {
        s = s.replace(&format!("{{{k}}}"), v);
    }
    s
}
// 例: tf(lang, "status.inserted_n", &[("n", "5")])
```

### 12.5 フォント

CJK フォントは egui のデフォルトに含まれない。`resources/fonts/NotoSansCJK-Regular.otf` 等を同梱して `eframe::Frame` の起動時に読み込む:

```rust
fn install_fonts(ctx: &egui::Context, font_path: Option<&Path>) {
    let mut fonts = egui::FontDefinitions::default();
    let bytes = match font_path {
        Some(p) => std::fs::read(p).ok(),
        None => Some(include_bytes!("../../resources/fonts/NotoSansCJK-Regular.otf").to_vec()),
    };
    if let Some(b) = bytes {
        fonts.font_data.insert("cjk".into(), egui::FontData::from_owned(b));
        fonts.families.entry(egui::FontFamily::Proportional).or_default().insert(0, "cjk".into());
        fonts.families.entry(egui::FontFamily::Monospace).or_default().push("cjk".into());
    }
    ctx.set_fonts(fonts);
}
```

ライセンス的に同梱できるフォントを選ぶこと (NotoSansCJK は SIL OFL なので同梱可)。

## 13. キーボードショートカット

### 13.1 一覧

| キー | 動作 | 有効条件 |
|------|------|----------|
| ↑ | 1 つ前のアイテムへ単選択移動 | テキスト入力非フォーカス時 |
| ↓ | 1 つ次のアイテムへ単選択移動 | 同上 |
| Shift+↑/↓ | 範囲選択拡張 | 同上 |
| PageUp/PageDown | 1 ページ移動 | 同上 |
| Home/End | 先頭/末尾へジャンプ | 同上 |
| Space | プレビュー再生・停止 | 選択あり、非フォーカス時 |
| Enter / KeypadEnter | 選択を Live のカーソル位置に挿入 | 選択あり、非フォーカス時 |
| Ctrl/Cmd+F | 検索ボックスにフォーカス | 常時 |
| Ctrl/Cmd+S | メタデータ即時 flush (= Save Metadata) | Source モード |
| Ctrl/Cmd+L | 言語切替 (en ↔ ja) | 常時 (任意) |
| Esc | テキスト入力中: 確定キャンセル / それ以外: ステータストースト消去 | |

### 13.2 ショートカット集中管理

```rust
// src/ui/shortcuts.rs

pub fn handle_global(app: &mut AppState, ctx: &egui::Context) {
    let focused = ctx.memory(|m| m.has_focus(egui::Id::NULL).then_some(()).is_none()
                              || ctx.wants_keyboard_input());
    if focused { return; }

    ctx.input(|i| {
        if i.key_pressed(egui::Key::ArrowUp)   { app.move_selection(-1, i.modifiers.shift); }
        if i.key_pressed(egui::Key::ArrowDown) { app.move_selection( 1, i.modifiers.shift); }
        if i.key_pressed(egui::Key::PageUp)    { app.move_selection(-20, i.modifiers.shift); }
        if i.key_pressed(egui::Key::PageDown)  { app.move_selection( 20, i.modifiers.shift); }
        if i.key_pressed(egui::Key::Home)      { app.move_to_top(i.modifiers.shift); }
        if i.key_pressed(egui::Key::End)       { app.move_to_bottom(i.modifiers.shift); }
        if i.key_pressed(egui::Key::Space)     { app.toggle_preview(); }
        if i.key_pressed(egui::Key::Enter)     { app.insert_via_keyboard(); }
    });

    if ctx.input(|i| i.modifiers.command_only() && i.key_pressed(egui::Key::F)) {
        app.focus_filter_box = true;
    }
    if ctx.input(|i| i.modifiers.command_only() && i.key_pressed(egui::Key::S)) {
        app.flush_metadata_now();
    }
}
```

### 13.3 衝突回避

- メニューバー側のアクセラレータ表示は OS 慣習に従う:
  - macOS: `⌘+S`, `⌘+F`
  - Windows / Linux: `Ctrl+S`, `Ctrl+F`
- プラットフォーム判定は `egui` が提供する `Modifiers::command_only()` で吸収できる

## 14. ビルド・配布

### 14.1 開発ビルド

```sh
cargo run --release
```

ログレベル切替: `RUST_LOG=abletonsrtbrowser=debug cargo run`

### 14.2 リリースビルド

```sh
cargo build --release
```

`target/release/abletonsrtbrowser` (Windows: `.exe`) が成果物。

### 14.3 macOS `.app` バンドル

`cargo-bundle` を使う:

```sh
cargo install cargo-bundle
cargo bundle --release
```

`Cargo.toml` に:

```toml
[package.metadata.bundle]
name = "AbletonSRTBrowser"
identifier = "io.github.naari3.abletonsrtbrowser"
icon = ["resources/icon.icns"]
copyright = "Copyright (c) 2026 naari3"
category = "public.app-category.music"
short_description = "SRT browser for Ableton Live"
long_description = "..."
osx_minimum_system_version = "12.0"
```

公証 (notarization) はオプション。配布時は `xcrun notarytool` で。

### 14.4 Windows `.exe`

`cargo build --release` 後、`target/release/abletonsrtbrowser.exe` をそのまま配布。アイコン埋め込みは `winres` で:

```toml
[build-dependencies]
winres = "0.1"
```

`build.rs`:

```rust
fn main() {
    if cfg!(target_os = "windows") {
        let mut res = winres::WindowsResource::new();
        res.set_icon("resources/icon.ico");
        res.compile().unwrap();
    }
}
```

コンソールウィンドウを抑止するには `main.rs` 先頭に:

```rust
#![cfg_attr(all(not(debug_assertions), target_os = "windows"), windows_subsystem = "windows")]
```

### 14.5 Linux AppImage

`cargo-appimage` または `appimage-builder` を使う。MVP では `tar.gz` 配布で十分。

### 14.6 Remote Script の配布

`crates/remote_script/` を zip にして release assets に同梱。インストール手順は `docs/REMOTE_SCRIPT_INSTALL.md`:

```
1. zip を展開すると AbletonSRTBrowser/ フォルダができる
2. 以下に配置:
   - macOS: ~/Music/Ableton/User Library/Remote Scripts/
   - Windows: Documents\Ableton\User Library\Remote Scripts\
3. Live を起動
4. Preferences → Link, Tempo & MIDI → Control Surface でドロップダウンから
   AbletonSRTBrowser を選択
5. AbletonSRTBrowser GUI を起動 → 自動接続
```

### 14.7 CI

GitHub Actions で 3 OS マトリクスビルド:

```yaml
# .github/workflows/build.yml
name: build
on: [push, pull_request]
jobs:
  build:
    strategy:
      matrix:
        os: [macos-latest, windows-latest, ubuntu-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo test --all
      - run: cargo build --release
      - uses: actions/upload-artifact@v4
        with:
          name: abletonsrtbrowser-${{ matrix.os }}
          path: target/release/abletonsrtbrowser*
```

リリースタグ push で `release.yml` が `.app` / `.exe` / Linux バイナリ + Remote Script zip をまとめて GitHub Release にアップロード。

## 15. 推奨実装順序 (フェーズ分け)

### Phase 0: スケルトン (1〜2 日)

1. `cargo new abletonsrtbrowser`、依存追加
2. `eframe::App` を実装し、ハロー egui ウィンドウを出す
3. `tracing-subscriber` でログを設定
4. CI 雛形を入れる (`cargo test`, `cargo clippy -- -D warnings`)

**完了基準**: 空ウィンドウが macOS / Windows / Linux で起動

### Phase 1: モデルとテスト (2〜3 日)

1. `model::srt` で SRT パーサ + ユニットテスト (BOM・CRLF・話者抽出)
2. `model::audio_match` の自動マッチング + ユニットテスト
3. `model::source::SrtSource::load` でパース → メタデータマージ
4. `model::filter::Filter` のフィルタロジック + テスト

**完了基準**: コマンドライン (テスト) のみで SRT を読み、フィルタが正しく効く

### Phase 2: 最低限の GUI (3〜5 日)

1. メニューバー (File メニューだけ動かす)
2. 左ペイン: Sources タブで SRT 一覧 (フォルダなし、フラットでよい)
3. 中央: アイテムテーブル (列・ソート・virtual scroll)
4. 検索ボックスのフィルタ反映
5. ↑↓ / Space (Space は仮で no-op)
6. 詳細ペイン (お気に入りトグルとタグだけ)
7. 永続化 (settings + metadata の atomic write)

**完了基準**: SRT を開いて検索・タグ付けができる

### Phase 3: オーディオプレビュー (1〜2 日)

1. `PreviewPlayer` 実装
2. Space キー / メニュー / 詳細ペインの再生ボタン
3. 音量設定ダイアログ
4. 範囲再生のテスト (短い wav と長い wav)

**完了基準**: 任意のアイテムをプレビュー再生・停止できる

### Phase 4: 挿入抽象 + .alc (3〜5 日)

1. `Inserter` trait 定義
2. `AlcInserter` 実装 + `.alc` テンプレート確定 (検証手順 6.2 を実行)
3. テスト用の hand-crafted `.alc` を Live にドロップして「クリップが意図通り配置されるか」を確認
4. `drag` クレートを組み込み、テーブル行ドラッグで `.alc` を drag-out
5. キーボード Enter は当面 .alc にフォールバック (Composite で `fallback_keyboard_to_alc=true`)

**完了基準**: アプリから Live の Arrangement にドラッグして、SRT 範囲のオーディオクリップが配置される

### Phase 5: Remote Script (3〜5 日)

1. `crates/remote_script/` の Python パッケージを書く
2. Live にインストール → Log.txt で起動を確認
3. `nc` で `{"op":"ping"}` → `{"op":"pong"}` を確認
4. `RemoteScriptInserter` の Rust クライアントを書く
5. `Track.create_audio_clip` で実際に挿入できるかを Live コンソールで先に確認
6. 接続管理 (起動時 connect、切断時の再試行、ステータスバー表示)
7. Composite Inserter で Keyboard を Remote へ振り分け

**完了基準**: Enter キーで Live のカーソル位置にクリップが挿入される

### Phase 6: ライブラリ機能 (2〜3 日)

1. `Library` モデル + フォルダツリー
2. 左ペイン Library タブ
3. ライブラリ横断検索 (中央テーブルが SRT 列を表示するモード)
4. ドラッグでの並び替え・フォルダ移動

**完了基準**: 複数 SRT を 1 つのライブラリにまとめて横断検索できる

### Phase 7: 仕上げ (継続)

1. i18n 完備 (en/ja)
2. ドラッグ&ドロップ受け入れ (`.srt`/`.wav`/フォルダ)
3. レイアウト永続化 (列幅・ペイン幅・展開状態)
4. トースト通知
5. アイコン作成、`cargo bundle` / `winres` 統合
6. `docs/REMOTE_SCRIPT_INSTALL.md` を仕上げる
7. README に GIF / スクリーンショット

**完了基準**: ReaSRTBrowser の README に書かれている全機能が動く

### 各フェーズの所要見積りについて

合計 15〜25 日 (1 人前提、慣れていれば下振れ、egui 初学なら上振れ)。Remote Script (Phase 5) は Live API の実機検証で詰まる可能性が一番高いので、Phase 4 (.alc) でリリース可能な状態を一度確保してから着手するのが安全。

## 16. 検証チェックリスト

### 16.1 ユニットテスト

- [ ] `srt::parse_srt` は BOM / CRLF / LF を正しく扱う
- [ ] `srt::parse_srt` は `,` `.` 両方の小数点記法を受ける
- [ ] `speaker::split_speaker` は `(話者)` `（話者）` 両方を抽出
- [ ] `audio_match::auto_match_audio` は完全一致を優先する
- [ ] `audio_match::auto_match_audio` は最短ファイル名で安定ソート
- [ ] `filter::Filter::parse` は `-tag` を exclude_tags に入れる
- [ ] `filter::matches` は exclude_tags が AND で効く
- [ ] `metadata::merge_meta` はキー (index,start,end,text) で同定する

### 16.2 .alc バックエンド

- [ ] 生成した `.alc` を `gunzip -c | xmllint --format -` で読める
- [ ] Live (11.x) で `.alc` をドロップしてクリップが配置される
- [ ] Live (12.x) で同上
- [ ] `start_marker` / `end_marker` が SRT 範囲と一致する (誤差 < 1ms)
- [ ] テンポを変えた状態でドロップしてもクリップ長が崩れない
- [ ] `WarpMode` が Off で挿入される
- [ ] 一時ファイル (`.alc`) は 24h で GC される
- [ ] ファイル名サニタイズ: `/` `\` `:` 等が含まれた display_name でも作成成功

### 16.3 Remote Script バックエンド

- [ ] Live 11 で Control Surface 一覧に表示される
- [ ] Live 12 で同上
- [ ] `nc 127.0.0.1 19823` で `{"op":"ping"}` → `{"op":"pong"}`
- [ ] `{"op":"get_state"}` で `tempo`, `song_time_beats` が取れる
- [ ] `subscribe` で `tempo` 変更がプッシュされる
- [ ] `insert_clips` でクリップが配置される
- [ ] `advance_cursor: true` で次回挿入位置が進む
- [ ] track_index 不正値で error が返る (クラッシュしない)
- [ ] 切断 → 再接続が `subscribe` を再送する
- [ ] Live 終了時に Rust 側が disconnect を検知してフォールバックに切り替える

### 16.4 GUI

- [ ] 1000+ アイテムの SRT でも検索がスムーズ (60fps 維持)
- [ ] 列ヘッダクリックでソートが切り替わる
- [ ] 列幅をドラッグして変更 → 再起動後も保持される
- [ ] 左ペイン幅・詳細ペイン高さが永続化される
- [ ] 言語切替が即座に反映される
- [ ] フォント変更後、CJK が表示される
- [ ] 高 DPI ディスプレイでフォントが滲まない
- [ ] 移植元の全メニュー項目が存在し動作する
- [ ] 全ショートカットが意図通り動く
- [ ] 入力フィールドフォーカス中はショートカットが誤発火しない
- [ ] エラー時にトースト表示され、アプリは落ちない

### 16.5 永続化

- [ ] アプリを kill しても破損 JSON が残らない (atomic write)
- [ ] settings.json が 500ms 以内にフラッシュされる
- [ ] metadata.json が 1500ms 以内にフラッシュされる
- [ ] Apply Offset 押下で即時フラッシュ
- [ ] アプリ終了時に dirty が全て吐かれる

### 16.6 マルチプラットフォーム

- [ ] macOS (12+) で `.app` バンドルが起動
- [ ] Windows (10+) で `.exe` がダブルクリックで起動 (コンソール窓なし)
- [ ] Ubuntu 22.04 / 24.04 でバイナリ単体で起動
- [ ] 3 OS いずれでも `.alc` drag-out が Ableton Live に届く

### 16.7 ライセンス

- [ ] 同梱フォントのライセンスが配布可 (NotoSansCJK = OFL-1.1)
- [ ] 全クレートの SPDX を README に列挙
- [ ] 元の ReaSRTBrowser が MIT なので、本プロジェクトも MIT を推奨

## 17. 参考リソース

### 17.1 既存実装

- **ReaSRTBrowser** (移植元): https://github.com/naari3/reasrtbrowser
  - 主に `src/ReaSRTBrowser.lua` (4474 行) と `src/reasrt/*.lua` を参照
  - 移植時は Lua の挙動を「機能要件」として読み、Rust らしく実装し直す

### 17.2 Ableton 関連

- **AbletonOSC**: https://github.com/ideoforms/AbletonOSC
  - 自前 Remote Script の実装パターンを学ぶリファレンス。`manager.py` の tick / disconnect / reload は流用可。
- **AbletonOSC NIME 2023 paper**: https://nime.org/proceedings/2023/nime2023_60.pdf
  - LOM 設計思想と OSC マッピングの背景
- **Creating your own Control Surface script (Ableton)**: https://help.ableton.com/hc/en-us/articles/206240184-Creating-your-own-Control-Surface-script
- **Installing third-party remote scripts (Ableton)**: https://help.ableton.com/hc/en-us/articles/209072009-Installing-third-party-remote-scripts
- **AbletonLive11_MIDIRemoteScripts (decompiled)**: https://github.com/gluon/AbletonLive11_MIDIRemoteScripts
  - `_Framework` の中身を読みたい時
- **ableton-lom-skill (Live 12.3 LOM)**: https://github.com/mikecfisher/ableton-lom-skill
  - `Track.create_audio_clip` 等のシグネチャ確認
- **Live 10.0.1 API documentation**: https://structure-void.com/PythonLiveAPI_documentation/Live10.0.1.xml
- **Debugging Ableton Remote Scripts**: https://forum.ableton.com/viewtopic.php?t=240667

### 17.3 .alc / .als 形式

- **fileformats.archiveteam.org Ableton**: http://fileformats.archiveteam.org/wiki/Ableton_Live
- **Sonic Bloom: Guide to Ableton Live File Formats**: https://sonicbloom.net/the-guide-to-ableton-live-file-formats/
- **Ableton Forum: Decoding ALS file format**: https://forum.ableton.com/viewtopic.php?t=121089
- **Using Live Clips (.alc files)**: https://help.ableton.com/hc/en-us/articles/209071569-Using-Live-Clips-alc-files

### 17.4 Rust ライブラリ

- **egui**: https://github.com/emilk/egui / https://docs.rs/egui
- **eframe**: https://docs.rs/eframe
- **egui_extras::TableBuilder**: https://docs.rs/egui_extras
- **egui_dnd**: https://crates.io/crates/egui_dnd
- **rodio**: https://github.com/RustAudio/rodio
- **symphonia**: https://github.com/pdeljanov/Symphonia
- **drag (tauri-apps)**: https://crates.io/crates/drag
- **rfd**: https://crates.io/crates/rfd
- **serde / serde_json**: https://serde.rs/
- **quick-xml**: https://docs.rs/quick-xml

### 17.5 Drag-out 周り

- **winit issue #1499 (file dragging)**: https://github.com/rust-windowing/winit/issues/1499
- **DroppedFile in egui**: https://docs.rs/egui/latest/egui/struct.DroppedFile.html

---

## 付録 A: 開発開始時の TODO チェックリスト

```markdown
- [ ] 新規リポジトリ作成 (例: github.com/<user>/abletonsrtbrowser)
- [ ] ライセンス: MIT (移植元と一致)
- [ ] README.md (1 段落の概要だけで OK)
- [ ] Cargo プロジェクトを作成 (Section 4 の Cargo.toml)
- [ ] resources/alc_template.xml を空ファイルで配置 (Phase 4 で確定)
- [ ] crates/remote_script/ ディレクトリを作って空の __init__.py / manager.py
- [ ] .github/workflows/build.yml を Section 14 の例で配置
- [ ] tracing-subscriber で main.rs の初期化
- [ ] eframe::run_native でハロー egui ウィンドウ
- [ ] CLAUDE.md を docs/ARCHITECTURE.md として一部抜粋
```

## 付録 B: よくある質問

**Q. `_Framework` を使わず Remote Script を書ける?**
A. 可能だが推奨しない。`ControlSurface` の lifecycle (build_midi_map / disconnect / can_lock_to_devices 等) を自分で全部書くことになる。`_Framework` は Live に同梱されており、これに依存しても外部依存にはならない。

**Q. `.alc` ではなく `.als` を生成して開かせる方が確実?**
A. `.als` は Live セット全体を表すので、開くと現在のセットが置き換わってしまう。「クリップを足す」用途には `.alc` が正解。

**Q. Live が起動していない状態で D&D したらどうなる?**
A. OS の D&D は Live の起動を促さない。ユーザに「Live を起動してください」と促すトーストを出す。Remote Script との接続状態が手がかりになる (接続なし = Live 不在の可能性が高い)。

**Q. egui のテーブルで 10 万行を出すと重くない?**
A. `TableBuilder` は virtual scroll に対応している (`body.rows(...)` パターン)。10 万行でも 60fps を維持できるが、行の高さを固定にする必要がある。可変高さは `body.heterogeneous_rows` を使うが性能注意。

**Q. なぜ macOS で `cacao` ではなく `drag` を使う?**
A. `cacao` は macOS 専用。`drag` (tauri-apps) は Windows / macOS / Linux を 1 つの API で扱える。winit / tao どちらにも対応している。

**Q. ReaSRTBrowser のメタデータ JSON と互換にすべき?**
A. しなくて良い (ハッシュ計算式が違う、フォーマットも微妙に変える)。互換が必要なら明示的にインポート機能を Phase 7 で追加。

---

このドキュメントは仕様書のスナップショットです。実装中に判明した制約 (特に `.alc` の必須属性、Live バージョン依存) は本書を更新してください。

