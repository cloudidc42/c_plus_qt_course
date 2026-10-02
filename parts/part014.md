# Part 014: Qt Model/View Architecture

## ขั้นตอนที่ 176-190

---

## ขั้นตอนที่ 176: Model/View Overview

```
Qt Model/View Architecture:

┌─────────────────────────────────────────────────┐
│                   View                          │
│  (QListView, QTableView, QTreeView, QComboBox)  │
│                     ↕ signals/slots             │
│                  Delegate                       │
│            (QAbstractItemDelegate)              │
│                     ↕                           │
│                   Model                         │
│    (QAbstractItemModel, QStandardItemModel,     │
│     QStringListModel, QFileSystemModel)         │
│                     ↕                           │
│                  Data Source                    │
│            (Database, File, Network)            │
└─────────────────────────────────────────────────┘

ข้อดี:
- แยก Logic (Model) ออกจาก Display (View)
- หนึ่ง Model สามารถแสดงผ่านหลาย View
- Sorting/Filtering ง่ายด้วย QSortFilterProxyModel
```

---

## ขั้นตอนที่ 177: QStringListModel

```cpp
#include <QApplication>
#include <QWidget>
#include <QHBoxLayout>
#include <QVBoxLayout>
#include <QListView>
#include <QStringListModel>
#include <QLineEdit>
#include <QPushButton>
#include <QLabel>
#include <QMessageBox>
#include <QSortFilterProxyModel>

class StringListDemo : public QWidget {
    Q_OBJECT
    
public:
    StringListDemo(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("QStringListModel Demo");
        setMinimumSize(500, 400);
        
        // Model
        model = new QStringListModel(this);
        model->setStringList({
            "Apple", "Banana", "Cherry", "Date", "Elderberry",
            "Fig", "Grape", "Honeydew", "Kiwi", "Lemon"
        });
        
        // Proxy for sorting and filtering
        proxy = new QSortFilterProxyModel(this);
        proxy->setSourceModel(model);
        proxy->setFilterCaseSensitivity(Qt::CaseInsensitive);
        proxy->sort(0, Qt::AscendingOrder);
        
        // View
        listView = new QListView();
        listView->setModel(proxy);
        
        // Controls
        auto* addEdit = new QLineEdit();
        addEdit->setPlaceholderText("เพิ่มรายการ...");
        
        auto* addBtn = new QPushButton("เพิ่ม");
        auto* removeBtn = new QPushButton("ลบที่เลือก");
        auto* clearBtn = new QPushButton("ล้างทั้งหมด");
        
        auto* filterEdit = new QLineEdit();
        filterEdit->setPlaceholderText("ค้นหา...");
        
        auto* countLabel = new QLabel("จำนวน: 10");
        
        // Layout
        auto* mainLayout = new QHBoxLayout(this);
        
        auto* rightPanel = new QVBoxLayout();
        rightPanel->addWidget(new QLabel("ค้นหา:"));
        rightPanel->addWidget(filterEdit);
        rightPanel->addWidget(listView);
        rightPanel->addWidget(countLabel);
        rightPanel->addWidget(addEdit);
        rightPanel->addWidget(addBtn);
        rightPanel->addWidget(removeBtn);
        rightPanel->addWidget(clearBtn);
        
        mainLayout->addLayout(rightPanel);
        
        // Connections
        connect(filterEdit, &QLineEdit::textChanged, proxy, 
                &QSortFilterProxyModel::setFilterFixedString);
        
        connect(addBtn, &QPushButton::clicked, [=]() {
            QString text = addEdit->text().trimmed();
            if (text.isEmpty()) return;
            
            int row = model->rowCount();
            model->insertRow(row);
            model->setData(model->index(row), text);
            addEdit->clear();
            
            updateCount(countLabel);
        });
        
        connect(addEdit, &QLineEdit::returnPressed, addBtn, &QPushButton::click);
        
        connect(removeBtn, &QPushButton::clicked, [=]() {
            QModelIndex idx = listView->currentIndex();
            if (idx.isValid()) {
                QModelIndex srcIdx = proxy->mapToSource(idx);
                model->removeRow(srcIdx.row());
                updateCount(countLabel);
            }
        });
        
        connect(clearBtn, &QPushButton::clicked, [=]() {
            if (QMessageBox::question(this, "ยืนยัน", "ล้างทั้งหมด?") == QMessageBox::Yes) {
                model->setStringList({});
                updateCount(countLabel);
            }
        });
        
        connect(model, &QAbstractItemModel::rowsInserted, [=]() {
            updateCount(countLabel);
        });
        
        connect(model, &QAbstractItemModel::rowsRemoved, [=]() {
            updateCount(countLabel);
        });
    }
    
private:
    QStringListModel* model;
    QSortFilterProxyModel* proxy;
    QListView* listView;
    
    void updateCount(QLabel* label) {
        label->setText(QString("จำนวน: %1 (แสดง: %2)")
            .arg(model->rowCount())
            .arg(proxy->rowCount()));
    }
};
```

