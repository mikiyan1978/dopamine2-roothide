# TrollFools = 「dylib注入IPA」のシンプル説明

## 本質

```
TrollFools とは：
「App Storeのアプリ (IPA) に、カスタムdylib を埋め込んで再署名したもの」
```

---

## IPA ファイルの構造

### 修正前（元のApp Store IPA）

```
YouTube.ipa (ZIP形式)
├── YouTube.app/
│   ├── Info.plist              (アプリ設定)
│   ├── YouTube                 (実行ファイル = Mach-Oバイナリ)
│   ├── Frameworks/
│   │   ├─ UIKit.framework
│   │   ├─ AVFoundation.framework
│   │   └─ ...（システムフレームワーク）
│   ├── Resources/
│   │   ├─ Images/
│   │   ├─ Localizable.strings
│   │   └─ ...
│   └─ _CodeSignature/
│       └─ CodeResources         (署名ファイル）
└── ... その他
```

### 修正後（TrollFools注入後）

```
YouTube.ipa.modified (ZIP形式)
├── YouTube.app/
│   ├── Info.plist
│   ├── YouTube                 ★修正: Load Commandに
│   │                             LC_LOAD_DYLIB追加
│   ├── Frameworks/
│   │   ├─ UIKit.framework
│   │   ├─ AVFoundation.framework
│   │   ├─ YouTubeExtension.dylib  ★新規: 注入dylib
│   │   └─ ...
│   ├── Resources/
│   └─ _CodeSignature/
│       └─ CodeResources         ★修正: 新しい署名
└── ...
```

---

## 修正プロセス（3ステップ）

### Step 1: IPA を ZIP として解凍

```bash
unzip YouTube.ipa -d YouTube.app.extracted
```

結果:
```
YouTube.app.extracted/
└── Payload/
    └── YouTube.app/
        └── YouTube (バイナリ)
```

### Step 2: YouTube バイナリの Mach-O を修正

**修正前**:
```
[Mach-O Header]
  ncmds = 15
  sizeofcmds = 1344
  
[Load Commands]
  ├─ LC_SEGMENT_64 (メモリマッピング)
  ├─ LC_LOAD_DYLIB /usr/lib/libobjc.A.dylib
  ├─ LC_LOAD_DYLIB /usr/lib/libSystem.dylib
  └─ ... 13個
```

**修正後**:
```
[Mach-O Header]
  ncmds = 16  ← +1
  sizeofcmds = 1392  ← +48
  
[Load Commands]
  ├─ LC_SEGMENT_64 (メモリマッピング)
  ├─ LC_RPATH @executable_path/Frameworks  ← 新規追加
  ├─ LC_LOAD_DYLIB @executable_path/Frameworks/YouTubeExtension.dylib  ← 新規追加
  ├─ LC_LOAD_DYLIB /usr/lib/libobjc.A.dylib
  ├─ LC_LOAD_DYLIB /usr/lib/libSystem.dylib
  └─ ... 14個
```

**何をしたか**:
- 新しい `LC_LOAD_DYLIB` コマンド追加
- パスは `@executable_path/Frameworks/YouTubeExtension.dylib`
- dyld が起動時に自動でこのdylibを読み込む

### Step 3: dylib をコピー + 再署名

```bash
# YouTubeExtension.dylib を Frameworks/ にコピー
cp YouTubeExtension.dylib YouTube.app.extracted/Payload/YouTube.app/Frameworks/

# バイナリを再署名
codesign -f -s - YouTube.app.extracted/Payload/YouTube.app/YouTube

# IPA を再作成
cd YouTube.app.extracted/Payload
zip -r ../../YouTube.ipa.modified YouTube.app
```

---

## 起動時のフロー

```
1. TrollStore が修正済み IPA をインストール
   ↓
2. ユーザーが YouTube アプリをタップ
   ↓
3. dyld が起動
   ├─ YouTube バイナリのMach-Oヘッダ読む
   ├─ Load Commandsをスキャン
   └─ LC_LOAD_DYLIB を見つける
   ↓
4. dyld が @executable_path/Frameworks/YouTubeExtension.dylib を探す
   ├─ @executable_path = /var/containers/.../YouTube.app/
   └─ 完全パス: /var/containers/.../YouTube.app/Frameworks/YouTubeExtension.dylib
   ↓
5. dyld が YouTubeExtension.dylib を mmap() + ロード
   ↓
6. YouTubeExtension.dylib の -init_routine 実行
   ├─ Substrate.framework 初期化
   └─ Logos %hook 登録
   ↓
7. YouTube アプリメイン関数へ
   └─ 既に %hook が適用済み状態で起動
```

---

## 脱獄との違い

### 脱獄環境（Dopamine）

```
BackgroundAction.dylib
  ↓
runningboardd に注入（システムwide）
  ↓
すべてのプロセスがフック対象
```

