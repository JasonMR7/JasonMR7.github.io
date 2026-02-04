---
layout: post
title: iOS 18 新能力：锁屏拍摄扩展与 ControlWidget 详解
date: 2024-06-10 18:49:00.000000000 +09:00
---

> 本文对应 **iOS 18** 与 **WWDC 2024** 推出的 Locked Camera Capture Extension、ControlWidget 等能力，成文于该特性发布节点。

---

## 一、背景介绍

iOS 18 新增的**锁屏拍摄扩展（Locked Camera Capture Extension）**，打破了传统相机功能的使用限制 —— 用户无需解锁设备，就能快速启动第三方应用的相机功能并完成内容捕捉。

这一扩展将应用的相机体验延伸至**控制中心、锁屏界面、动作按钮**三大场景，大幅降低用户调用核心相机功能的操作成本，为具备拍摄能力的第三方应用提供了更便捷的用户触达路径。

![锁屏拍摄扩展与 ControlWidget](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera1.webp)

---

## 二、前置知识：ControlWidget 详解

要掌握锁屏拍摄扩展，首先需要理解它的「入口载体」——**ControlWidget**。

<video src="https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera2.mov" controls playsinline></video>

在 iOS 18 中，苹果为控制中心、锁定屏幕开放了更多交互能力，用户无需打开完整应用，就能在这些系统界面直接调用轻量化控件按钮完成核心操作，这类控件按钮就是 **ControlWidget**。

### 2.1 如何创建 ControlWidget

创建 ControlWidget 需基于 Xcode 操作，采用 **SwiftUI** 实现，核心步骤如下：

![创建 ControlWidget](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera3.webp)

1. 在 Xcode 中为项目添加 Target，选择「**Widget Extension**」；
2. 定义继承自 `WidgetBundle` 的类，声明要添加的 ControlWidget；
3. 实现自定义 ControlWidget，配置交互按钮与对应的 Intent 动作。

```swift
// 主入口：声明 Widget Bundle
@main
struct ControlWidgetBundle: WidgetBundle {
    var body: some Widget {
        ScanControlWidget() // 自定义的扫码 ControlWidget
    }
}

// 自定义扫码 ControlWidget
struct ScanControlWidget: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(
            kind: "com.alipay.scan" // 唯一标识，建议采用反向域名格式
        ) {
            // 自定义按钮样式：文字+系统图标（支持自定义 SVG）
            ControlWidgetButton(action: ScanIntent()) {
                Label("扫码",
                      systemImage: "qrcode.viewfinder")
            }
        }
    }
}
```

### 2.2 配套 Intent 与工具类实现

ControlWidget 的点击动作依赖 **AppIntent** 驱动，同时需注意 **Target 归属配置**：

以下 `ScanIntent.swift` 和 `ScanManager.swift` 会被主应用和 WidgetExtension 共同引用，需在 Xcode 的「**Target Membership**」中同时勾选「主 Target」和「WidgetExtension」，否则会出现编译报错。

```swift
// 扫码动作对应的 Intent
struct ScanIntent: AppIntent {
    // Intent 显示标题
    static var title: LocalizedStringResource = "扫码"
    
    // 核心执行逻辑：与主应用进行交互
    func perform() async throws -> some IntentResult {
        // 调用主 Target 中的工具类，传递数据
        ScanManager.shared.updateData(newMessage: "Hello, World!\nReceive a message from control widget")
        return .result();
    }
    
    // 配置：执行 Intent 时打开主应用
    static let openAppWhenRun: Bool = true
}

// 主应用中的数据管理单例（用于与 Extension 交互）
class ScanManager: ObservableObject {
    static var shared = ScanManager() // 单例实例
    @Published var message: String = "Hello, World!" // 可观察数据，用于界面刷新
    
    // 更新数据方法
    func updateData(newMessage: String) {
        message = newMessage
    }
}
```


### 2.3 效果展示

<video src="https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera4.mov" controls playsinline></video>

运行项目后，即可在设备的控制中心或锁屏界面看到创建的 `ScanControlWidget`，点击即可触发扫码 Intent 并打开主应用。

---

## 三、核心能力：LockedCameraCapture 锁屏拍摄扩展

### 3.1 功能概述

iOS 18 提供的 **LockedCameraCapture** 框架，专为第三方相机类应用打造 —— 它允许应用从锁屏直接打开轻量化拍摄界面（无需进入完整主应用），用户解锁后则无缝跳转至主应用，实现「锁屏快速拍摄」与「主应用深度编辑」的衔接。

### 3.2 如何创建锁屏拍摄扩展

1. Xcode 中添加 Target，选择「**Capture Extension**」；
2. 实现继承自 `LockedCameraCaptureExtension` 的主类，配置拍摄场景；
3. 采用 SwiftUI 构建自定义拍摄视图，复用主应用视图可保证 UI 一致性；
4. **关键配置**：在 Extension 和主应用的 `Info.plist` 中同时添加摄像头访问权限（`NSCameraUsageDescription`），否则无法正常调用相机。

```swift
// 锁屏拍摄扩展主入口
@main
struct LockedExtension: LockedCameraCaptureExtension {
    var body: some LockedCameraCaptureExtensionScene {
        // 配置拍摄场景，传递拍摄会话实例
        LockedCameraCaptureUIScene { session in
            LockedCameraCaptureView(session: session)
        }
    }
}

// 自定义锁屏拍摄视图
struct LockedCameraCaptureView: View {
    let session: LockedCameraCaptureSession // 锁屏拍摄会话，管理相机状态、拍摄动作等
    
    var body: some View {
        // 复用主应用的 ContentView，保证 UI 风格一致
        ContentView(configProvider: AppStorageConfigProvider(session))
            .environment(\.scenePhase, .active) // 配置场景状态为活跃
            .environment(\.openMainApp, OpenMainAppAction(session: session)) // 注入打开主应用的动作
    }
}
```

