# Part 041: Real-World Project — Task Management App

## ขั้นตอนที่ 581-595

---

## ขั้นตอนที่ 581: Project Structure

```
TaskManager/
├── CMakeLists.txt
├── main.cpp
├── app/
│   ├── Application.h / .cpp       # App singleton
├── models/
│   ├── Task.h / .cpp
│   ├── TaskListModel.h / .cpp     # QAbstractListModel
│   ├── Project.h / .cpp
├── services/
│   ├── TaskService.h / .cpp       # Business logic
│   ├── DatabaseService.h / .cpp
│   ├── NotificationService.h / .cpp
├── ui/
│   ├── MainWindow.h / .cpp
│   ├── TaskBoard.h / .cpp         # Kanban view
│   ├── TaskCard.h / .cpp          # Drag-drop card
│   ├── TaskDialog.h / .cpp
│   ├── ProjectSidebar.h / .cpp
└── resources.qrc
```

---

## ขั้นตอนที่ 582: Task Model

```cpp
#include <QDateTime>
#include <QColor>

// === Task data class ===
class Task {
public:
    enum class Priority { Low, Medium, High, Critical };
    enum class Status   { Todo, InProgress, Review, Done };
    
    int id = 0;
    QString title;
    QString description;
    Priority priority = Priority::Medium;
    Status status = Status::Todo;
    QDate dueDate;
    int projectId = 0;
    QString assignee;
    QStringList tags;
    QDateTime createdAt;
    QDateTime updatedAt;
    bool archived = false;
    
    static QString priorityText(Priority p) {
        switch (p) {
        case Priority::Low:      return "ต่ำ";
        case Priority::Medium:   return "ปานกลาง";
        case Priority::High:     return "สูง";
        case Priority::Critical: return "วิกฤต";
        }
        return "";
    }
    
    static QColor priorityColor(Priority p) {
        switch (p) {
        case Priority::Low:      return QColor("#27ae60");
        case Priority::Medium:   return QColor("#3498db");
        case Priority::High:     return QColor("#e67e22");
        case Priority::Critical: return QColor("#e74c3c");
        }
        return Qt::gray;
    }
    
    static QString statusText(Status s) {
        switch (s) {
        case Status::Todo:       return "รอดำเนินการ";
        case Status::InProgress: return "กำลังทำ";
        case Status::Review:     return "รอตรวจสอบ";
        case Status::Done:       return "เสร็จสิ้น";
        }
        return "";
    }
    
    bool isOverdue() const {
        return dueDate.isValid() && dueDate < QDate::currentDate()
               && status != Status::Done;
    }
    
    QJsonObject toJson() const {
        QJsonObject obj;
        obj["id"]          = id;
        obj["title"]       = title;
        obj["description"] = description;
        obj["priority"]    = static_cast<int>(priority);
        obj["status"]      = static_cast<int>(status);
        obj["due_date"]    = dueDate.isValid() ? dueDate.toString(Qt::ISODate) : "";
        obj["project_id"]  = projectId;
        obj["assignee"]    = assignee;
        obj["tags"]        = QJsonArray::fromStringList(tags);
        obj["created_at"]  = createdAt.toISOString();
        obj["archived"]    = archived;
        return obj;
    }
    
    static Task fromJson(const QJsonObject& obj) {
        Task t;
        t.id          = obj["id"].toInt();
        t.title       = obj["title"].toString();
        t.description = obj["description"].toString();
        t.priority    = static_cast<Priority>(obj["priority"].toInt());
        t.status      = static_cast<Status>(obj["status"].toInt());
        t.dueDate     = QDate::fromString(obj["due_date"].toString(), Qt::ISODate);
        t.projectId   = obj["project_id"].toInt();
        t.assignee    = obj["assignee"].toString();
        t.archived    = obj["archived"].toBool();
        
        for (const auto& tag : obj["tags"].toArray()) {
            t.tags << tag.toString();
        }
        return t;
    }
};
Q_DECLARE_METATYPE(Task)
```

---

## ขั้นตอนที่ 583: Task List Model

