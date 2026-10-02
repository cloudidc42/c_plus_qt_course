# Part 015: Qt Database (QtSQL)

## ขั้นตอนที่ 191-205

---

## ขั้นตอนที่ 191: Qt SQL Overview

```
Qt SQL Modules:
  ├── QSqlDatabase  - Connection management
  ├── QSqlQuery     - Execute SQL directly
  ├── QSqlTableModel - Model for single table
  ├── QSqlRelationalTableModel - With foreign keys
  └── QSqlQueryModel - Read-only query result

Supported Databases:
  ├── SQLite  (built-in, driver: QSQLITE)
  ├── MySQL   (driver: QMYSQL)
  ├── PostgreSQL (driver: QPSQL)
  ├── Oracle  (driver: QOCI)
  └── ODBC    (driver: QODBC)
```

**เพิ่ม QtSql ใน .pro:**
```qmake
QT += sql
```

---

## ขั้นตอนที่ 192: SQLite Connection & Basic Operations

```cpp
#include <QApplication>
#include <QSqlDatabase>
#include <QSqlQuery>
#include <QSqlError>
#include <QSqlRecord>
#include <QDebug>
#include <QFile>

class Database {
private:
    QSqlDatabase db;
    
public:
    Database(const QString& dbPath = "myapp.db") {
        db = QSqlDatabase::addDatabase("QSQLITE");
        db.setDatabaseName(dbPath);
    }
    
    bool open() {
        if (!db.open()) {
            qCritical() << "DB open failed:" << db.lastError().text();
            return false;
        }
        return true;
    }
    
    void close() { db.close(); }
    
    bool createTables() {
        QSqlQuery q;
        
        // สร้างตาราง users
        bool ok = q.exec(R"(
            CREATE TABLE IF NOT EXISTS users (
                id      INTEGER PRIMARY KEY AUTOINCREMENT,
                name    TEXT NOT NULL,
                email   TEXT UNIQUE NOT NULL,
                age     INTEGER,
                role    TEXT DEFAULT 'user',
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        )");
        if (!ok) { qCritical() << "Create users failed:" << q.lastError(); return false; }
        
        // สร้างตาราง products
        ok = q.exec(R"(
            CREATE TABLE IF NOT EXISTS products (
                id          INTEGER PRIMARY KEY AUTOINCREMENT,
                name        TEXT NOT NULL,
                price       REAL NOT NULL,
                stock       INTEGER DEFAULT 0,
                category    TEXT
            )
        )");
        if (!ok) { qCritical() << "Create products failed:" << q.lastError(); return false; }
        
        // สร้างตาราง orders
        ok = q.exec(R"(
            CREATE TABLE IF NOT EXISTS orders (
                id          INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id     INTEGER NOT NULL,
                product_id  INTEGER NOT NULL,
                quantity    INTEGER NOT NULL,
                total       REAL NOT NULL,
                order_date  DATETIME DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id)    REFERENCES users(id),
                FOREIGN KEY (product_id) REFERENCES products(id)
            )
        )");
        if (!ok) { qCritical() << "Create orders failed:" << q.lastError(); return false; }
        
        qDebug() << "Tables created OK";
        return true;
    }
    
    // === INSERT ===
    int insertUser(const QString& name, const QString& email, int age, const QString& role = "user") {
        QSqlQuery q;
        q.prepare("INSERT INTO users (name, email, age, role) VALUES (?, ?, ?, ?)");
        q.addBindValue(name);
        q.addBindValue(email);
        q.addBindValue(age);
        q.addBindValue(role);
        
        if (!q.exec()) {
            qCritical() << "Insert user failed:" << q.lastError().text();
            return -1;
        }
        return q.lastInsertId().toInt();
    }
    
    int insertProduct(const QString& name, double price, int stock, const QString& category) {
        QSqlQuery q;
        q.prepare("INSERT INTO products (name, price, stock, category) VALUES (:name, :price, :stock, :category)");
        q.bindValue(":name", name);
        q.bindValue(":price", price);
        q.bindValue(":stock", stock);
        q.bindValue(":category", category);
        
        if (!q.exec()) {
            qCritical() << "Insert product failed:" << q.lastError().text();
            return -1;
        }
        return q.lastInsertId().toInt();
    }
    
    // === SELECT ===
    QList<QVariantMap> getUsers() {
        QList<QVariantMap> result;
        QSqlQuery q("SELECT id, name, email, age, role, created_at FROM users ORDER BY id");
        
        while (q.next()) {
            QVariantMap user;
            user["id"]    = q.value("id");
            user["name"]  = q.value("name");
            user["email"] = q.value("email");
            user["age"]   = q.value("age");
            user["role"]  = q.value("role");
            user["created_at"] = q.value("created_at");
            result << user;
        }
        return result;
    }
    
    QVariantMap getUserById(int id) {
        QSqlQuery q;
        q.prepare("SELECT * FROM users WHERE id = ?");
        q.addBindValue(id);
        q.exec();
        
        if (q.next()) {
            QVariantMap user;
            QSqlRecord rec = q.record();
            for (int i = 0; i < rec.count(); i++) {
                user[rec.fieldName(i)] = q.value(i);
            }
            return user;
        }
        return {};
    }
    
    // === UPDATE ===
    bool updateUserRole(int id, const QString& role) {
        QSqlQuery q;
        q.prepare("UPDATE users SET role = ? WHERE id = ?");
        q.addBindValue(role);
        q.addBindValue(id);
        return q.exec();
    }
    
    // === DELETE ===
    bool deleteUser(int id) {
        QSqlQuery q;
        q.prepare("DELETE FROM users WHERE id = ?");
        q.addBindValue(id);
        return q.exec();
    }
    
    // === TRANSACTION ===
    bool transferOrder(int userId, int productId, int qty) {
        if (!db.transaction()) return false;
        
        try {
            // ตรวจสอบ stock
            QSqlQuery checkQ;
            checkQ.prepare("SELECT stock, price FROM products WHERE id = ?");
            checkQ.addBindValue(productId);
            checkQ.exec();
            
            if (!checkQ.next()) throw std::runtime_error("Product not found");
            
            int stock = checkQ.value("stock").toInt();
            double price = checkQ.value("price").toDouble();
            
            if (stock < qty) throw std::runtime_error("Insufficient stock");
            
            // ลด stock
            QSqlQuery updateQ;
            updateQ.prepare("UPDATE products SET stock = stock - ? WHERE id = ?");
            updateQ.addBindValue(qty);
            updateQ.addBindValue(productId);
            if (!updateQ.exec()) throw std::runtime_error("Update stock failed");
            
            // สร้าง order
            QSqlQuery orderQ;
            orderQ.prepare("INSERT INTO orders (user_id, product_id, quantity, total) VALUES (?,?,?,?)");
            orderQ.addBindValue(userId);
            orderQ.addBindValue(productId);
            orderQ.addBindValue(qty);
            orderQ.addBindValue(price * qty);
            if (!orderQ.exec()) throw std::runtime_error("Insert order failed");
            
            db.commit();
            qDebug() << "Order created successfully";
            return true;
            
        } catch (const std::exception& e) {
            db.rollback();
            qCritical() << "Transaction failed:" << e.what();
            return false;
        }
    }
    
    // === AGGREGATE QUERY ===
    QVariantMap getStatistics() {
        QVariantMap stats;
        
        QSqlQuery q;
        
        q.exec("SELECT COUNT(*) as total, AVG(age) as avg_age FROM users");
        if (q.next()) {
            stats["user_count"] = q.value("total");
            stats["avg_age"] = q.value("avg_age");
        }
        
        q.exec("SELECT COUNT(*) as total, SUM(stock * price) as value FROM products");
        if (q.next()) {
            stats["product_count"] = q.value("total");
            stats["inventory_value"] = q.value("value");
        }
        
        q.exec("SELECT COUNT(*) as total, SUM(total) as revenue FROM orders");
        if (q.next()) {
            stats["order_count"] = q.value("total");
            stats["revenue"] = q.value("revenue");
        }
        
        return stats;
    }
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    Database db;
    if (!db.open()) return 1;
    db.createTables();
    
    // Insert sample data
    int u1 = db.insertUser("สมชาย ใจดี",  "somchai@example.com", 28, "admin");
    int u2 = db.insertUser("สมหญิง รักดี", "somying@example.com", 25);
    int u3 = db.insertUser("อนุชา เก่งกาจ", "anucha@example.com",  32);
    
    int p1 = db.insertProduct("Laptop",    29990, 10, "Electronics");
    int p2 = db.insertProduct("Mouse",      990, 50, "Electronics");
    int p3 = db.insertProduct("Keyboard",  1490, 30, "Electronics");
    
    qDebug() << "Created users:" << u1 << u2 << u3;
    qDebug() << "Created products:" << p1 << p2 << p3;
    
    // Get all users
    auto users = db.getUsers();
    qDebug() << "\nAll Users:";
    for (const auto& u : users) {
        qDebug() << " -" << u["id"].toInt() << u["name"].toString() 
                 << u["role"].toString();
    }
    
    // Create order
    db.transferOrder(u1, p1, 1);
    db.transferOrder(u2, p2, 3);
    
    // Update
    db.updateUserRole(u2, "admin");
    
    // Statistics
    auto stats = db.getStatistics();
    qDebug() << "\nStatistics:";
    qDebug() << "Users:" << stats["user_count"].toInt();
    qDebug() << "Avg age:" << stats["avg_age"].toDouble();
    qDebug() << "Products:" << stats["product_count"].toInt();
    qDebug() << "Revenue:" << stats["revenue"].toDouble();
    
    return 0;
}
```

