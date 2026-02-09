**修改时间:2026.1.26**

**参与者:刘志钰**

# ROS2 基础编程



## (1) 面向对象编程

面向对象的语言都可以创建类，所谓的类就是对事物的一种封装。

```
#include <string>
#include "rclcpp/rclcpp.hpp"

class PersonNode : public rclcpp::Node
{
private:
    std::string name_;
    int age_;

public:
    PersonNode(const std::string &node_name, const std::string &name, const int &age) 
        : Node(node_name)
    {
        this->name_ = name;
        this->age_ = age;
    };

    void eat(const std::string &food_name)
    {
        RCLCPP_INFO(this->get_logger(), "我是 %s，今年 %d 岁，我现在正在吃 %s",
            name_.c_str(), age_, food_name.c_str());
    };
};

int main(int argc, char **argv)
{
    rclcpp::init(argc, argv);
    auto node = std::make_shared<PersonNode>("cpp_node", "法外狂徒张三", 18);
    node->eat("鱼香ROS");
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```

### 代码解释

- **`class PersonNode : public rclcpp::Node`**
  - `:` 是继承符号，`public` 是访问说明符，`rclcpp::Node` 是基类。
  - 标识 `PersonNode` 类以 `public` 方式继承自 `rclcpp::Node` 类。
- **`private`**
  - 这部分代码开始定义 `PersonNode` 类的私有成员区域，私有成员只能在类内部访问（例如可以在类的内部调用私有函数，但是不能在外部比如 `main` 函数调用私有函数，会报错）。
- **`RCLCPP_INFO()`**
  - 宏，用于记录一条日志信息（是 `rclcpp` 库中的）。
- **`this->get_logger()`**
  - `this` 是 C++ 中的一个关键字，指向当前对象（即调用成员函数的那个对象）。
  - `get_logger()` 是 `rclcpp::Node` 类的一个成员函数，用于获取当前节点的日志记录器对象。
- **`rclcpp::init`**
  - 初始化 ROS2 系统。
- **`auto node = std::make_shared<PersonNode>()`**
  - `auto`: 类型推导关键字，当使用 `auto` 声明变量时，编译器会根据变量的初始化表达式自动推断其类型。
  - `std::make_shared`: C++ 标准库 `<memory>` 头文件中的一个函数模板，用于在堆上分配对象并返回一个 `std::shared_ptr` 智能指针来管理该对象。
  - **智能指针**：会自动管理所指向对象的生命周期，当没有任何 `std::shared_ptr` 指向该对象时，对象会被自动释放，从而避免了内存泄露的问题。
  - `<PersonNode>`: 指向的是 `PersonNode` 类，不是成员函数 `PersonNode`。
- **`rclcpp::spin(node)`**
  - 使程序进入事件循环，等待并处理 ROS2 的各种事件，直到节点被关闭或遇到其他终止条件才会结束（例如 `ctrl+c` 中断信号）。
- **`std`**
  - `std::string` 可以直接使用，因为包含了 `<string>` 头文件。
  - `std::make_shared`（定义在 `<memory>` 头文件中）可以直接使用，因为 `rclcpp/rclcpp.hpp` 这个头文件可能间接包含了 `<memory>`。
- **`c_str()`**
  - `std::string` 类的一个成员函数。
  - 作用是返回一个指向以空字符 `\0` 结尾的字符数组的指针。简单来说，它把 `std::string` 对象转换为 C 风格的字符串。
  - 注意：在使用 `RCLCPP_INFO` 等类 printf 风格函数时，`std::string` 都要使用 `.c_str()` 转换（平常使用流操作符 `<<` 则不需要）。

------

## (2) C++ 新特性

### -1 自动类型推导

- 对应 Python 中的动态类型（注意：Python 的 `eval()` 是执行字符串表达式，不是类型推导）。
- **`auto` 关键字**：可以在给变量赋值时，根据等号右边返回的类型自动推导变量的类型。

### -2 智能指针

可以动态分配内存，避免内存泄漏和空指针等问题。智能指针是在头文件 `<memory>` 的 `std` 命名空间中定义的。

- **三种类型**：`std::unique_ptr`、`std::shared_ptr` 和 `std::weak_ptr`。
- **原理**：该指针会记录指向同一个资源的指针数量，当数量为 0 时会自动释放内存，这样一来就不会出现提前释放或者忘记释放的情况。

### -3 Lambda 表达式

语法结构：

```
[capture list](parameters) -> return_type { function body }
```

- Lambda 表达式是 C++11 引入的一种匿名函数，没有名字，但是也可以像正常函数一样调用。
- `capture list`: 捕获列表，可以用于捕获外部变量。
- `parameters`: 参数列表。
- `return_type`: 返回值类型。
- `function body`: 函数体。
- **注意**：部分高级 Lambda 语法可能需要导入 algorithm 库 (`#include <algorithm>`)。