```cpp
class TaskListModel : public QAbstractListModel {
    Q_OBJECT
    
public:
    enum Roles {
        IdRole = Qt::UserRole + 1,
        TitleRole, DescriptionRole,
        PriorityRole, PriorityTextRole, PriorityColorRole,
        StatusRole, StatusTextRole,
        DueDateRole, OverdueRole,
        AssigneeRole, TagsRole,
        TaskRole,   // full Task object
    };
    
    explicit TaskListModel(QObject* parent = nullptr) : QAbstractListModel(parent) {}
    
    int rowCount(const QModelIndex&) const override { return m_tasks.size(); }
    
    QVariant data(const QModelIndex& index, int role) const override {
        if (!index.isValid() || index.row() >= m_tasks.size()) return {};
        const Task& task = m_tasks[index.row()];
        
        switch (role) {
        case IdRole:           return task.id;
        case TitleRole:        return task.title;
        case DescriptionRole:  return task.description;
        case PriorityRole:     return static_cast<int>(task.priority);
        case PriorityTextRole: return Task::priorityText(task.priority);
        case PriorityColorRole:return Task::priorityColor(task.priority);
        case StatusRole:       return static_cast<int>(task.status);
        case StatusTextRole:   return Task::statusText(task.status);
        case DueDateRole:      return task.dueDate;
        case OverdueRole:      return task.isOverdue();
        case AssigneeRole:     return task.assignee;
        case TagsRole:         return task.tags;
        case TaskRole:         return QVariant::fromValue(task);
        default: return {};
        }
    }
    
    bool setData(const QModelIndex& index, const QVariant& value, int role) override {
        if (!index.isValid() || index.row() >= m_tasks.size()) return false;
        Task& task = m_tasks[index.row()];
        
        switch (role) {
        case TitleRole:       task.title = value.toString(); break;
        case StatusRole:      task.status = static_cast<Task::Status>(value.toInt()); break;
        case PriorityRole:    task.priority = static_cast<Task::Priority>(value.toInt()); break;
        case AssigneeRole:    task.assignee = value.toString(); break;
        default: return false;
        }
        
        task.updatedAt = QDateTime::currentDateTime();
        emit dataChanged(index, index, {role});
        return true;
    }
    
    QHash<int, QByteArray> roleNames() const override {
        return {
            {IdRole,           "taskId"},
            {TitleRole,        "title"},
            {DescriptionRole,  "description"},
            {PriorityRole,     "priority"},
            {PriorityTextRole, "priorityText"},
            {PriorityColorRole,"priorityColor"},
            {StatusRole,       "status"},
            {StatusTextRole,   "statusText"},
            {DueDateRole,      "dueDate"},
            {OverdueRole,      "overdue"},
            {AssigneeRole,     "assignee"},
            {TagsRole,         "tags"},
        };
    }
    
    Qt::ItemFlags flags(const QModelIndex& index) const override {
        if (!index.isValid()) return Qt::NoItemFlags;
        return Qt::ItemIsEnabled | Qt::ItemIsSelectable | Qt::ItemIsEditable
               | Qt::ItemIsDragEnabled | Qt::ItemIsDropEnabled;
    }
    
    Qt::DropActions supportedDropActions() const override {
        return Qt::MoveAction | Qt::CopyAction;
    }
    
    // Data manipulation
    void setTasks(const QList<Task>& tasks) {
        beginResetModel();
        m_tasks = tasks;
        endResetModel();
    }
    
    void addTask(const Task& task) {
        beginInsertRows({}, m_tasks.size(), m_tasks.size());
        m_tasks.append(task);
        endInsertRows();
    }
    
    void updateTask(const Task& task) {
        for (int i = 0; i < m_tasks.size(); ++i) {
            if (m_tasks[i].id == task.id) {
                m_tasks[i] = task;
                emit dataChanged(index(i), index(i));
                return;
            }
        }
    }
    
    void removeTask(int id) {
        for (int i = 0; i < m_tasks.size(); ++i) {
            if (m_tasks[i].id == id) {
                beginRemoveRows({}, i, i);
                m_tasks.removeAt(i);
                endRemoveRows();
                return;
            }
        }
    }
    
    QList<Task> tasks() const { return m_tasks; }
    
    Task* taskById(int id) {
        for (auto& t : m_tasks) {
            if (t.id == id) return &t;
        }
        return nullptr;
    }
    
private:
    QList<Task> m_tasks;
};
```

---

## ขั้นตอนที่ 584: Kanban Board Widget

