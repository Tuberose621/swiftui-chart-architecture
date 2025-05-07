## Swift 相关知识点学习


## 一、Main Actor 隔离

```swift
.task {
    await withTaskGroup(of: Void.self) { group in
        group.addTask {
            uploadData()
        }

        group.addTask {
            // Disable Idle Timer to avoid auto-locking screen  
            UIApplication.shared.isIdleTimerDisabled = true
        }
    }
}

```

遇到的这个编译错误：

```
Main actor-isolated class property 'shared' can not be mutated from a Sendable closure
```

是因为在 `.task` 这个 `modifier` 的异步闭包中访问了 `UIApplication.shared.isIdleTimerDisabled = true`，而这个属性 **受 Main Actor 隔离（Main actor-isolated）**，不能在一个并发上下文中（比如 `TaskGroup` 内部）直接修改。

---

### ⚠️ 原因分析：

- `UIApplication.shared` 是 `MainActor` 隔离的，也就是说只能在主线程上访问和修改它的属性。
- `.task` 是并发上下文，`Swift` 认为这个闭包是 `@Sendable`，意味着它可能在任意线程运行。
- 在 `TaskGroup` 的闭包中调用 `UIApplication.shared.isIdleTimerDisabled = true`，就是从非主线程修改主线程隔离属性，因此编译器报错。

---

### ✅ 解决方案：

#### 方法一：在主线程中设置 `isIdleTimerDisabled`

你可以把它包裹到 `MainActor.run` 中运行：

```swift
group.addTask {
    await MainActor.run {
        UIApplication.shared.isIdleTimerDisabled = true
    }
}
```

#### 方法二：把 `UIApplication.shared.isIdleTimerDisabled = true` 放到 `.onAppear` 或 `DispatchQueue.main.async` 中

其实完全可以避免放在 `task` 的 `group` 中，像这样写也可以：

```swift
.onAppear {
    DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
        self.onboarding()
        UIApplication.shared.isIdleTimerDisabled = true
    }
}
```

或者直接：

```swift
.onAppear {
    UIApplication.shared.isIdleTimerDisabled = true
}
```

---

### ✅ 总结建议

由于设置 `isIdleTimerDisabled` 本身并不耗时，也不需要并发处理，**直接放在 `.onAppear` 中或者使用 `MainActor.run` 包裹，确保线程安全即可。**

---


### 一、先解释：`@MainActor` 是什么？

#### 👉 它的作用：
`@MainActor` 是 `Swift` 的一种 **actor 隔离机制**，用于确保标记的变量或方法 **只能在主线程上被访问或修改**，适用于：

- `UI` 相关的属性（比如 `SwiftUI / UIKit` 的 `UI` 状态）
- 线程安全性要求高的变量（尤其和主线程交互）

```swift
@MainActor
var title: String  // 只能在主线程读取或写入
```

在原来的代码里，不用加 `@MainActor` 也可以，只要**保证在主线程上更新 UI 状态或平台变量即可**。

---
---


#### 同步耗时操作阻塞主线程

```swift

init() {
    // Check if UserDefaults contains platform key
    if let platform = UserDefaults.standard.string(forKey: "Platform"), let platform = Platform(rawValue: platform) {
        self.platform = platform
    } else {
        self.platform = MyCommand.shared.searchPlatform()
        UserDefaults.standard.set(self.platform.rawValue, forKey: "Platform")
    }

    Task {
        switch self.platform {
        case .platform10xx:
            let _ = MyCommand.shared.initXXXListener()
        case .platform20xx:
            let _ = MyCommand.shared.initXXXEvent()
        }
    }
}
```

这个 `MyCommand.shared.searchPlatform()` 是执行命令，返回平台是`platform10xx`还是 `platform20xx`。可能比较耗时，使用异步线程调用就解决了这个问题。它为什么会影响 `view` 的启动。

---

🧠 一、为什么 `MyCommand.shared.searchPlatform()` 会影响 `view` 启动？

