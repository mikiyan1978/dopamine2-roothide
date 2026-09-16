# TrollFools 非脱獄dylib注入メカニズム - 詳細解析

## 概要

TrollFools（開発者: 82Flex）は、**脱獄なしのiOSアプリ内にdylibを注入する**唯一の実用的ツール。App Storeアプリをサイドロードしながら機能拡張を行える。

**重要**: 脱獄環境では**ここまで複雑な仕組みは不要**。本解析は「脱獄なしではどこまで複雑か」を理解するため。

---

## Part 1: 前提知識 - Mach-O Binary Format

### Mach-Oとは

iOS実行ファイルの形式。以下で構成：

```
[Mach-O Header]
    ├─ magic: 0xfeedfacf (32-bit) / 0xfeedfacf (64-bit)
    ├─ cputype: CPU_TYPE_ARM (0x7) / CPU_TYPE_ARM64 (0x0100000c)
    ├─ filetype: MH_EXECUTE (2) / MH_DYLIB (6) / etc.
    ├─ ncmds: Load command数
    ├─ sizeofcmds: 全Load commandのバイト数
    └─ flags: MH_PIE, MH_DYLDLINK, ...

[Load Commands] ← 動的リンカ用メタデータ
    ├─ LC_SEGMENT_64         (メモリマッピング)
    ├─ LC_DYLD_INFO_ONLY     (実行時バインディング情報)
    ├─ LC_LOAD_DYLIB         ★ 依存dylib指定
    ├─ LC_LOAD_WEAK_DYLIB    ★ 弱参照dylib指定
    ├─ LC_CODE_SIGNATURE     (コード署名)
    ├─ LC_ENCRYPTION_INFO_64 (暗号化フラグ)
    └─ ...

[Sections]
    ├─ __text                 (実行可能コード)
    ├─ __data                 (初期化済みデータ)
    ├─ __cstring              (C文字列リテラル)
    └─ ...

[Symbol Table / Dynamic Symbol Table]
    └─ (dyld用の外部シンボル解決情報)
```

### Load Commandsの役割

dyldが起動時に**実行ファイルのLoad Commandsを読む**：

```
1. dyld が実行ファイルのMach-Oヘッダを解析
2. LC_LOAD_DYLIB コマンドをスキャン
3. 各dylib のパス（例: @executable_path/Frameworks/MyDylib.dylib）を抽出
4. 順番に dylib を mmap() してメモリにロード
5. シンボルテーブルを構築
6. リロケーション情報を適用
7. 初期化関数(-init_routine) を実行
```

**TrollFoolsの戦略**: 
- アプリのバイナリに新しい `LC_LOAD_DYLIB` コマンドを挿入
- dylib を `@executable_path/Frameworks/` に配置
- dyld が自動で読み込む

---

## Part 2: TrollFools - InjectorV3の3つの段階

### 概要フロー

```
[入力] アプリ.ipa + 注入dylib群
       ↓
[InjectorV3初期化]
       ├─ bundleURL検証
       ├─ 実行ファイル検出
       ├─ Frameworks/ディレクトリ検出
       ├─ appID・teamID抽出
       └─ ログディレクトリ作成
       ↓
[前処理: preprocessAssets()]
       ├─ dylib・framework・その他を分類
       ├─ Substrate.framework自動組み込み
       ├─ ファイル削除・パス正規化
       └─ arm64/arm64e スライス検出
       ↓
[Mach-O解析: injectInto()]
       ├─ 保護レベル検出 (CS_HARD, CS_RESTRICT, ...)
       ├─ 利用可能な実行ファイル選択
       ├─ Load Command挿入前の検証
       └─ Atomic操作開始
       ↓
[Load Command操作]
       ├─ cmdInsertLoadCommandRuntimePath()
       │  └─ @executable_path/Frameworks を追加
       ├─ cmdInsertLoadCommandDylib()
       │  └─ LC_LOAD_DYLIB コマンド挿入
       ├─ cmdChangeLoadCommandDylib()
       │  └─ パス正規化 (@rpath → @loader_path等)
       └─ [各dylib順に実行]
       ↓
[署名・永続化]
       ├─ 暫定バイナリを本体に置き換え
       ├─ /var/mobile/Library/TrollFools/PersistentPlugins/{bundleID}/ に保存
       └─ 次回起動時も同じ注入を適用
       ↓
[出力] 修正済み.ipa
```