---

## ขั้นตอนที่ 178: QStandardItemModel (Table)

```cpp
#include <QTableView>
#include <QStandardItemModel>
#include <QHeaderView>
#include <QMenu>

class TableDemo : public QWidget {
    Q_OBJECT
    
public:
    TableDemo(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("QTableView Demo");
        setMinimumSize(600, 400);
        
        // Model
        model = new QStandardItemModel(0, 5, this);
        model->setHorizontalHeaderLabels({
            "รหัส", "ชื่อ", "แผนก", "เงินเดือน", "วันเริ่มงาน"
        });
        
        // Add sample data
        addEmployee(1, "สมชาย ใจดี",      "IT",          60000, "2020-01-15");
        addEmployee(2, "สมหญิง สุขใจ",    "Marketing",   55000, "2019-06-20");
        addEmployee(3, "อนุชา รักษ์ดี",   "Finance",     70000, "2018-03-10");
        addEmployee(4, "วิไล รักงาน",     "HR",          50000, "2021-09-05");
        addEmployee(5, "ประยุทธ เก่งกาจ", "Engineering", 85000, "2017-12-01");
        
        // Proxy for sorting
        proxy = new QSortFilterProxyModel(this);
        proxy->setSourceModel(model);
        
        // View
        tableView = new QTableView();
        tableView->setModel(proxy);
        tableView->setSortingEnabled(true);
        tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
        tableView->setAlternatingRowColors(true);
        tableView->horizontalHeader()->setStretchLastSection(true);
        tableView->verticalHeader()->setVisible(false);
        tableView->setContextMenuPolicy(Qt::CustomContextMenu);
        
        // Style
        tableView->setStyleSheet(R"(
            QTableView {
                gridline-color: #ddd;
                background: white;
                alternate-background-color: #f8f9fa;
            }
            QTableView::item:selected {
                background: #0078d4;
                color: white;
            }
            QHeaderView::section {
                background: #2c3e50;
                color: white;
                padding: 6px;
                border: none;
                border-right: 1px solid #34495e;
            }
        )");
        
        // Context menu
        connect(tableView, &QTableView::customContextMenuRequested, this,
                &TableDemo::showContextMenu);
        
        // Summary
        summaryLabel = new QLabel(updateSummary());
        
        // Controls
        auto* addBtn = new QPushButton("เพิ่มพนักงาน");
        auto* deleteBtn = new QPushButton("ลบที่เลือก");
        auto* searchEdit = new QLineEdit();
        searchEdit->setPlaceholderText("ค้นหาชื่อ...");
        
        connect(addBtn, &QPushButton::clicked, this, &TableDemo::addNewEmployee);
        connect(deleteBtn, &QPushButton::clicked, this, &TableDemo::deleteSelected);
        connect(searchEdit, &QLineEdit::textChanged, [=](const QString& text) {
            proxy->setFilterKeyColumn(1);  // ค้นหาในคอลัมน์ชื่อ
            proxy->setFilterFixedString(text);
        });
        
        connect(model, &QAbstractItemModel::dataChanged, [=]() {
            summaryLabel->setText(updateSummary());
        });
        
        // Layout
        auto* layout = new QVBoxLayout(this);
        auto* btnLayout = new QHBoxLayout();
        btnLayout->addWidget(new QLabel("ค้นหา:"));
        btnLayout->addWidget(searchEdit);
        btnLayout->addStretch();
        btnLayout->addWidget(addBtn);
        btnLayout->addWidget(deleteBtn);
        
        layout->addLayout(btnLayout);
        layout->addWidget(tableView);
        layout->addWidget(summaryLabel);
    }
    
private:
    QStandardItemModel* model;
    QSortFilterProxyModel* proxy;
    QTableView* tableView;
    QLabel* summaryLabel;
    int nextId = 6;
    
    void addEmployee(int id, const QString& name, const QString& dept,
                     int salary, const QString& date) {
        QList<QStandardItem*> row;
        
        auto* idItem = new QStandardItem(QString::number(id));
        idItem->setTextAlignment(Qt::AlignCenter);
        idItem->setData(id, Qt::UserRole);  // สำหรับ sorting
        
        auto* nameItem = new QStandardItem(name);
        auto* deptItem = new QStandardItem(dept);
        
        auto* salaryItem = new QStandardItem(
            QString("%1").arg(salary, 0, 10, QChar(' ')));
        salaryItem->setTextAlignment(Qt::AlignRight | Qt::AlignVCenter);
        salaryItem->setData(salary, Qt::UserRole);
        
        // Color coding by salary
        if (salary >= 80000) {
            salaryItem->setForeground(QBrush(Qt::darkGreen));
            salaryItem->setFont(QFont("", -1, QFont::Bold));
        } else if (salary >= 60000) {
            salaryItem->setForeground(QBrush(Qt::darkBlue));
        }
        
        auto* dateItem = new QStandardItem(date);
        dateItem->setTextAlignment(Qt::AlignCenter);
        
        row << idItem << nameItem << deptItem << salaryItem << dateItem;
        model->appendRow(row);
    }
    
    void addNewEmployee() {
        addEmployee(nextId++, "ชื่อใหม่ " + QString::number(nextId-1),
                    "Department", 50000, "2024-01-01");
        
        // Scroll to new row
        tableView->scrollToBottom();
        summaryLabel->setText(updateSummary());
    }
    
    void deleteSelected() {
        QModelIndexList selected = tableView->selectionModel()->selectedRows();
        if (selected.isEmpty()) return;
        
        if (QMessageBox::question(this, "ยืนยัน", "ลบพนักงานที่เลือก?") != QMessageBox::Yes)
            return;
        
        // Delete from bottom to top (ป้องกัน index shift)
        QList<int> rows;
        for (const auto& idx : selected) {
            rows << proxy->mapToSource(idx).row();
        }
        std::sort(rows.rbegin(), rows.rend());
        for (int row : rows) {
            model->removeRow(row);
        }
        summaryLabel->setText(updateSummary());
    }
    
    void showContextMenu(const QPoint& pos) {
        QModelIndex idx = tableView->indexAt(pos);
        if (!idx.isValid()) return;
        
        QMenu menu;
        menu.addAction("แก้ไข", this, &TableDemo::editSelected);
        menu.addAction("ลบ", this, &TableDemo::deleteSelected);
        menu.addSeparator();
        menu.addAction("คัดลอกชื่อ", [=]() {
            QModelIndex nameIdx = proxy->index(idx.row(), 1);
            QString name = proxy->data(nameIdx).toString();
            QApplication::clipboard()->setText(name);
        });
        menu.exec(tableView->viewport()->mapToGlobal(pos));
    }
    
    void editSelected() {
        QModelIndex idx = tableView->currentIndex();
        if (idx.isValid()) {
            tableView->edit(idx);
        }
    }
    
    QString updateSummary() {
        double totalSalary = 0;
        for (int r = 0; r < model->rowCount(); r++) {
            totalSalary += model->item(r, 3)->data(Qt::UserRole).toDouble();
        }
        return QString("พนักงาน: %1 คน | เงินเดือนรวม: %2 บาท")
            .arg(model->rowCount())
            .arg(totalSalary, 0, 'f', 0);
    }
};
```

