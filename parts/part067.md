# Part 067: Appendix A — Qt Quick Reference

## ขั้นตอนที่ 971-985

---

## ขั้นตอนที่ 971: Qt Class Quick Reference

```cpp
// ===== QObject Essentials =====
class MyObject : public QObject {
    Q_OBJECT
    Q_PROPERTY(int count READ count WRITE setCount NOTIFY countChanged)
    
public:
    explicit MyObject(QObject* parent = nullptr);
    int count() const { return m_count; }
    
public slots:
    void setCount(int v) {
        if (m_count != v) {
            m_count = v;
            emit countChanged(v);
        }
    }
    
signals:
    void countChanged(int count);
    
private:
    int m_count{0};
};

// Connect patterns
// Lambda (modern)
connect(obj, &MyObject::countChanged, this, [this](int v) { … });

// Method pointer
connect(obj, &MyObject::countChanged, other, &OtherClass::onCountChanged);

// Disconnect
QMetaObject::Connection c = connect(…);
disconnect(c);

// ===== Containers Quick Reference =====
QList<T>         // Dynamic array (like std::vector in Qt6)
QVector<T>       // Alias for QList<T> in Qt6
QMap<K,V>        // Sorted key-value (std::map)
QHash<K,V>       // Unordered (std::unordered_map) — faster lookup
QSet<T>          // Unique values (std::unordered_set)
QQueue<T>        // FIFO queue
QStack<T>        // LIFO stack
QPair<A,B>       // Two values
QVariant         // Any Qt value
QVariantMap      // QMap<QString, QVariant>
QVariantList     // QList<QVariant>
QStringList      // QList<QString>

// ===== QString Quick Reference =====
QString s = "Hello, %1! You are %2 years old.";
s = s.arg("สมชาย").arg(25);

s.contains("Hello");              // true
s.startsWith("Hello");            // true
s.endsWith("old.");               // true
s.toLower() / s.toUpper();
s.trimmed();                      // remove leading/trailing whitespace
s.simplified();                   // single spaces inside
s.split(",");                     // → QStringList
s.replace("old", "young");
s.toInt() / s.toDouble();
s.toUtf8();                       // → QByteArray
QString::number(42);
QString::number(3.14, 'f', 2);    // "3.14"

// ===== QFile / QDir =====
QFile f(":/data/config.json");    // resource file
QFile f(QStandardPaths::writableLocation(
    QStandardPaths::AppDataLocation) + "/data.db");
f.open(QFile::ReadOnly);
QByteArray data = f.readAll();
f.close();

QDir dir = QDir::home();
dir.mkpath("myapp/data");
QStringList files = dir.entryList({"*.txt"}, QDir::Files);

// ===== QDateTime =====
QDateTime now = QDateTime::currentDateTime();
QDateTime utcNow = QDateTime::currentDateTimeUtc();

now.toString(Qt::ISODate);            // "2024-01-15T10:30:00"
now.toString("dd/MM/yyyy HH:mm");     // "15/01/2024 10:30"
now.toMSecsSinceEpoch();              // milliseconds since epoch
QDateTime::fromMSecsSinceEpoch(ms);

QDate today = QDate::currentDate();
today.addDays(30);
today.daysTo(other);

// ===== QSettings =====
QSettings settings;  // uses appName + orgName

settings.setValue("window/geometry", saveGeometry());
settings.setValue("user/name", "สมชาย");
settings.setValue("app/language", "th");

QByteArray geo = settings.value("window/geometry").toByteArray();
QString name = settings.value("user/name", "Guest").toString();
```

---

## ขั้นตอนที่ 972: Widget Quick Reference

