# Part 055: Capstone — ERP Main Window & Navigation

## ขั้นตอนที่ 791-805

---

## ขั้นตอนที่ 791: Side Navigation Bar

```cpp
// ui/main/sidenavbar.h
#pragma once
#include <QWidget>
#include <QPainter>
#include <QScrollArea>

struct NavItem {
    QString id;
    QString icon;     // unicode char from icon font (or Qt icon name)
    QString label;
    int badge = 0;    // notification badge count
    QList<NavItem> children;
};

class SideNavBar : public QWidget {
    Q_OBJECT
    
public:
    explicit SideNavBar(QWidget* parent = nullptr) : QWidget(parent) {
        setFixedWidth(220);
        setObjectName("SideNavBar");
        setStyleSheet(R"(
            #SideNavBar {
                background: #1E2A38;
            }
        )");
        
        setupItems();
        setupUi();
    }
    
    void setBadge(const QString& itemId, int count) {
        for (auto& item : m_items) {
            if (item.id == itemId) {
                item.badge = count;
                break;
            }
        }
        update();
        if (auto* btn = m_buttons.value(itemId)) {
            if (count > 0) {
                btn->setText(QString("%1  (%2)").arg(btn->property("baseText").toString()).arg(count));
            } else {
                btn->setText(btn->property("baseText").toString());
            }
        }
    }
    
signals:
    void pageSelected(const QString& pageId);
    
private:
    void setupItems() {
        m_items = {
            {"dashboard",  "📊", "แดชบอร์ด"},
            {"customers",  "👥", "ลูกค้า"},
            {"products",   "📦", "สินค้า"},
            {"orders",     "📋", "คำสั่งซื้อ"},
            {"invoices",   "📄", "ใบแจ้งหนี้"},
            {"employees",  "🏢", "พนักงาน"},
            {"reports",    "📈", "รายงาน"},
            {"settings",   "⚙️", "ตั้งค่า"}
        };
    }
    
    void setupUi() {
        auto* vlay = new QVBoxLayout(this);
        vlay->setContentsMargins(0, 0, 0, 0);
        vlay->setSpacing(0);
        
        // Logo area
        auto* logoWidget = new QWidget(this);
        logoWidget->setFixedHeight(72);
        logoWidget->setStyleSheet("background: #16202C;");
        
        auto* logoLayout = new QHBoxLayout(logoWidget);
        auto* logoLabel = new QLabel("🏢 ERP Pro", logoWidget);
        logoLabel->setStyleSheet("color: white; font-size: 16px; font-weight: bold;");
        logoLayout->addWidget(logoLabel);
        
        vlay->addWidget(logoWidget);
        
        // Navigation items
        auto* scroll = new QScrollArea(this);
        scroll->setWidgetResizable(true);
        scroll->setFrameStyle(QFrame::NoFrame);
        scroll->setHorizontalScrollBarPolicy(Qt::ScrollBarAlwaysOff);
        scroll->setStyleSheet("background: transparent;");
        
        auto* itemsContainer = new QWidget;
        itemsContainer->setStyleSheet("background: transparent;");
        auto* itemsLayout = new QVBoxLayout(itemsContainer);
        itemsLayout->setContentsMargins(8, 8, 8, 8);
        itemsLayout->setSpacing(2);
        
        QString firstId;
        
        for (const auto& item : m_items) {
            if (firstId.isEmpty()) firstId = item.id;
            
            auto* btn = new QPushButton(
                QString("%1  %2").arg(item.icon).arg(item.label),
                itemsContainer);
            btn->setProperty("navId", item.id);
            btn->setProperty("baseText", QString("%1  %2").arg(item.icon).arg(item.label));
            btn->setCheckable(true);
            btn->setAutoExclusive(true);
            btn->setMinimumHeight(44);
            btn->setCursor(Qt::PointingHandCursor);
            btn->setStyleSheet(R"(
                QPushButton {
                    color: #BDC3C7;
                    background: transparent;
                    border: none;
                    border-radius: 8px;
                    padding: 8px 12px;
                    text-align: left;
                    font-size: 13px;
                }
                QPushButton:hover {
                    background: #2C3E50;
                    color: white;
                }
                QPushButton:checked {
                    background: #3498DB;
                    color: white;
                    font-weight: bold;
                }
            )");
            
            m_buttons[item.id] = btn;
            itemsLayout->addWidget(btn);
            
            connect(btn, &QPushButton::clicked, this, [=] {
                emit pageSelected(item.id);
            });
        }
        
        itemsLayout->addStretch();
        scroll->setWidget(itemsContainer);
        vlay->addWidget(scroll);
        
        // User info footer
        auto* footer = new QWidget(this);
        footer->setFixedHeight(60);
        footer->setStyleSheet("background: #16202C; border-top: 1px solid #2C3E50;");
        
        auto* footerLayout = new QHBoxLayout(footer);
        
        auto* avatar = new QLabel("👤", footer);
        avatar->setFixedSize(36, 36);
        avatar->setAlignment(Qt::AlignCenter);
        avatar->setStyleSheet("background: #3498DB; border-radius: 18px; color: white; font-size: 18px;");
        
        auto* userInfo = new QWidget(footer);
        auto* userLayout = new QVBoxLayout(userInfo);
        userLayout->setContentsMargins(0, 0, 0, 0);
        userLayout->setSpacing(0);
        
        auto* nameLabel = new QLabel("ผู้ดูแลระบบ", userInfo);
        nameLabel->setStyleSheet("color: white; font-weight: bold; font-size: 12px;");
        
        auto* roleLabel = new QLabel("Administrator", userInfo);
        roleLabel->setStyleSheet("color: #7F8C8D; font-size: 11px;");
        
        userLayout->addWidget(nameLabel);
        userLayout->addWidget(roleLabel);
        
        footerLayout->addWidget(avatar);
        footerLayout->addWidget(userInfo);
        
        vlay->addWidget(footer);
    }
    
    QList<NavItem> m_items;
    QMap<QString, QPushButton*> m_buttons;
};
```