---

## ขั้นตอนที่ 179: Custom Model

```cpp
// Custom Model สำหรับ Contact List
struct Contact {
    QString name;
    QString phone;
    QString email;
    bool favorite;
};

class ContactModel : public QAbstractTableModel {
    Q_OBJECT
    
private:
    QList<Contact> contacts;
    
public:
    enum Columns { Name = 0, Phone, Email, Favorite, ColumnCount };
    
    explicit ContactModel(QObject* parent = nullptr) : QAbstractTableModel(parent) {}
    
    // === Required overrides ===
    int rowCount(const QModelIndex& parent = QModelIndex()) const override {
        if (parent.isValid()) return 0;
        return contacts.size();
    }
    
    int columnCount(const QModelIndex& parent = QModelIndex()) const override {
        if (parent.isValid()) return 0;
        return ColumnCount;
    }
    
    QVariant data(const QModelIndex& index, int role = Qt::DisplayRole) const override {
        if (!index.isValid() || index.row() >= contacts.size()) return {};
        
        const Contact& c = contacts[index.row()];
        
        switch (role) {
            case Qt::DisplayRole:
                switch (index.column()) {
                    case Name:     return c.name;
                    case Phone:    return c.phone;
                    case Email:    return c.email;
                    case Favorite: return c.favorite ? "★" : "☆";
                }
                break;
            
            case Qt::TextAlignmentRole:
                if (index.column() == Favorite)
                    return Qt::AlignCenter;
                break;
            
            case Qt::ForegroundRole:
                if (index.column() == Favorite && c.favorite)
                    return QBrush(QColor(255, 200, 0));
                break;
            
            case Qt::FontRole:
                if (index.column() == Favorite) {
                    QFont f; f.setPointSize(14); return f;
                }
                break;
            
            case Qt::UserRole:  // Raw data for sorting
                if (index.column() == Favorite) return c.favorite;
                break;
        }
        return {};
    }
    
    QVariant headerData(int section, Qt::Orientation orientation, 
                        int role = Qt::DisplayRole) const override {
        if (role != Qt::DisplayRole || orientation != Qt::Horizontal) return {};
        switch (section) {
            case Name:     return "ชื่อ";
            case Phone:    return "โทรศัพท์";
            case Email:    return "อีเมล";
            case Favorite: return "★";
        }
        return {};
    }
    
    bool setData(const QModelIndex& index, const QVariant& value, 
                 int role = Qt::EditRole) override {
        if (!index.isValid() || role != Qt::EditRole) return false;
        
        Contact& c = contacts[index.row()];
        switch (index.column()) {
            case Name:     c.name = value.toString(); break;
            case Phone:    c.phone = value.toString(); break;
            case Email:    c.email = value.toString(); break;
            case Favorite: c.favorite = value.toBool(); break;
            default: return false;
        }
        
        emit dataChanged(index, index, {role});
        return true;
    }
    
    Qt::ItemFlags flags(const QModelIndex& index) const override {
        if (!index.isValid()) return Qt::NoItemFlags;
        return Qt::ItemIsEnabled | Qt::ItemIsSelectable | Qt::ItemIsEditable;
    }
    
    // === Custom methods ===
    void addContact(const Contact& c) {
        beginInsertRows({}, contacts.size(), contacts.size());
        contacts.append(c);
        endInsertRows();
    }
    
    void removeContact(int row) {
        if (row < 0 || row >= contacts.size()) return;
        beginRemoveRows({}, row, row);
        contacts.removeAt(row);
        endRemoveRows();
    }
    
    void toggleFavorite(int row) {
        if (row < 0 || row >= contacts.size()) return;
        contacts[row].favorite = !contacts[row].favorite;
        auto idx = index(row, Favorite);
        emit dataChanged(idx, idx);
    }
    
    const Contact& getContact(int row) const {
        return contacts[row];
    }
};
```