### Stage 1: InjectorV3初期化

**ファイル**: InjectorV3.swift (135行)

```swift
init(_ bundleURL: URL, loggerType: LoggerType = .file) throws {
    self.bundleURL = bundleURL
    
    // 実行ファイル検出（アーキテクチャ自動判定）
    let executableURL = try locateExecutableInBundle(bundleURL)
    
    // Frameworks ディレクトリ検出
    let frameworksDirectoryURL = try locateFrameworksDirectoryInBundle(bundleURL)
    
    // bundle ID + team ID抽出（コード署名から）
    let appID = try identifierOfBundle(bundleURL)
    let teamID = try teamIdentifierOfMachO(executableURL) ?? ""
    
    // 一時ディレクトリ生成（UUID-based）
    temporaryDirectoryURL = Self.temporaryRoot
        .appendingPathComponent(UUID().uuidString, isDirectory: true)
    
    try? FileManager.default.createDirectory(
        at: temporaryDirectoryURL, 
        withIntermediateDirectories: true
    )
}
```

**重要な検出処理**:
- `locateExecutableInBundle()`: Mach-Oバイナリを探す（app内に複数ある可能性）
- `identifierOfBundle()`: Info.plist から CFBundleIdentifier を抽出
- `teamIdentifierOfMachO()`: Mach-Oの code signing slot から Team ID を取得

**Dopamineでの対応**: この段階では iOS/脱獄検出なし。ただし dylib内では脱獄判定可能。

### Stage 2: 前処理 - preprocessAssets()

**InjectorV3+Inject.swift の冒頭**

```swift
private func preprocessAssets() throws {
    // 1. dylib/framework/その他を分類
    let dylibURLs: [URL] = /* .dylib ファイル抽出 */
    let frameworks: [URL] = /* .framework バンドル抽出 */
    
    // 2. Substrate.framework自動組み込み
    //    （TrollFoolsにbundledされている）
    let substratePath = Bundle.main
        .path(forResource: "Substrate", ofType: "framework")
    if let substratePath = substratePath {
        try FileManager.default.copyItem(
            atPath: substratePath,
            toPath: frameworksDirectoryURL
                .appendingPathComponent("Substrate.framework").path
        )
    }
    
    // 3. arm64/arm64e スライス検出
    for dylib in dylibURLs {
        let arch = try detectArchitecture(dylib)
        // arm64 のみ: continue
        // arm64e: PAC検証が必要
    }
}
```

**なぜSubstrate組み込みか**:
- Logos %hook を使うには Substrate.framework が必須
- TrollFools は自動でbundle化している
- ユーザーが手動でSubstrate入手の手間が減る

### Stage 3: 実際のdylib注入 - injectInto()

#### 3.1: 保護レベル検出

```swift
private func detectProtectedMachO(at url: URL) -> Bool {
    // Mach-Oヘッダから flags を読む
    let data = try Data(contentsOf: url)
    let header = (data.withUnsafeBytes { ptr in
        ptr.load(as: mach_header_64.self)
    })
    
    // flags から code sign 情報を抽出
    let codeSignOffset = /* LC_CODE_SIGNATURE を検索 */
    let codeSignData = data[codeSignOffset...]
    
    // CS_HARD フラグをチェック
    if (csData.flags & CS_HARD) != 0 {
        return true  // 硬化済み（リンカが署名検証）
    }
    
    if (csData.flags & CS_RESTRICT) != 0 {
        return true  // 制限付き実行
    }
    
    if (csData.flags & CS_KILL) != 0 {
        return true  // 無効署名で即座にkill
    }
    
    return false
}
```

