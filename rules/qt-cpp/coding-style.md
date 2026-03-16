# Qt C++11 编码规范

## 概述

本规范基于 C++ Core Guidelines 和 Qt 官方最佳实践，适用于 Qt C++11/14/17 项目开发。

## 文件组织

### 头文件

```cpp
// widget.h
#ifndef PROJECT_WIDGET_H
#define PROJECT_WIDGET_H

#include <QWidget>  // Qt 头文件
#include <QLabel>
#include <QPushButton>

// 前向声明
class DataModel;

class Widget : public QWidget {
    Q_OBJECT
public:
    explicit Widget(QWidget *parent = nullptr);
    ~Widget();

signals:
    void dataChanged(const QString &text);

public slots:
    void updateDisplay(const QString &text);

private:
    void initUI();

    QLabel *m_label;
    QPushButton *m_button;
    DataModel *m_model;
};

#endif // PROJECT_WIDGET_H
```

### 源文件

```cpp
// widget.cpp
#include "widget.h"
#include "datamodel.h"

#include <QVBoxLayout>

Widget::Widget(QWidget *parent)
    : QWidget(parent)
    , m_label(new QLabel(this))
    , m_button(new QPushButton(this))
    , m_model(new DataModel(this))
{
    initUI();
}

Widget::~Widget() = default;

void Widget::initUI() {
    auto *layout = new QVBoxLayout(this);
    layout->addWidget(m_label);
    layout->addWidget(m_button);

    connect(m_button, &QPushButton::clicked,
            this, &Widget::onButtonClicked);
}
```

## 命名规范

### 类和类型

```cpp
// 使用 PascalCase
class MyWidget;
class DataModel;
struct ConfigOptions;
typedef QList<QString> StringList;
using StringList = QList<QString>;
```

### 变量

```cpp
// 成员变量：m_ 前缀 + camelCase
int m_count;
QString m_name;
QList<Item*> m_items;

// 全局变量：g_ 前缀
int g_counter;

// 常量：k 前缀或全大写
const int kMaxCount = 100;
static const QString kDefaultName = "default";

// 参数和局部变量：camelCase
void process(const QString &inputName, int &outputCount);
```

### 函数

```cpp
// 函数：PascalCase
void UpdateDisplay();
QString GetName() const;

// 槽函数：on 前缀 + 对象名 + 操作
void onButtonClicked();
void onLineEditChanged(const QString &text);
```

## Qt 特定规则

### Q_OBJECT 宏

```cpp
// 所有需要信号槽的类必须包含 Q_OBJECT
class MyClass : public QObject {
    Q_OBJECT
public:
    // ...
};
```

### 信号槽连接

```cpp
// 推荐：新式函数指针连接
connect(sender, &Sender::signal,
        receiver, &Receiver::slot);

// 避免：旧式字符串连接
connect(sender, SIGNAL(signal()),
        receiver, SLOT(slot()));

// Lambda 连接（注意捕获）
connect(timer, &QTimer::timeout, this, [this]() {
    updateUI();
});
```

### 内存管理

```cpp
// 推荐：设置父对象
QPushButton *button = new QPushButton("Click", parentWidget);

// 或使用智能指针
QScopedPointer<QLabel> label(new QLabel("text"));
std::unique_ptr<MyClass> obj = std::make_unique<MyClass>();
```

## C++11 特性

### auto 关键字

```cpp
// 推荐：类型明确时使用 auto
auto widget = new QWidget;
auto list = QStringList();

// 避免：类型不明确时使用 auto
auto value = getSomeValue();  // 不确定返回类型
```

### nullptr

```cpp
// 使用 nullptr 而不是 NULL
QWidget *widget = nullptr;
int *ptr = nullptr;

// 不使用
QWidget *widget = NULL;
```

### 范围 for 循环

```cpp
// 推荐：范围 for
for (const QString &item : items) {
    process(item);
}

// 不推荐
for (int i = 0; i < items.size(); ++i) {
    process(items.at(i));
}
```

### Lambda 表达式

```cpp
// 正确捕获
connect(timer, &QTimer::timeout, [this]() {
    updateData();
});

// 避免捕获 this 的危险用法
connect(timer, &QTimer::timeout, [&data]() {  // data 可能已销毁
    process(data);
});
```

### override 关键字

```cpp
class MyWidget : public QWidget {
protected:
    void keyPressEvent(QKeyEvent *event) override {
        // 实现
    }
};
```

### = default / = delete

```cpp
class NonCopyable {
public:
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
};
```

## 代码格式化

### 缩进和空格

```cpp
// 使用 4 空格缩进
void Function() {
    if (condition) {
        doSomething();
    }
}

// 大括号风格：K&R
void Function() {
    if (condition) {
        doSomething();
    } else {
        doOther();
    }
}

// 命名空间不增加缩进
namespace MyNamespace {
class MyClass {
public:
    void method();
};
}
```

### 行长度

- 最大 100 字符
- 过长时使用续行缩进

```cpp
void longFunctionName(
        int parameter1,
        int parameter2,
        int parameter3);
```

## 注释

```cpp
// 公共 API 文档
//! Processes user data
//! \param userId The user identifier
//! \return true on success
bool processUserData(int userId);

// 实现注释
// TODO: Optimize for large datasets
// FIXME: Handle edge case
```

## Qt 模块导入

```cpp
// 最小化包含，按需导入
#include <QCoreApplication>  // 核心
#include <QDebug>            // 调试
#include <QFile>             // 文件
#include <QLabel>           // UI
```

## 错误处理

```cpp
// 使用异常处理构造函数
MyClass::MyClass() try : m_resource(nullptr) {
    m_resource = acquire();
} catch (const std::exception &e) {
    qWarning() << "Failed:" << e.what();
    throw;
}

// 参数验证
void setValue(int value) {
    if (value < 0 || value > 100) {
        qWarning() << "Invalid value:" << value;
        return;
    }
    m_value = value;
}
```

## 测试

- 使用 Qt Test 框架
- 测试命名：test_<功能>
- 每个类至少有一个测试文件

```cpp
class TestMyClass : public QObject {
    Q_OBJECT
private slots:
    void init();
    void cleanup();
    void testInitialState();
    void testSetValue();
};
