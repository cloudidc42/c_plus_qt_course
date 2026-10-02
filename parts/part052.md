# Part 052: Capstone Project — Enterprise ERP System (Part 1)

## ขั้นตอนที่ 746-760

---

## ขั้นตอนที่ 746: ERP System Architecture

```
Enterprise ERP System — โครงสร้างระบบ:

Modules:
  ├── Core (shared models, DB, logging, settings)
  ├── CRM   (Customers, Contacts, Opportunities)
  ├── Inventory (Products, Warehouses, Stock movements)
  ├── Sales (Orders, Invoices, Payments)
  ├── HR (Employees, Attendance, Payroll)
  └── Dashboard (KPIs, Charts, Reports)

Architecture Pattern:
  - Modular plugin architecture
  - Repository pattern per module
  - EventBus for cross-module communication
  - QML UI with C++ backend models
  - SQLite for local, PostgreSQL for server

Directory Structure:
  erp/
  ├── core/
  │   ├── database/
  │   ├── models/
  │   └── services/
  ├── modules/
  │   ├── crm/
  │   ├── inventory/
  │   ├── sales/
  │   └── hr/
  ├── ui/
  │   ├── main/
  │   ├── components/
  │   └── themes/
  └── tests/
```

---

## ขั้นตอนที่ 747: Core Database Layer