**検出対象フラグ**:
- `CS_HARD`: ハードコードされた署名検証
- `CS_RESTRICT`: 実行時に署名再検証
- `CS_KILL`: 署名不正で即座プロセス終了
- `CS_PLATFORM_BINARY`: プラットフォームバイナリ（修正不可）

**脱獄環境での無効化**: 脱獄されていれば、署名チェック自体がpatchされている場合が多い。ただしTrollFoolsは非脱獄対応なので、これらをすべてチェックする必要がある。

#### 3.2: Load Command挿入メカニズム

**鍵となる3つの操作**:

##### 操作1: @executable_path/Frameworks追加

```swift
func cmdInsertLoadCommandRuntimePath() {
    // LC_RPATH コマンドを新規作成
    let rpathCmd = linkedit_data_command(
        cmd: LC_RPATH,
        cmdsize: UInt32(MemoryLayout<linkedit_data_command>.size 
                       + "@executable_path/Frameworks".utf8.count + 1),
        dataoff: ...,
        datasize: ...
    )
    
    // Load Commands領域に挿入
    // （他のコマンドをずらして詰める）
    insertLoadCommand(rpathCmd)
}
```

**なぜ必要**:
- dyld が dylib を検索する際、`@executable_path` は実行ファイルのディレクトリ
- `@executable_path/Frameworks` に dylib を配置すれば dyld が自動検索

##### 操作2: LC_LOAD_DYLIB挿入

```swift
func cmdInsertLoadCommandDylib(dylibPath: String, 
                               weak: Bool = false) {
    let cmd: UInt32 = weak ? LC_LOAD_WEAK_DYLIB : LC_LOAD_DYLIB
    
    // dylib_command 構造体構築
    var dylibCmd = dylib_command(
        cmd: cmd,
        cmdsize: UInt32(
            MemoryLayout<dylib_command>.size + 
            dylibPath.utf8.count + 1
        ),
        dylib: dylib(
            name: lc_str(offset: 24),  // dylib_command直後から文字列
            timestamp: 2,               // 未使用（互換性のため）
            current_version: 0x00010000,
            compatibility_version: 0x00010000
        )
    )
    
    // Load Commands領域に挿入
    var data = Data()
    data.append(withUnsafeBytes(of: &dylibCmd) { Data($0) })
    data.append(dylibPath.utf8)
    data.append(0)  // null終端
    
    insertLoadCommand(data)
}
```

**LC_LOAD_DYLIB vs LC_LOAD_WEAK_DYLIB**:
- `LC_LOAD_DYLIB`: dylib が見つからない → アプリクラッシュ
- `LC_LOAD_WEAK_DYLIB`: dylib が見つからない → 無視して続行（graceful degradation）

TrollFools では両方をサポート（オプション）。

##### 操作3: パス正規化

```swift
func cmdChangeLoadCommandDylib(oldPath: String, 
                               newPath: String) {
    // 既存の LC_LOAD_DYLIB コマンドを検索
    let lcLoadCmd = findLoadCommand(cmd: LC_LOAD_DYLIB)
    
    // パス文字列を置き換え（サイズ変更があるので詰め込む）
    var newCmds = UnsafeMutableBufferPointer<UInt8>(...)
    
    // 古いパスを新しいパスに置き換え
    // 例: /usr/lib/system/libsystem_kernel.dylib 
    //  → @executable_path/Frameworks/libsystem_kernel.dylib
    replaceString(in: &newCmds, old: oldPath, new: newPath)
}
```

**なぜ正規化**:
- dylib が元々 `/var/containers/Bundle/Application/xxx/...` という絶対パスを持つ場合、他デバイスでは無効
- `@executable_path/` に統一することでポータビリティ確保

#### 3.3: バイナリの上書き

