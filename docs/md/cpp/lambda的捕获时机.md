# lambda的捕获时机

---

所有 `lambda` 表达式的捕获（无论是引用 `[&]`、值 `[=]`，还是混合捕获）都是在 `lambda` 表达式被定义时发生的，而不是在调用时。

👉如果 lambda 捕获了某个变量（比如通过 `[&]` 捕获 `count`），不论你在后续**哪里调用这个 lambda**，哪怕调用现场有个**同名变量**，它引用和操作的都是**定义时捕获的那个变量**`count`。

> 代码示例

```clike
#include <iostream>
#include <functional>

void pass_and_call(std::function<void()> f) {
    int count = 999; // 同名变量
    f(); // 调用 lambda
}

int main() {
    int count = 0;
    auto f = [&]() {
        ++count;
        std::cout << "count = " << count << "\n";
    };

    pass_and_call(f); // 输出的是主函数中的 count，而不是 pass_and_call 中的
}

```

> 输出

```
count = 1
```

<br>

>  总结

- **lambda 是闭包对象（closure）**，它会“记住”它定义时的上下文。
- 所以你传它去别的地方运行，它也能“带着上下文一起走”。
- **捕获引用 `[&]` 有风险**，一旦原始变量超出作用域，它指向的引用就悬空了。