**1つのTweak → 複数アプリに効果**

### 非脱獄環境（TrollFools）

```
YouTubeExtension.dylib
  ↓
YouTube バイナリに埋め込み
  ↓
YouTube起動時のみ自動ロード
```

**各アプリごとにIPA修正が必要**

---

## 実例：YouTube 広告ブロッカー

### ユースケース

```
App Store YouTube では広告が必須
  ↓
TrollFools で広告ブロッカーdylibを注入
  ↓
修正済み YouTube インストール
  ↓
広告スキップ機能が組み込まれた YouTube として動作
```

### 実装例

```objc
// YouTubeExtension.dylib 内の Logos %hook

%hook YTInterstitialAdRenderer  // 広告表示クラス
- (void)renderContainerWithContext:(id)context {
    // 広告は表示しない（早期リターン）
    return;
}
%end

%hook YTPlayerViewController
- (void)playVideo {
    NSLog(@"✓ Video playback (skip ads)");
    %orig;  // 元のメソッド実行
}
%end
```

---

## IPAでなく「アプリそのもの」を修正する場合

### インストール済みアプリへの直接パッチ

```
/var/containers/Bundle/Application/XXXX/YouTube.app/YouTube
  (バイナリ)
  ↓
TrollFools が同じ操作
  ├─ Mach-O Load Command 追加
  ├─ dylib を @executable_path/Frameworks/ に配置
  └─ dyld が自動ロード
```

**デバイス上で直接修正する場合**:
```bash
# SSH で device に接続
ssh root@device.local

# インストール済みアプリを探す
find /var/containers -name "YouTube" -type f 2>/dev/null

# バイナリを修正
/path/to/TrollFoolsInjector \
  --app /var/containers/Bundle/Application/XXXX/YouTube.app \
  --dylib /tmp/YouTubeExtension.dylib

# アプリ再起動
killall -9 YouTube
```

---

## コア概念：@executable_path の自動解決

### 従来（絶対パス）

```
Load Command: 
  LC_LOAD_DYLIB /var/containers/Bundle/Application/ABC123/YouTube.app/Frameworks/YouTubeExtension.dylib

問題：
  ・デバイスAのパス ≠ デバイスBのパス
  ・毎回異なるディレクトリになる
  ・バイナリを複数作る必要がある
```

### TrollFools方式（@executable_path）

```
Load Command:
  LC_LOAD_DYLIB @executable_path/Frameworks/YouTubeExtension.dylib

dyld の自動展開：
  デバイスA: @executable_path → /var/containers/.../YouTube.app/
  デバイスB: @executable_path → /private/var/containers/.../YouTube.app/
  
結果：
  ・1つのバイナリで複数デバイス対応
  ・相対パスなので portable
```

---

## まとめ

```
TrollFools の本質
=================

┌────────────────────────────────┐
│ 元のApp Store IPA              │
│  ├─ YouTube (バイナリ)         │
│  ├─ Frameworks/                │
│  └─ Resources/                 │
└────────────────────────────────┘
            ↓ [TrollFools 処理]
┌────────────────────────────────┐
│ 修正済み IPA                    │
│  ├─ YouTube (修正版)           │
│  │   Load Command追加          │
│  ├─ YouTubeExtension.dylib ← 新規
│  ├─ Frameworks/                │
│  └─ Resources/                 │
└────────────────────────────────┘
            ↓ [再署名]
┌────────────────────────────────┐
│ 署名済み修正IPA                 │
│  (デバイスにインストール可能)    │
└────────────────────────────────┘
            ↓ [インストール]
        デバイス上で
            ↓ [起動]
        dyld が自動で
        YouTubeExtension.dylib ロード
            ↓
        %hook が有効
```

---

## 技術的ポイント

| 項目 | 説明 |
|------|------|
| **IPA** | iOS app package (ZIP形式) |
| **Mach-O** | iOS実行ファイル形式 |
| **Load Command** | dyld用のメタデータ |
| **LC_LOAD_DYLIB** | 依存dylib指定コマンド |
| **@executable_path** | アプリバンドルへの相対パス |
| **dyld** | Dynamic loader（起動時にdylib読み込む） |
| **ad-hoc署名** | TrollStore対応の再署名方式 |

---

## 脱獄環境での不要な理由

```
脱獄 Tweak の場合：
- runningboardd に直接注入
- 全プロセスに効果
- IPA修正不要（システムレベル）

非脱獄 TrollFools の場合：
- 各IPA個別に修正
- 修正したアプリのみ効果
- IPA解凍 → 修正 → 再署名 が必須
```

**結論**: TrollFools = 「脱獄できない環境での代替案」

---

あなたの理解（「dylib注入IPA」）で完全に正しいです！