✅ 原因归结为一点：**它是一个**“**同步耗时操作**”，放在 **初始化阶段（`init`）阻塞主线程**，间接破坏了 `AVKit` 或 `UI` 的初始化时机。

#### 🔍 具体影响路径：

1. `init()` 是在主线程上调用的，如果其中执行了耗时操作（如 `searchPlatform()` 是同步执行 `shell` 命令或 `I/O` 的话）——
2. 会 **阻塞主线程**，导致整个 `app UI` 构建、系统服务（如 `AVAudioSession`、`AVPlayer`、`AVPiPController`）初始化延迟甚至错过系统期望的生命周期阶段。
3. 特别是 **PiP view初始化涉及的系统服务需要在合适的生命周期点启动，且依赖主线程的“响应性”**，此时如果主线程卡住，就会出现系统组件初始化失败，或者系统认为“你不支持 `PiP`”。

✅ 将 `searchPlatform()` 放到异步线程后，系统组件终于能按时正常初始化，自然也就不会影响 `PiP` 启动了。

```swift
init() {
    Task {
        await self.setupPlatformt()
    }
}

private func setupPlatform() async {
    // 1. Get platform (cache first)
    let detectedPlatform: Platform
    if let cached = UserDefaults.standard.string(forKey: "Platform"),
       let cachedPlatform = Platform(rawValue: cached) {
        detectedPlatform = cachedPlatform
    } else {
        detectedPlatform = await withCheckedContinuation { continuation in
            DispatchQueue.global(qos: .userInitiated).async {
                let result = MyCommand.shared.searchPlatform()
                continuation.resume(returning: result)
            }
        }
        
        // Caching platform values
        UserDefaults.standard.set(detectedPlatform.rawValue, forKey: "Platform")
    }
    
    // 2. Update local variables 
    self.platform = detectedPlatform
    
    // 3. Execute initialization commands according to the platform
    switch detectedPlatform {
        case .platform10xx:
            let _ = MyCommand.shared.initXXXListener()
        case .platform20xx:
            let _ = MyCommand.shared.initXXXEvent()
    }
}
```

---


### ✅ 1. `init()` 方法一般用来做什么？

#### 💡 `init()` 的本质职责：

> 构造出**有效可用的对象初始状态**，不建议在其中执行副作用重的逻辑，尤其是：
- 耗时操作，异步任务，系统权限请求，网络请求 / 文件访问，主线程 `UI` 修改

#### ❗为什么？

- `init()` 是同步执行的，若在其内执行耗时逻辑会阻塞主线程，尤其在 `App` 启动阶段；
- 如果 `init()` 失败或者行为不可预测，会导致对象构造异常，使程序逻辑失控；
- 异步操作在 `init()` 内很难正确捕获、管理生命周期（比如 `weak self`、任务取消等）；
- 构造函数不支持 `async`，所以只能硬塞 `Task` 或 `DispatchQueue`，这会让副作用不可控。

---

#### ✅ 合理的 `init()` 用法 ✅

| 合法的行为 | 说明 |
|------------|------|
| 设置默认值 | 初始化本地属性，比如 `self.platform = .unknown` |
| 从内存中恢复状态 | 例如 `UserDefaults` / 本地缓存 |
| 创建依赖对象 | 比如 `Logger()`、`URLSession()` |
| 做轻量判断逻辑 | 不会阻塞线程 |

---

#### ❌ 不推荐的 `init()` 行为 ❌

| 不推荐的行为 | 原因 |
|--------------|------|
| 执行 `shell` 命令 / `I/O` 操作 | 易阻塞主线程，且不易捕获异常 |
| 配置系统服务（如 `AVAudioSession`） | 初始化时系统组件可能尚未准备好 |
| 创建 `AVPlayer / PiP Controller` 等系统资源 | 这些往往依赖系统生命周期和 `UI` 状态 |
| 启动异步任务并期望依赖其结果 | 初始化结束后没法等结果完成，不可预测 |

