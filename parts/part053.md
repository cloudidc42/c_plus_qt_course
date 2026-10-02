# Part 053: Capstone — ERP Sales Order System

## ขั้นตอนที่ 761-775

---

## ขั้นตอนที่ 761: Product Repository & Model

```cpp
// modules/inventory/product.h
#pragma once
#include <QString>
#include <QJsonObject>

struct Product {
    int id = 0;
    QString code;
    QString name;
    QString category;
    QString unit = "pcs";
    double costPrice = 0;
    double sellPrice = 0;
    double stockQty = 0;
    double minStock = 0;
    QString status = "active";
    QDateTime createdAt;
    
    bool isActive() const { return status == "active"; }
    bool isLowStock() const { return stockQty <= minStock; }
    double stockValue() const { return stockQty * costPrice; }
    
    static Product fromMap(const QVariantMap& m) {
        Product p;
        p.id        = m["id"].toInt();
        p.code      = m["code"].toString();
        p.name      = m["name"].toString();
        p.category  = m["category"].toString();
        p.unit      = m["unit"].toString();
        p.costPrice = m["cost_price"].toDouble();
        p.sellPrice = m["sell_price"].toDouble();
        p.stockQty  = m["stock_qty"].toDouble();
        p.minStock  = m["min_stock"].toDouble();
        p.status    = m["status"].toString();
        return p;
    }
    
    QJsonObject toJson() const {
        return {
            {"id", id}, {"code", code}, {"name", name},
            {"category", category}, {"unit", unit},
            {"sellPrice", sellPrice}, {"stockQty", stockQty}
        };
    }
};

class ProductRepository : public QObject {
    Q_OBJECT
    
public:
    explicit ProductRepository(QObject* parent = nullptr)
        : QObject(parent), db(Database::instance()) {}
    
    QList<Product> all(bool activeOnly = true) {
        QString sql = "SELECT * FROM products";
        if (activeOnly) sql += " WHERE status='active'";
        sql += " ORDER BY name";
        
        QList<Product> result;
        for (const auto& row : db.fetchAll(sql)) result.append(Product::fromMap(row));
        return result;
    }
    
    QList<Product> search(const QString& q, int limit = 50) {
        QString like = "%" + q + "%";
        QList<Product> result;
        for (const auto& row : db.fetchAll(
                "SELECT * FROM products WHERE (name LIKE ? OR code LIKE ?) AND status='active'"
                " ORDER BY name LIMIT ?", {like, like, limit})) {
            result.append(Product::fromMap(row));
        }
        return result;
    }
    
    QList<Product> lowStockProducts() {
        QList<Product> result;
        for (const auto& row : db.fetchAll(
                "SELECT * FROM products WHERE stock_qty <= min_stock AND status='active'"
                " ORDER BY stock_qty")) {
            result.append(Product::fromMap(row));
        }
        return result;
    }
    
    Product findById(int id) {
        return Product::fromMap(db.fetchOne("SELECT * FROM products WHERE id=?", {id}));
    }
    
    bool create(Product& p) {
        bool ok = db.exec(
            "INSERT INTO products(code,name,category,unit,cost_price,sell_price,"
            "stock_qty,min_stock,status,created_at,updated_at) VALUES(?,?,?,?,?,?,?,?,?,?,?)",
            {p.code, p.name, p.category, p.unit, p.costPrice, p.sellPrice,
             p.stockQty, p.minStock, p.status,
             QDateTime::currentDateTime().toString(Qt::ISODate),
             QDateTime::currentDateTime().toString(Qt::ISODate)}
        );
        if (ok) {
            p.id = db.lastInsertId();
            emit productCreated(p);
        }
        return ok;
    }
    
    bool adjustStock(int productId, double qty, const QString& reason = {}) {
        return db.transaction([&] {
            bool ok = db.exec(
                "UPDATE products SET stock_qty=stock_qty+?, updated_at=? WHERE id=?",
                {qty, QDateTime::currentDateTime().toString(Qt::ISODate), productId}
            );
            if (ok) emit stockAdjusted(productId, qty, reason);
            return ok;
        });
    }
    
signals:
    void productCreated(const Product& p);
    void stockAdjusted(int productId, double qty, const QString& reason);
    
private:
    Database& db;
};
```

