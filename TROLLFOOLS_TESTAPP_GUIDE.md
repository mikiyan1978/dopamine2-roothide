# TrollFools 仕組みテストアプリ - 実装ガイド

## 前提条件チェック

### ✅ 脱獄済み環境（Dopamine等）の場合

**推奨**: BackgroundAction方式で十分  
- TrollFoolsのMach-O操作は不要
- runningboardd フックの方がシンプル
- 実装: 3行のコードで完結

### ❌ 非脱獄環境（TrollStore）の場合

**必須**: TrollFoolsのMach-O操作が必須  
- 難易度: 高い（バイナリレベル操作）
- 開発環境: Xcode + Swift
- テスト環境: iOS 14+ のデバイス + TrollStore

### ✅ 学習目的での環境構築の場合

**推奨**: macOS上での「Mach-O操作シミュレータ」  
- 実デバイス不要
- 完全に制御可能
- 脱獄・署名問題なし

---

## 案1: 脱獄環境での簡易dylib注入テスト

**推奨対象**: Dopamine環境（iPhone 14 Pro + iOS 16.2）

### 1.1 テスト用dylib作成（Theos）

```bash
mkdir -p ~/theos-test
cd ~/theos-test
$THEOS/bin/nic.pl
# プロンプト選択: Tweak (iphone/tweak)
# Project: TestDylibInjection
```

#### Makefile

```makefile
TARGET := iphone:clang:latest:15.0
THEOS_PACKAGE_SCHEME = rootless
ARCHS = arm64 arm64e

include $(THEOS)/makefiles/common.mk

TWEAK_NAME = TestDylibInjection
TestDylibInjection_FILES = Tweak.xm
TestDylibInjection_INSTALL_PATH = /usr/lib/TweakInject

include $(THEOS_MAKE_PATH)/tweak.mk
```

#### Tweak.xm（最小実装）

```objc
#import <Foundation/Foundation.h>

// 注入成功ログ
%ctor {
    NSLog(@"🎯 TestDylibInjection loaded!");
    NSLog(@"Process: %@", [[NSBundle mainBundle] bundleIdentifier]);
    NSLog(@"PID: %d", getpid());
}

// サンプルフック（UIView）
%hook UIView
- (void)layoutSubviews {
    NSLog(@"✓ UIView layoutSubviews called");
    %orig;
}
%end
```

#### ビルド

```bash
make package
# → ../Packages/TestDylibInjection_0.0.1_iphoneos-arm64.deb

# デバイスにインストール
scp ../Packages/*.deb root@device.local:/tmp/
ssh root@device.local dpkg -i /tmp/TestDylibInjection*.deb

# 自動Respring
```

#### テスト

```bash
# デバイスで
ssh root@device.local

# ログ確認
log stream --predicate 'process contains[cd] "Safari"' --level debug

# または
tail -f /var/mobile/Library/Logs/System.log | grep TestDylibInjection
```

**結果**: Safari起動時に `🎯 TestDylibInjection loaded!` が出現

---

## 案2: 非脱獄用Mach-O操作テストツール（macOS）

**推奨対象**: Xcode環境がある Mac

### 2.1 プロジェクト構成

```
TrollFoolsTestApp/
├── MachOModifier.swift         (Mach-O操作ロジック)
├── DylibInjector.swift         (dylib注入メイン)
├── InjectionTarget.app/        (テスト対象アプリ)
└── TestDylib.dylib             (注入するdylib)
```

### 2.2 MachOModifier.swift（簡略版）