```cpp
// core/database/database.h
#pragma once
#include <QSqlDatabase>
#include <QSqlQuery>
#include <QSqlError>
#include <QVariantMap>
#include <functional>

class Database : public QObject {
    Q_OBJECT
    
public:
    static Database& instance() {
        static Database db;
        return db;
    }
    
    bool connect(const QString& dbPath) {
        auto db = QSqlDatabase::addDatabase("QSQLITE");
        db.setDatabaseName(dbPath);
        
        if (!db.open()) {
            qCritical() << "DB open error:" << db.lastError().text();
            return false;
        }
        
        // Performance pragmas
        exec("PRAGMA journal_mode=WAL");
        exec("PRAGMA synchronous=NORMAL");
        exec("PRAGMA cache_size=10000");
        exec("PRAGMA foreign_keys=ON");
        
        runMigrations();
        return true;
    }
    
    bool exec(const QString& sql, const QVariantList& params = {}) {
        QSqlQuery q;
        q.prepare(sql);
        for (int i = 0; i < params.size(); ++i) {
            q.bindValue(i, params[i]);
        }
        if (!q.exec()) {
            qWarning() << "SQL Error:" << q.lastError().text();
            qWarning() << "SQL:" << sql;
            return false;
        }
        return true;
    }
    
    QList<QVariantMap> fetchAll(const QString& sql, const QVariantList& params = {}) {
        QSqlQuery q;
        q.prepare(sql);
        for (int i = 0; i < params.size(); ++i) {
            q.bindValue(i, params[i]);
        }
        
        QList<QVariantMap> results;
        if (!q.exec()) {
            qWarning() << "SQL Error:" << q.lastError().text();
            return results;
        }
        
        auto record = q.record();
        int cols = record.count();
        
        while (q.next()) {
            QVariantMap row;
            for (int i = 0; i < cols; ++i) {
                row[record.fieldName(i)] = q.value(i);
            }
            results.append(row);
        }
        return results;
    }
    
    QVariantMap fetchOne(const QString& sql, const QVariantList& params = {}) {
        auto results = fetchAll(sql, params);
        return results.isEmpty() ? QVariantMap{} : results.first();
    }
    
    QVariant scalar(const QString& sql, const QVariantList& params = {}) {
        QSqlQuery q;
        q.prepare(sql);
        for (int i = 0; i < params.size(); ++i) {
            q.bindValue(i, params[i]);
        }
        if (q.exec() && q.next()) return q.value(0);
        return {};
    }
    
    qint64 lastInsertId() {
        return QSqlQuery().lastInsertId().toLongLong();
    }
    
    bool transaction(std::function<bool()> fn) {
        QSqlDatabase::database().transaction();
        if (fn()) {
            QSqlDatabase::database().commit();
            return true;
        }
        QSqlDatabase::database().rollback();
        return false;
    }
    
private:
    void runMigrations() {
        exec(R"(CREATE TABLE IF NOT EXISTS _migrations (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL UNIQUE,
            applied_at TEXT NOT NULL
        ))");
        
        QStringList migrations = {
            "001_create_customers",
            "002_create_products",
            "003_create_orders",
            "004_create_employees"
        };
        
        for (const auto& name : migrations) {
            auto exists = scalar(
                "SELECT COUNT(*) FROM _migrations WHERE name=?", {name});
            
            if (exists.toInt() == 0) {
                applyMigration(name);
                exec("INSERT INTO _migrations(name, applied_at) VALUES(?,?)",
                     {name, QDateTime::currentDateTime().toString(Qt::ISODate)});
            }
        }
    }
    
    void applyMigration(const QString& name) {
        if (name == "001_create_customers") {
            exec(R"(CREATE TABLE customers (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                code TEXT NOT NULL UNIQUE,
                name TEXT NOT NULL,
                email TEXT,
                phone TEXT,
                address TEXT,
                credit_limit REAL DEFAULT 0,
                balance REAL DEFAULT 0,
                status TEXT DEFAULT 'active',
                created_at TEXT,
                updated_at TEXT
            ))");
        }
        else if (name == "002_create_products") {
            exec(R"(CREATE TABLE products (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                code TEXT NOT NULL UNIQUE,
                name TEXT NOT NULL,
                category TEXT,
                unit TEXT DEFAULT 'pcs',
                cost_price REAL DEFAULT 0,
                sell_price REAL DEFAULT 0,
                stock_qty REAL DEFAULT 0,
                min_stock REAL DEFAULT 0,
                status TEXT DEFAULT 'active',
                created_at TEXT,
                updated_at TEXT
            ))");
        }
        else if (name == "003_create_orders") {
            exec(R"(CREATE TABLE sales_orders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                order_no TEXT NOT NULL UNIQUE,
                customer_id INTEGER REFERENCES customers(id),
                order_date TEXT,
                due_date TEXT,
                status TEXT DEFAULT 'draft',
                subtotal REAL DEFAULT 0,
                discount REAL DEFAULT 0,
                tax REAL DEFAULT 0,
                total REAL DEFAULT 0,
                notes TEXT,
                created_at TEXT,
                updated_at TEXT
            ))");
            
            exec(R"(CREATE TABLE sales_order_items (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                order_id INTEGER REFERENCES sales_orders(id) ON DELETE CASCADE,
                product_id INTEGER REFERENCES products(id),
                qty REAL NOT NULL,
                unit_price REAL NOT NULL,
                discount REAL DEFAULT 0,
                line_total REAL NOT NULL
            ))");
        }
        else if (name == "004_create_employees") {
            exec(R"(CREATE TABLE employees (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                code TEXT NOT NULL UNIQUE,
                first_name TEXT NOT NULL,
                last_name TEXT NOT NULL,
                department TEXT,
                position TEXT,
                email TEXT,
                phone TEXT,
                hire_date TEXT,
                salary REAL DEFAULT 0,
                status TEXT DEFAULT 'active',
                created_at TEXT,
                updated_at TEXT
            ))");
        }
    }
};
```

---

## ขั้นตอนที่ 748: Customer Model & Repository