```swift
func applyChanges() {
    // 1. 修正内容をステージングファイルに書き込み
    try modifiedBinaryData.write(to: stagingURL)
    
    // 2. 署名検証（修正後のコード署名が有効か）
    let signatureValid = try verifySignature(stagingURL)
    if !signatureValid {
        try FileManager.default.removeItem(at: stagingURL)
        throw InjectionError.signatureValidationFailed
    }
    
    // 3. Atomic置き換え（エラーなら rollback）
    try? FileManager.default.removeItem(at: executableURL)
    try FileManager.default.copyItem(
        at: stagingURL,
        to: executableURL
    )
}
```

**Atomic性**:
- `staging` → 本体の2ステップ
- 途中でエラー → staging を削除し、本体は未修正
- 途中でクラッシュ → 古い本体が残る

---

## Part 3: 実行時 - dyld の自動ロード

### dyld が実行ファイルを起動した時

```
1. dyld が実行ファイルのMach-Oヘッダ読み込み
2. Load Commands を順番に処理
   ├─ LC_SEGMENT_64 → メモリマッピング
   ├─ LC_LOAD_DYLIB @executable_path/Frameworks/MyDylib.dylib
   │  └─ dyld が MyDylib.dylib を検索・mmap・ロード
   ├─ LC_LOAD_DYLIB /usr/lib/libc++.dylib
   │  └─ システムdylib ロード
   └─ ...
3. 全dylib のシンボル解決（lazy binding）
4. 初期化関数実行
   ├─ LC_ROUTINES_64 の関数
   ├─ 各dylib の+load()
   └─ __mod_init_func セクション
```

**TrollFools注入dylib の場合**:
```
[MyDylib.dylib -init_routine]
    ↓
[Substrate.framework -init_routine]
    ↓
[Substrate が MSHookMessageEx 機能初期化]
    ↓
[MyDylib の static constructor で Logos %hook登録]
    ↓
[アプリメイン関数 main() へ]
```

### @executable_path の解決

```c
// dyld内部での処理（簡略化）

const char *dylibPath = "@executable_path/Frameworks/MyDylib.dylib";

// @executable_path を実行ファイルの絶対パスに置き換え
char *expanded = malloc(PATH_MAX);
snprintf(expanded, PATH_MAX, "%s/Frameworks/MyDylib.dylib", 
         dirname(executablePath));
// 結果: /var/containers/Bundle/Application/XXX/App.app/Frameworks/MyDylib.dylib

// ファイルを検索・stat
struct stat st;
if (stat(expanded, &st) == 0) {
    // ファイル存在 → mmap
    void *handle = mmap(NULL, st.st_size, PROT_READ, 
                        MAP_PRIVATE, fd, 0);
}
```

**各デバイスでの自動適応**:
- デバイスAの実行ファイルパス: `/var/containers/.../A.app/MyDylib.dylib`
- デバイスBの実行ファイルパス: `/private/var/containers/.../A.app/MyDylib.dylib`
- 両方とも `@executable_path` が自動解決される

---

## Part 4: Core Trust / Signature Bypass の仕組み

### TrollStore が可能にしたこと

TrollFools が **非脱獄で** dylib注入できる理由:

1. **TrollStore** (Trolltech制作) が存在
   - App Store バイパス機構
   - 独自の署名検証ロジック
   - "Root Helper" 経由での privileged操作

2. **修正されたアプリの再署名**
   ```swift
   // アプリが修正された場合
   let codesignTask = Process()
   codesignTask.executableURL = URL(fileURLWithPath: "/usr/bin/codesign")
   codesignTask.arguments = [
       "-f", "-s", "-",  // - = ad-hoc署名
       binaryPath
   ]
   try codesignTask.run()
   ```

3. **Ad-hoc 署名**
   - 開発者証明書不要
   - `codesign -f -s - app` で即署名
   - ローカル実行のみ有効（配布不可）

### Core Trust とは

