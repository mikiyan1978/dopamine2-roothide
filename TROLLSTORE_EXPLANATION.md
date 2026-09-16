# TrollStore - 非脱獄でのアプリサイドロード基盤

## TrollStore とは

```
TrollStore = 「脱獄なしで、改ざんされたIPAをデバイスにインストールできるツール」
```

開発者: Trolltech  
GitHub: `https://github.com/34306/TrollStore`

---

## 何ができるのか

### App Store では不可能なこと

```
❌ App Store:
   └─ 署名されたIPAのみ受け付け
   └─ Apple以外の署名は拒否
   └─ App改ざん検出 → 即削除

✅ TrollStore:
   ├─ 独自署名IPAをインストール可能
   ├─ App Store版アプリを修正して再インストール
   ├─ 同じアプリの複数バージョン共存
   └─ カスタムdylib埋め込みアプリ実行
```

---

## TrollStore の仕組み

### Core Trust Bypass

```
通常のiOS:
  app起動 → dyld → Code署名検証
          → 署名OK? → 実行 / NG? → kill

TrollStore環境:
  app起動 → dyld → 署名チェック（独自実装）
          → TrollStore許可リスト確認
          → 許可なら実行
          → 署名スキップ
```

**重要**: TrollStore は iOS 15.0 以降の特定のバージョンの「脆弱性」を利用

---

## インストール方法

### 前提

- iOS 15.0 - 16.6.1 (特定バージョン)
- または iOS 17.0+ の一部バージョン
- **脱獄不要**（重要）

### 手順

```bash
# 1. TrollStore インストーラーをダウンロード
# https://github.com/34306/TrollStore/releases
# TrollStore.ipa をダウンロード

# 2. AltStore / Sideloadly などを使ってインストール
#    （AltStore経由でTrollStoreをサイドロード）

# 3. インストール完了後、TrollStoreアプリが起動可能に

# 4. TrollStore内から他のIPAをインストール
#    ├─ 修正済みYouTube.ipa
#    ├─ 広告ブロッカー付きTikTok.ipa
#    └─ カスタムTwitterクライアント.ipa
```

---

## TrollFools との関係

```
TrollStore + TrollFools = 完全な非脱獄カスタマイズ環境

┌─────────────────────────────────────────┐
│ TrollStore                              │
│ (App Store app を改ざんしてインストール)  │
│                                         │
│ ┌───────────────────────────────────┐   │
│ │ TrollFools                        │   │
│ │ (IPA内にdylib埋め込み)            │   │
│ │  1. IPA解凍                       │   │
│ │  2. Mach-O Load Command追加       │   │
│ │  3. dylib配置                     │   │
│ │  4. 再署名                        │   │
│ └───────────────────────────────────┘   │
│                                         │
│ 結果: YouTubeExtension.dylib            │
│      (広告ブロック、UI改ざん等)        │
│      が組み込まれたYouTube             │
└─────────────────────────────────────────┘
```

---

## 実例：YouTube をカスタマイズ

### ワークフロー

```
Step 1: YouTube.ipa を入手
  └─ App Store から抽出 / 配布サイトから

Step 2: TrollFools でカスタマイズ
  ├─ AdBlocker.dylib を埋め込み
  ├─ Mach-O Load Command 追加
  └─ 再署名
  結果: YouTube_Modified.ipa

Step 3: TrollStore を開く
  └─ YouTube_Modified.ipa をドラッグ&ドロップ

Step 4: インストール
  └─ TrollStore が署名検証をスキップしてインストール

Step 5: YouTube 起動
  ├─ dyld が AdBlocker.dylib を自動ロード
  ├─ %hook が広告クラスをインターセプト
  └─ 広告なしで再生
```

---

## TrollStore でインストール可能なアプリ

### 修正可能なカテゴリ

```
✅ 可能:
  ├─ App Store 公式アプリ（YouTube, TikTok等）
  ├─ 広告ブロッカー埋め込み
  ├─ UI改ざん
  ├─ API キー管理ツール埋め込み
  ├─ ログ出力dylib
  └─ ローカルHTTPプロキシ埋め込み

❌ 不可能:
  ├─ 高度なセキュリティアプリ（VPN等）
  ├─ Health/HomeKit など特殊entitlements
  ├─ App Store以外の署名
  └─ Kernel level の操作
```

**理由**: TrollStore は「App Store app の再署名」のみ。
System entitlements や Kernel access は提供しない。

---

## TrollStore と脱獄の関係

### 非脱獄でのTrollStore

```
iOS 15-16:
  └─ CVE-2023-XXXXX の脆弱性を利用
  └─ Core Trust Bypass → 署名チェック無視
  └─ 脱獄不要で動作

iOS 17+:
  └─ 同様の脆弱性を利用（バージョン限定）
  └─ バージョン更新で使えなくなる可能性
```