---

## ขั้นตอนที่ 193: QSqlTableModel

```cpp
#include <QSqlTableModel>
#include <QTableView>
#include <QHeaderView>

class ProductManager : public QWidget {
    Q_OBJECT
    
public:
    ProductManager(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("Product Manager");
        setMinimumSize(700, 500);
        
        // Model
        model = new QSqlTableModel(this);
        model->setTable("products");
        model->setEditStrategy(QSqlTableModel::OnFieldChange);  // Save on change
        model->select();
        
        // Header labels
        model->setHeaderData(0, Qt::Horizontal, "ID");
        model->setHeaderData(1, Qt::Horizontal, "ชื่อสินค้า");
        model->setHeaderData(2, Qt::Horizontal, "ราคา");
        model->setHeaderData(3, Qt::Horizontal, "สต็อก");
        model->setHeaderData(4, Qt::Horizontal, "หมวดหมู่");
        
        // View
        tableView = new QTableView();
        tableView->setModel(model);
        tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
        tableView->setSortingEnabled(true);
        tableView->horizontalHeader()->setStretchLastSection(true);
        tableView->verticalHeader()->setVisible(false);
        tableView->setColumnHidden(0, false);  // แสดง ID
        
        // Controls
        auto* addBtn    = new QPushButton("เพิ่มสินค้า");
        auto* deleteBtn = new QPushButton("ลบ");
        auto* revertBtn = new QPushButton("ยกเลิก");
        auto* submitBtn = new QPushButton("บันทึก");
        
        auto* filterEdit = new QLineEdit();
        filterEdit->setPlaceholderText("กรองหมวดหมู่...");
        
        auto* lowStockBtn = new QPushButton("แสดงสต็อกต่ำ");
        
        // Connections
        connect(addBtn, &QPushButton::clicked, [=]() {
            int row = model->rowCount();
            model->insertRow(row);
            tableView->selectRow(row);
            tableView->edit(model->index(row, 1));
        });
        
        connect(deleteBtn, &QPushButton::clicked, [=]() {
            QModelIndex idx = tableView->currentIndex();
            if (idx.isValid()) model->removeRow(idx.row());
        });
        
        connect(revertBtn, &QPushButton::clicked, model, &QSqlTableModel::revertAll);
        connect(submitBtn, &QPushButton::clicked, [=]() {
            if (model->submitAll()) {
                statusLabel->setText("บันทึกแล้ว");
            } else {
                statusLabel->setText("Error: " + model->lastError().text());
            }
        });
        
        connect(filterEdit, &QLineEdit::textChanged, [=](const QString& text) {
            if (text.isEmpty()) {
                model->setFilter("");
            } else {
                model->setFilter(QString("category LIKE '%%1%'").arg(text));
            }
            model->select();
        });
        
        connect(lowStockBtn, &QPushButton::clicked, [=]() {
            model->setFilter("stock < 10");
            model->select();
            statusLabel->setText("แสดงสินค้าที่สต็อก < 10");
        });
        
        statusLabel = new QLabel("พร้อมใช้งาน");
        
        // Layout
        auto* layout = new QVBoxLayout(this);
        auto* btnBar = new QHBoxLayout();
        btnBar->addWidget(new QLabel("กรอง:"));
        btnBar->addWidget(filterEdit);
        btnBar->addWidget(lowStockBtn);
        btnBar->addStretch();
        btnBar->addWidget(addBtn);
        btnBar->addWidget(deleteBtn);
        btnBar->addWidget(revertBtn);
        btnBar->addWidget(submitBtn);
        
        layout->addLayout(btnBar);
        layout->addWidget(tableView);
        layout->addWidget(statusLabel);
    }
    
private:
    QSqlTableModel* model;
    QTableView* tableView;
    QLabel* statusLabel;
};
```