### 3.3 两种触发效果展示

**（锁屏状态触发效果）**

![锁屏状态触发效果](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera5.webp)

iOS 设备锁屏界面，点击「扫码 / 拍摄」ControlWidget 后，直接进入轻量化拍摄界面（无主应用启动动画），标注出拍摄取景框、快门按钮等核心元素。

**（解锁状态触发效果）**

<video src="https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera6.mov" controls playsinline></video>

iOS 设备解锁后的主屏幕 / 控制中心，点击「扫码 / 拍摄」ControlWidget 后，直接启动应用主界面并进入拍摄功能，展示主应用的完整拍摄界面。

完成上述配置后，点击对应的 ControlWidget 会出现两种场景：

- **设备锁屏状态**：直接进入 Capture Extension 轻量化拍摄界面；
- **设备解锁状态**：直接跳转至应用主界面的对应拍摄功能。

### 3.4 专属相机捕获意图：CameraCaptureIntent

锁屏拍摄扩展的触发，同样依赖 ControlWidget，但对应的 Intent 需继承 iOS 18 新的系统意图 **CameraCaptureIntent**（区别于普通的 `AppIntent`）。

同时需注意 **Target 归属配置升级**：该 Intent 会被「主 Target」「WidgetExtension」「Capture Extension」三方引用，需在 Xcode 的「Target Membership」中同时勾选这三个 Target，避免编译报错。

```swift
// 锁屏拍摄专属 Intent
struct AppCaptureIntent: CameraCaptureIntent {
    // 自定义 App 上下文（可序列化），用于传递轻量配置信息
    struct MyAppContext: Codable {
        var cameraPosition: CameraPosition = .front // 相机位置：默认前置
    }
    
    // 关联自定义 App 上下文
    typealias AppContext = MyAppContext
    
    // Intent 配置信息
    static let title: LocalizedStringResource = "AppCaptureIntent"
    static let description = IntentDescription("Capture photos with MyApp.")
    
    // 核心执行逻辑
    @MainActor
    func perform() async throws -> some IntentResult {
        return .result()
    }
}
```

---

## 四、关键难点：锁屏拍摄扩展与主应用的数据交互

传统 Extension 通常通过 **App Group** 机制共享 UserDefaults 和文件，但锁屏拍摄扩展因隐私安全限制，**禁止使用 App Group 功能**。

针对「拍摄照片信息传输」这一核心需求，目前有两种可行方案：

### 4.1 方案一：CameraCaptureIntent.appContext（轻量数据传输）

如上述 `AppCaptureIntent` 中定义的 `MyAppContext`，就是 `CameraCaptureIntent` 提供的轻量数据传递载体，可通过 `AppCaptureIntent.updateAppContext(appContext)` 方法更新上下文信息。

**注意事项：**

- **数据上限**：仅支持 **4KB 以内**的轻量数据，无法直接传输照片、视频等大文件；
- **适用场景**：传递拍摄配置信息，如相机前后置切换、拍摄模式（照片 / 视频）、分辨率设置等。

### 4.2 方案二：PhotoKit（大文件 / 照片传输）

这是传输拍摄照片的**核心方案**，核心逻辑是「**先存系统相册，再主应用读取**」，步骤如下：

1. 锁屏拍摄扩展中完成拍摄后，通过 **PhotoKit** 将照片 / 视频存储到系统相册；
2. 调用 `LockedCameraCaptureSession` 的专属方法，解锁设备并打开主应用：

```swift
// 解锁并打开主应用
session.openApplication(for: NSUserActivity(activityType: NSUserActivityTypeLockedCameraCapture))
```

3. 主应用启动后，通过 PhotoKit 读取系统相册中刚存储的拍摄内容，进行后续编辑、上传等操作。

![数据交互示意](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera7.webp)

![PhotoKit 与主应用衔接](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera8.webp)

![数据流转流程](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera9.webp)

---

## 五、思考与展望

![锁屏拍摄扩展场景](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera10.webp)

![ControlWidget 与流量入口](https://github.com/JasonMR7/JasonMR7.github.io/raw/master/assets/images/2024-06-10-iOS-18-锁屏拍摄扩展与-ControlWidget-详解/camera11.webp)

iOS 18 开放控制中心、锁屏首页的控件能力，不仅为用户提供了更便捷的操作体验，更让这两个场景成为第三方 iOS 应用的**全新流量入口**。

以支付宝为例，其核心功能如「扫码」「探一探」「付款码」等，都非常适合配置对应的 ControlWidget：

- **抢占流量入口**：让用户在锁屏 / 控制中心一键调用核心功能，减少操作路径，提升用户粘性；
- **培养用户心智**：让用户形成「锁屏点击 = 快速使用支付宝核心功能」的使用习惯，强化品牌认知；
- **延伸功能场景**：结合锁屏拍摄扩展，还可拓展「锁屏快速扫码付款」「锁屏拍摄识别发票」等新场景。

---

## 六、参考资料

- [WWDC 2024 - Build a great Lock Screen camera capture experience](https://developer.apple.com/videos/play/wwdc2024/)
- [WWDC 2024 - Extend your app's controls across the system](https://developer.apple.com/videos/play/wwdc2024/)
- [JuniperPhoton / LockedCameraCaptureExtensionDemo](https://github.com/JuniperPhoton/LockedCameraCaptureExtensionDemo)