```swift
import Foundation

class MachOModifier {
    let binaryURL: URL
    var binaryData: Data
    
    init(binaryURL: URL) throws {
        self.binaryURL = binaryURL
        self.binaryData = try Data(contentsOf: binaryURL)
    }
    
    // Mach-O ヘッダ構造体
    struct MachHeader64 {
        var magic: UInt32        // 0xfeedfacf
        var cputype: Int32       // 0x0100000c (ARM64)
        var cpusubtype: Int32
        var filetype: UInt32     // 2 (MH_EXECUTE)
        var ncmds: UInt32        // Load Command 数
        var sizeofcmds: UInt32   // 合計サイズ
        var flags: UInt32
        var reserved: UInt32
    }
    
    // Load Command 基本構造
    struct LoadCommand {
        var cmd: UInt32          // LC_LOAD_DYLIB など
        var cmdsize: UInt32
    }
    
    // dylib_command 構造
    struct DylibCommand {
        var cmd: UInt32
        var cmdsize: UInt32
        var nameOffset: UInt32
        var timestamp: UInt32
        var currentVersion: UInt32
        var compatibilityVersion: UInt32
    }
    
    // 1. Mach-O ヘッダ読み込み
    func readHeader() -> MachHeader64 {
        return binaryData.withUnsafeBytes { ptr in
            ptr.load(as: MachHeader64.self)
        }
    }
    
    // 2. Load Command を全て列挙
    func enumerateLoadCommands() -> [(offset: Int, cmd: LoadCommand)] {
        var results: [(offset: Int, cmd: LoadCommand)] = []
        
        let header = readHeader()
        var offset = MemoryLayout<MachHeader64>.size
        
        for _ in 0..<header.ncmds {
            let cmd = binaryData.withUnsafeBytes { ptr in
                ptr.load(fromByteOffset: offset, as: LoadCommand.self)
            }
            results.append((offset: offset, cmd: cmd))
            offset += Int(cmd.cmdsize)
        }
        
        return results
    }
    
    // 3. LC_LOAD_DYLIB コマンドを探す
    func findDylibCommand(containing: String) -> (offset: Int, cmd: LoadCommand)? {
        let cmds = enumerateLoadCommands()
        let LC_LOAD_DYLIB: UInt32 = 0xc  // Apple定数
        
        for (offset, cmd) in cmds {
            if cmd.cmd == LC_LOAD_DYLIB {
                // dylib_command の直後に文字列がある
                let stringOffset = offset + MemoryLayout<DylibCommand>.size
                let stringData = binaryData.subdata(in: stringOffset..<binaryData.count)
                let path = String(cString: stringData)
                
                if path.contains(containing) {
                    return (offset, cmd)
                }
            }
        }
        return nil
    }
    
    // 4. 新しい LC_LOAD_DYLIB コマンドを挿入
    func insertDylibCommand(path: String) throws {
        let LC_LOAD_DYLIB: UInt32 = 0xc
        let LC_RPATH: UInt32 = 0x1c
        
        // dylib パス + null terminator
        let pathData = (path as NSString).utf8String!
        let pathLength = strlen(pathData) + 1
        
        // dylib_command + パス文字列
        var dylibCmd = DylibCommand(
            cmd: LC_LOAD_DYLIB,
            cmdsize: UInt32(MemoryLayout<DylibCommand>.size + pathLength),
            nameOffset: 24,  // dylib_command の直後
            timestamp: 2,
            currentVersion: 0x00010000,
            compatibilityVersion: 0x00010000
        )
        
        // コマンドをバイナリに追加
        var newData = Data()
        newData.append(withUnsafeBytes(of: &dylibCmd) { Data($0) })
        newData.append(Data(bytes: pathData, count: pathLength))
        
        // Mach-O ヘッダの ncmds, sizeofcmds を更新
        updateMachHeader(
            incrementNcmds: 1,
            incrementSizeofcmds: Int(dylibCmd.cmdsize)
        )
        
        // Load Commands 領域の末尾に追加
        binaryData.append(newData)
        
        print("✓ Inserted LC_LOAD_DYLIB: \(path)")
    }
    
    // 5. Mach-O ヘッダ更新
    private func updateMachHeader(incrementNcmds: UInt32, incrementSizeofcmds: Int) {
        var header = readHeader()
        header.ncmds += incrementNcmds
        header.sizeofcmds += UInt32(incrementSizeofcmds)
        
        let headerData = withUnsafeBytes(of: &header) { Data($0) }
        binaryData.replaceSubrange(0..<MemoryLayout<MachHeader64>.size, with: headerData)
    }
    
    // 6. 修正されたバイナリを保存
    func saveToBinary(at url: URL) throws {
        try binaryData.write(to: url)
        print("✓ Binary saved to \(url.path)")
    }
}
```

### 2.3 DylibInjector.swift（メイン処理）