```cpp
// ===== Layout Cheatsheet =====

// QVBoxLayout — แนวตั้ง
auto* vl = new QVBoxLayout(parent);
vl->addWidget(w1);
vl->addWidget(w2);
vl->addStretch();          // push widgets to top
vl->setSpacing(8);
vl->setContentsMargins(12, 12, 12, 12);

// QHBoxLayout — แนวนอน
auto* hl = new QHBoxLayout;
hl->addWidget(label);
hl->addWidget(edit);
hl->addStretch(1);         // weight for stretch

// QGridLayout
auto* gl = new QGridLayout;
gl->addWidget(w, row, col, rowspan, colspan);
gl->setColumnStretch(1, 1);  // column 1 gets all extra space
gl->setRowMinimumHeight(0, 40);

// QFormLayout — label + field pairs
auto* fl = new QFormLayout;
fl->addRow("ชื่อ:", nameEdit);
fl->addRow("อีเมล:", emailEdit);
fl->setFieldGrowthPolicy(QFormLayout::ExpandingFieldsGrow);

// ===== Common Widgets =====
// QPushButton
auto* btn = new QPushButton("บันทึก", parent);
btn->setIcon(QIcon(":/icons/save.png"));
btn->setShortcut(QKeySequence::Save);
btn->setDefault(true);     // Enter key triggers it
connect(btn, &QPushButton::clicked, this, &MyClass::onSave);

// QLineEdit
auto* edit = new QLineEdit(parent);
edit->setPlaceholderText("ใส่อีเมลที่นี่...");
edit->setEchoMode(QLineEdit::Password);
edit->setMaxLength(100);
edit->setValidator(new QIntValidator(0, 999, edit));
edit->setInputMask("000-000-0000");  // phone mask
connect(edit, &QLineEdit::returnPressed, btn, &QPushButton::click);

// QComboBox
auto* combo = new QComboBox(parent);
combo->addItems({"ตัวเลือก A", "ตัวเลือก B", "ตัวเลือก C"});
combo->addItem(QIcon(":/icons/star.png"), "Premium", 42);  // with data
combo->setCurrentIndex(1);
combo->currentData().toInt();   // get associated data
connect(combo, &QComboBox::currentIndexChanged, …);

// QTableWidget (simple)
auto* table = new QTableWidget(rows, cols, parent);
table->setHorizontalHeaderLabels({"ชื่อ", "ราคา", "จำนวน"});
table->setItem(r, c, new QTableWidgetItem("ข้อมูล"));
table->setSelectionBehavior(QAbstractItemView::SelectRows);
table->setEditTriggers(QAbstractItemView::NoEditTriggers);
table->horizontalHeader()->setSectionResizeMode(
    0, QHeaderView::Stretch);

// QSplitter
auto* splitter = new QSplitter(Qt::Horizontal, parent);
splitter->addWidget(leftPanel);
splitter->addWidget(rightPanel);
splitter->setStretchFactor(0, 1);
splitter->setStretchFactor(1, 3);
splitter->setSizes({200, 600});

// QTabWidget
auto* tabs = new QTabWidget(parent);
tabs->addTab(page1, QIcon(":/icons/home.png"), "หน้าหลัก");
tabs->addTab(page2, "ลูกค้า");
tabs->setTabPosition(QTabWidget::North);
tabs->setTabsClosable(true);
connect(tabs, &QTabWidget::tabCloseRequested,
        tabs, &QTabWidget::removeTab);

// QScrollArea
auto* scroll = new QScrollArea(parent);
scroll->setWidget(contentWidget);
scroll->setWidgetResizable(true);
scroll->setHorizontalScrollBarPolicy(Qt::ScrollBarAlwaysOff);
```

---

## ขั้นตอนที่ 973: SQL Quick Reference

```cpp
// ===== QSqlDatabase Setup =====
QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE");
db.setDatabaseName("/path/to/database.db");
db.open();

// ===== QSqlQuery =====
QSqlQuery q(db);

// Prepared statement with named bindings
q.prepare("INSERT INTO customers (name, email) VALUES (:name, :email)");
q.bindValue(":name", "สมชาย");
q.bindValue(":email", "somchai@example.com");
q.exec();
int id = q.lastInsertId().toInt();

// SELECT with iteration
q.prepare("SELECT id, name, email FROM customers WHERE status = :status");
q.bindValue(":status", "active");
q.exec();
while (q.next()) {
    int id = q.value("id").toInt();
    QString name = q.value("name").toString();
    QString email = q.value("email").toString();
}

// Transaction
db.transaction();
try {
    // multiple queries
    db.commit();
} catch (...) {
    db.rollback();
}

// ===== QSqlTableModel =====
auto* model = new QSqlTableModel(parent, db);
model->setTable("customers");
model->setFilter("status = 'active'");
model->setSort(1, Qt::AscendingOrder);  // sort by column 1
model->select();

model->setHeaderData(0, Qt::Horizontal, "รหัส");
model->setHeaderData(1, Qt::Horizontal, "ชื่อ");

auto* view = new QTableView;
view->setModel(model);

// Edit
model->setData(model->index(row, col), "new value");
model->submitAll();

// Insert
int r = model->rowCount();
model->insertRow(r);
model->setData(model->index(r, 1), "ชื่อใหม่");
model->submitAll();
```

---

## ขั้นตอนที่ 974: CMakeLists.txt Template

```cmake
cmake_minimum_required(VERSION 3.21)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

# C++ Standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Qt auto tools
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

# Find Qt
find_package(Qt6 6.5 REQUIRED COMPONENTS
    Core Gui Widgets Sql Network Charts WebSockets Concurrent
)

# Sources
file(GLOB_RECURSE SOURCES src/*.cpp src/*.h)

# Resources
qt_add_resources(RESOURCES resources/resources.qrc)

# Executable
qt_add_executable(MyApp
    src/main.cpp
    ${SOURCES}
    ${RESOURCES}
)

# macOS/Windows platform specific
set_target_properties(MyApp PROPERTIES
    WIN32_EXECUTABLE TRUE
    MACOSX_BUNDLE TRUE
    MACOSX_BUNDLE_GUI_IDENTIFIER "com.mycompany.myapp"
    MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
)

# Include dirs
target_include_directories(MyApp PRIVATE src)

# Link Qt
target_link_libraries(MyApp PRIVATE
    Qt6::Core Qt6::Gui Qt6::Widgets
    Qt6::Sql Qt6::Network Qt6::Charts
    Qt6::WebSockets Qt6::Concurrent
)

# Warnings
if(MSVC)
    target_compile_options(MyApp PRIVATE /W4)
else()
    target_compile_options(MyApp PRIVATE -Wall -Wextra -Wpedantic)
endif()

# Tests
enable_testing()
add_subdirectory(tests)

# Deploy
qt_generate_deploy_app_script(
    TARGET MyApp
    OUTPUT_SCRIPT deploy_script
    NO_UNSUPPORTED_PLATFORM_ERROR
)
install(SCRIPT ${deploy_script})
```