---

## ขั้นตอนที่ 194-205: Full CRUD Application

```cpp
// === Full-featured CRUD with SQLite ===
class ContactBook : public QMainWindow {
    Q_OBJECT
    
public:
    ContactBook(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Contact Book");
        setMinimumSize(900, 600);
        
        setupDatabase();
        setupUi();
        setupConnections();
        loadContacts();
    }
    
private:
    QSqlDatabase db;
    QSqlQueryModel* queryModel;
    QTableView* tableView;
    
    // Form fields
    QLineEdit* nameEdit;
    QLineEdit* phoneEdit;
    QLineEdit* emailEdit;
    QComboBox* groupCombo;
    QTextEdit* notesEdit;
    
    QLabel* statusLabel;
    QLineEdit* searchEdit;
    
    int selectedId = -1;
    
    void setupDatabase() {
        db = QSqlDatabase::addDatabase("QSQLITE");
        db.setDatabaseName("contacts.db");
        db.open();
        
        QSqlQuery q;
        q.exec(R"(
            CREATE TABLE IF NOT EXISTS contacts (
                id      INTEGER PRIMARY KEY AUTOINCREMENT,
                name    TEXT NOT NULL,
                phone   TEXT,
                email   TEXT,
                group_name TEXT DEFAULT 'Personal',
                notes   TEXT,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        )");
        
        // Sample data
        int count = 0;
        QSqlQuery cnt("SELECT COUNT(*) FROM contacts");
        if (cnt.next()) count = cnt.value(0).toInt();
        
        if (count == 0) {
            q.exec("INSERT INTO contacts (name, phone, email, group_name) VALUES "
                   "('สมชาย ใจดี', '081-234-5678', 'somchai@email.com', 'Family')");
            q.exec("INSERT INTO contacts (name, phone, email, group_name) VALUES "
                   "('สมหญิง รักดี', '089-876-5432', 'somying@email.com', 'Work')");
            q.exec("INSERT INTO contacts (name, phone, email, group_name) VALUES "
                   "('อนุชา เก่งกาจ', '062-111-2222', 'anucha@email.com', 'Friend')");
        }
    }
    
    void setupUi() {
        // Central widget
        auto* central = new QWidget(this);
        setCentralWidget(central);
        
        auto* mainLayout = new QHBoxLayout(central);
        
        // === Left: List ===
        auto* leftPanel = new QVBoxLayout();
        
        searchEdit = new QLineEdit();
        searchEdit->setPlaceholderText("ค้นหา...");
        
        tableView = new QTableView();
        tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
        tableView->setSelectionMode(QAbstractItemView::SingleSelection);
        tableView->setEditTriggers(QAbstractItemView::NoEditTriggers);
        tableView->verticalHeader()->setVisible(false);
        tableView->horizontalHeader()->setStretchLastSection(true);
        tableView->setAlternatingRowColors(true);
        
        leftPanel->addWidget(searchEdit);
        leftPanel->addWidget(tableView);
        
        // === Right: Form ===
        auto* rightPanel = new QVBoxLayout();
        
        auto* formGroup = new QGroupBox("รายละเอียด");
        auto* form = new QFormLayout(formGroup);
        
        nameEdit  = new QLineEdit();
        phoneEdit = new QLineEdit();
        emailEdit = new QLineEdit();
        groupCombo = new QComboBox();
        groupCombo->addItems({"Personal", "Work", "Family", "Friend", "Other"});
        notesEdit = new QTextEdit();
        notesEdit->setMaximumHeight(80);
        
        form->addRow("ชื่อ:", nameEdit);
        form->addRow("โทร:", phoneEdit);
        form->addRow("อีเมล:", emailEdit);
        form->addRow("กลุ่ม:", groupCombo);
        form->addRow("หมายเหตุ:", notesEdit);
        
        auto* saveBtn   = new QPushButton("บันทึก");
        auto* deleteBtn = new QPushButton("ลบ");
        auto* clearBtn  = new QPushButton("ล้าง");
        
        saveBtn->setStyleSheet("background: #4CAF50; color: white; padding: 6px 20px;");
        deleteBtn->setStyleSheet("background: #f44336; color: white; padding: 6px 20px;");
        
        auto* btnLayout = new QHBoxLayout();
        btnLayout->addWidget(saveBtn);
        btnLayout->addWidget(deleteBtn);
        btnLayout->addWidget(clearBtn);
        
        rightPanel->addWidget(formGroup);
        rightPanel->addLayout(btnLayout);
        rightPanel->addStretch();
        
        statusLabel = new QLabel("พร้อมใช้งาน");
        rightPanel->addWidget(statusLabel);
        
        mainLayout->addLayout(leftPanel, 2);
        mainLayout->addLayout(rightPanel, 1);
        
        // Store buttons for connections
        this->findChild<QPushButton*>("saveBtn")?
            this->findChild<QPushButton*>("saveBtn") :
            saveBtn->setObjectName("saveBtn");
        saveBtn->setObjectName("saveBtn");
        deleteBtn->setObjectName("deleteBtn");
        clearBtn->setObjectName("clearBtn");
    }
    
    void setupConnections() {
        connect(searchEdit, &QLineEdit::textChanged, this, &ContactBook::loadContacts);
        
        connect(tableView->selectionModel(), 
                &QItemSelectionModel::currentRowChanged,
                this, &ContactBook::onRowSelected);
        
        connect(findChild<QPushButton*>("saveBtn"), &QPushButton::clicked,
                this, &ContactBook::saveContact);
        connect(findChild<QPushButton*>("deleteBtn"), &QPushButton::clicked,
                this, &ContactBook::deleteContact);
        connect(findChild<QPushButton*>("clearBtn"), &QPushButton::clicked,
                this, &ContactBook::clearForm);
    }
    
    void loadContacts() {
        QString search = searchEdit->text().trimmed();
        
        QString sql = "SELECT id, name, phone, email, group_name FROM contacts";
        if (!search.isEmpty()) {
            sql += QString(" WHERE name LIKE '%%1%' OR phone LIKE '%%1%' OR email LIKE '%%1%'")
                   .arg(search);
        }
        sql += " ORDER BY name";
        
        queryModel = new QSqlQueryModel(this);
        queryModel->setQuery(sql);
        queryModel->setHeaderData(0, Qt::Horizontal, "ID");
        queryModel->setHeaderData(1, Qt::Horizontal, "ชื่อ");
        queryModel->setHeaderData(2, Qt::Horizontal, "โทร");
        queryModel->setHeaderData(3, Qt::Horizontal, "อีเมล");
        queryModel->setHeaderData(4, Qt::Horizontal, "กลุ่ม");
        
        tableView->setModel(queryModel);
        tableView->setColumnHidden(0, true);  // ซ่อน ID
        
        statusLabel->setText(QString("พบ %1 รายการ").arg(queryModel->rowCount()));
    }
    
    void onRowSelected(const QModelIndex& current, const QModelIndex&) {
        if (!current.isValid()) return;
        
        selectedId = queryModel->data(queryModel->index(current.row(), 0)).toInt();
        
        QSqlQuery q;
        q.prepare("SELECT * FROM contacts WHERE id = ?");
        q.addBindValue(selectedId);
        q.exec();
        
        if (q.next()) {
            nameEdit->setText(q.value("name").toString());
            phoneEdit->setText(q.value("phone").toString());
            emailEdit->setText(q.value("email").toString());
            groupCombo->setCurrentText(q.value("group_name").toString());
            notesEdit->setPlainText(q.value("notes").toString());
        }
    }
    
    void saveContact() {
        QString name = nameEdit->text().trimmed();
        if (name.isEmpty()) {
            statusLabel->setText("กรุณาใส่ชื่อ");
            return;
        }
        
        QSqlQuery q;
        if (selectedId > 0) {
            q.prepare("UPDATE contacts SET name=?, phone=?, email=?, group_name=?, notes=? WHERE id=?");
            q.addBindValue(name);
            q.addBindValue(phoneEdit->text());
            q.addBindValue(emailEdit->text());
            q.addBindValue(groupCombo->currentText());
            q.addBindValue(notesEdit->toPlainText());
            q.addBindValue(selectedId);
            q.exec();
            statusLabel->setText("อัปเดตแล้ว: " + name);
        } else {
            q.prepare("INSERT INTO contacts (name, phone, email, group_name, notes) VALUES (?,?,?,?,?)");
            q.addBindValue(name);
            q.addBindValue(phoneEdit->text());
            q.addBindValue(emailEdit->text());
            q.addBindValue(groupCombo->currentText());
            q.addBindValue(notesEdit->toPlainText());
            q.exec();
            statusLabel->setText("เพิ่มแล้ว: " + name);
        }
        
        loadContacts();
    }
    
    void deleteContact() {
        if (selectedId <= 0) return;
        if (QMessageBox::question(this, "ยืนยัน", "ลบรายชื่อนี้?") != QMessageBox::Yes) return;
        
        QSqlQuery q;
        q.prepare("DELETE FROM contacts WHERE id = ?");
        q.addBindValue(selectedId);
        q.exec();
        
        clearForm();
        loadContacts();
        statusLabel->setText("ลบแล้ว");
    }
    
    void clearForm() {
        selectedId = -1;
        nameEdit->clear();
        phoneEdit->clear();
        emailEdit->clear();
        groupCombo->setCurrentIndex(0);
        notesEdit->clear();
    }
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    ContactBook book;
    book.show();
    
    return app.exec();
}
```

---

## สรุป Part 015

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt SQL - การเชื่อมต่อ SQLite
2. ✅ QSqlQuery - Execute SQL
3. ✅ Transaction
4. ✅ QSqlTableModel
5. ✅ Full CRUD Application

---

⬅️ [Part 014](part014.md) | ➡️ [Part 016: Qt Network](part016.md)