### -4 函数包装器 `std::function`

`std::function` 是 C++11 引入的一种通用函数包装器，它可以存储任意可调用对象（函数、函数指针、Lambda 表达式等）并提供统一的调用接口。

- 头文件：`<functional>`。
- 在外部定义的函数称为自由函数。

**示例：**

```
std::function<void(const std::string &)> save3 = std::bind(&FileSave::save_with_member_fun, &file_save, std::placeholders::_1);
```

**解释：**

- **`std::function`**：C++ 标准库中的通用多态函数包装器，可存储、复制和调用任何可调用对象。
- **`<void(const std::string &)>`**：模板参数，指定了所包装的可调用对象的签名（返回 void，参数为 string 引用）。
- **`std::bind`**：将成员函数变成一个 `std::function` 的对象。
  - 正常调用成员函数是 `对象.函数` (如 `file_save.save_with_member_fun`)。
  - 这里用 `bind` 将成员函数 `FileSave::save_with_member_fun` 与对象 `file_save` 绑定在一起。
- **`std::placeholders::_1`**：占位符。
  - 预留一个位置传递函数的参数。
  - 当调用 `save3` 时，传入的第一个参数将被转发给 `FileSave::save_with_member_fun` 函数。
- **`&`**：通常紧跟在类型名之后，写在变量名之前（取地址或引用）。
- **库依赖**：需要导入 `<functional>` 库才能使用。

------

## (3) 多线程与回调函数

### Python 示例

```
import threading
import requests

class Download:
    def download(self, url, callback):
        print(f'线程 :{threading.get_ident()} 开始下载：{url}')
        response = requests.get(url)
        response.encoding = 'utf-8'
        callback(url, response.text)
        
    def start_download(self, url, callback):
        thread = threading.Thread(target=self.download, args=(url, callback))
        thread.start()

def download_finish_callback(url, result):
    print(f'{url} 下载完成，共：{len(result)} 字，内容为：{result[:5]}...')

def main():
    d = Download()
    d.start_download('http://localhost:8000/novel1.txt', download_finish_callback)
    d.start_download('http://localhost:8000/novel2.txt', download_finish_callback)
    d.start_download('http://localhost:8000/novel3.txt', download_finish_callback)

# 注意：thread = threading.Thread(target=self.download, args=(url, callback))
```

> **提示**：使用 `python3 -m http.server` 可以启动一个本地的 HTTP 服务器（只能找到启动该命令的文件夹里面的内容）。

### C++ 示例

```
#include <iostream>
#include <thread>
#include <chrono>
#include <functional>
#include <cpp-httplib/httplib.h>

class Download {
public:
    void download(const std::string &host, const std::string &path, const std::function<void(const std::string &, const std::string &)>& callback) {
        std::cout << " 线程 ID: " << std::this_thread::get_id() << std::endl;
        httplib::Client client(host);
        auto response = client.Get(path);
        if (response && response->status == 200) {
            callback(path, response->body);
        }
    }
    
    void start_download(const std::string &host, const std::string &path, const std::function<void(const std::string &, const std::string &)>& callback) {
        auto download_fun = std::bind(&Download::download, this, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3);
        std::thread download_thread(download_fun, host, path, callback);
        download_thread.detach();
    }
};

int main() {
    Download download;
    auto download_finish_callback = [](const std::string &path, const std::string &result) -> void {
        std::cout << " 下载完成：" << path << " 共：" << result.length() << " 字，内容为：" << result.substr(0, 16) << std::endl;
    };
    
    download.start_download("http://localhost:8000", "/novel1.txt", download_finish_callback);
    download.start_download("http://localhost:8000", "/novel2.txt", download_finish_callback);
    download.start_download("http://localhost:8000", "/novel3.txt", download_finish_callback);
    
    std::this_thread::sleep_for(std::chrono::milliseconds(1000 * 10));
    return 0;
}
```

#### 关键点解释

1. **`download_thread.detach()`**
   - 将线程对象 `download_thread` 与执行的线程分离，使其能够在后台独立运行。
2. **依赖库准备**
   - 需要下载 C++ 的 HTTP 请求库 `cpp-httplib`，该库只需要引入头文件即可使用。
   - 下载命令：`git clone https://gitee.com/ohhuo/cpp-httplib.git`
   - 将下载库放在 include 目录下。
3. **CMakeLists.txt 配置**
   - 下载完成后，需要在 `CMakeLists.txt` 中添加 `include_directories(include)` 指令，指定 include 文件夹为头文件目录。
4. **头文件说明**
   - `thread`: 线程相关。
   - `chrono`: 时间相关。
   - `functional`: 函数包装器。
   - `cpp-httplib/httplib.h`: 用于下载。
5. **差异**
   - C++ 和 Python 的字符长度统计方式不同，所以最终显示的字数可能不同。