---

### 🧠 2. 那耗时或异步操作放在哪更合适？

- 放在 **`Task` 中，异步执行**
- 放在 **`viewDidAppear`、`.task` modifier、`.onAppear`** 中
- 放在 **`init()` 后的 setup()/load()/start()** 方法中，延后主动调用

---
---


## 二、private(set) 和  didSet

这是 Swift 中一种非常**简洁且实用的语法组合**


### 完整语法结构分析

```swift
private(set) var currentSession: MySession? {
    didSet {
        if let currentSession {
            // ...
        }
    }
}
```

---

#### 1. `private(set)`：访问权限控制
- **含义**：`currentSession` 对**外部可读**，但只有**当前作用域（比如类）内部可以修改**。
- 目的：**保护这个值只能被类内部逻辑控制修改，外部只能观察（只读）**。

---

#### 2. `var currentSession: MySession?`
- **属性定义**：这是一个可选类型（`MySession?`），表示可能有值，也可能是 `nil`。

---

#### 3. `didSet` 是属性观察器
- **作用**：一旦 `currentSession` 被赋新值，`didSet` 块就会自动执行。
- 用于**做副作用的处理**，比如：记录日志、更新相关状态、触发其他逻辑。

---

### 4. `if let currentSession` 是 **简化版的 Optional Binding**
```swift
if let currentSession {
    // ...
}
```
等价于：
```swift
if let currentSession = currentSession {
    // ...
}
```
Swift 5.7+ 中支持这种 **简写绑定形式**。

---

### 5. `didSet` 内的逻辑说明

```swift
self.sessionStart = Date()
```
- 设置会话开始时间。

```swift
self.logPath = currentSession.folderPath + "/myLog.txt"
```
- 设置日志路径，基于当前的测试会话文件夹路径。

---

## 📌 小结：这是一个“状态属性 + 观察器 + 封装写权限”的典型用法

它实现了：
- 状态更新控制
- 自动执行副作用（如时间记录、日志输出）
- 属性值变化的封装保护（`private(set)`）
- 函数式逻辑与日志的结合

---

### 📚 可借鉴的场景
- 登录用户信息：更新 `currentUser` 时触发 `UI` 更新
- 网络会话对象：更新 `session` 时触发日志与状态同步
- 设备连接状态：更新连接状态时重设内部依赖项

---
---

## 三、串行队列 DispatchQueue

```swift
let queue = DispatchQueue(label: "com.test.queue", qos: .userInitiated)
```

这个代码中的 `DispatchQueue` 是一个串行队列（`serial queue`）。在这行代码中，`DispatchQueue(label: "com.test.queue", qos: .userInitiated)` 创建了一个串行队列，但它的使用是否异步取决于如何在队列中调度任务。

### 解释：
1. **串行队列（Serial Queue）**:
   - 串行队列按顺序执行任务，一个任务在前一个任务完成后才会执行下一个任务。即使将多个任务放入队列，它们也会逐一执行，而不会并行。
   - 通过传递 `label` 来创建一个串行队列。这里的 `com.test.queue` 只是队列的名称，`qos: .userInitiated` 是队列的质量服务（`Quality of Service`）级别，表示任务是用户发起的，应该优先执行。

2. **为什么不是异步的**：
   - 当创建了一个队列，但队列的调度模式是决定异步还是同步的。在这行代码中，队列本身没有指定任务的调度方式。
   - 如果在队列中调度任务时使用的是 **`async`**，那么任务就是异步执行的；如果使用的是 **`sync`**，则是同步执行。

例如：
```swift
let queue = DispatchQueue(label: "com.test.queue", qos: .userInitiated)

// 异步调度任务
queue.async {
    // 异步任务
    print("This is async task")
}

// 同步调度任务
queue.sync {
    // 同步任务
    print("This is sync task")
}
```

