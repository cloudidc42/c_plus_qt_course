# Part 024: Advanced Qt Patterns & Architecture

## ขั้นตอนที่ 326-340

---

## ขั้นตอนที่ 326: MVC Architecture ใน Qt

```cpp
// === Model ===
class UserModel : public QAbstractListModel {
    Q_OBJECT
    
public:
    enum Roles {
        IdRole = Qt::UserRole + 1,
        NameRole,
        EmailRole,
        RoleRole,
        ActiveRole
    };
    
    struct User {
        int id;
        QString name, email, role;
        bool active;
    };
    
private:
    QList<User> m_users;
    
public:
    explicit UserModel(QObject* parent = nullptr) : QAbstractListModel(parent) {}
    
    int rowCount(const QModelIndex& parent = {}) const override {
        return parent.isValid() ? 0 : m_users.size();
    }
    
    QVariant data(const QModelIndex& idx, int role = Qt::DisplayRole) const override {
        if (!idx.isValid() || idx.row() >= m_users.size()) return {};
        const User& u = m_users[idx.row()];
        
        switch (role) {
            case Qt::DisplayRole: return u.name;
            case IdRole:    return u.id;
            case NameRole:  return u.name;
            case EmailRole: return u.email;
            case RoleRole:  return u.role;
            case ActiveRole: return u.active;
        }
        return {};
    }
    
    bool setData(const QModelIndex& idx, const QVariant& value, int role) override {
        if (!idx.isValid()) return false;
        User& u = m_users[idx.row()];
        
        switch (role) {
            case NameRole:   u.name = value.toString(); break;
            case EmailRole:  u.email = value.toString(); break;
            case RoleRole:   u.role = value.toString(); break;
            case ActiveRole: u.active = value.toBool(); break;
            default: return false;
        }
        emit dataChanged(idx, idx, {role});
        return true;
    }
    
    Qt::ItemFlags flags(const QModelIndex& idx) const override {
        if (!idx.isValid()) return Qt::NoItemFlags;
        return Qt::ItemIsEnabled | Qt::ItemIsSelectable | Qt::ItemIsEditable;
    }
    
    QHash<int, QByteArray> roleNames() const override {
        return {
            {IdRole,    "id"},
            {NameRole,  "name"},
            {EmailRole, "email"},
            {RoleRole,  "userRole"},
            {ActiveRole,"active"}
        };
    }
    
    // CRUD
    void addUser(const User& u) {
        beginInsertRows({}, m_users.size(), m_users.size());
        m_users.append(u);
        endInsertRows();
    }
    
    void removeUser(int row) {
        if (row < 0 || row >= m_users.size()) return;
        beginRemoveRows({}, row, row);
        m_users.removeAt(row);
        endRemoveRows();
    }
    
    const User& user(int row) const { return m_users[row]; }
    QList<User>& users() { return m_users; }
    
    // Load from JSON
    void loadFromJson(const QJsonArray& arr) {
        beginResetModel();
        m_users.clear();
        for (const QJsonValue& v : arr) {
            QJsonObject obj = v.toObject();
            m_users.append({
                obj["id"].toInt(),
                obj["name"].toString(),
                obj["email"].toString(),
                obj["role"].toString("user"),
                obj["active"].toBool(true)
            });
        }
        endResetModel();
    }
    
    QJsonArray toJson() const {
        QJsonArray arr;
        for (const User& u : m_users) {
            arr.append(QJsonObject{
                {"id",     u.id},
                {"name",   u.name},
                {"email",  u.email},
                {"role",   u.role},
                {"active", u.active}
            });
        }
        return arr;
    }
};

// === Controller ===
class UserController : public QObject {
    Q_OBJECT
    
private:
    UserModel* m_model;
    QSortFilterProxyModel* m_proxy;
    int m_nextId = 1;
    
public:
    explicit UserController(QObject* parent = nullptr) : QObject(parent) {
        m_model = new UserModel(this);
        m_proxy = new QSortFilterProxyModel(this);
        m_proxy->setSourceModel(m_model);
        m_proxy->setFilterCaseSensitivity(Qt::CaseInsensitive);
        m_proxy->setFilterKeyColumn(-1);  // all columns
        
        loadSampleData();
    }
    
    UserModel* model() { return m_model; }
    QSortFilterProxyModel* proxyModel() { return m_proxy; }
    
    bool addUser(const QString& name, const QString& email, const QString& role) {
        if (name.isEmpty() || email.isEmpty()) {
            emit errorOccurred("Name and email are required");
            return false;
        }
        
        // Check duplicate email
        for (int i = 0; i < m_model->rowCount(); i++) {
            if (m_model->user(i).email == email) {
                emit errorOccurred("Email already exists");
                return false;
            }
        }
        
        m_model->addUser({m_nextId++, name, email, role, true});
        emit userAdded(name);
        return true;
    }
    
    void removeUser(int proxyRow) {
        QModelIndex srcIdx = m_proxy->mapToSource(m_proxy->index(proxyRow, 0));
        QString name = m_model->user(srcIdx.row()).name;
        m_model->removeUser(srcIdx.row());
        emit userRemoved(name);
    }
    
    void toggleActive(int proxyRow) {
        QModelIndex srcIdx = m_proxy->mapToSource(m_proxy->index(proxyRow, 0));
        bool current = m_model->user(srcIdx.row()).active;
        m_model->setData(srcIdx, !current, UserModel::ActiveRole);
    }
    
    void setFilter(const QString& text) {
        m_proxy->setFilterFixedString(text);
    }
    
    void sortBy(int column, Qt::SortOrder order = Qt::AscendingOrder) {
        m_proxy->sort(column, order);
    }
    
signals:
    void userAdded(const QString& name);
    void userRemoved(const QString& name);
    void errorOccurred(const QString& msg);
    
private:
    void loadSampleData() {
        QJsonArray data = QJsonDocument::fromJson(R"([
            {"id":1, "name":"สมชาย ใจดี",   "email":"somchai@example.com",  "role":"admin",   "active":true},
            {"id":2, "name":"สมหญิง รักดี", "email":"somying@example.com",  "role":"user",    "active":true},
            {"id":3, "name":"อนุชา เก่งกาจ","email":"anucha@example.com",   "role":"manager", "active":false},
            {"id":4, "name":"วิไล สวยงาม",  "email":"wilai@example.com",    "role":"user",    "active":true}
        ])").toJson(QJsonDocument::Compact)).array();
        m_model->loadFromJson(data);
        m_nextId = 5;
    }
};

// === View ===
class UserView : public QWidget {
    Q_OBJECT
    
public:
    UserView(UserController* ctrl, QWidget* parent = nullptr)
        : QWidget(parent), controller(ctrl) {
        
        setupUi();
        connectSignals();
    }
    
private:
    UserController* controller;
    QTableView* tableView;
    QLineEdit* searchEdit;
    QLabel* statusLabel;
    
    void setupUi() {
        setWindowTitle("User Management");
        setMinimumSize(700, 500);
        
        tableView = new QTableView();
        tableView->setModel(controller->proxyModel());
        tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
        tableView->setAlternatingRowColors(true);
        tableView->horizontalHeader()->setStretchLastSection(true);
        tableView->verticalHeader()->setVisible(false);
        
        searchEdit = new QLineEdit();
        searchEdit->setPlaceholderText("ค้นหา...");
        
        auto* addBtn = new QPushButton("เพิ่มผู้ใช้");
        auto* removeBtn = new QPushButton("ลบ");
        auto* toggleBtn = new QPushButton("เปิด/ปิดใช้งาน");
        
        statusLabel = new QLabel("Ready");
        statusLabel->setStyleSheet("color: #27ae60;");
        
        auto* layout = new QVBoxLayout(this);
        
        auto* toolBar = new QHBoxLayout();
        toolBar->addWidget(searchEdit);
        toolBar->addStretch();
        toolBar->addWidget(addBtn);
        toolBar->addWidget(removeBtn);
        toolBar->addWidget(toggleBtn);
        
        layout->addLayout(toolBar);
        layout->addWidget(tableView);
        layout->addWidget(statusLabel);
        
        connect(searchEdit, &QLineEdit::textChanged, 
                controller, &UserController::setFilter);
        
        connect(addBtn, &QPushButton::clicked, this, &UserView::addUser);
        
        connect(removeBtn, &QPushButton::clicked, [=]() {
            auto rows = tableView->selectionModel()->selectedRows();
            if (!rows.isEmpty()) {
                controller->removeUser(rows.first().row());
            }
        });
        
        connect(toggleBtn, &QPushButton::clicked, [=]() {
            auto rows = tableView->selectionModel()->selectedRows();
            if (!rows.isEmpty()) {
                controller->toggleActive(rows.first().row());
            }
        });
    }
    
    void connectSignals() {
        connect(controller, &UserController::userAdded, [this](const QString& name) {
            showStatus("เพิ่ม: " + name, true);
        });
        
        connect(controller, &UserController::userRemoved, [this](const QString& name) {
            showStatus("ลบ: " + name, true);
        });
        
        connect(controller, &UserController::errorOccurred, [this](const QString& msg) {
            showStatus("Error: " + msg, false);
            QMessageBox::warning(this, "Error", msg);
        });
    }
    
    void addUser() {
        QDialog dlg(this);
        dlg.setWindowTitle("Add User");
        auto* form = new QFormLayout(&dlg);
        
        auto* nameEdit = new QLineEdit();
        auto* emailEdit = new QLineEdit();
        auto* roleCombo = new QComboBox();
        roleCombo->addItems({"user", "admin", "manager"});
        
        form->addRow("ชื่อ:", nameEdit);
        form->addRow("อีเมล:", emailEdit);
        form->addRow("บทบาท:", roleCombo);
        
        auto* buttons = new QDialogButtonBox(QDialogButtonBox::Ok | QDialogButtonBox::Cancel);
        form->addRow(buttons);
        
        connect(buttons, &QDialogButtonBox::accepted, &dlg, &QDialog::accept);
        connect(buttons, &QDialogButtonBox::rejected, &dlg, &QDialog::reject);
        
        if (dlg.exec() == QDialog::Accepted) {
            controller->addUser(nameEdit->text(), emailEdit->text(), roleCombo->currentText());
        }
    }
    
    void showStatus(const QString& msg, bool ok) {
        statusLabel->setText(msg);
        statusLabel->setStyleSheet(ok ? "color: #27ae60;" : "color: #e74c3c;");
    }
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    UserController controller;
    UserView view(&controller);
    view.show();
    
    return app.exec();
}
```

---

## สรุป Part 024

ใน Part นี้คุณได้เรียนรู้:

1. ✅ MVC Architecture ใน Qt
2. ✅ Custom QAbstractListModel
3. ✅ Controller with business logic
4. ✅ View with QTableView
5. ✅ Complete User Management App

---

⬅️ [Part 023](part023.md) | ➡️ [Part 025: Qt Concurrency Advanced](part025.md)
