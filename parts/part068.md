# Part 068: Appendix B — C++ Best Practices & Step 1000

## ขั้นตอนที่ 986-1000

---

## ขั้นตอนที่ 986: Modern C++ Best Practices

```cpp
// ===== Resource Management =====

// ✅ ใช้ smart pointers — ไม่ใช้ raw new/delete
auto obj = std::make_unique<MyClass>(arg1, arg2);
auto shared = std::make_shared<Resource>();

// ✅ ใช้ RAII สำหรับทุก resource
class FileHandle {
public:
    explicit FileHandle(const QString& path)
        : m_file(path) { m_file.open(QFile::ReadWrite); }
    ~FileHandle() { m_file.close(); }
    QFile& get() { return m_file; }
private:
    QFile m_file;
};

// ✅ Rule of Zero — ถ้าใช้ smart pointers ไม่ต้อง define destructor/copy/move
class MyClass {
    std::unique_ptr<Widget> m_widget;
    std::vector<Data> m_data;
    QString m_name;
    // Compiler generates correct copy/move/destructor
};

// ===== Const Correctness =====

// ✅ const สำหรับ member functions ที่ไม่เปลี่ยน state
class Rectangle {
public:
    double area() const { return m_w * m_h; }  // const
    void resize(double w, double h) { m_w = w; m_h = h; }  // non-const
private:
    double m_w{}, m_h{};
};

// ✅ const references สำหรับ parameters ขนาดใหญ่
void processCustomers(const QList<Customer>& customers);
QString formatName(const QString& first, const QString& last);

// ✅ [[nodiscard]] สำหรับ return values สำคัญ
[[nodiscard]] bool save();          // caller must check return
[[nodiscard]] std::optional<User> findUser(int id);

// ===== Type Safety =====

// ✅ Strong typedefs ด้วย enum class
enum class CustomerId : int {};
enum class ProductId : int {};
// ป้องกัน: void foo(CustomerId cid, ProductId pid) ≠ foo(pid, cid) 

// ✅ std::optional สำหรับ nullable values
std::optional<Customer> findCustomer(int id);
if (auto cust = findCustomer(42)) {
    process(*cust);
}

// ✅ std::variant สำหรับ sum types
using Result = std::variant<Customer, QString>;  // success or error message
Result result = loadCustomer(id);
std::visit(overloaded{
    [](const Customer& c) { display(c); },
    [](const QString& err) { showError(err); }
}, result);

// ===== Performance Tips =====

// ✅ Reserve capacity สำหรับ containers
QList<Customer> customers;
customers.reserve(1000);  // avoid re-allocations

// ✅ std::move สำหรับ large objects
void addCustomer(Customer&& c) {
    m_customers.push_back(std::move(c));  // no copy
}

// ✅ std::string_view / QStringView สำหรับ read-only access
bool startsWith(QStringView str, QStringView prefix) {
    return str.startsWith(prefix);  // no allocation
}

// ✅ Avoid premature optimization — measure first
// Use: Profiler, perf, Valgrind, Qt Creator Analyzer
```

---

## ขั้นตอนที่ 987: Qt-Specific Best Practices