---

## ขั้นตอนที่ 975: Common Patterns Cheatsheet

```cpp
// ===== Singleton Pattern =====
class AppService {
public:
    static AppService& instance() {
        static AppService inst;
        return inst;
    }
    AppService(const AppService&) = delete;
    AppService& operator=(const AppService&) = delete;
private:
    AppService() = default;
};

// ===== RAII Pattern =====
class DatabaseTransaction {
public:
    explicit DatabaseTransaction(QSqlDatabase& db) : m_db(db) {
        m_db.transaction();
    }
    ~DatabaseTransaction() {
        if (!m_committed) m_db.rollback();
    }
    void commit() { m_db.commit(); m_committed = true; }
private:
    QSqlDatabase& m_db;
    bool m_committed{false};
};

// ===== Builder Pattern =====
class QueryBuilder {
public:
    QueryBuilder& from(const QString& table) { m_table = table; return *this; }
    QueryBuilder& where(const QString& cond) { m_conditions << cond; return *this; }
    QueryBuilder& orderBy(const QString& col, bool asc = true) {
        m_order = col + (asc ? " ASC" : " DESC");
        return *this;
    }
    QueryBuilder& limit(int n) { m_limit = n; return *this; }
    
    QString build() const {
        QString sql = "SELECT * FROM " + m_table;
        if (!m_conditions.isEmpty())
            sql += " WHERE " + m_conditions.join(" AND ");
        if (!m_order.isEmpty())
            sql += " ORDER BY " + m_order;
        if (m_limit > 0)
            sql += " LIMIT " + QString::number(m_limit);
        return sql;
    }
    
private:
    QString m_table;
    QStringList m_conditions;
    QString m_order;
    int m_limit{0};
};

// Usage: QueryBuilder().from("customers").where("status='active'").limit(10).build()

// ===== Observer via Qt Signals =====
class StockService : public QObject {
    Q_OBJECT
public:
    void updateStock(int productId, int qty) {
        m_stocks[productId] = qty;
        emit stockChanged(productId, qty);
        if (qty < 10) emit lowStock(productId, qty);
    }
signals:
    void stockChanged(int productId, int qty);
    void lowStock(int productId, int qty);
private:
    QMap<int, int> m_stocks;
};

// ===== State Machine Pattern =====
class OrderFSM : public QObject {
    Q_OBJECT
    Q_PROPERTY(QString state READ state NOTIFY stateChanged)
    
    enum State { Draft, Confirmed, Shipped, Delivered, Cancelled };
    
public:
    bool confirm() { return transition(Draft, Confirmed, [this]{ sendConfirmEmail(); }); }
    bool ship()    { return transition(Confirmed, Shipped, [this]{ printShippingLabel(); }); }
    bool deliver() { return transition(Shipped, Delivered, [this]{ sendDeliveryNotify(); }); }
    bool cancel()  {
        if (m_state == Draft || m_state == Confirmed) {
            m_state = Cancelled;
            emit stateChanged(state());
            return true;
        }
        return false;
    }
    
    QString state() const {
        static QMap<State, QString> names = {
            {Draft, "draft"}, {Confirmed, "confirmed"},
            {Shipped, "shipped"}, {Delivered, "delivered"},
            {Cancelled, "cancelled"}
        };
        return names[m_state];
    }
    
signals:
    void stateChanged(const QString& state);
    
private:
    template<typename Fn>
    bool transition(State from, State to, Fn&& action) {
        if (m_state != from) return false;
        std::forward<Fn>(action)();
        m_state = to;
        emit stateChanged(state());
        return true;
    }
    
    State m_state{Draft};
};
```

---

## สรุป Part 067

Quick reference ครอบคลุม:

1. ✅ QObject, Q_PROPERTY, Signals & Slots patterns
2. ✅ Qt Containers: QList, QMap, QHash, QSet
3. ✅ QString operations
4. ✅ Layout managers + common widgets
5. ✅ SQL: QSqlQuery, QSqlTableModel
6. ✅ CMakeLists.txt template สำหรับ production
7. ✅ Design patterns: Singleton, RAII, Builder, Observer, State Machine

---

⬅️ [Part 066](part066.md) | ➡️ [Part 068: Appendix B — C++ Best Practices](part068.md)