```
┌─────────────────────────────────────────┐
│ dyld/loader                             │
│   ↓                                      │
│ Code Signature 検証                      │
│   ├─ mmap した dylib のハッシュ計算      │
│   ├─ Mach-O署名スロットと比較           │
│   └─ コード署名フラグチェック           │
│       └─ CS_HARD / CS_RESTRICT / ...  │
│   ↓                                      │
│ デジタル署名が有効？                     │
│   ├─ YES → 実行許可                     │
│   └─ NO → 署名不正エラー/kill          │
└─────────────────────────────────────────┘

脱獄環境では dyld自体が patch されている：
└─ 署名チェック → 常に pass
```

**非脱獄での対策**:
TrollStore は以下を実現：
- 修正されたアプリを正規署名で再署名
- インストール後も署名維持
- dyld が署名検証時に OK を返す

---

## Part 5: Substrate.framework - Logos %hook の実装

### Substrate とは

Cydia Substrate (元Cydia MobileSubstrate) = Runtime method swizzling フレームワーク

```c
// 基本API: MSHookMessageEx()

// Old = 元のメソッド実装
// New = 新しい実装
// Old = NULL の場合、元の実装を取得して返す

MSHookMessageEx(
    NSClassFromString(@"UIApplication"),  // 対象クラス
    @selector(applicationDidFinishLaunchingWithOptions:),
    (IMP)new_applicationDidFinishLaunchingWithOptions,
    (IMP *)&old_applicationDidFinishLaunchingWithOptions
);
```

### Logos が生成するコード

```objc
// Logos 記法
%hook UIApplication
- (BOOL)applicationDidFinishLaunchingWithOptions:(NSDictionary *)options {
    NSLog(@"Before orig");
    BOOL result = %orig;
    NSLog(@"After orig");
    return result;
}
%end
```

↓ **Logos前処理器が下記に変換**

```objc
// 元の実装へのポインタ
static BOOL (*old_applicationDidFinishLaunchingWithOptions)(
    id, SEL, NSDictionary *);

// 新しい実装
static BOOL new_applicationDidFinishLaunchingWithOptions(
    id self, SEL _cmd, NSDictionary *options) {
    NSLog(@"Before orig");
    BOOL result = old_applicationDidFinishLaunchingWithOptions(
        self, _cmd, options);
    NSLog(@"After orig");
    return result;
}

// %ctor 内で hook 登録
MSHookMessageEx(
    NSClassFromString(@"UIApplication"),
    @selector(applicationDidFinishLaunchingWithOptions:),
    (IMP)new_applicationDidFinishLaunchingWithOptions,
    (IMP *)&old_applicationDidFinishLaunchingWithOptions
);
```

### Runtime Method Swizzling の実装

```c
// MSHookMessageEx 内部（簡略版）

// 1. 既存メソッドを method_getImplementation() で取得
original_IMP = method_getImplementation(
    class_getInstanceMethod(targetClass, selector)
);

// 2. 元の実装を caller に返す
if (original_p != NULL) {
    *original_p = original_IMP;
}

// 3. 新しい実装に置き換え
method_setImplementation(
    class_getInstanceMethod(targetClass, selector),
    new_IMP
);
```

**重要**: これは **Objective-C ランタイム** の機能を使っている
- コンパイル時の最適化なし
- 動的ディスパッチ（vtableではなく objc_msgSend）
- 実行時にメソッドテーブルを修正

---

## Part 6: Dopamine環境での TrollFools の制限

### 脱獄済みでの差

脱獄環境ではこのような複雑さは**不要**:

| 項目 | 脱獄環境 | 非脱獄環境 |
|------|---------|----------|
| dylib注入 | runningboardd フック（プロセス全体） | アプリバイナリ改ざん（個別） |
| Code署名 | 無視・ByPass | 再署名必須 |
| Root アクセス | ✓ | × |
| /var 書き込み | ✓ | × (App Group のみ) |
| 複数プロセス | 1つの Tweak → 全プロセス | 各アプリで個別 Tweak |