```cpp
// ===== Object Lifetime =====

// ✅ Qt parent-child ownership สำหรับ QObject
auto* widget = new QPushButton("OK", parent);  // parent owns it
// widget deleted when parent deleted — ไม่ต้อง delete เอง

// ✅ อย่า stack-allocate QWidget
// ❌ QPushButton btn("OK"); // อาจมีปัญหา lifetime
// ✅ QPushButton* btn = new QPushButton("OK", this);

// ✅ ใช้ unique_ptr สำหรับ non-QObject QObject-unowned objects
auto model = std::make_unique<MyModel>();
view->setModel(model.get());  // view doesn't own it

// ===== Signal-Slot Patterns =====

// ✅ Qt::ConnectionType สำหรับ thread safety
connect(worker, &Worker::finished, this, &UI::onFinished,
        Qt::QueuedConnection);  // cross-thread — safe

// ✅ Disconnect เมื่อ object deleted
// Qt จัดการโดยอัตโนมัติถ้าทั้ง sender/receiver เป็น QObject

// ✅ Lambda ใน connect — ระวัง dangling captures
connect(button, &QPushButton::clicked, this, [this]() {
    // ✅ 'this' tracked — Qt disconnects if 'this' is deleted
    m_model->save();
});

// ❌ Dangerous:
// connect(button, &QPushButton::clicked, [this]() { … }); // no context!

// ===== Model/View =====

// ✅ emit dataChanged ด้วย roles ที่เปลี่ยนจริง
void MyModel::updateItem(int row) {
    m_data[row].value = newValue;
    QModelIndex idx = index(row);
    emit dataChanged(idx, idx, {Qt::DisplayRole, ValueRole});
    // ❌ ไม่ควร: emit dataChanged ทั้ง model เมื่อเปลี่ยนแค่ 1 item
}

// ✅ beginInsertRows/endInsertRows สำหรับ insert
void MyModel::addItem(const T& item) {
    int row = m_data.size();
    beginInsertRows(QModelIndex{}, row, row);
    m_data.append(item);
    endInsertRows();
}

// ===== Painting =====

// ✅ setRenderHint สำหรับ anti-aliasing
void MyWidget::paintEvent(QPaintEvent*) {
    QPainter p(this);
    p.setRenderHints(QPainter::Antialiasing |
                     QPainter::TextAntialiasing |
                     QPainter::SmoothPixmapTransform);
    // ...
}

// ✅ Cache QPixmap/QImage — ไม่สร้างใหม่ทุก paintEvent
class MyWidget : public QWidget {
    void paintEvent(QPaintEvent*) override {
        if (m_cache.isNull()) rebuildCache();
        QPainter(this).drawPixmap(0, 0, m_cache);
    }
    void rebuildCache() {
        m_cache = QPixmap(size());
        QPainter p(&m_cache);
        // expensive drawing...
    }
    QPixmap m_cache;
};
```

---

## ขั้นตอนที่ 988: Code Organization

```
โครงสร้าง project แนะนำ:

MyApp/
├── CMakeLists.txt
├── README.md
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── src/
│   ├── main.cpp
│   ├── core/
│   │   ├── database/
│   │   │   ├── database.h / .cpp
│   │   │   └── migration.h / .cpp
│   │   └── eventbus.h / .cpp
│   ├── models/
│   │   ├── customer.h
│   │   └── product.h
│   ├── repositories/
│   │   ├── customerrepository.h / .cpp
│   │   └── productrepository.h / .cpp
│   ├── services/
│   │   ├── authservice.h / .cpp
│   │   └── reportservice.h / .cpp
│   ├── network/
│   │   ├── httpclient.h / .cpp
│   │   └── websocketclient.h / .cpp
│   └── ui/
│       ├── main/
│       │   └── mainwindow.h / .cpp
│       ├── widgets/
│       │   └── customwidgets.h / .cpp
│       └── dialogs/
│           └── customerdialog.h / .cpp
├── qml/
│   ├── main.qml
│   └── components/
│       └── Button.qml
├── resources/
│   ├── resources.qrc
│   ├── icons/
│   └── fonts/
├── translations/
│   ├── app_th.ts
│   └── app_en.ts
├── tests/
│   ├── CMakeLists.txt
│   ├── test_customer.cpp
│   └── test_repository.cpp
└── plugins/
    └── report_pdf/
        ├── CMakeLists.txt
        └── reportpdfplugin.cpp
```

---

## ขั้นตอนที่ 989: Debugging Techniques

