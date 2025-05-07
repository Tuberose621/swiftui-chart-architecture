## Swift 中的函数式编程/Task 和 async/await 

### 一、函数式编程风格

在下面的代码中：
```swift
let entries = try store
    .getEntries(at: position)
    .compactMap { $0 as? OSLogEntryLog }
    .filter { $0.subsystem == Bundle.main.bundleIdentifier! }
    .filter { $0.date >= date }
    .map { "[\(self.dateFormatter.string(from: $0.date))][\((OSLogLevel(rawValue: $0.level.rawValue) ?? .undefined).title)][\($0.category)] \($0.composedMessage)" }
```
- **compactMap**：将集合中每个元素转换为某类型，如果转换失败则过滤掉 nil。  
- **filter**：筛选集合中满足条件的元素。  
- **map**：将集合中每个元素转换成另一种形式。  
- **reduce**：（在其他地方使用）将集合中所有元素累加或合并成一个单一值。

这种写法属于函数式编程风格，使代码更加简洁、链式操作逻辑清晰，同时利用 `Swift` 提供的高阶函数实现数据处理。

---

下面详细解释一下 Task、async 和 await 的区别和使用场景：

---

### 二、异步任务 Task 与 async/await 的使用

#### 1. Task 和异步执行

- **Task { ... }**  
  - 当我们使用 `Task { ... }` 时，代码块会在异步上下文中执行，这意味着里面的代码不会阻塞主线程。  
  - 无论我们在 `Task` 内部是否使用 `await`，这个 `Task` 本身都是异步启动的，代码块会在后台执行。

---

#### 2. async 与 await 的作用

- **async 函数**：  
  - 一个用 `async` 修饰的函数表示它包含异步操作，调用时必须以异步方式执行。  
  - 例如：
    ```swift
    func someAsyncFunction() async throws -> String {
        // 模拟一个异步操作，例如网络请求
        try await Task.sleep(nanoseconds: 1_000_000_000)
        return "Hello, Async!"
    }
    ```
  - 使用 `async` 函数可以编写非阻塞代码，但调用它时需要用 `await` 来等待结果。

- **await 的作用**：  
  - `await` 表示“暂停当前任务”，直到异步函数完成并返回结果。  
  - 如果不加 `await`，我们只能得到一个尚未完成的异步操作，不会直接获得结果。例如：
    ```swift
    // 错误示例：无法直接获得结果，因为 someAsyncFunction() 是异步的
    let result = someAsyncFunction() // result 是一个“待完成”的异步任务
    ```
  - 加上 `await` 后：
    ```swift
    let result = try await someAsyncFunction()
    print("Result: \(result)")
    ```
    这表示当前 `Task` 会等待 `someAsyncFunction()` 完成后再继续执行，保证我们拿到了函数返回的结果。

---

#### 3. 使用场景与结合

- **不使用 await 的情况**  
  - 如果我们调用一个异步函数但不需要等待其结果，可以不加 await。例如启动一个后台任务，不关心返回值：
    ```swift
    Task {
        someAsyncFunction() // 这里不等待结果，任务在后台执行
    }
    ```
  - 但如果后续代码依赖该函数的结果，就必须使用 await，否则数据未就绪时就继续执行了。

- **结合 Task 和 async/await 的示例**  
  - 当我们需要在后台执行异步任务，并等待返回结果时，可以这样写：
    ```swift
    Task {
        do {
            let result = try await someAsyncFunction()
            print("Result: \(result)")
        } catch {
            print("Error: \(error)")
        }
    }
    ```
  - 这种写法确保了 `Task` 内部在执行到 `await` 处时，会暂停等待结果，等函数返回后再继续执行后续代码，同时不会阻塞主线程。

- **async/await 使用场景**  
  - 网络请求：调用网络 `API`、解析 `JSON` 等耗时操作时。
  - 文件 `I/O`：异步读取/写入大文件。
  - 任何耗时操作：例如数据库查询、长时间计算等。

**下面给出几个常见的 `async` 函数使用场景及代码示例**

-   **1. 网络请求**
    -   **场景：**  向服务器发送 `HTTP` 请求并等待响应。这种操作通常会耗时，使用 `async/await` 可以写出简洁的代码。

    -   **示例代码：**

```swift
// 定义一个 async 函数，使用 URLSession 发起网络请求
func fetchData(from url: URL) async throws -> Data {
    // URLSession 的 data(from:) 方法本身就是 async 的
    let (data, _) = try await URLSession.shared.data(from: url)
    return data
}

// 调用示例
Task {
    do {
        let url = URL(string: "https://api.example.com/data")!
        let data = try await fetchData(from: url)
        print("Received data: \(data.count) bytes")
    } catch {
        print("Network request failed: \(error)")
    }
}
```

---

-   **2. 文件读写**
    -   **场景：**  读取或写入大文件时不阻塞主线程，可以用 `async` 函数来执行 `I/O` 操作。

    -   **示例代码：**

```swift
func readFileAsync(at url: URL) async throws -> String {
    // 使用 Swift 的 FileManager 读取文件数据（示例中模拟异步延时操作）
    try await Task.sleep(nanoseconds: 500_000_000)  // 模拟异步耗时操作
    let data = try Data(contentsOf: url)
    guard let content = String(data: data, encoding: .utf8) else {
        throw NSError(domain: "ReadError", code: 0, userInfo: nil)
    }
    return content
}

Task {
    do {
        let fileURL = URL(fileURLWithPath: "/path/to/file.txt")
        let content = try await readFileAsync(at: fileURL)
        print("File content: \(content)")
    } catch {
        print("Failed to read file: \(error)")
    }
}
```

---

-   **3. 异步数据库查询**

    -   **场景：**  在后台查询数据库数据，并在查询完成后更新 `UI`。使用 `async/await` 使代码逻辑清晰。

    -   **示例代码：**