- `async` 会让任务在队列中异步执行，意味着它会立即返回，不会等待任务完成。
- `sync` 会让任务同步执行，意味着它会阻塞当前线程，直到任务完成。

总结：
- `DispatchQueue(label: "com.test.queue")` 本身只是创建了一个队列，决定是否异步或同步是在调度任务时选择 `async` 或 `sync` 来决定的。

---
---

## 四、stride(from:through:by:) 函数学习

`stride(from:through:by:)` 是 `Swift` 中用来生成一个数值序列的函数。它通过指定起始值、结束值和步长来创建一个有规律的数值序列。你可以使用它来创建一个范围内的数值集合，步长可以是正数也可以是负数。

### 函数签名：

```swift
func stride(from start: T, through end: T, by step: T) -> StrideTo<T> where T : Strideable
```

### 参数说明：

- `from start: T`：起始值。
- `through end: T`：结束值，序列会包含该值。
- `by step: T`：步长值，用于确定数值递增或递减的幅度。

### 返回值：

- 返回一个 `StrideTo<T>` 类型的序列，这个序列是一个步进的数值集合。

### 示例代码：
1. **正向步进（递增）**：
   生成从 0 到 10（包括 10）之间的数，步长为 2。
   
   ```swift
   let numbers = stride(from: 0, through: 10, by: 2)
   for number in numbers {
       print(number)
   }
   ```
   输出：
   ```
   0
   2
   4
   6
   8
   10
   ```

2. **负向步进（递减）**：
   生成从 10 到 0（包括 0）之间的数，步长为 -2。
   
   ```swift
   let numbers = stride(from: 10, through: 0, by: -2)
   for number in numbers {
       print(number)
   }
   ```
   输出：
   ```
   10
   8
   6
   4
   2
   0
   ```

3. **通过 `...` 创建闭区间**：
   `stride(from:through:by:)` 是一个闭区间（`through`），意味着它会包含结束值。

### 使用场景：
`stride` 适用于需要创建有规律的数值序列时，例如：
- 生成均匀分布的数值（比如绘图时的坐标点）。
- 处理循环或者需要指定步长的逻辑。

### 总结：
`stride(from:through:by:)` 函数是 `Swift` 提供的一个非常灵活的工具，可以用于生成带有特定步长的数值序列。它非常适合需要精确控制范围和步长的场景。

---
---

## 五、split(separator:maxSplits:omittingEmptySubsequences:)

这个方法用来把一个字符串分割成多个子字符串（`[Substring]`），根据指定的“分隔符”来进行切割。

### 🧾 函数签名：
```swift
func split(
    separator: Character,
    maxSplits: Int = .max,
    omittingEmptySubsequences: Bool = true
) -> [Substring]
```

---

### 📌 参数说明：

- `separator`: 分隔符字符。字符串会按照这个字符进行拆分。
- `maxSplits`: 最多分割多少次。默认是 `.max`，表示不限次数。
- `omittingEmptySubsequences`: 是否忽略空的子串（如连续两个分隔符之间没有内容），默认是 `true`。

---

### 🔍 这个例子的解析：
```swift
let components = content.split(separator: "?", maxSplits: 1)
```

等价于：
```swift
let components = content.split(separator: "?", maxSplits: 1, omittingEmptySubsequences: true)
```

- 把 `content` 这个字符串按 `?` 分隔；
- 最多分一次（只分一次，也就是得到两部分）；
- 忽略空字符串（如果 `?` 在最开始或中间连续出现，不会保留空部分）。

---

### 🧪 示例：

#### 例 1：正常分割
```swift
let content = "abc?def?ghi"
let parts = content.split(separator: "?", maxSplits: 1)
// parts = ["abc", "def?ghi"]
```

> 只分一次。第一次遇到 `?` 就切一刀，剩下的部分原样保留。

---

#### 例 2：分割多次（默认值）
```swift
let content = "a?b?c?d"
let parts = content.split(separator: "?")
// parts = ["a", "b", "c", "d"]
```