```cpp
// modules/crm/customer.h
#pragma once
#include <QString>
#include <QDateTime>
#include <QJsonObject>

struct Customer {
    int id = 0;
    QString code;
    QString name;
    QString email;
    QString phone;
    QString address;
    double creditLimit = 0;
    double balance = 0;
    QString status = "active";
    QDateTime createdAt;
    QDateTime updatedAt;
    
    bool isValid() const { return !name.isEmpty(); }
    
    bool isActive() const { return status == "active"; }
    
    QJsonObject toJson() const {
        return {
            {"id", id},
            {"code", code},
            {"name", name},
            {"email", email},
            {"phone", phone},
            {"address", address},
            {"creditLimit", creditLimit},
            {"balance", balance},
            {"status", status}
        };
    }
    
    static Customer fromMap(const QVariantMap& m) {
        Customer c;
        c.id          = m["id"].toInt();
        c.code        = m["code"].toString();
        c.name        = m["name"].toString();
        c.email       = m["email"].toString();
        c.phone       = m["phone"].toString();
        c.address     = m["address"].toString();
        c.creditLimit = m["credit_limit"].toDouble();
        c.balance     = m["balance"].toDouble();
        c.status      = m["status"].toString();
        c.createdAt   = QDateTime::fromString(m["created_at"].toString(), Qt::ISODate);
        c.updatedAt   = QDateTime::fromString(m["updated_at"].toString(), Qt::ISODate);
        return c;
    }
};

// modules/crm/customerrepository.h
class CustomerRepository : public QObject {
    Q_OBJECT
    
public:
    explicit CustomerRepository(QObject* parent = nullptr)
        : QObject(parent), db(Database::instance()) {}
    
    QList<Customer> all(bool activeOnly = false) {
        QString sql = "SELECT * FROM customers";
        if (activeOnly) sql += " WHERE status='active'";
        sql += " ORDER BY name";
        
        QList<Customer> result;
        for (const auto& row : db.fetchAll(sql)) {
            result.append(Customer::fromMap(row));
        }
        return result;
    }
    
    QList<Customer> search(const QString& query, int limit = 50) {
        QString like = "%" + query + "%";
        auto rows = db.fetchAll(
            "SELECT * FROM customers WHERE name LIKE ? OR code LIKE ? OR email LIKE ?"
            " ORDER BY name LIMIT ?",
            {like, like, like, limit}
        );
        
        QList<Customer> result;
        for (const auto& row : rows) result.append(Customer::fromMap(row));
        return result;
    }
    
    Customer findById(int id) {
        return Customer::fromMap(
            db.fetchOne("SELECT * FROM customers WHERE id=?", {id}));
    }
    
    Customer findByCode(const QString& code) {
        return Customer::fromMap(
            db.fetchOne("SELECT * FROM customers WHERE code=?", {code}));
    }
    
    bool create(Customer& c) {
        c.createdAt = QDateTime::currentDateTime();
        c.updatedAt = c.createdAt;
        
        bool ok = db.exec(
            "INSERT INTO customers(code,name,email,phone,address,credit_limit,balance,status,"
            "created_at,updated_at) VALUES(?,?,?,?,?,?,?,?,?,?)",
            {c.code, c.name, c.email, c.phone, c.address,
             c.creditLimit, c.balance, c.status,
             c.createdAt.toString(Qt::ISODate),
             c.updatedAt.toString(Qt::ISODate)}
        );
        
        if (ok) {
            c.id = db.lastInsertId();
            emit customerCreated(c);
        }
        return ok;
    }
    
    bool update(const Customer& c) {
        bool ok = db.exec(
            "UPDATE customers SET code=?,name=?,email=?,phone=?,address=?,"
            "credit_limit=?,balance=?,status=?,updated_at=? WHERE id=?",
            {c.code, c.name, c.email, c.phone, c.address,
             c.creditLimit, c.balance, c.status,
             QDateTime::currentDateTime().toString(Qt::ISODate),
             c.id}
        );
        if (ok) emit customerUpdated(c);
        return ok;
    }
    
    bool remove(int id) {
        bool ok = db.exec("DELETE FROM customers WHERE id=?", {id});
        if (ok) emit customerDeleted(id);
        return ok;
    }
    
    int count(bool activeOnly = false) {
        QString sql = "SELECT COUNT(*) FROM customers";
        if (activeOnly) sql += " WHERE status='active'";
        return db.scalar(sql).toInt();
    }
    
    double totalBalance() {
        return db.scalar("SELECT SUM(balance) FROM customers WHERE status='active'")
            .toDouble();
    }
    
    QString generateCode() {
        int next = db.scalar("SELECT COUNT(*)+1 FROM customers").toInt();
        return QString("CUST-%1").arg(next, 5, 10, QChar('0'));
    }
    
signals:
    void customerCreated(const Customer& c);
    void customerUpdated(const Customer& c);
    void customerDeleted(int id);
    
private:
    Database& db;
};
```

---

## ขั้นตอนที่ 749: Customer Table Model

