# Part 042: Task App — Dialog, Service & Main Window

## ขั้นตอนที่ 596-610

---

## ขั้นตอนที่ 596: Task Dialog

```cpp
class TaskDialog : public QDialog {
    Q_OBJECT
    
public:
    enum Mode { Create, Edit };
    
    explicit TaskDialog(Mode mode, QWidget* parent = nullptr)
        : QDialog(parent), m_mode(mode) {
        
        setWindowTitle(mode == Create ? "New Task" : "Edit Task");
        setMinimumWidth(500);
        
        setupUi();
    }
    
    void setTask(const Task& task) {
        m_task = task;
        loadFromTask();
    }
    
    Task task() const {
        Task t = m_task;
        t.title       = titleEdit->text().trimmed();
        t.description = descEdit->toPlainText().trimmed();
        t.priority    = static_cast<Task::Priority>(priorityCombo->currentIndex());
        t.status      = static_cast<Task::Status>(statusCombo->currentIndex());
        t.assignee    = assigneeEdit->text().trimmed();
        t.dueDate     = dueDateEdit->date();
        
        // Tags
        t.tags.clear();
        for (const QString& tag : tagsEdit->text().split(',', Qt::SkipEmptyParts)) {
            t.tags << tag.trimmed();
        }
        
        return t;
    }
    
private:
    void setupUi() {
        auto* layout = new QVBoxLayout(this);
        auto* form = new QFormLayout();
        form->setLabelAlignment(Qt::AlignRight);
        
        titleEdit = new QLineEdit();
        titleEdit->setPlaceholderText("Task title...");
        
        descEdit = new QTextEdit();
        descEdit->setMaximumHeight(100);
        descEdit->setPlaceholderText("Description (optional)...");
        
        priorityCombo = new QComboBox();
        for (int i = 0; i <= static_cast<int>(Task::Priority::Critical); ++i) {
            priorityCombo->addItem(Task::priorityText(static_cast<Task::Priority>(i)));
        }
        priorityCombo->setCurrentIndex(1); // Medium default
        
        statusCombo = new QComboBox();
        for (int i = 0; i <= static_cast<int>(Task::Status::Done); ++i) {
            statusCombo->addItem(Task::statusText(static_cast<Task::Status>(i)));
        }
        
        assigneeEdit = new QLineEdit();
        assigneeEdit->setPlaceholderText("Assignee name...");
        
        dueDateEdit = new QDateEdit(QDate::currentDate().addDays(7));
        dueDateEdit->setCalendarPopup(true);
        dueDateEdit->setDisplayFormat("dd/MM/yyyy");
        
        auto* clearDateBtn = new QCheckBox("No deadline");
        
        tagsEdit = new QLineEdit();
        tagsEdit->setPlaceholderText("tag1, tag2, tag3...");
        
        form->addRow("Title*:", titleEdit);
        form->addRow("Description:", descEdit);
        form->addRow("Priority:", priorityCombo);
        form->addRow("Status:", statusCombo);
        form->addRow("Assignee:", assigneeEdit);
        form->addRow("Due Date:", dueDateEdit);
        form->addRow("", clearDateBtn);
        form->addRow("Tags:", tagsEdit);
        
        auto* btnBox = new QDialogButtonBox(
            QDialogButtonBox::Ok | QDialogButtonBox::Cancel);
        
        layout->addLayout(form);
        layout->addWidget(btnBox);
        
        connect(clearDateBtn, &QCheckBox::toggled, [this](bool noDue) {
            dueDateEdit->setEnabled(!noDue);
            if (noDue) dueDateEdit->setDate(QDate());
        });
        
        connect(btnBox->button(QDialogButtonBox::Ok), &QPushButton::clicked,
                this, &TaskDialog::validate);
        connect(btnBox->button(QDialogButtonBox::Cancel), &QPushButton::clicked,
                this, &QDialog::reject);
        
        titleEdit->setFocus();
    }
    
    void loadFromTask() {
        titleEdit->setText(m_task.title);
        descEdit->setPlainText(m_task.description);
        priorityCombo->setCurrentIndex(static_cast<int>(m_task.priority));
        statusCombo->setCurrentIndex(static_cast<int>(m_task.status));
        assigneeEdit->setText(m_task.assignee);
        
        if (m_task.dueDate.isValid()) {
            dueDateEdit->setDate(m_task.dueDate);
        }
        
        tagsEdit->setText(m_task.tags.join(", "));
    }
    
    void validate() {
        if (titleEdit->text().trimmed().isEmpty()) {
            QMessageBox::warning(this, "Validation", "Title is required.");
            titleEdit->setFocus();
            return;
        }
        accept();
    }
    
    Mode m_mode;
    Task m_task;
    
    QLineEdit* titleEdit;
    QTextEdit* descEdit;
    QComboBox* priorityCombo;
    QComboBox* statusCombo;
    QLineEdit* assigneeEdit;
    QDateEdit* dueDateEdit;
    QLineEdit* tagsEdit;
};
```