> 默认是 `maxSplits: .max`，会尽可能多地切。

---

#### 例 3：保留空子串
```swift
let content = "a??b"
let parts = content.split(separator: "?", omittingEmptySubsequences: false)
// parts = ["a", "", "b"]
```

---

### 🧠 总结记忆法：

- `split(separator:)` → 用某个字符拆字符串。
- `maxSplits:` → 控制“最多切几刀”。
- `omittingEmptySubsequences:` → 是否丢掉空的部分。

---
---

##  六、Swift 中使用 static

### 在 Swift 中频繁使用 `static` 有什么问题？

`static` 方法是**类型方法**，它们属于类型（`class`/`struct`/`enum`），而不属于某个实例对象。频繁使用 `static` 方法有一些 **潜在问题**：

### ❌ 可能的问题
1. **无法访问实例属性**
   - `static` 方法无法访问 `self` 或实例变量，这意味着**所有需要实例数据的逻辑都不能放在 `static` 方法中**。
   - 例如：
     ```swift
     class Example {
         var name = "Swift"

         static func printName() {
             print(name) // ❌ 编译错误，不能访问实例属性
         }
     }
     ```
   - **解决方案**：如果方法需要访问实例数据，就不应该使用 `static`，应该用**实例方法**。

2. **不适用于需要继承的情况**
   - `static` 方法**不能被子类重写**，而 `class func` 可以。
   - 例如：
     ```swift
     class Parent {
         static func greet() {
             print("Hello from Parent")
         }
     }

     class Child: Parent {
         override static func greet() {  // ❌ 报错，static 方法不能被 override
             print("Hello from Child")
         }
     }
     ```
   - **解决方案**：
     - 如果方法需要**允许子类重写**，请使用 `class func` 代替 `static func`。

3. **容易导致全局状态污染**
   - 过多 `static` 方法会让类逐渐变成**工具类（Utility Class）**，但大部分时候，**面向对象设计更鼓励使用实例方法**。
   - 例如：
     ```swift
     class Utility {
         static func format(date: Date) -> String { ... }
         static func log(message: String) { ... }
         static func validateEmail(_ email: String) -> Bool { ... }
     }
     ```
   - **问题**：
     - 这些方法全是 `static`，导致 `Utility` 变成了一个**全局工具类**。
     - **缺乏面向对象的封装性**，无法在不同实例间存储状态。
     - **难以扩展**：如果需要不同的 `log` 级别、不同的 `date format`，就必须增加参数或新增方法，代码维护困难。

   - **解决方案**：
     - **如果方法属于特定的对象实例，尽量用实例方法**。
     - **如果方法需要继承，使用 `class func`**。

---

### 什么时候适合使用 `static`？
虽然 `static` 可能带来上述问题，但**在以下场景中使用 `static` 是合适的**：

#### ✅ 1. 工具方法（Utility Methods）
- **不依赖实例，不需要存储状态** 的方法，适合用 `static`：
  ```swift
  struct MathUtil {
      static func square(_ value: Double) -> Double {
          return value * value
      }
  }

  let result = MathUtil.square(4) // ✅ 4 * 4 = 16
  ```
- `square(_:)` 方法不需要 `MathUtil` 的实例，因此 `static` 是合理的选择。

#### ✅ 2. 单例模式
- **全局唯一的实例**，适合用 `static`：
  ```swift
  class Logger {
      static let shared = Logger()

      private init() { } // 禁止外部创建实例

      func log(_ message: String) {
          print("[LOG]: \(message)")
      }
  }

  Logger.shared.log("App started") // ✅ 访问单例对象
  ```
- `static let shared` 确保 `Logger` 只有一个实例（单例模式）。
- 这里 `log(_:)` 是 **实例方法**，因为它可能需要维护日志级别等状态。

#### ✅ 3. 枚举中的静态常量
- `static` 适用于 **枚举中定义常量**：
  ```swift
  enum API {
      static let baseURL = "https://api.example.com"
      static let timeout = 30.0
  }

  let url = API.baseURL // ✅
  ```
