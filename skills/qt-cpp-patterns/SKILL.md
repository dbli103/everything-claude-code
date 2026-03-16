---
name: qt-cpp-patterns
description: Qt C++11 设计模式和最佳实践。用于编写、审查或重构 Qt C++ 代码。涵盖信号槽、智能指针、模型/视图、MVC 模式等。
origin: ECC
---

# Qt C++11 设计模式与最佳实践

全面的 Qt C++11 设计模式，涵盖信号槽机制、智能指针、模型/视图架构、MVC 模式等。

## 何时使用

- 编写新的 Qt C++ 类、控件、对话框
- 审查或重构现有的 Qt C++ 代码
- 在 Qt 项目中做出架构决策
- 选择 Qt 特定的功能（如信号槽 vs 回调）

### 何时不使用

- 非 Qt C++ 项目
- 纯 C 项目
- Qt Quick/QML 项目（使用 Qt Quick 模式）

## 核心原则

1. **使用新式信号槽连接** — 使用函数指针而非字符串
2. **RAII 资源管理** — 利用 Qt 的父子对象系统
3. **智能指针** — 结合 QScopedPointer 和 std::unique_ptr
4. **const 正确性** — 默认使用 const 成员函数
5. **模型/视图分离** — 使用 QAbstractItemModel

## 信号槽模式

### 新式连接（推荐）

```cpp
// 推荐：使用函数指针连接
connect(sender, &Sender::valueChanged,
        receiver, &Receiver::updateValue);

// Lambda 连接
connect(timer, &QTimer::timeout, this, [this]() {
    updateDisplay();
});

// 带上下文的 lambda（防止内存泄漏）
connect(timer, &QTimer::timeout, this, [this]() {
    updateDisplay();
}, Qt::QueuedConnection);
```

### 旧式连接（避免）

```cpp
// 避免：运行时解析，无类型检查
connect(sender, SIGNAL(valueChanged(QString)),
        receiver, SLOT(updateDisplay(QString)));

// 避免：使用字符串 lambda
connect(timer, SIGNAL(timeout()), this, SLOT(update()));
```

### 信号槽线程亲和性

```cpp
// 跨线程连接需要 QueuedConnection
connect(threadWorker, &Worker::resultReady,
        this, &MainWindow::handleResult,
        Qt::QueuedConnection);

// 直接连接用于同线程
connect(btn, &QPushButton::clicked,
        this, &MainWindow::onButtonClicked,
        Qt::DirectConnection);
```

## 内存管理

### Qt 父子对象系统

```cpp
// 好：设置父对象，自动清理
QWidget *window = new QWidget;
QPushButton *button = new QPushButton("Click", window);  // 父对象是 window

// 当 window 被删除时，button 也会被自动删除
delete window;  // 清理所有子对象
```

### 智能指针

```cpp
// QScopedPointer：Qt 风格的独占所有权
QScopedPointer<QLabel> label(new QLabel("text"));

// std::unique_ptr：C++11 标准，更灵活
std::unique_ptr<QPushButton> button(new QPushButton("Click"));
std::unique_ptr<QPushButton> button2 = std::make_unique<QPushButton>("Click");

// QSharedPointer：共享所有权
QSharedPointer<MyClass> ptr = QSharedPointer<MyClass>::create();
```

### 避免的内存管理模式

```cpp
// 避免：裸指针，容易泄漏
void badMethod() {
    QLabel *label = new QLabel("text");
    // 如果发生异常或提前返回，内存泄漏
}

// 好：使用智能指针或父对象
void goodMethod() {
    QScopedPointer<QLabel> label(new QLabel("text"));
    // 自动清理
}

// 好：使用父对象
void goodMethod2(QWidget *parent) {
    QLabel *label = new QLabel("text", parent);
    // parent 负责清理
}
```

## 模型/视图模式

### 基本模型实现

```cpp
class MyModel : public QAbstractListModel {
    Q_OBJECT
public:
    explicit MyModel(QObject *parent = nullptr) : QAbstractListModel(parent) {}

    int rowCount(const QModelIndex &parent = QModelIndex()) const override {
        return items.size();
    }

    QVariant data(const QModelIndex &index, int role = Qt::DisplayRole) const override {
        if (!index.isValid() || index.row() >= items.size())
            return QVariant();

        if (role == Qt::DisplayRole)
            return items[index.row()];
        return QVariant();
    }

    // 可选：支持编辑
    bool setData(const QModelIndex &index, const QVariant &value, int role) override {
        if (index.isValid() && role == Qt::EditRole) {
            items[index.row()] = value.toString();
            emit dataChanged(index, index, {role});
            return true;
        }
        return false;
    }

    Qt::ItemFlags flags(const QModelIndex &index) const override {
        return QAbstractListModel::flags(index) | Qt::ItemIsEditable;
    }

private:
    QStringList items;
};
```