---

## ขั้นตอนที่ 597: Task Service

```cpp
class TaskService : public QObject {
    Q_OBJECT
    
public:
    explicit TaskService(QObject* parent = nullptr) : QObject(parent) {
        initDatabase();
        loadAll();
    }
    
    QList<Task> tasks() const { return m_tasks; }
    
    QList<Task> filtered(const QString& query = "",
                          std::optional<Task::Status> status = std::nullopt,
                          std::optional<Task::Priority> priority = std::nullopt) const {
        QList<Task> result;
        for (const Task& t : m_tasks) {
            if (t.archived) continue;
            
            if (!query.isEmpty()) {
                bool match = t.title.contains(query, Qt::CaseInsensitive)
                          || t.description.contains(query, Qt::CaseInsensitive)
                          || t.assignee.contains(query, Qt::CaseInsensitive)
                          || t.tags.join(" ").contains(query, Qt::CaseInsensitive);
                if (!match) continue;
            }
            
            if (status && t.status != *status) continue;
            if (priority && t.priority != *priority) continue;
            
            result << t;
        }
        return result;
    }
    
    std::optional<Task> findById(int id) const {
        for (const Task& t : m_tasks) {
            if (t.id == id) return t;
        }
        return std::nullopt;
    }
    
    int create(const Task& task) {
        Task newTask = task;
        newTask.id = ++m_nextId;
        newTask.createdAt = QDateTime::currentDateTime();
        newTask.updatedAt = QDateTime::currentDateTime();
        
        m_tasks.append(newTask);
        saveToFile();
        
        emit taskCreated(newTask);
        return newTask.id;
    }
    
    bool update(const Task& task) {
        for (auto& t : m_tasks) {
            if (t.id == task.id) {
                Task updated = task;
                updated.updatedAt = QDateTime::currentDateTime();
                t = updated;
                saveToFile();
                emit taskUpdated(updated);
                return true;
            }
        }
        return false;
    }
    
    bool updateStatus(int id, Task::Status newStatus) {
        auto task = findById(id);
        if (!task) return false;
        task->status = newStatus;
        return update(*task);
    }
    
    bool archive(int id) {
        for (auto& t : m_tasks) {
            if (t.id == id) {
                t.archived = true;
                saveToFile();
                emit taskArchived(id);
                return true;
            }
        }
        return false;
    }
    
    bool remove(int id) {
        for (int i = 0; i < m_tasks.size(); ++i) {
            if (m_tasks[i].id == id) {
                m_tasks.removeAt(i);
                saveToFile();
                emit taskDeleted(id);
                return true;
            }
        }
        return false;
    }
    
    // Statistics
    QMap<Task::Status, int> countByStatus() const {
        QMap<Task::Status, int> counts;
        for (const Task& t : m_tasks) {
            if (!t.archived) counts[t.status]++;
        }
        return counts;
    }
    
    QList<Task> overdueTasks() const {
        QList<Task> result;
        for (const Task& t : m_tasks) {
            if (t.isOverdue()) result << t;
        }
        return result;
    }
    
signals:
    void taskCreated(const Task& task);
    void taskUpdated(const Task& task);
    void taskDeleted(int id);
    void taskArchived(int id);
    
private:
    QString dataPath() const {
        return QDir::appDataLocation() + "/tasks.json";
    }
    
    void initDatabase() {
        QDir().mkpath(QDir::appDataLocation());
    }
    
    void loadAll() {
        QFile file(dataPath());
        if (!file.exists() || !file.open(QIODevice::ReadOnly)) {
            seedSampleData();
            return;
        }
        
        auto doc = QJsonDocument::fromJson(file.readAll());
        m_nextId = doc.object()["next_id"].toInt(1);
        
        for (const auto& val : doc.object()["tasks"].toArray()) {
            m_tasks << Task::fromJson(val.toObject());
        }
    }
    
    void saveToFile() {
        QJsonArray arr;
        for (const Task& t : m_tasks) arr.append(t.toJson());
        
        QJsonObject root;
        root["next_id"] = m_nextId;
        root["tasks"] = arr;
        
        QFile file(dataPath());
        file.open(QIODevice::WriteOnly);
        file.write(QJsonDocument(root).toJson());
    }
    
    void seedSampleData() {
        QList<Task> samples = {
            {0, "Setup project structure", "Initialize CMake and Qt project", Task::Priority::High, Task::Status::Done},
            {0, "Design database schema", "Create tables for users, tasks, projects", Task::Priority::Medium, Task::Status::InProgress},
            {0, "Implement authentication", "", Task::Priority::Critical, Task::Status::Todo},
            {0, "Write unit tests", "Cover service layer with tests", Task::Priority::Medium, Task::Status::Todo},
            {0, "Deploy to production", "", Task::Priority::High, Task::Status::Todo},
        };
        
        for (auto& s : samples) create(s);
    }
    
    QList<Task> m_tasks;
    int m_nextId = 1;
};
```