---

## ขั้นตอนที่ 762: Sales Order Model

```cpp
// modules/sales/salesorder.h
#pragma once
#include <QList>
#include <QString>
#include <QDateTime>

struct OrderItem {
    int id = 0;
    int orderId = 0;
    int productId = 0;
    QString productCode;
    QString productName;
    QString unit;
    double qty = 0;
    double unitPrice = 0;
    double discount = 0;
    double lineTotal = 0;
    
    void recalculate() {
        lineTotal = qty * unitPrice * (1.0 - discount / 100.0);
    }
    
    static OrderItem fromMap(const QVariantMap& m) {
        OrderItem item;
        item.id          = m["id"].toInt();
        item.orderId     = m["order_id"].toInt();
        item.productId   = m["product_id"].toInt();
        item.qty         = m["qty"].toDouble();
        item.unitPrice   = m["unit_price"].toDouble();
        item.discount    = m["discount"].toDouble();
        item.lineTotal   = m["line_total"].toDouble();
        // Join fields
        item.productCode = m["product_code"].toString();
        item.productName = m["product_name"].toString();
        item.unit        = m["unit"].toString();
        return item;
    }
};

struct SalesOrder {
    int id = 0;
    QString orderNo;
    int customerId = 0;
    QString customerName;
    QString customerCode;
    QDate orderDate;
    QDate dueDate;
    QString status = "draft";  // draft, confirmed, shipped, invoiced, paid, cancelled
    double subtotal = 0;
    double discountPct = 0;
    double taxPct = 7;
    double total = 0;
    QString notes;
    QDateTime createdAt;
    
    QList<OrderItem> items;
    
    void recalculate() {
        subtotal = 0;
        for (auto& item : items) {
            item.recalculate();
            subtotal += item.lineTotal;
        }
        double discountAmt = subtotal * discountPct / 100.0;
        double taxAmt = (subtotal - discountAmt) * taxPct / 100.0;
        total = subtotal - discountAmt + taxAmt;
    }
    
    bool isDraft() const { return status == "draft"; }
    bool isEditable() const { return status == "draft" || status == "confirmed"; }
    bool isCancellable() const { return status != "paid" && status != "cancelled"; }
    
    QString statusLabel() const {
        static const QMap<QString, QString> labels{
            {"draft",     "ร่าง"},
            {"confirmed", "ยืนยันแล้ว"},
            {"shipped",   "จัดส่งแล้ว"},
            {"invoiced",  "ออกใบแจ้งหนี้แล้ว"},
            {"paid",      "ชำระแล้ว"},
            {"cancelled", "ยกเลิก"}
        };
        return labels.value(status, status);
    }
    
    static SalesOrder fromMap(const QVariantMap& m) {
        SalesOrder o;
        o.id           = m["id"].toInt();
        o.orderNo      = m["order_no"].toString();
        o.customerId   = m["customer_id"].toInt();
        o.customerName = m["customer_name"].toString();
        o.customerCode = m["customer_code"].toString();
        o.orderDate    = QDate::fromString(m["order_date"].toString(), Qt::ISODate);
        o.dueDate      = QDate::fromString(m["due_date"].toString(), Qt::ISODate);
        o.status       = m["status"].toString();
        o.subtotal     = m["subtotal"].toDouble();
        o.discountPct  = m["discount"].toDouble();
        o.taxPct       = m["tax"].toDouble();
        o.total        = m["total"].toDouble();
        o.notes        = m["notes"].toString();
        o.createdAt    = QDateTime::fromString(m["created_at"].toString(), Qt::ISODate);
        return o;
    }
};
```

---

## ขั้นตอนที่ 763: Sales Order Repository