```swift
import Foundation

class DylibInjector {
    let targetAppPath: String
    let dylibPath: String
    
    init(targetAppPath: String, dylibPath: String) {
        self.targetAppPath = targetAppPath
        self.dylibPath = dylibPath
    }
    
    func inject() throws {
        // 1. アプリバンドル内の実行ファイルを探す
        let infoPlistPath = "\(targetAppPath)/Info.plist"
        guard FileManager.default.fileExists(atPath: infoPlistPath) else {
            throw NSError(domain: "InjectionError", 
                         code: 1, 
                         userInfo: [NSLocalizedDescriptionKey: "Not an app bundle"])
        }
        
        // 2. CFBundleExecutable を読み込む
        let infoPlist = NSDictionary(contentsOfFile: infoPlistPath) ?? [:]
        guard let executableName = infoPlist["CFBundleExecutable"] as? String else {
            throw NSError(domain: "InjectionError", code: 2)
        }
        
        let binaryPath = "\(targetAppPath)/\(executableName)"
        let binaryURL = URL(fileURLWithPath: binaryPath)
        
        print("📦 Target app: \(targetAppPath)")
        print("🔧 Executable: \(binaryPath)")
        print("💉 Injecting: \(dylibPath)")
        
        // 3. Mach-O を読み込み
        let modifier = try MachOModifier(binaryURL: binaryURL)
        
        // 4. @executable_path/Frameworks/TestDylib.dylib を挿入
        let dylibRelativePath = "@executable_path/Frameworks/TestDylib.dylib"
        try modifier.insertDylibCommand(path: dylibRelativePath)
        
        // 5. バイナリを上書き（バックアップ作成）
        let backupURL = binaryURL.appendingPathExtension("backup")
        try FileManager.default.copyItem(at: binaryURL, to: backupURL)
        print("💾 Backup: \(backupURL.path)")
        
        // 6. 修正されたバイナリを保存
        try modifier.saveToBinary(at: binaryURL)
        
        // 7. 実際のdylibをコピー
        let frameworksDir = "\(targetAppPath)/Frameworks"
        try FileManager.default.createDirectory(
            atPath: frameworksDir,
            withIntermediateDirectories: true
        )
        
        let destDylibPath = "\(frameworksDir)/TestDylib.dylib"
        try FileManager.default.copyItem(
            atPath: dylibPath,
            toPath: destDylibPath
        )
        
        print("✅ Injection complete!")
        print("   Binary: \(binaryPath)")
        print("   Dylib: \(destDylibPath)")
    }
}
```

### 2.4 使用例

```swift
// main.swift
import Foundation

let injector = DylibInjector(
    targetAppPath: "/Applications/TestApp.app",
    dylibPath: "/tmp/TestDylib.dylib"
)

do {
    try injector.inject()
} catch {
    print("❌ Injection failed: \(error)")
}
```

### 2.5 実行

```bash
swift DylibInjector.swift MachOModifier.swift main.swift

# 出力例:
# 📦 Target app: /Applications/TestApp.app
# 🔧 Executable: /Applications/TestApp.app/TestApp
# 💉 Injecting: /tmp/TestDylib.dylib
# ✓ Inserted LC_LOAD_DYLIB: @executable_path/Frameworks/TestDylib.dylib
# ✓ Binary saved to /Applications/TestApp.app/TestApp
# ✅ Injection complete!
```

---

## 案3: Xcode プロジェクトでの統合テスト

**推奨対象**: 完全なE2Eテスト環境

### 3.1 Xcodeプロジェクト構成

```
TrollFoolsDemo.xcodeproj/
├── TargetApp (iOS App)
│   ├── Main.storyboard
│   └── ViewController.swift
├── TestDylib (Dylib)
│   ├── TestDylib.m
│   └── Substrate.framework (bundled)
└── InjectorCLI (Command-line Tool)
    ├── main.swift
    ├── MachOModifier.swift
    └── DylibInjector.swift
```

### 3.2 最小限のテストアプリ (TargetApp)

```swift
// ViewController.swift
import UIKit

class ViewController: UIViewController {
    @IBOutlet weak var label: UILabel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        label.text = "Hello, TrollFools!"
        
        // dylib注入後、このメソッドがフック対象
        testMethod()
    }
    
    func testMethod() {
        print("📱 testMethod called")
    }
}
```

### 3.3 テストDylib (Objective-C)

```objc
// TestDylib.m
#import <Foundation/Foundation.h>

%ctor {
    NSLog(@"🎯 TestDylib injected!");
}

%hook ViewController
- (void)testMethod {
    NSLog(@"🎣 Hook intercepted testMethod");
    %orig;
    NSLog(@"✓ testMethod completed");
}
%end
```

---

## 案4: PoC「Mach-O viewer」（学習用・実装簡単）

**推奨対象**: 概念理解したい人向け

### 目標: Load Commands を GUI で表示するだけ