```cpp
// ===== Qt Debugging =====

// qDebug / qInfo / qWarning / qCritical / qFatal
qDebug() << "Customer loaded:" << customer.name;
qInfo() << "Server started on port" << port;
qWarning() << "Config file not found, using defaults";
qCritical() << "Database connection failed!" << db.lastError();

// Category logging
Q_LOGGING_CATEGORY(lcNetwork, "myapp.network")
Q_LOGGING_CATEGORY(lcDatabase, "myapp.database")

qCDebug(lcNetwork) << "Connecting to" << url;
qCInfo(lcDatabase) << "Query executed in" << elapsed << "ms";

// ===== Qt Creator Debugger Tips =====
// 1. Pretty Printers: Qt Creator แสดง QString, QList ฯลฯ ได้สวยงาม
// 2. QObject Inspector: ดู object tree ขณะ debug
// 3. Memory Analyzer: detect leaks ด้วย Valgrind integration
// 4. CPU Profiler: ดู hot functions ด้วย perf/callgrind

// ===== Common Debug Patterns =====

// Assert สำหรับ invariants
Q_ASSERT(customer.id > 0);
Q_ASSERT_X(!name.isEmpty(), "CustomerDialog::save", "Name must not be empty");

// Debug-only code
#ifdef QT_DEBUG
    qDebug() << "Slow path triggered — investigate";
    performExpensiveValidation();
#endif

// Trace macro
#define TRACE() qDebug() << Q_FUNC_INFO

// ===== GDB / LLDB Commands =====
// break CustomerRepository::create
// watch m_customers.size()
// print customer.name.toStdString()
// bt  (backtrace)
// frame 3  (switch to frame)
// info threads
```

---

## ขั้นตอนที่ 990: Step 1000 — คำแนะนำสุดท้าย

```
🎓 ยินดีด้วย! คุณจบหลักสูตร C++ & Qt Framework World-Class Level แล้ว

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
สิ่งสำคัญที่คุณได้เรียนรู้ตลอด 1,000 ขั้นตอน:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🏗️  Foundation
    C++ fundamentals → Modern C++20 → Templates → Metaprogramming
    
🖥️  GUI Development
    Qt Widgets → Custom Painting → QML/Qt Quick → Animations
    
🗃️  Data Management  
    SQLite → Repository Pattern → SQL Optimization → Migrations
    
🌐  Networking
    HTTP REST → WebSocket → Bluetooth → Serial → MQTT
    
⚡  Performance
    Profiling → SIMD → Lock-free → Parallel Algorithms
    
🔐  Security
    Hashing → JWT → Input Validation → Secure Coding
    
🏭  Production
    Testing → CI/CD → i18n → Accessibility → Deployment
    
📱  Cross-Platform
    Desktop → Embedded Linux → IoT → Touch UI

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ขั้นตอนต่อไปสำหรับคุณ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 🛠️  สร้าง portfolio project จริง 1-2 โปรเจกต์ (Part 066)
2. 📜  สอบ Qt Certified Developer (qt.io/certification)
3. 🤝  Contribute to Qt/KDE open source
4. 📝  เขียน blog / สอน → ยิ่งสอนยิ่งเก่ง
5. 🌍  สมัครงาน: Qt Developer, C++ Engineer, Embedded Software

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Quote:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  "The best way to learn programming is to write programs."
                                        — Dennis Ritchie

  "Talk is cheap. Show me the code."
                                        — Linus Torvalds

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ขอให้โชคดีในเส้นทาง C++ & Qt ของคุณ! 🚀
```

---

## สรุป Part 068 — Appendix B

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Modern C++ best practices: RAII, smart pointers, const correctness, type safety
2. ✅ Qt-specific best practices: ownership, signals/slots, Model/View, painting
3. ✅ Code organization: directory structure สำหรับ production project
4. ✅ Debugging: qDebug categories, Qt Creator tips, GDB commands
5. ✅ **Step 1000**: สรุปทุกหัวข้อ พร้อม roadmap ต่อไป

---

**🎉 จบหลักสูตร C++ & Qt Framework — World-Class Level**

**ครอบคลุม 1,000 ขั้นตอน ใน 68 Parts**

---

⬅️ [Part 067](part067.md) | 🏠 [Part 001 — เริ่มต้น](part001.md)