### Dopamine rootless での TrollFools 利用

```
[Dopamine rootless]
  ├─ /var/jb に jailbreak files
  ├─ /usr/lib/TweakInject に Tweak dylib
  └─ MSHookMessageEx で hook

[TrollFools]
  ├─ 各アプリバイナリに dylib 埋め込み
  ├─ @executable_path/Frameworks/ に配置
  └─ dyld が自動ロード
```

**の間に関係**:
- TrollFools は脱獄状態と **独立して動作** 可能
- 同じアプリに対して両方適用することもできる
- ただし両方から同じメソッド hook するとコンフリクト

---

## Part 7: TrollFools の応用例

### 例1: App Store YouTube に機能追加

```
[YouTube.app(App Store版)]
  + @executable_path/Frameworks/YouTubeExtension.dylib
  
[YouTubeExtension.dylib]
  ├─ Substrate.framework に依存
  └─ %hook:
      ├─ -[YTMainViewController viewDidLoad]
      ├─ -[YTPlayerViewController play]
      └─ その他多数
```

### 例2: iOS 14.xでの Tweak配布

```
脱獄できない環境でも、アプリ内に Tweak dylib を埋め込めば実質Tweak化
```

---

## Part 8: TrollFools の本質的な制限

### 1. Sandbox 内での操作のみ

```
❌ できない:
├─ /var/mobile に直接書き込み
├─ /private/var に log 出力
├─ システムフレームワーク改ざん
├─ 他のプロセスへのアクセス
└─ kernel call

✅ できる:
├─ NSUserDefaults (App Group)
├─ アプリ内ディレクトリへの読み書き
├─ 独自の dylib 注入
└─ 他のアプリの dylib に依存しない操作
```

### 2. Entitlements 拡張できず

```
アプリが元々持っていない entitlement は追加不可:
❌ ネットワーク拡張
❌ VPN フィルタリング
❌ HomeKit ブリッジ
❌ Health データアクセス

✅ 元々あれば:
├─ Photos (Photo entitlement あり)
├─ Camera (Camera entitlement あり)
└─ Location (NSLocationWhenInUseUsageDescription)
```

### 3. dylib サイズの制限

```
アプリバンドルサイズ制限:
- App Store: 4 GB (~4000 MB)
- 大規模dylib追加で容量圧迫

修正後のサイズ:
YouTube: 200+ MB → +50 MB dylib = 250+ MB
```

---

## まとめ: TrollFools の革新性

### 非脱獄で何を実現したか

```
従来: App Store app は固定機能のみ
      ┌──────────────────┐
      │ YouTube.app      │
      │ (App Store)      │
      └──────────────────┘
      
TrollFools 後:
      ┌──────────────────────────────┐
      │ YouTube.app (modified)       │
      ├──────────────────────────────┤
      │ + カスタムUI Tweak           │
      │ + API キー管理Tweak          │
      │ + 再生機能拡張Tweak          │
      │ + その他gadget              │
      └──────────────────────────────┘
```

### 技術的優秀性

1. **Mach-O フォーマットの完全理解** - Load Command 挿入
2. **dyld の自動ロード機構を利用** - 外部ツール不要
3. **Atomic エラーハンドリング** - 途中失敗時の rollback
4. **arm64/arm64e 自動判定** - PAC対応検討
5. **Substrate 自動バンドル化** - ユーザー手間削減

### 脱獄環境での TrollFools の無意味さ

脱獄では Tweak がシステム全体に注入される：
```
BackgroundAction.dylib
  ├─ RunningBoardd に注入 → 全プロセスを管理
  ├─ SpringBoard に注入 → UI全体を管理
  └─ 各アプリに注入 → アプリ個別制御

一つの Tweak で複数層制御 → TrollFools より効率的
```

---

**結論**: TrollFools は「脱獄できない環境での Tweak 代替案」。脱獄可能なら BackgroundAction のような「層状防御」が採用でき、複雑な Mach-O 操作は不要。