### 使用视图

```cpp
// 创建模型和视图
MyModel *model = new MyModel(this);
QListView *view = new QListView;
view->setModel(model);

// 添加数据
model->insertRows(0, 1);
model->setData(model->index(0), "New Item", Qt::EditRole);
```

## Lambda 最佳实践

### 正确捕获

```cpp
// 好：按值捕获基本类型
int count = 0;
connect(timer, &QTimer::timeout, [count]() {
    qDebug() << "Count:" << count;
});

// 好：按引用捕获，但注意生命周期
connect(timer, &QTimer::timeout, this, [&data]() {
    processData(data);  // 确保 this 存活
});

// 好：明确捕获特定变量
connect(timer, &QTimer::timeout, this, [=]() {
    qDebug() << value;
});

// 避免：不当捕获导致悬空引用
connect(timer, &QTimer::timeout, [&result]() {  // result 可能在 lambda 执行时已销毁
    result.setValue(compute());
});
```

### 带参数的 Lambda

```cpp
// 使用 qOverload 处理重载
connect(lineEdit, &QLineEdit::textChanged,
        this, qOverload<const QString &>(&MyWidget::onTextChanged));
```

## Q_OBJECT 与元对象

### 必须使用 Q_OBJECT 的情况

```cpp
// 需要信号槽的类
class MyWidget : public QWidget {
    Q_OBJECT  // 必须！
signals:
    void mySignal(const QString &text);
public slots:
    void mySlot(const QString &text) { /* ... */ }
};
```

### 动态属性

```cpp
class MyWidget : public QWidget {
    Q_OBJECT
    Q_PROPERTY(int value READ value WRITE setValue)
public:
    int value() const { return m_value; }
    void setValue(int v) { m_value = v; }
private:
    int m_value;
};

// 使用动态属性
MyWidget *w = new MyWidget;
w->setProperty("author", "John");
QString author = w->property("author").toString();
```

## 错误处理

### 异常安全

```cpp
// 构造函数中的异常处理
MyClass::MyClass() try : m_resource(nullptr) {
    m_resource = acquireResource();
    // 可能抛出异常的代码
} catch (const std::exception &e) {
    qWarning() << "Construction failed:" << e.what();
    throw;  // 重新抛出
}

// 使用 qCheckPtr 处理分配失败
void *ptr = qCheckPointer malloc(sz);
if (!ptr) {
    qFatal("Allocation failed!");
}
```

### 验证输入

```cpp
void MyWidget::setValue(int value) {
    if (value < 0 || value > 100) {
        qWarning() << "Invalid value:" << value;
        return;
    }
    m_value = value;
    emit valueChanged(m_value);
}
```

## 性能优化

### 避免不必要复制

```cpp
// 避免：通过值传递大型对象
void process(const QString &data) { /* ... */ }

// 好：通过 const 引用
void process(const QString &data);

// 好：使用 QStringView（C++17）
void process(QStringView data);
```

### 延迟加载

```cpp
class MyWidget : public QWidget {
    Q_OBJECT
public:
    MyWidget(QWidget *parent = nullptr) : QWidget(parent) {}

    // 延迟创建
    QPushButton* button() {
        if (!m_button) {
            m_button = new QPushButton("Click", this);
            connect(m_button, &QPushButton::clicked, this, &MyWidget::onClick);
        }
        return m_button;
    }

private:
    QPushButton *m_button = nullptr;
};
```

### 避免过度布局

```cpp
// 好：使用尺寸策略
button->setSizePolicy(QSizePolicy::Fixed, QSizePolicy::Fixed);

// 好：设置最小/最大尺寸
widget->setMinimumSize(100, 50);
widget->setMaximumSize(200, 100);
```

## 快速参考清单

完成 Qt C++ 工作前检查：

- [ ] 使用新式函数指针信号槽连接（不要用 SIGNAL/SLOT 宏）
- [ ] 正确设置父对象或使用智能指针
- [ ] Q_OBJECT 宏用于需要信号槽的类
- [ ] 使用 const 成员函数（除非需要修改）
- [ ] Lambda 捕获正确（避免悬空引用）
- [ ] 跨线程信号槽使用 QueuedConnection
- [ ] 使用模型/视图分离数据展示
- [ ] 避免在主线程执行耗时操作
- [ ] 使用 qDebug/qWarning 而不是 printf
- [ ] 验证用户输入

## 相关资源

- Qt 官方文档：https://doc.qt.io/
- Qt 示例：https://doc.qt.io/qt-6/qtexamples.html
- Qt 教程：https://doc.qt.io/qt-6/qtwidgets-index.html