- **避免用 `case` 表示常量**，因为 `case` 主要用于枚举值。

#### ✅ 4. 类型级计算属性
- `static` 可以用于 **全局不可变的数据**：
  ```swift
  struct Configuration {
      static var defaultTimeout: TimeInterval = 60.0
  }

  print(Configuration.defaultTimeout) // ✅ 60.0
  ```
- **注意**：如果可能改变，考虑 `class var`，这样子类可以重写：
  ```swift
  class BaseConfig {
      class var defaultTimeout: TimeInterval { return 60.0 }
  }

  class CustomConfig: BaseConfig {
      override class var defaultTimeout: TimeInterval { return 120.0 }
  }
  ```

#### ✅ 5. `static` 作为工厂方法
- **如果创建对象需要特殊逻辑**，可以用 `static func`：
  ```swift
  struct User {
      let name: String

      static func createGuestUser() -> User {
          return User(name: "Guest")
      }
  }

  let guest = User.createGuestUser() // ✅
  ```
- **相比 `init()` 方法，工厂方法可以定制创建逻辑**。

---

### 什么时候不适合用 `static`？
| 场景 | `static` 适合？ | 推荐做法 |
|------|-------------|----------|
| 需要访问实例属性 | ❌ 不适合 | 用实例方法 |
| 可能需要被子类重写 | ❌ 不适合 | 用 `class func` |
| 可能存储状态 | ❌ 不适合 | 用实例方法 |
| 需要创建对象（工厂方法） | ✅ 适合 | 可用 `static func` |
| 纯工具方法 | ✅ 适合 | 可用 `static func` |

---

### 代码如何优化？
在 `ReportConfigModel` 里，频繁使用 `static`，但它们大部分是**不依赖实例的数据解析逻辑**，所以**`static` 是合适的**。

✅ 但如果以后 `booleanValue` 需要**根据实例的设定解析数据**，建议改成实例方法：
```swift
func booleanValue(forKey key: String, in dictionary: NSDictionary) -> Bool {
    (dictionary[key] as? Bool) ?? ((dictionary[key] as? String)?.lowercased() == "true")
}
```
这样**更灵活，避免滥用 `static`**。

---

### 总结
- **适合 `static` 的场景**：
  - **工具方法**（不需要实例状态）。
  - **全局常量**（如 API 地址）。
  - **单例模式**（如 `Logger.shared`）。
  - **类型级计算属性**（如 `Configuration.defaultTimeout`）。
  - **工厂方法**（如 `User.createGuestUser()`）。

- **不适合 `static` 的场景**：
  - **方法依赖实例数据**，应使用**实例方法**。
  - **方法可能需要被子类重写**，应使用 `class func`。

---

**✅ 结论：**
在 `ReportConfigModel` 里，`booleanValue` 目前是工具方法，**用 `static` 没问题**。  
但如果未来它需要访问 `self`，就应该改成实例方法。**合理使用 `static`，避免让代码变成全是工具函数的“过程式编程”**。


---
---

## 七、数字系统及其转换

#### 1 基本概念

- **二进制（Binary）**  
  - **定义**：使用 `0` 和 `1` 两个数字表示信息，是计算机内部的基本数制（基数为 `2`）。  
  - **应用**：所有计算机数据最终都以二进制形式存储和运算。

- **十进制（Decimal）**  
  - **定义**：我们日常生活中使用的数字系统，基数为 `10`，由 `0～9` 十个数字组成。  
  - **应用**：人类最直观的数制，用于计算和计量。

- **十六进制（Hexadecimal）**  
  - **定义**：一种基数为 `16` 的数制，使用 `0～9` 和 `A～F`（代表 `10～15`）来表示。  
  - **应用**：在计算机系统中常用来表示内存地址、颜色值、以及底层硬件接口数据，因为它可以更紧凑地表示二进制数据（每 `1` 个十六进制数字等于 `4` 个二进制位）。