```cpp
class TaskCard : public QFrame {
    Q_OBJECT
    
public:
    explicit TaskCard(const Task& task, QWidget* parent = nullptr)
        : QFrame(parent), m_task(task) {
        
        setFrameStyle(QFrame::Box | QFrame::Raised);
        setLineWidth(1);
        setCursor(Qt::OpenHandCursor);
        setAcceptDrops(false);
        
        auto* layout = new QVBoxLayout(this);
        layout->setContentsMargins(10, 8, 10, 8);
        layout->setSpacing(4);
        
        // Priority indicator + title
        auto* topRow = new QHBoxLayout();
        
        priorityBar = new QLabel("█");
        priorityBar->setStyleSheet(QString("color: %1; font-size: 16px;")
            .arg(Task::priorityColor(task.priority).name()));
        
        titleLabel = new QLabel(task.title);
        titleLabel->setWordWrap(true);
        titleLabel->setStyleSheet("font-weight: bold; font-size: 13px;");
        
        topRow->addWidget(priorityBar);
        topRow->addWidget(titleLabel, 1);
        layout->addLayout(topRow);
        
        // Description
        if (!task.description.isEmpty()) {
            auto* desc = new QLabel(task.description);
            desc->setWordWrap(true);
            desc->setStyleSheet("color: #666; font-size: 11px;");
            desc->setMaximumHeight(40);
            layout->addWidget(desc);
        }
        
        // Tags
        if (!task.tags.isEmpty()) {
            auto* tagRow = new QHBoxLayout();
            tagRow->setSpacing(4);
            
            for (const QString& tag : task.tags) {
                auto* tagLabel = new QLabel(tag);
                tagLabel->setStyleSheet(
                    "background: #e8f4fd; color: #2980b9; "
                    "padding: 1px 6px; border-radius: 8px; font-size: 11px;");
                tagRow->addWidget(tagLabel);
            }
            tagRow->addStretch();
            layout->addLayout(tagRow);
        }
        
        // Footer: due date + assignee
        auto* footer = new QHBoxLayout();
        
        if (task.dueDate.isValid()) {
            QString dueTxt = task.dueDate.toString("dd MMM");
            auto* dueLabel = new QLabel(dueTxt);
            
            if (task.isOverdue()) {
                dueLabel->setStyleSheet("color: #e74c3c; font-size: 11px; font-weight: bold;");
                dueLabel->setText("⚠ " + dueTxt);
            } else {
                dueLabel->setStyleSheet("color: #999; font-size: 11px;");
            }
            footer->addWidget(dueLabel);
        }
        
        footer->addStretch();
        
        if (!task.assignee.isEmpty()) {
            auto* assigneeLabel = new QLabel(task.assignee.left(1).toUpper());
            assigneeLabel->setFixedSize(22, 22);
            assigneeLabel->setAlignment(Qt::AlignCenter);
            assigneeLabel->setStyleSheet(
                "background: #3498db; color: white; "
                "border-radius: 11px; font-size: 11px; font-weight: bold;");
            assigneeLabel->setToolTip(task.assignee);
            footer->addWidget(assigneeLabel);
        }
        
        layout->addLayout(footer);
        
        updateStyle();
    }
    
    const Task& task() const { return m_task; }
    
signals:
    void editRequested(int taskId);
    void deleteRequested(int taskId);
    void statusChanged(int taskId, Task::Status newStatus);
    
protected:
    void mousePressEvent(QMouseEvent* event) override {
        if (event->button() == Qt::LeftButton) {
            m_dragStart = event->pos();
        }
        QFrame::mousePressEvent(event);
    }
    
    void mouseMoveEvent(QMouseEvent* event) override {
        if (event->buttons() & Qt::LeftButton) {
            if ((event->pos() - m_dragStart).manhattanLength() >= QApplication::startDragDistance()) {
                startDrag();
            }
        }
    }
    
    void mouseDoubleClickEvent(QMouseEvent*) override {
        emit editRequested(m_task.id);
    }
    
    void contextMenuEvent(QContextMenuEvent* event) override {
        QMenu menu(this);
        menu.addAction("Edit", [this]() { emit editRequested(m_task.id); });
        menu.addSeparator();
        
        auto* statusMenu = menu.addMenu("Move to");
        for (int i = 0; i <= static_cast<int>(Task::Status::Done); ++i) {
            auto* act = statusMenu->addAction(Task::statusText(static_cast<Task::Status>(i)));
            auto status = static_cast<Task::Status>(i);
            connect(act, &QAction::triggered, [this, status]() {
                emit statusChanged(m_task.id, status);
            });
        }
        
        menu.addSeparator();
        menu.addAction("Delete", [this]() { emit deleteRequested(m_task.id); });
        menu.exec(event->globalPos());
    }
    
private:
    void updateStyle() {
        QString bg = m_task.isOverdue() ? "#fff5f5" : "white";
        setStyleSheet(QString(
            "TaskCard { background: %1; border-radius: 6px; "
            "border: 1px solid #e0e0e0; } "
            "TaskCard:hover { border-color: #3498db; }").arg(bg));
    }
    
    void startDrag() {
        auto* drag = new QDrag(this);
        auto* mimeData = new QMimeData();
        mimeData->setData("application/x-task-id",
            QByteArray::number(m_task.id));
        drag->setMimeData(mimeData);
        
        QPixmap pixmap(size());
        render(&pixmap);
        drag->setPixmap(pixmap.scaled(200, 80, Qt::KeepAspectRatio));
        drag->setHotSpot(m_dragStart);
        
        drag->exec(Qt::MoveAction);
    }
    
    Task m_task;
    QLabel* titleLabel;
    QLabel* priorityBar;
    QPoint m_dragStart;
};
```