```cpp
// modules/crm/customertablemodel.h
#pragma once
#include <QAbstractTableModel>
#include "customerrepository.h"

class CustomerTableModel : public QAbstractTableModel {
    Q_OBJECT
    
public:
    enum Column {
        ColCode = 0, ColName, ColEmail, ColPhone,
        ColBalance, ColCreditLimit, ColStatus,
        ColumnCount
    };
    
    explicit CustomerTableModel(CustomerRepository* repo, QObject* parent = nullptr)
        : QAbstractTableModel(parent), m_repo(repo)
    {
        reload();
        
        connect(repo, &CustomerRepository::customerCreated, this, [=] { reload(); });
        connect(repo, &CustomerRepository::customerUpdated, this, [=] { reload(); });
        connect(repo, &CustomerRepository::customerDeleted, this, [=] { reload(); });
    }
    
    void reload(const QString& searchQuery = {}) {
        beginResetModel();
        if (searchQuery.isEmpty()) {
            m_data = m_repo->all();
        } else {
            m_data = m_repo->search(searchQuery);
        }
        endResetModel();
    }
    
    int rowCount(const QModelIndex& = {}) const override { return m_data.size(); }
    int columnCount(const QModelIndex& = {}) const override { return ColumnCount; }
    
    QVariant headerData(int section, Qt::Orientation orientation, int role = Qt::DisplayRole) const override {
        if (orientation == Qt::Vertical || role != Qt::DisplayRole) return {};
        
        switch (section) {
            case ColCode:        return "รหัส";
            case ColName:        return "ชื่อลูกค้า";
            case ColEmail:       return "อีเมล";
            case ColPhone:       return "โทรศัพท์";
            case ColBalance:     return "ยอดค้างชำระ";
            case ColCreditLimit: return "วงเงินเครดิต";
            case ColStatus:      return "สถานะ";
            default: return {};
        }
    }
    
    QVariant data(const QModelIndex& index, int role = Qt::DisplayRole) const override {
        if (!index.isValid() || index.row() >= m_data.size()) return {};
        
        const auto& c = m_data[index.row()];
        
        if (role == Qt::DisplayRole) {
            switch (index.column()) {
                case ColCode:        return c.code;
                case ColName:        return c.name;
                case ColEmail:       return c.email;
                case ColPhone:       return c.phone;
                case ColBalance:     return QString("%L1").arg(c.balance, 0, 'f', 2);
                case ColCreditLimit: return QString("%L1").arg(c.creditLimit, 0, 'f', 2);
                case ColStatus:      return c.status == "active" ? "ใช้งาน" : "ไม่ใช้งาน";
                default: return {};
            }
        }
        
        if (role == Qt::UserRole) return c.id;
        
        if (role == Qt::TextAlignmentRole) {
            if (index.column() == ColBalance || index.column() == ColCreditLimit) {
                return static_cast<int>(Qt::AlignRight | Qt::AlignVCenter);
            }
        }
        
        if (role == Qt::ForegroundRole) {
            if (index.column() == ColStatus) {
                return c.isActive() ? QColor("#2ECC71") : QColor("#E74C3C");
            }
            if (index.column() == ColBalance && c.balance > 0) {
                return QColor("#E74C3C");
            }
        }
        
        return {};
    }
    
    Customer customerAt(int row) const {
        return (row >= 0 && row < m_data.size()) ? m_data[row] : Customer{};
    }
    
private:
    CustomerRepository* m_repo;
    QList<Customer> m_data;
};
```

---

## ขั้นตอนที่ 750: Customer Form Dialog