### 脱獄環境でのTrollStore

```
Dopamine脱獄済み + TrollStore:
  ├─ 脱獄なし TrollStore よりセキュアな環境
  ├─ Tweak + TrollFools 両方使用可能
  ├─ Tweak は システムwide に効果
  └─ TrollFools は 個別アプリに効果

併用メリット:
  ├─ BackgroundAction (Tweak) で全アプリをバックグラウンド保護
  └─ YouTubeExtension.dylib で YouTube だけ広告ブロック
```

---

## 注意点・制限事項

### 脆弱性ベース = バージョン依存

```
TrollStore が動作するiOSバージョン:
  ✅ iOS 15.0 - 15.8
  ✅ iOS 16.0 - 16.6.1
  ❌ iOS 16.7+           (脆弱性修正)
  ✅ iOS 17.0 - 17.4.1   (別の脆弱性)
  ❌ iOS 17.5+           (修正予定)
  
ユーザー側での対策なし：
  └─ iOS更新 → 使えなくなる可能性
```

### 署名の有効期限

```
TrollStore 署名:
  └─ ローカルのみ有効（7日？）
  └─ 配布不可
  └─ デバイス固有

App Store 署名:
  └─ 全デバイスで有効
  └─ 配布可能
  └─ 長期有効
```

**つまり**:
- TrollStore は「自分のデバイスでカスタマイズして使う」ツール
- 他人に配布するなら App Store 署名が必須

---

## TrollStore の実装テクニック

### What TrollStore Does

```c
// 簡略化

// 1. IPAを検証
bool isValidIPA(NSString *ipaPath) {
    // Info.plist 読み込み
    // Mach-O バイナリ確認
    // 既にインストール済みか確認
    return true/false;
}

// 2. インストール処理
void installIPA(NSString *ipaPath) {
    // Step 1: IPA を /var/containers/Bundle/Application/ に展開
    extractIPAToApplications(ipaPath);
    
    // Step 2: Code Signature を TrollStore 署名に置き換え
    //         （この部分が脆弱性を利用）
    replaceSignatureWithTrollStoreSignature();
    
    // Step 3: dyld cache 無効化（古いバージョン削除）
    invalidateLaunchdCache();
    
    // Step 4: SpringBoard に通知
    //         （ホーム画面にアイコン出現）
    notifySpringBoard();
}

// 3. 起動時の署名チェック
bool checkSignature(NSString *appPath) {
    // dyld が署名検証の代わりに、
    // TrollStore の許可リストを確認
    return isInTrollStoreRegistry(appPath);
}
```

---

## 実践的なワークフロー

### 完全なセットアップ

```bash
# 1. TrollStore インストール
#    └─ Sideloadly で TrollStore.ipa をインストール

# 2. YouTube.ipa を取得
#    ├─ AppStore から抽出（iMazing等）
#    ├─ または配布サイトからDL
#    └─ YouTube_v18.42.00.ipa

# 3. TrollFools で修正
swift TrollFoolsInjector \
    --input YouTube_v18.42.00.ipa \
    --dylib AdBlocker.dylib \
    --output YouTube_AdFree.ipa

# 4. TrollStore で install
#    └─ YouTube_AdFree.ipa を TrollStore にドラッグ
#    └─ "Install" ボタン
#    └─ 数秒で完了

# 5. YouTube 起動
#    └─ 広告なしで再生可能
```

---

## 脱獄 vs TrollStore 比較

| 項目 | 脱獄（Dopamine） | TrollStore |
|------|-----------------|-----------|
| **必要物** | iOS脆弱性 | iOS脆弱性 |
| **Root取得** | ✅ | ❌ |
| **Tweak** | ✅ | ❌ |
| **IPA修正** | ✅ | ✅ |
| **System修正** | ✅ | ❌ |
| **複数アプリ同時** | ✅ | ✅ |
| **永続性** | ✅ | ⚠️ (7日署名) |
| **配布可能** | ❌ | ❌ |
| **難易度** | 高 | 低 |

---

## まとめ

```
TrollStore の位置づけ
=======================

App Store
  (Apple 署名のみ)
      ↑
      └─ TrollStore
         (脆弱性利用して署名検証ByPass)
         
TrollStore 内から:
  ├─ 素のIPA インストール
  ├─ TrollFools で修正したIPA インストール
  └─ dylib 埋め込みアプリ実行可能
```

**重要**: 
- TrollStore = 「署名検証をスキップするツール」
- TrollFools = 「IPA内にdylib埋め込むツール」
- 両方揃ってはじめて「非脱獄カスタマイズ」完成

---

**Dopamine環境なら TrollStore 不要**（脱獄済みなので）
ただし BackgroundAction + TrollFools の組み合わせも可能