---

## ขั้นตอนที่ 585: Kanban Column

```cpp
class KanbanColumn : public QWidget {
    Q_OBJECT
    
public:
    KanbanColumn(Task::Status status, const QString& title, QWidget* parent = nullptr)
        : QWidget(parent), m_status(status) {
        
        setAcceptDrops(true);
        setMinimumWidth(260);
        
        auto* layout = new QVBoxLayout(this);
        layout->setSpacing(0);
        layout->setContentsMargins(0, 0, 0, 0);
        
        // Header
        auto* header = new QWidget();
        header->setStyleSheet(headerStyle(status));
        auto* hLayout = new QHBoxLayout(header);
        
        titleLabel = new QLabel(title);
        titleLabel->setStyleSheet("color: white; font-weight: bold; font-size: 14px;");
        
        countLabel = new QLabel("0");
        countLabel->setStyleSheet(
            "background: rgba(255,255,255,0.3); color: white; "
            "padding: 1px 8px; border-radius: 10px; font-size: 12px;");
        
        hLayout->addWidget(titleLabel);
        hLayout->addStretch();
        hLayout->addWidget(countLabel);
        
        // Card area
        cardArea = new QWidget();
        auto* cardLayout = new QVBoxLayout(cardArea);
        cardLayout->setSpacing(8);
        cardLayout->setAlignment(Qt::AlignTop);
        
        auto* scroll = new QScrollArea();
        scroll->setWidget(cardArea);
        scroll->setWidgetResizable(true);
        scroll->setFrameShape(QFrame::NoFrame);
        scroll->setHorizontalScrollBarPolicy(Qt::ScrollBarAlwaysOff);
        
        layout->addWidget(header);
        layout->addWidget(scroll, 1);
        
        // Add button
        auto* addBtn = new QPushButton("+ Add Task");
        addBtn->setStyleSheet(
            "background: transparent; border: 2px dashed #ccc; "
            "padding: 8px; color: #999; border-radius: 4px;");
        
        layout->addWidget(addBtn);
        
        connect(addBtn, &QPushButton::clicked, [this]() {
            emit addTaskRequested(m_status);
        });
    }
    
    void addCard(const Task& task) {
        auto* card = new TaskCard(task, cardArea);
        
        connect(card, &TaskCard::editRequested, this, &KanbanColumn::editTaskRequested);
        connect(card, &TaskCard::deleteRequested, this, &KanbanColumn::deleteTaskRequested);
        connect(card, &TaskCard::statusChanged, this, &KanbanColumn::taskStatusChanged);
        
        cardArea->layout()->addWidget(card);
        updateCount();
    }
    
    void clearCards() {
        while (auto* item = cardArea->layout()->takeAt(0)) {
            delete item->widget();
            delete item;
        }
        updateCount();
    }
    
    Task::Status status() const { return m_status; }
    
signals:
    void addTaskRequested(Task::Status status);
    void editTaskRequested(int taskId);
    void deleteTaskRequested(int taskId);
    void taskStatusChanged(int taskId, Task::Status newStatus);
    void taskDropped(int taskId, Task::Status newStatus);
    
protected:
    void dragEnterEvent(QDragEnterEvent* event) override {
        if (event->mimeData()->hasFormat("application/x-task-id")) {
            event->acceptProposedAction();
            setStyleSheet("border: 2px solid #3498db;");
        }
    }
    
    void dragLeaveEvent(QDragLeaveEvent*) override {
        setStyleSheet("");
    }
    
    void dropEvent(QDropEvent* event) override {
        setStyleSheet("");
        
        int taskId = event->mimeData()->data("application/x-task-id").toInt();
        emit taskDropped(taskId, m_status);
        event->acceptProposedAction();
    }
    
private:
    void updateCount() {
        int count = cardArea->layout()->count();
        countLabel->setText(QString::number(count));
    }
    
    QString headerStyle(Task::Status s) {
        static const QMap<Task::Status, QString> colors = {
            {Task::Status::Todo,       "#95a5a6"},
            {Task::Status::InProgress, "#3498db"},
            {Task::Status::Review,     "#e67e22"},
            {Task::Status::Done,       "#27ae60"},
        };
        return QString("background: %1; padding: 12px; border-radius: 4px 4px 0 0;")
            .arg(colors.value(s, "#95a5a6"));
    }
    
    Task::Status m_status;
    QWidget* cardArea;
    QLabel* titleLabel;
    QLabel* countLabel;
};
```