```cpp
// modules/crm/customerdialog.h
#pragma once
#include <QDialog>
#include <QFormLayout>
#include <QLineEdit>
#include <QDoubleSpinBox>
#include <QComboBox>
#include <QTextEdit>
#include <QDialogButtonBox>
#include <QMessageBox>
#include "customer.h"

class CustomerDialog : public QDialog {
    Q_OBJECT
    
public:
    explicit CustomerDialog(QWidget* parent = nullptr) : QDialog(parent) {
        setupUi();
    }
    
    CustomerDialog(const Customer& customer, QWidget* parent = nullptr)
        : QDialog(parent), m_customer(customer)
    {
        setupUi();
        loadCustomer();
        setWindowTitle("แก้ไขข้อมูลลูกค้า");
    }
    
    Customer customer() const { return m_customer; }
    
private slots:
    void validate() {
        QStringList errors;
        
        if (m_nameEdit->text().trimmed().isEmpty()) {
            errors << "กรุณากรอกชื่อลูกค้า";
        }
        
        if (!m_emailEdit->text().isEmpty()) {
            QRegularExpression emailRx(R"([^@]+@[^@]+\.[^@]+)");
            if (!emailRx.match(m_emailEdit->text()).hasMatch()) {
                errors << "รูปแบบอีเมลไม่ถูกต้อง";
            }
        }
        
        if (!errors.isEmpty()) {
            QMessageBox::warning(this, "ข้อมูลไม่ครบถ้วน", errors.join("\n"));
            return;
        }
        
        saveToCustomer();
        accept();
    }
    
private:
    void setupUi() {
        setWindowTitle("เพิ่มลูกค้าใหม่");
        setMinimumWidth(500);
        
        auto* layout = new QFormLayout(this);
        layout->setSpacing(10);
        
        m_codeEdit = new QLineEdit(this);
        m_codeEdit->setPlaceholderText("จะสร้างอัตโนมัติถ้าเว้นว่าง");
        
        m_nameEdit = new QLineEdit(this);
        m_nameEdit->setPlaceholderText("ชื่อบริษัท / บุคคล");
        
        m_emailEdit = new QLineEdit(this);
        m_phoneEdit = new QLineEdit(this);
        
        m_addressEdit = new QTextEdit(this);
        m_addressEdit->setMaximumHeight(80);
        
        m_creditLimitSpin = new QDoubleSpinBox(this);
        m_creditLimitSpin->setRange(0, 99999999);
        m_creditLimitSpin->setDecimals(2);
        m_creditLimitSpin->setSuffix(" บาท");
        m_creditLimitSpin->setGroupSeparatorShown(true);
        
        m_statusCombo = new QComboBox(this);
        m_statusCombo->addItems({"ใช้งาน", "ไม่ใช้งาน"});
        
        layout->addRow("รหัสลูกค้า:", m_codeEdit);
        layout->addRow("ชื่อลูกค้า *:", m_nameEdit);
        layout->addRow("อีเมล:", m_emailEdit);
        layout->addRow("โทรศัพท์:", m_phoneEdit);
        layout->addRow("ที่อยู่:", m_addressEdit);
        layout->addRow("วงเงินเครดิต:", m_creditLimitSpin);
        layout->addRow("สถานะ:", m_statusCombo);
        
        auto* btns = new QDialogButtonBox(
            QDialogButtonBox::Save | QDialogButtonBox::Cancel, this);
        layout->addRow(btns);
        
        connect(btns, &QDialogButtonBox::accepted, this, &CustomerDialog::validate);
        connect(btns, &QDialogButtonBox::rejected, this, &QDialog::reject);
    }
    
    void loadCustomer() {
        m_codeEdit->setText(m_customer.code);
        m_nameEdit->setText(m_customer.name);
        m_emailEdit->setText(m_customer.email);
        m_phoneEdit->setText(m_customer.phone);
        m_addressEdit->setPlainText(m_customer.address);
        m_creditLimitSpin->setValue(m_customer.creditLimit);
        m_statusCombo->setCurrentIndex(m_customer.isActive() ? 0 : 1);
    }
    
    void saveToCustomer() {
        m_customer.code        = m_codeEdit->text().trimmed();
        m_customer.name        = m_nameEdit->text().trimmed();
        m_customer.email       = m_emailEdit->text().trimmed();
        m_customer.phone       = m_phoneEdit->text().trimmed();
        m_customer.address     = m_addressEdit->toPlainText().trimmed();
        m_customer.creditLimit = m_creditLimitSpin->value();
        m_customer.status      = (m_statusCombo->currentIndex() == 0) ? "active" : "inactive";
    }
    
    Customer m_customer;
    QLineEdit* m_codeEdit;
    QLineEdit* m_nameEdit;
    QLineEdit* m_emailEdit;
    QLineEdit* m_phoneEdit;
    QTextEdit* m_addressEdit;
    QDoubleSpinBox* m_creditLimitSpin;
    QComboBox* m_statusCombo;
};
```

---

## สรุป Part 052

ใน Part นี้คุณได้เรียนรู้:

1. ✅ ERP System Architecture — modular, multi-layer
2. ✅ Core Database layer ด้วย SQLite, WAL mode, migrations
3. ✅ Customer struct + Repository pattern + signals
4. ✅ CustomerTableModel ด้วย QAbstractTableModel
5. ✅ CustomerDialog ด้วย QFormLayout + validation

---

⬅️ [Part 051](part051.md) | ➡️ [Part 053: Capstone — ERP Sales Order System](part053.md)