---

## ขั้นตอนที่ 792: Customers Page

```cpp
// ui/pages/customerspage.h
#pragma once
#include <QWidget>
#include <QToolBar>
#include <QStatusBar>

class CustomersPage : public QWidget {
    Q_OBJECT
    
public:
    explicit CustomersPage(CustomerRepository* repo, QWidget* parent = nullptr)
        : QWidget(parent), m_repo(repo)
    {
        setupUi();
        reload();
        
        connect(repo, &CustomerRepository::customerCreated, this, [=] { reload(); });
        connect(repo, &CustomerRepository::customerUpdated, this, [=] { reload(); });
        connect(repo, &CustomerRepository::customerDeleted, this, [=] { reload(); });
    }
    
private slots:
    void addCustomer() {
        CustomerDialog dlg(this);
        if (dlg.exec() != QDialog::Accepted) return;
        
        Customer c = dlg.customer();
        if (c.code.isEmpty()) {
            c.code = m_repo->generateCode();
        }
        
        if (!m_repo->create(c)) {
            QMessageBox::critical(this, "ข้อผิดพลาด", "ไม่สามารถบันทึกข้อมูลได้");
        }
    }
    
    void editCustomer() {
        int row = m_tableView->currentIndex().row();
        if (row < 0) return;
        
        Customer c = m_model->customerAt(row);
        CustomerDialog dlg(c, this);
        
        if (dlg.exec() != QDialog::Accepted) return;
        
        if (!m_repo->update(dlg.customer())) {
            QMessageBox::critical(this, "ข้อผิดพลาด", "ไม่สามารถบันทึกข้อมูลได้");
        }
    }
    
    void deleteCustomer() {
        int row = m_tableView->currentIndex().row();
        if (row < 0) return;
        
        Customer c = m_model->customerAt(row);
        
        auto reply = QMessageBox::question(this, "ยืนยันการลบ",
            QString("ต้องการลบลูกค้า '%1' หรือไม่?").arg(c.name),
            QMessageBox::Yes | QMessageBox::No);
        
        if (reply == QMessageBox::Yes) {
            m_repo->remove(c.id);
        }
    }
    
    void reload(const QString& query = {}) {
        m_model->reload(query);
        updateStatusBar();
    }
    
private:
    void setupUi() {
        auto* layout = new QVBoxLayout(this);
        layout->setContentsMargins(0, 0, 0, 0);
        layout->setSpacing(0);
        
        // Toolbar
        auto* toolbar = new QToolBar(this);
        toolbar->setIconSize(QSize(20, 20));
        toolbar->setMovable(false);
        toolbar->setStyleSheet("QToolBar { border-bottom: 1px solid #ddd; padding: 4px; }");
        
        auto* addAction = toolbar->addAction(QIcon::fromTheme("list-add"), "เพิ่ม");
        auto* editAction = toolbar->addAction(QIcon::fromTheme("document-edit"), "แก้ไข");
        auto* deleteAction = toolbar->addAction(QIcon::fromTheme("edit-delete"), "ลบ");
        toolbar->addSeparator();
        
        auto* exportAction = toolbar->addAction(QIcon::fromTheme("document-export"), "ส่งออก Excel");
        
        toolbar->addSeparator();
        
        m_searchEdit = new QLineEdit(this);
        m_searchEdit->setPlaceholderText("ค้นหาลูกค้า...");
        m_searchEdit->setClearButtonEnabled(true);
        m_searchEdit->setFixedWidth(250);
        toolbar->addWidget(m_searchEdit);
        
        layout->addWidget(toolbar);
        
        // Table
        m_model = new CustomerTableModel(m_repo, this);
        m_proxyModel = new QSortFilterProxyModel(this);
        m_proxyModel->setSourceModel(m_model);
        m_proxyModel->setSortCaseSensitivity(Qt::CaseInsensitive);
        
        m_tableView = new QTableView(this);
        m_tableView->setModel(m_proxyModel);
        m_tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
        m_tableView->setAlternatingRowColors(true);
        m_tableView->setSortingEnabled(true);
        m_tableView->horizontalHeader()->setStretchLastSection(true);
        m_tableView->verticalHeader()->setDefaultSectionSize(32);
        m_tableView->setContextMenuPolicy(Qt::CustomContextMenu);
        m_tableView->sortByColumn(CustomerTableModel::ColName, Qt::AscendingOrder);
        
        layout->addWidget(m_tableView, 1);
        
        // Status bar
        m_statusLabel = new QLabel(this);
        m_statusLabel->setStyleSheet("padding: 4px 8px; color: #666;");
        layout->addWidget(m_statusLabel);
        
        // Connections
        connect(addAction, &QAction::triggered, this, &CustomersPage::addCustomer);
        connect(editAction, &QAction::triggered, this, &CustomersPage::editCustomer);
        connect(deleteAction, &QAction::triggered, this, &CustomersPage::deleteCustomer);
        
        connect(m_tableView, &QTableView::doubleClicked, this, [=] {
            editCustomer();
        });
        
        connect(m_tableView, &QTableView::customContextMenuRequested, this, [=](const QPoint& pos) {
            auto* menu = new QMenu(this);
            menu->addAction("แก้ไข", this, &CustomersPage::editCustomer);
            menu->addAction("ลบ", this, &CustomersPage::deleteCustomer);
            menu->exec(m_tableView->viewport()->mapToGlobal(pos));
        });
        
        connect(m_searchEdit, &QLineEdit::textChanged, this, [=](const QString& q) {
            m_model->reload(q);
            updateStatusBar();
        });
        
        connect(exportAction, &QAction::triggered, this, &CustomersPage::exportToExcel);
    }
    
    void updateStatusBar() {
        m_statusLabel->setText(
            QString("แสดง %1 รายการ จากทั้งหมด %2 ราย")
                .arg(m_model->rowCount())
                .arg(m_repo->count()));
    }
    
    void exportToExcel() {
        QString path = QFileDialog::getSaveFileName(
            this, "บันทึกไฟล์", QDir::homePath() + "/customers.csv",
            "CSV Files (*.csv)");
        
        if (path.isEmpty()) return;
        
        QFile file(path);
        if (!file.open(QIODevice::WriteOnly | QIODevice::Text)) {
            QMessageBox::critical(this, "ข้อผิดพลาด", "ไม่สามารถบันทึกไฟล์ได้");
            return;
        }
        
        QTextStream out(&file);
        out.setEncoding(QStringConverter::Utf8);
        out << "\xEF\xBB\xBF";  // UTF-8 BOM for Excel
        
        // Headers
        QStringList headers;
        for (int col = 0; col < m_model->columnCount(); ++col) {
            headers << m_model->headerData(col, Qt::Horizontal).toString();
        }
        out << headers.join(",") << "\n";
        
        // Data
        for (int row = 0; row < m_model->rowCount(); ++row) {
            QStringList fields;
            for (int col = 0; col < m_model->columnCount(); ++col) {
                QString val = m_model->data(m_model->index(row, col)).toString();
                if (val.contains(',') || val.contains('"') || val.contains('\n')) {
                    val = "\"" + val.replace("\"", "\"\"") + "\"";
                }
                fields << val;
            }
            out << fields.join(",") << "\n";
        }
        
        QMessageBox::information(this, "สำเร็จ",
            QString("ส่งออกข้อมูลเรียบร้อย: %1").arg(path));
    }
    
    CustomerRepository* m_repo;
    CustomerTableModel* m_model;
    QSortFilterProxyModel* m_proxyModel;
    QTableView* m_tableView;
    QLineEdit* m_searchEdit;
    QLabel* m_statusLabel;
};
```