---

## ขั้นตอนที่ 598: Main Window

```cpp
class TaskManagerWindow : public QMainWindow {
    Q_OBJECT
    
public:
    TaskManagerWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Task Manager");
        setMinimumSize(1200, 700);
        
        service = new TaskService(this);
        
        setupUi();
        setupMenus();
        refreshBoard();
        
        connect(service, &TaskService::taskCreated, this, [this](const Task&) { refreshBoard(); });
        connect(service, &TaskService::taskUpdated, this, [this](const Task&) { refreshBoard(); });
        connect(service, &TaskService::taskDeleted, this, [this](int)         { refreshBoard(); });
    }
    
private:
    void setupUi() {
        // Sidebar
        auto* sidebar = new QWidget();
        sidebar->setFixedWidth(220);
        sidebar->setStyleSheet("background: #2c3e50;");
        
        auto* sLayout = new QVBoxLayout(sidebar);
        
        auto* appTitle = new QLabel("📋 Task Manager");
        appTitle->setStyleSheet("color: white; font-size: 18px; font-weight: bold; padding: 16px;");
        sLayout->addWidget(appTitle);
        
        auto* navList = new QListWidget();
        navList->setStyleSheet(
            "QListWidget { background: transparent; border: none; }"
            "QListWidget::item { color: white; padding: 10px 16px; }"
            "QListWidget::item:selected { background: rgba(255,255,255,0.15); border-radius: 4px; }"
            "QListWidget::item:hover { background: rgba(255,255,255,0.08); }");
        
        navList->addItem("All Tasks");
        navList->addItem("In Progress");
        navList->addItem("Review");
        navList->addItem("Done");
        navList->addItem("Overdue");
        navList->setCurrentRow(0);
        sLayout->addWidget(navList, 1);
        
        // Stats
        statsWidget = new QWidget();
        statsWidget->setStyleSheet("color: white; padding: 8px;");
        sLayout->addWidget(statsWidget);
        
        // Content area
        auto* content = new QWidget();
        auto* cLayout = new QVBoxLayout(content);
        cLayout->setContentsMargins(0, 0, 0, 0);
        
        // Top bar
        auto* topBar = new QHBoxLayout();
        topBar->setContentsMargins(12, 8, 12, 8);
        
        searchEdit = new QLineEdit();
        searchEdit->setPlaceholderText("Search tasks...");
        searchEdit->setStyleSheet("padding: 6px; border-radius: 4px;");
        
        auto* addBtn = new QPushButton("+ New Task");
        addBtn->setStyleSheet("background: #27ae60; color: white; padding: 8px 20px; border-radius: 4px; font-weight: bold;");
        
        topBar->addWidget(searchEdit, 1);
        topBar->addWidget(addBtn);
        
        board = new TaskBoard(this);
        
        cLayout->addLayout(topBar);
        cLayout->addWidget(board, 1);
        
        // Main layout
        auto* mainLayout = new QHBoxLayout();
        mainLayout->setSpacing(0);
        mainLayout->addWidget(sidebar);
        mainLayout->addWidget(content, 1);
        
        auto* central = new QWidget();
        central->setLayout(mainLayout);
        setCentralWidget(central);
        
        // Connect
        connect(addBtn, &QPushButton::clicked, [this]() {
            showTaskDialog(Task::Status::Todo);
        });
        
        connect(searchEdit, &QLineEdit::textChanged, this, &TaskManagerWindow::onSearch);
        
        connect(board, &TaskBoard::editTaskRequested,  this, &TaskManagerWindow::editTask);
        connect(board, &TaskBoard::taskDeleted,        this, &TaskManagerWindow::deleteTask);
        connect(board, &TaskBoard::taskStatusChanged,  this, &TaskManagerWindow::changeStatus);
        connect(board, &TaskBoard::addTaskRequested,   this, &TaskManagerWindow::showTaskDialog);
        
        connect(navList, &QListWidget::currentRowChanged, [this](int row) {
            filterTasks(row);
        });
    }
    
    void setupMenus() {
        auto* fileMenu = menuBar()->addMenu("File");
        fileMenu->addAction("New Task", QKeySequence::New, [this]() {
            showTaskDialog(Task::Status::Todo);
        });
        fileMenu->addSeparator();
        fileMenu->addAction("Quit", QKeySequence::Quit, this, &QWidget::close);
    }
    
    void refreshBoard() {
        QString filter = searchEdit ? searchEdit->text() : "";
        board->setTasks(service->filtered(filter));
        updateStats();
    }
    
    void updateStats() {
        auto counts = service->countByStatus();
        int total = 0;
        for (auto c : counts) total += c;
        
        auto* layout = statsWidget->layout();
        if (!layout) {
            layout = new QVBoxLayout(statsWidget);
        }
        
        QLayoutItem* item;
        while ((item = layout->takeAt(0)) != nullptr) {
            delete item->widget();
            delete item;
        }
        
        auto* totalLabel = new QLabel(QString("Total: %1 tasks").arg(total));
        totalLabel->setStyleSheet("color: #bdc3c7; font-size: 12px; padding: 4px 16px;");
        layout->addWidget(totalLabel);
        
        int overdue = service->overdueTasks().size();
        if (overdue > 0) {
            auto* overdueLabel = new QLabel(QString("⚠ %1 overdue").arg(overdue));
            overdueLabel->setStyleSheet("color: #e74c3c; font-size: 12px; padding: 4px 16px;");
            layout->addWidget(overdueLabel);
        }
    }
    
    void showTaskDialog(Task::Status defaultStatus = Task::Status::Todo) {
        auto* dlg = new TaskDialog(TaskDialog::Create, this);
        Task t;
        t.status = defaultStatus;
        dlg->setTask(t);
        
        if (dlg->exec() == QDialog::Accepted) {
            service->create(dlg->task());
        }
    }
    
    void editTask(int id) {
        auto task = service->findById(id);
        if (!task) return;
        
        auto* dlg = new TaskDialog(TaskDialog::Edit, this);
        dlg->setTask(*task);
        
        if (dlg->exec() == QDialog::Accepted) {
            service->update(dlg->task());
        }
    }
    
    void deleteTask(int id) {
        service->remove(id);
    }
    
    void changeStatus(int id, Task::Status newStatus) {
        service->updateStatus(id, newStatus);
    }
    
    void onSearch(const QString& text) {
        board->setTasks(service->filtered(text));
    }
    
    void filterTasks(int navIndex) {
        static const QMap<int, std::optional<Task::Status>> filters = {
            {0, std::nullopt},
            {1, Task::Status::InProgress},
            {2, Task::Status::Review},
            {3, Task::Status::Done},
        };
        
        if (navIndex == 4) { // Overdue
            board->setTasks(service->overdueTasks());
        } else {
            board->setTasks(service->filtered("", filters.value(navIndex, std::nullopt)));
        }
    }
    
    TaskService* service;
    TaskBoard* board;
    QLineEdit* searchEdit;
    QWidget* statsWidget;
};
```

---

## สรุป Part 042

ใน Part นี้คุณได้เรียนรู้:

1. ✅ TaskDialog สำหรับ create/edit ด้วย validation
2. ✅ TaskService ด้วย JSON persistence + filtering + stats
3. ✅ TaskManagerWindow ครบด้วย Sidebar, Search, Stats
4. ✅ Signal routing ระหว่าง Board → Service → Refresh

---

⬅️ [Part 041](part041.md) | ➡️ [Part 043: Real-World Project — Chat Application](part043.md)