```cpp
// modules/sales/salesorderrepository.h
class SalesOrderRepository : public QObject {
    Q_OBJECT
    
public:
    explicit SalesOrderRepository(QObject* parent = nullptr)
        : QObject(parent), db(Database::instance()) {}
    
    QList<SalesOrder> all(const QString& status = {}, int limit = 100) {
        QString sql = R"(
            SELECT so.*, c.name AS customer_name, c.code AS customer_code
            FROM sales_orders so
            LEFT JOIN customers c ON c.id = so.customer_id
        )";
        
        QVariantList params;
        if (!status.isEmpty()) {
            sql += " WHERE so.status=?";
            params << status;
        }
        
        sql += " ORDER BY so.order_date DESC LIMIT ?";
        params << limit;
        
        QList<SalesOrder> result;
        for (const auto& row : db.fetchAll(sql, params)) {
            result.append(SalesOrder::fromMap(row));
        }
        return result;
    }
    
    SalesOrder findById(int id) {
        auto row = db.fetchOne(R"(
            SELECT so.*, c.name AS customer_name, c.code AS customer_code
            FROM sales_orders so
            LEFT JOIN customers c ON c.id = so.customer_id
            WHERE so.id=?
        )", {id});
        
        SalesOrder order = SalesOrder::fromMap(row);
        
        // Load items
        auto itemRows = db.fetchAll(R"(
            SELECT soi.*, p.code AS product_code, p.name AS product_name, p.unit
            FROM sales_order_items soi
            LEFT JOIN products p ON p.id = soi.product_id
            WHERE soi.order_id=?
            ORDER BY soi.id
        )", {id});
        
        for (const auto& ir : itemRows) {
            order.items.append(OrderItem::fromMap(ir));
        }
        
        return order;
    }
    
    bool create(SalesOrder& order) {
        order.recalculate();
        order.createdAt = QDateTime::currentDateTime();
        
        return db.transaction([&] {
            bool ok = db.exec(R"(
                INSERT INTO sales_orders(order_no,customer_id,order_date,due_date,
                status,subtotal,discount,tax,total,notes,created_at,updated_at)
                VALUES(?,?,?,?,?,?,?,?,?,?,?,?)
            )", {order.orderNo, order.customerId,
                 order.orderDate.toString(Qt::ISODate),
                 order.dueDate.toString(Qt::ISODate),
                 order.status, order.subtotal, order.discountPct, order.taxPct,
                 order.total, order.notes,
                 order.createdAt.toString(Qt::ISODate),
                 order.createdAt.toString(Qt::ISODate)});
            
            if (!ok) return false;
            order.id = db.lastInsertId();
            
            for (auto& item : order.items) {
                item.orderId = order.id;
                ok = db.exec(
                    "INSERT INTO sales_order_items(order_id,product_id,qty,unit_price,discount,line_total)"
                    " VALUES(?,?,?,?,?,?)",
                    {item.orderId, item.productId, item.qty,
                     item.unitPrice, item.discount, item.lineTotal});
                if (!ok) return false;
            }
            
            emit orderCreated(order);
            return true;
        });
    }
    
    bool updateStatus(int orderId, const QString& newStatus) {
        bool ok = db.exec(
            "UPDATE sales_orders SET status=?, updated_at=? WHERE id=?",
            {newStatus, QDateTime::currentDateTime().toString(Qt::ISODate), orderId}
        );
        if (ok) emit orderStatusChanged(orderId, newStatus);
        return ok;
    }
    
    QString generateOrderNo() {
        QString prefix = "SO" + QDate::currentDate().toString("yyyyMM");
        int count = db.scalar(
            "SELECT COUNT(*)+1 FROM sales_orders WHERE order_no LIKE ?",
            {prefix + "%"}).toInt();
        return prefix + QString("%1").arg(count, 4, 10, QChar('0'));
    }
    
    // KPI queries
    double totalSalesThisMonth() {
        QDate from = QDate(QDate::currentDate().year(), QDate::currentDate().month(), 1);
        return db.scalar(
            "SELECT SUM(total) FROM sales_orders WHERE status!='cancelled'"
            " AND order_date>=?",
            {from.toString(Qt::ISODate)}).toDouble();
    }
    
    int orderCountByStatus(const QString& status) {
        return db.scalar(
            "SELECT COUNT(*) FROM sales_orders WHERE status=?",
            {status}).toInt();
    }
    
signals:
    void orderCreated(const SalesOrder& order);
    void orderStatusChanged(int orderId, const QString& status);
    
private:
    Database& db;
};
```