---

## ขั้นตอนที่ 793: ERP Main Window

```cpp
// ui/main/erpmainwindow.h
#pragma once
#include <QMainWindow>
#include <QStackedWidget>
#include "sidenavbar.h"
#include "../dashboard/erpdashboard.h"
#include "../pages/customerspage.h"

class ErpMainWindow : public QMainWindow {
    Q_OBJECT
    
public:
    explicit ErpMainWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("ERP Pro — ระบบจัดการธุรกิจ");
        setMinimumSize(1280, 720);
        
        initRepositories();
        setupUi();
        restoreGeometry();
        
        // Show dashboard first
        m_navBar->findChild<QPushButton*>("", Qt::FindDirectChildrenOnly);
        showPage("dashboard");
    }
    
    ~ErpMainWindow() {
        QSettings s;
        s.setValue("mainwindow/geometry", saveGeometry());
        s.setValue("mainwindow/state", saveState());
    }
    
private slots:
    void showPage(const QString& pageId) {
        m_currentPage = pageId;
        
        if (!m_pages.contains(pageId)) {
            createPage(pageId);
        }
        
        m_stack->setCurrentWidget(m_pages[pageId]);
        
        // Update header title
        static const QMap<QString, QString> titles{
            {"dashboard", "แดชบอร์ด"},
            {"customers", "ลูกค้า"},
            {"products",  "สินค้า"},
            {"orders",    "คำสั่งซื้อ"},
            {"invoices",  "ใบแจ้งหนี้"},
            {"employees", "พนักงาน"},
            {"reports",   "รายงาน"},
            {"settings",  "ตั้งค่า"}
        };
        
        m_pageTitle->setText(titles.value(pageId, pageId));
    }
    
private:
    void initRepositories() {
        QString dbPath = QStandardPaths::writableLocation(
            QStandardPaths::AppDataLocation) + "/erp.db";
        QDir().mkpath(QFileInfo(dbPath).path());
        
        Database::instance().connect(dbPath);
        
        m_customers = new CustomerRepository(this);
        m_products  = new ProductRepository(this);
        m_orders    = new SalesOrderRepository(this);
    }
    
    void setupUi() {
        auto* central = new QWidget(this);
        setCentralWidget(central);
        
        auto* hlay = new QHBoxLayout(central);
        hlay->setContentsMargins(0, 0, 0, 0);
        hlay->setSpacing(0);
        
        // Side nav
        m_navBar = new SideNavBar(this);
        hlay->addWidget(m_navBar);
        
        // Main content area
        auto* contentArea = new QWidget(this);
        auto* vlay = new QVBoxLayout(contentArea);
        vlay->setContentsMargins(0, 0, 0, 0);
        vlay->setSpacing(0);
        
        // Top bar
        auto* topBar = new QWidget(contentArea);
        topBar->setFixedHeight(52);
        topBar->setStyleSheet(
            "background: white; border-bottom: 1px solid #E0E0E0;");
        
        auto* topLayout = new QHBoxLayout(topBar);
        topLayout->setContentsMargins(16, 0, 16, 0);
        
        m_pageTitle = new QLabel("แดชบอร์ด", topBar);
        m_pageTitle->setStyleSheet("font-size: 18px; font-weight: bold;");
        
        auto* spacer = new QWidget(topBar);
        spacer->setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Preferred);
        
        // Global search
        auto* globalSearch = new QLineEdit(topBar);
        globalSearch->setPlaceholderText("ค้นหา...");
        globalSearch->setClearButtonEnabled(true);
        globalSearch->setFixedWidth(200);
        globalSearch->setStyleSheet(
            "QLineEdit { border: 1px solid #ddd; border-radius: 16px; padding: 4px 12px; }");
        
        // Notification button
        auto* notifBtn = new QPushButton("🔔", topBar);
        notifBtn->setFlat(true);
        notifBtn->setStyleSheet("font-size: 18px; padding: 4px;");
        
        topLayout->addWidget(m_pageTitle);
        topLayout->addWidget(spacer);
        topLayout->addWidget(globalSearch);
        topLayout->addWidget(notifBtn);
        
        // Page stack
        m_stack = new QStackedWidget(contentArea);
        
        vlay->addWidget(topBar);
        vlay->addWidget(m_stack, 1);
        
        hlay->addWidget(contentArea, 1);
        
        // Connect nav
        connect(m_navBar, &SideNavBar::pageSelected, this, &ErpMainWindow::showPage);
    }
    
    void createPage(const QString& pageId) {
        QWidget* page = nullptr;
        
        if (pageId == "dashboard") {
            page = new ErpDashboard(m_customers, m_products, m_orders, this);
        } else if (pageId == "customers") {
            page = new CustomersPage(m_customers, this);
        } else if (pageId == "products") {
            page = createPlaceholderPage("สินค้า — กำลังพัฒนา");
        } else if (pageId == "orders") {
            page = createPlaceholderPage("คำสั่งซื้อ — กำลังพัฒนา");
        } else if (pageId == "employees") {
            page = createPlaceholderPage("พนักงาน — กำลังพัฒนา");
        } else {
            page = createPlaceholderPage(pageId + " — กำลังพัฒนา");
        }
        
        if (page) {
            m_stack->addWidget(page);
            m_pages[pageId] = page;
        }
    }
    
    QWidget* createPlaceholderPage(const QString& text) {
        auto* w = new QWidget(this);
        auto* lay = new QVBoxLayout(w);
        auto* label = new QLabel(text, w);
        label->setAlignment(Qt::AlignCenter);
        label->setStyleSheet("font-size: 24px; color: #999;");
        lay->addWidget(label);
        return w;
    }
    
    void restoreGeometry() {
        QSettings s;
        if (s.contains("mainwindow/geometry")) {
            QMainWindow::restoreGeometry(s.value("mainwindow/geometry").toByteArray());
            restoreState(s.value("mainwindow/state").toByteArray());
        } else {
            resize(1280, 720);
            showMaximized();
        }
    }
    
    // Repositories
    CustomerRepository* m_customers;
    ProductRepository* m_products;
    SalesOrderRepository* m_orders;
    
    // UI
    SideNavBar* m_navBar;
    QStackedWidget* m_stack;
    QLabel* m_pageTitle;
    QMap<QString, QWidget*> m_pages;
    QString m_currentPage;
};
```

---

## สรุป Part 055

ใน Part นี้คุณได้เรียนรู้:

1. ✅ SideNavBar ด้วย QPushButton (checkable + auto-exclusive) + logo + user footer
2. ✅ CustomersPage — toolbar, search, QTableView ด้วย proxy model, context menu, CSV export
3. ✅ ErpMainWindow — side nav + top bar + QStackedWidget + lazy page creation
4. ✅ Geometry persistence ด้วย QSettings
5. ✅ Page factory pattern สำหรับ lazy initialization

---

⬅️ [Part 054](part054.md) | ➡️ [Part 056: World-Class — Plugin Architecture](part056.md)