---

## ขั้นตอนที่ 180-190: Custom Delegate

```cpp
// Custom Delegate สำหรับ rendering และ editing
class ContactDelegate : public QStyledItemDelegate {
    Q_OBJECT
    
public:
    ContactDelegate(QObject* parent = nullptr) : QStyledItemDelegate(parent) {}
    
    void paint(QPainter* painter, const QStyleOptionViewItem& option,
               const QModelIndex& index) const override {
        
        if (index.column() == ContactModel::Name) {
            QStyleOptionViewItem opt = option;
            initStyleOption(&opt, index);
            
            // Background
            if (opt.state & QStyle::State_Selected) {
                painter->fillRect(opt.rect, opt.palette.highlight());
            } else if (opt.state & QStyle::State_MouseOver) {
                painter->fillRect(opt.rect, QColor(240, 240, 255));
            }
            
            // Avatar circle
            QRect rect = opt.rect;
            int r = rect.height() - 8;
            QRect avatarRect(rect.left() + 4, rect.top() + 4, r, r);
            
            painter->save();
            painter->setRenderHint(QPainter::Antialiasing);
            
            QString name = index.data().toString();
            QChar letter = name.isEmpty() ? '?' : name[0].toUpper();
            
            // Generate color from name
            int h = name.length() > 0 ? qHash(name) % 360 : 0;
            painter->setBrush(QColor::fromHsv(h, 150, 200));
            painter->setPen(Qt::NoPen);
            painter->drawEllipse(avatarRect);
            
            painter->setPen(Qt::white);
            painter->setFont(QFont("Arial", r / 2, QFont::Bold));
            painter->drawText(avatarRect, Qt::AlignCenter, letter);
            
            painter->restore();
            
            // Name text
            QRect textRect(avatarRect.right() + 8, rect.top(), 
                          rect.right() - avatarRect.right() - 8, rect.height());
            painter->setPen(opt.state & QStyle::State_Selected 
                           ? opt.palette.highlightedText().color() 
                           : Qt::black);
            painter->setFont(QFont("Arial", 11, QFont::DemiBold));
            painter->drawText(textRect, Qt::AlignVCenter, name);
        } else {
            QStyledItemDelegate::paint(painter, option, index);
        }
    }
    
    QSize sizeHint(const QStyleOptionViewItem& option, 
                   const QModelIndex& index) const override {
        if (index.column() == ContactModel::Name) {
            return QSize(200, 40);
        }
        return QStyledItemDelegate::sizeHint(option, index);
    }
    
    QWidget* createEditor(QWidget* parent, const QStyleOptionViewItem&,
                          const QModelIndex& index) const override {
        if (index.column() == ContactModel::Favorite) {
            auto* cb = new QCheckBox(parent);
            return cb;
        }
        auto* edit = new QLineEdit(parent);
        
        if (index.column() == ContactModel::Phone) {
            edit->setValidator(new QRegularExpressionValidator(
                QRegularExpression("[0-9\\-\\+\\(\\) ]+"), edit));
        } else if (index.column() == ContactModel::Email) {
            // Simple email validation
        }
        
        return edit;
    }
    
    void setEditorData(QWidget* editor, const QModelIndex& index) const override {
        if (index.column() == ContactModel::Favorite) {
            auto* cb = qobject_cast<QCheckBox*>(editor);
            if (cb) cb->setChecked(index.data(Qt::UserRole).toBool());
        } else {
            auto* edit = qobject_cast<QLineEdit*>(editor);
            if (edit) edit->setText(index.data().toString());
        }
    }
    
    void setModelData(QWidget* editor, QAbstractItemModel* model,
                      const QModelIndex& index) const override {
        if (index.column() == ContactModel::Favorite) {
            auto* cb = qobject_cast<QCheckBox*>(editor);
            if (cb) model->setData(index, cb->isChecked());
        } else {
            auto* edit = qobject_cast<QLineEdit*>(editor);
            if (edit) model->setData(index, edit->text());
        }
    }
};
```

---

## สรุป Part 014

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Model/View Architecture
2. ✅ QStringListModel + QListView + QSortFilterProxyModel
3. ✅ QStandardItemModel + QTableView
4. ✅ Custom QAbstractTableModel
5. ✅ Custom QStyledItemDelegate

---

⬅️ [Part 013](part013.md) | ➡️ [Part 015: Qt Database](part015.md)