```swift
func queryDatabaseAsync(query: String) async throws -> [String] {
    // 模拟数据库查询，使用 Task.sleep 模拟耗时
    try await Task.sleep(nanoseconds: 1_000_000_000)  // 模拟耗时 1 秒
    // 假设查询结果返回字符串数组
    return ["Result 1", "Result 2", "Result 3"]
}

Task {
    do {
        let results = try await queryDatabaseAsync(query: "SELECT * FROM table")
        print("Database query results: \(results)")
    } catch {
        print("Database query failed: \(error)")
    }
}
```

---

-   **4. 并行任务组合**

    -   **场景：**  需要同时执行多个异步任务，并等待它们全部完成后再继续处理，比如同时发起多个网络请求。
    -   **示例代码：**

```swift
func fetchData(from url: URL) async throws -> Data {
    let (data, _) = try await URLSession.shared.data(from: url)
    return data
}

Task {
    do {
        async let data1 = fetchData(from: URL(string: "https://api.example.com/data1")!)
        async let data2 = fetchData(from: URL(string: "https://api.example.com/data2")!)
        // 使用 await 同时等待两个异步请求完成
        let (result1, result2) = try await (data1, data2)
        print("Received data1: \(result1.count) bytes, data2: \(result2.count) bytes")
    } catch {
        print("Error in fetching data: \(error)")
    }
}
```

在这个例子中，`async let` 用于并行启动多个异步任务，`await (data1, data2)` 则同时等待所有任务完成。

---

**5. 总结 async/await 与 Task**

- **Task { ... }**：  
  创建一个异步任务，在后台异步执行代码块。无论 `Task` 内部是否使用 `await`，代码块都会在异步上下文中运行，不会阻塞主线程。

- **async 函数**：  
  用 `async` 声明的函数表示该函数内部可能存在异步操作。调用 `async` 函数时必须使用 `await`，以确保等待其完成并获取结果。

- **await**：  
  用于暂停当前任务直到异步操作完成，返回结果后继续执行。对于依赖结果的代码必须使用 `await`。

- **结合使用**：  
  `Task` 内部可以调用 `async` 函数，并使用 `await` 等待结果，从而使得异步代码写起来像同步代码，但不会阻塞主线程。

---

#### 总结

- **Task { ... }** 创建的代码块总是异步执行，不管里面是否有 `await`。
- **await** 用于等待 `async` 函数返回结果，保证后续代码能使用正确的返回值。
- 结合使用时，我们通常在 `Task` 块中调用 `async` 函数，并用 `await` 等待结果，这样既能在后台执行又能以同步风格写代码。

**通过这种方式，可以避免阻塞主线程，同时又能确保依赖异步操作的代码按照预期顺序执行。**

---

### 三、Task 和 async/await 以及 GCD/NSOperationQueue

#### 1. Task 和 async/await 的关系

- **Task { ... }** 本身创建了一个异步上下文，在该上下文中代码会在后台执行，不阻塞主线程。  
- **async 函数** 表示某个函数内部可能存在异步操作，它可以在需要时挂起当前任务直到操作完成，然后恢复执行。调用 `async` 函数必须使用 `await` 来等待其返回结果，这样能够使代码的流程看起来像同步代码，但实际上是非阻塞的。

**为什么 Task 内部还需要调用 async 函数？**  
- 即使 `Task` 已经在后台运行，`Task` 内部的代码可能调用的是一个耗时操作（例如网络请求、文件读写或数据库查询），而这些操作已经以 `async` 函数的方式封装。使用 `await` 可以暂停 `Task` 内部代码的执行，直到 `async` 函数返回结果，然后再继续后续处理。这种写法使得代码结构更清晰、错误处理更方便，同时能确保依赖异步结果的代码能正确运行。

---

#### 2. 使用场景与优势

- **提高代码可读性和结构性**  
  `async/await` 提供了结构化并发，代码逻辑看起来像同步代码，避免了回调地狱和嵌套 `GCD block` 的混乱。

- **简化错误处理**  
  `async` 函数支持抛出异常，使用 `try/await` 可以简化错误传播，而不用像传统 `GCD` 那样手动传递 `error` 对象。

- **减少 GCD/NSOperationQueue 的复杂性**  
  在许多常见的异步操作场景下（如网络请求、文件读写等），我们可以直接用 `async/await` 来替代 `GCD` 或 `NSOperationQueue`。苹果目前推荐新项目尽量采用 `Swift` 的异步编程模型（`Task、async/await`），因为它更加现代和结构化。

---

#### 3. 现有工具的价值

- **GCD 和 NSOperationQueue 依然有使用价值**  
  - **底层操作**：在一些底层系统操作或需要精细控制调度策略的场景下，`GCD` 仍然非常高效。
  - **兼容性**：老项目和一些第三方库可能仍然依赖 `GCD`。
  - **混合使用**：在新项目中，可以使用 `async/await` 处理大部分异步任务，同时在必要时用 `GCD` 来完成一些低层或系统级任务。

苹果的目标是逐步引入并推广 `Swift Concurrency（Task、async/await）`的使用，因为它带来了更好的代码结构、错误处理和并发控制，但 `GCD` 依然是系统底层的一部分。

---

#### 4. 结论

- **Task** 创建了一个异步上下文，而 **async/await** 是对具体异步函数的调用方式。  
- 当我们需要依赖异步函数的返回结果或确保顺序执行时，必须使用 `await`。  
- 苹果推荐使用 `async/await`（结合 `Task`）来写新项目中的并发代码，因为它简洁、结构化，同时可以替代大部分 `GCD/NSOperationQueue` 的用法，但 `GCD` 仍然在底层有不可替代的作用。