```swift
import Cocoa

class MachOViewer: NSViewController {
    @IBOutlet weak var textView: NSTextView!
    
    func openBinary() {
        let openPanel = NSOpenPanel()
        openPanel.begin { [weak self] result in
            if result == .OK, let url = openPanel.url {
                self?.displayMachO(url)
            }
        }
    }
    
    func displayMachO(_ url: URL) {
        do {
            let data = try Data(contentsOf: url)
            var output = "📊 Mach-O Analysis\n\n"
            
            // ヘッダ解析
            let header = data.withUnsafeBytes { ptr in
                ptr.load(as: MachHeader64.self)
            }
            
            output += "Magic: 0x\(String(header.magic, radix: 16))\n"
            output += "CPU Type: \(header.cputype)\n"
            output += "File Type: \(header.filetype)\n"
            output += "Num Commands: \(header.ncmds)\n"
            output += "Size of Commands: \(header.sizeofcmds)\n\n"
            
            // Load Commands 列挙
            output += "Load Commands:\n"
            output += "─" * 50 + "\n"
            
            var offset = MemoryLayout<MachHeader64>.size
            for i in 0..<header.ncmds {
                let cmd = data.withUnsafeBytes { ptr in
                    ptr.load(fromByteOffset: offset, as: LoadCommand.self)
                }
                
                let cmdName = cmdTypeName(cmd.cmd)
                output += "[\(i)] \(cmdName)\n"
                output += "    Offset: 0x\(String(offset, radix: 16))\n"
                output += "    Size: \(cmd.cmdsize) bytes\n\n"
                
                offset += Int(cmd.cmdsize)
            }
            
            textView.string = output
            
        } catch {
            textView.string = "❌ Error: \(error)"
        }
    }
    
    private func cmdTypeName(_ cmd: UInt32) -> String {
        switch cmd {
        case 0x1: return "LC_SEGMENT"
        case 0x19: return "LC_SEGMENT_64"
        case 0xc: return "LC_LOAD_DYLIB"
        case 0x1c: return "LC_RPATH"
        case 0x5: return "LC_DYLINKER"
        default: return "LC_UNKNOWN(0x\(String(cmd, radix: 16)))"
        }
    }
}
```

**結果**: Safari.app などを開くと Load Commands が可視化される

---

## 推奨実装順序

### 環境別

**脱獄済み(Dopamine)**:
```
案1 (Theos dylib) → 最簡単・即テスト可能
└─ 5分でテスト完了
```

**脱獄なし + Xcode環境**:
```
案2 (macOS Swift) → 中程度・理解深い
└─ 1時間程度で動作確認
```

**学習目的のみ**:
```
案4 (PoC Viewer) → 最簡単
└─ 30分で Load Commands 理解
```

**完全なエコシステム構築**:
```
案3 (Xcode統合) → 最複雑・本格的
└─ 3時間程度で E2E テスト
```

---

## 主な実装ポイント

### 1. Mach-O 操作の核

```swift
// 3ステップで十分
1. binaryData.withUnsafeBytes で ヘッダ読み込み
2. offset += cmdsize で Load Commands イテレート
3. binaryData.replaceSubrange で修正・保存
```

### 2. Load Command 形式

```
全てのコマンドは共通構造:
┌─────────────────┐
│ cmd (4 bytes)   │ ← コマンド種別 (LC_LOAD_DYLIB など)
│ cmdsize (4 bytes)│ ← このコマンド全体のサイズ
│ ... 可変長データ  │
└─────────────────┘
```

### 3. dylib パス

```
@executable_path/Frameworks/MyDylib.dylib

dyld が起動時に自動展開:
/var/containers/.../App.app/Frameworks/MyDylib.dylib
```

### 4. 最小の Dylib（ログのみ）

```objc
%ctor {
    NSLog(@"Dylib loaded!");
}
```

署名・複雑な hook 不要で動作確認可能

---

## まとめ

| 案 | 難易度 | テスト時間 | 学習効果 | 推奨環境 |
|---|-------|---------|--------|--------|
| 案1 (Theos) | ⭐ | 5分 | 低 | 脱獄済み |
| 案2 (Swift) | ⭐⭐ | 1h | 高 | macOS + Xcode |
| 案3 (Xcode) | ⭐⭐⭐ | 3h | 最高 | iOS dev環境 |
| 案4 (PoC) | ⭐ | 30分 | 中 | macOS |

**最初の1歩**: 脱獄済みなら **案1 (Theos)** で即テスト可能！