---

## ขั้นตอนที่ 764: Sales Order Form

```cpp
// modules/sales/salesorderdialog.h
#pragma once
#include <QDialog>
#include <QTableWidget>
#include <QHeaderView>
#include <QCompleter>
#include "salesorder.h"

class OrderItemsWidget : public QWidget {
    Q_OBJECT
    
public:
    explicit OrderItemsWidget(ProductRepository* products, QWidget* parent = nullptr)
        : QWidget(parent), m_products(products)
    {
        auto* layout = new QVBoxLayout(this);
        
        // Toolbar
        auto* toolbar = new QHBoxLayout;
        auto* addBtn = new QPushButton(QIcon::fromTheme("list-add"), "เพิ่มสินค้า", this);
        auto* removeBtn = new QPushButton(QIcon::fromTheme("list-remove"), "ลบรายการ", this);
        toolbar->addWidget(addBtn);
        toolbar->addWidget(removeBtn);
        toolbar->addStretch();
        
        // Items table
        m_table = new QTableWidget(0, 6, this);
        m_table->setHorizontalHeaderLabels({
            "สินค้า", "หน่วย", "จำนวน", "ราคา/หน่วย", "ส่วนลด%", "ยอดรวม"
        });
        m_table->horizontalHeader()->setSectionResizeMode(0, QHeaderView::Stretch);
        m_table->setSelectionBehavior(QAbstractItemView::SelectRows);
        m_table->setEditTriggers(QAbstractItemView::DoubleClicked
                                 | QAbstractItemView::AnyKeyPressed);
        
        layout->addLayout(toolbar);
        layout->addWidget(m_table);
        
        connect(addBtn, &QPushButton::clicked, this, &OrderItemsWidget::addRow);
        connect(removeBtn, &QPushButton::clicked, this, &OrderItemsWidget::removeSelectedRow);
        connect(m_table, &QTableWidget::cellChanged, this, &OrderItemsWidget::recalculateRow);
    }
    
    void setItems(const QList<OrderItem>& items) {
        m_table->setRowCount(0);
        for (const auto& item : items) {
            int row = m_table->rowCount();
            m_table->insertRow(row);
            populateRow(row, item);
        }
    }
    
    QList<OrderItem> items() const {
        QList<OrderItem> result;
        for (int row = 0; row < m_table->rowCount(); ++row) {
            OrderItem item;
            item.productName = m_table->item(row, 0)->text();
            item.productId   = m_table->item(row, 0)->data(Qt::UserRole).toInt();
            item.unit        = m_table->item(row, 1)->text();
            item.qty         = m_table->item(row, 2)->text().toDouble();
            item.unitPrice   = m_table->item(row, 3)->text().toDouble();
            item.discount    = m_table->item(row, 4)->text().toDouble();
            item.recalculate();
            result.append(item);
        }
        return result;
    }
    
    double total() const {
        double sum = 0;
        for (int row = 0; row < m_table->rowCount(); ++row) {
            if (auto* it = m_table->item(row, 5)) {
                sum += it->text().toDouble();
            }
        }
        return sum;
    }
    
signals:
    void totalChanged(double total);
    
private slots:
    void addRow() {
        // Product search dialog
        QDialog dlg(this);
        dlg.setWindowTitle("เลือกสินค้า");
        dlg.setMinimumWidth(500);
        
        auto* lay = new QVBoxLayout(&dlg);
        auto* searchEdit = new QLineEdit(&dlg);
        searchEdit->setPlaceholderText("ค้นหาสินค้า...");
        
        auto* listWidget = new QListWidget(&dlg);
        auto* btns = new QDialogButtonBox(
            QDialogButtonBox::Ok | QDialogButtonBox::Cancel, &dlg);
        
        lay->addWidget(searchEdit);
        lay->addWidget(listWidget);
        lay->addWidget(btns);
        
        auto loadProducts = [&](const QString& query) {
            listWidget->clear();
            auto products = query.isEmpty()
                ? m_products->all()
                : m_products->search(query);
            
            for (const auto& p : products) {
                auto* item = new QListWidgetItem(
                    QString("[%1] %2 — ราคา: %3 (คงเหลือ: %4 %5)")
                        .arg(p.code, p.name)
                        .arg(p.sellPrice, 0, 'f', 2)
                        .arg(p.stockQty)
                        .arg(p.unit));
                item->setData(Qt::UserRole, QVariant::fromValue(p));
                listWidget->addItem(item);
            }
        };
        
        loadProducts({});
        
        connect(searchEdit, &QLineEdit::textChanged, loadProducts);
        connect(btns, &QDialogButtonBox::accepted, &dlg, &QDialog::accept);
        connect(btns, &QDialogButtonBox::rejected, &dlg, &QDialog::reject);
        connect(listWidget, &QListWidget::itemDoubleClicked, &dlg, &QDialog::accept);
        
        if (dlg.exec() != QDialog::Accepted) return;
        
        auto* selected = listWidget->currentItem();
        if (!selected) return;
        
        auto product = selected->data(Qt::UserRole).value<Product>();
        
        int row = m_table->rowCount();
        m_table->insertRow(row);
        
        OrderItem item;
        item.productId   = product.id;
        item.productName = product.name;
        item.unit        = product.unit;
        item.qty         = 1;
        item.unitPrice   = product.sellPrice;
        item.recalculate();
        
        populateRow(row, item);
    }
    
    void removeSelectedRow() {
        int row = m_table->currentRow();
        if (row >= 0) {
            m_table->removeRow(row);
            emit totalChanged(total());
        }
    }
    
    void recalculateRow(int row, int col) {
        if (col != 2 && col != 3 && col != 4) return;
        
        double qty   = m_table->item(row, 2) ? m_table->item(row, 2)->text().toDouble() : 0;
        double price = m_table->item(row, 3) ? m_table->item(row, 3)->text().toDouble() : 0;
        double disc  = m_table->item(row, 4) ? m_table->item(row, 4)->text().toDouble() : 0;
        double lineTotal = qty * price * (1.0 - disc / 100.0);
        
        auto* totalItem = new QTableWidgetItem(QString::number(lineTotal, 'f', 2));
        totalItem->setFlags(totalItem->flags() & ~Qt::ItemIsEditable);
        totalItem->setTextAlignment(Qt::AlignRight | Qt::AlignVCenter);
        
        m_table->blockSignals(true);
        m_table->setItem(row, 5, totalItem);
        m_table->blockSignals(false);
        
        emit totalChanged(total());
    }
    
    void populateRow(int row, const OrderItem& item) {
        auto makeItem = [](const QString& text, bool editable = true) {
            auto* i = new QTableWidgetItem(text);
            if (!editable) i->setFlags(i->flags() & ~Qt::ItemIsEditable);
            i->setTextAlignment(Qt::AlignRight | Qt::AlignVCenter);
            return i;
        };
        
        auto* nameItem = new QTableWidgetItem(item.productName);
        nameItem->setData(Qt::UserRole, item.productId);
        nameItem->setFlags(nameItem->flags() & ~Qt::ItemIsEditable);
        nameItem->setTextAlignment(Qt::AlignLeft | Qt::AlignVCenter);
        
        m_table->setItem(row, 0, nameItem);
        m_table->setItem(row, 1, makeItem(item.unit, false));
        m_table->setItem(row, 2, makeItem(QString::number(item.qty)));
        m_table->setItem(row, 3, makeItem(QString::number(item.unitPrice, 'f', 2)));
        m_table->setItem(row, 4, makeItem(QString::number(item.discount)));
        m_table->setItem(row, 5, makeItem(QString::number(item.lineTotal, 'f', 2), false));
    }
    
    QTableWidget* m_table;
    ProductRepository* m_products;
};
```

---

## สรุป Part 053

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Product struct + Repository (search, low stock, adjust stock)
2. ✅ SalesOrder + OrderItem structs ด้วย recalculate()
3. ✅ SalesOrderRepository (create, update status, KPIs)
4. ✅ OrderItemsWidget ด้วย QTableWidget + product search dialog
5. ✅ Auto-calculate line totals + grand total

---

⬅️ [Part 052](part052.md) | ➡️ [Part 054: Capstone — ERP Dashboard & Reports](part054.md)