---

## ขั้นตอนที่ 586-595: Main Kanban Board

```cpp
class TaskBoard : public QWidget {
    Q_OBJECT
    
public:
    explicit TaskBoard(QWidget* parent = nullptr) : QWidget(parent) {
        auto* layout = new QHBoxLayout(this);
        layout->setSpacing(12);
        layout->setContentsMargins(12, 12, 12, 12);
        
        // Create columns for each status
        QMap<Task::Status, QString> columns = {
            {Task::Status::Todo,       "รอดำเนินการ"},
            {Task::Status::InProgress, "กำลังทำ"},
            {Task::Status::Review,     "รอตรวจสอบ"},
            {Task::Status::Done,       "เสร็จสิ้น"},
        };
        
        for (auto it = columns.begin(); it != columns.end(); ++it) {
            auto* col = new KanbanColumn(it.key(), it.value(), this);
            
            connect(col, &KanbanColumn::addTaskRequested,
                    this, &TaskBoard::onAddTask);
            connect(col, &KanbanColumn::editTaskRequested,
                    this, &TaskBoard::editTaskRequested);
            connect(col, &KanbanColumn::deleteTaskRequested,
                    this, &TaskBoard::onDeleteTask);
            connect(col, &KanbanColumn::taskDropped,
                    this, &TaskBoard::onTaskDropped);
            
            layout->addWidget(col, 1);
            m_columns[it.key()] = col;
        }
    }
    
    void setTasks(const QList<Task>& tasks) {
        // Clear all columns
        for (auto* col : m_columns) col->clearCards();
        
        // Place tasks in columns
        for (const Task& task : tasks) {
            if (!task.archived && m_columns.contains(task.status)) {
                m_columns[task.status]->addCard(task);
            }
        }
    }
    
signals:
    void editTaskRequested(int taskId);
    void taskStatusChanged(int taskId, Task::Status newStatus);
    void taskDeleted(int taskId);
    void addTaskRequested(Task::Status status);
    
private slots:
    void onAddTask(Task::Status status) {
        emit addTaskRequested(status);
    }
    
    void onDeleteTask(int taskId) {
        if (QMessageBox::question(this, "Confirm", "Delete this task?") == QMessageBox::Yes) {
            emit taskDeleted(taskId);
        }
    }
    
    void onTaskDropped(int taskId, Task::Status newStatus) {
        emit taskStatusChanged(taskId, newStatus);
    }
    
    QMap<Task::Status, KanbanColumn*> m_columns;
};
```

---

## สรุป Part 041

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Task data class ด้วย Priority/Status enum + JSON serialization
2. ✅ TaskListModel (QAbstractListModel) ครบด้วย roles และ drag-drop
3. ✅ TaskCard widget ด้วย drag ได้ + context menu
4. ✅ KanbanColumn ด้วย drop zone + header
5. ✅ TaskBoard ครบ 4 columns + signal routing

---

⬅️ [Part 040](part040.md) | ➡️ [Part 042: Real-World Project — Task App Backend & Dialog](part042.md)