#### 2 数字转换方法

- **二进制 ⇔ 十进制**  
  - **二进制转十进制**：对二进制数的每一位，乘以相应的 `2` 的幂次方再求和。  
    例如，二进制 `1011 = 1×2³ + 0×2² + 1×2¹ + 1×2⁰ = 8 + 0 + 2 + 1 = 11`。  
  - **十进制转二进制**：将十进制数不断除以 `2`，记录余数，再将余数倒序排列。  
    例如，`11 ÷ 2 = 5 余 1，5 ÷ 2 = 2 余 1，2 ÷ 2 = 1 余 0，1 ÷ 2 = 0 余 1，倒序得到 1011`。

- **十进制 ⇔ 十六进制**  
  - **十进制转十六进制**：将十进制数不断除以 `16`，记录余数（`10～15` 分别用 `A～F` 表示），余数倒序排列。  
    例如，`255 ÷ 16 = 15 余 15（F），15 ÷ 16 = 0 余 15（F），得到 FF`。  
  - **十六进制转十进制**：对每一位乘以 `16` 的幂次方再求和。  
    例如，`FF = 15×16¹ + 15×16⁰ = 240 + 15 = 255`。

- **二进制 ⇔ 十六进制**  
  - 每 `4` 个二进制位对应 `1` 个十六进制位。例如，二进制 `1011 1100 分成 1011（B）和 1100（C），即十六进制 BC`。

#### 3 应用场景

- **硬件接口**：很多低层设备接口返回的是十六进制数据，使用十六进制便于阅读和调试。  
- **调试输出**：在调试工具中，经常需要将二进制数据以十六进制格式显示，便于快速定位问题。  
- **数据存储和传输**：序列化数据时，将内部数据转换为文本格式（如 `JSON`）时，使用 `Codable` 可以自动处理枚举的 `raw value`。

---


---

## 八、Xcode 调试工具：LLDB、Console 和 OSLog

#### 1 LLDB

- **概念**：  
  `LLDB` 是 `Xcode` 使用的调试器，用于设置断点、检查变量、跟踪函数调用等调试操作。
- **用途**：  
  - **设置断点**：在代码中暂停执行，检查内存、变量状态等。
  - **命令行调试**：通过 `LLDB` 命令（如 `po`、`bt`）查看对象、打印堆栈信息等。
- **使用方法**：  
  - 在 `Xcode` 中点击代码行号设置断点；  
  - 在调试控制台中输入 `LLDB` 命令，例如：
    - `po someVariable`（打印变量描述）  
    - `breakpoint set --name functionName`（设置断点）  
    - `thread backtrace`（查看调用堆栈）

#### 2 Console

- **概念**：  
  `macOS` 的 `Console` 应用用于查看系统日志和应用日志。  
- **用途**：  
  - 查看 `Xcode`、系统和应用产生的日志信息。  
  - 过滤日志、搜索特定关键字，帮助调试问题。
- **使用方法**：  
  - 打开 `macOS` 的 `Console` 应用；  
  - 在搜索栏中输入相关关键字（例如项目的子系统名称、进程名称等）；  
  - 观察日志中是否有异常信息或错误提示。

#### 3 OSLog

- **概念**：  
  `OSLog` 是苹果提供的新型日志系统，支持结构化日志记录。  
- **用途**：  
  - 用于在代码中记录调试、信息、错误日志；  
  - 支持高性能日志写入，并可通过 `Console` 或 `OSLogStore API` 进行查询。
- **使用方法**：  
  - 在代码中通过 `Logger` 类记录日志，例如：
    ```swift
    let logger = Logger(subsystem: "com.example.myapp", category: "network")
    logger.info("Network request started")
    logger.error("Network error: \(error.localizedDescription)")
    ```
  - 在 `Console` 应用中查看日志，或者使用 `OSLogStore API` 导出日志。

---