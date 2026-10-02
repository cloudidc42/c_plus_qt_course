# Part 021: Design Patterns in Qt

## ขั้นตอนที่ 281-295

---

## ขั้นตอนที่ 281: Singleton Pattern

```cpp
// === Qt Singleton ===
class AppConfig : public QObject {
    Q_OBJECT
    
private:
    QSettings* settings;
    QMap<QString, QVariant> cache;
    
    AppConfig() : QObject(nullptr) {
        settings = new QSettings("MyCompany", "MyApp", this);
        qDebug() << "AppConfig initialized";
    }
    
    ~AppConfig() = default;
    AppConfig(const AppConfig&) = delete;
    AppConfig& operator=(const AppConfig&) = delete;
    
public:
    static AppConfig* instance() {
        static AppConfig* inst = nullptr;
        if (!inst) {
            inst = new AppConfig();
        }
        return inst;
    }
    
    QVariant get(const QString& key, const QVariant& defaultVal = {}) const {
        if (cache.contains(key)) return cache[key];
        return settings->value(key, defaultVal);
    }
    
    void set(const QString& key, const QVariant& value) {
        cache[key] = value;
        settings->setValue(key, value);
        emit configChanged(key, value);
    }
    
    QString theme() const { return get("theme", "light").toString(); }
    void setTheme(const QString& t) { set("theme", t); }
    
    QString language() const { return get("language", "th").toString(); }
    void setLanguage(const QString& l) { set("language", l); }
    
    QSize windowSize() const {
        return get("windowSize", QSize(800, 600)).toSize();
    }
    
signals:
    void configChanged(const QString& key, const QVariant& value);
};

// Usage
void useConfig() {
    auto* cfg = AppConfig::instance();
    cfg->setTheme("dark");
    qDebug() << "Theme:" << cfg->theme();
    
    // Connect to changes
    QObject::connect(cfg, &AppConfig::configChanged, 
                     [](const QString& k, const QVariant& v) {
        qDebug() << "Config changed:" << k << "=" << v;
    });
}
```

---

## ขั้นตอนที่ 282: Observer Pattern (Signal/Slot)

```cpp
// Qt's Signal/Slot IS Observer Pattern
// แต่นี่คือ custom implementation ที่ละเอียดขึ้น

class EventSystem : public QObject {
    Q_OBJECT
    
public:
    using Handler = std::function<void(const QVariant&)>;
    
    static EventSystem* instance() {
        static EventSystem inst;
        return &inst;
    }
    
    // Subscribe to event
    QMetaObject::Connection subscribe(const QString& event, QObject* receiver, 
                                       Handler handler) {
        auto* wrapper = new QObject(receiver);
        
        connect(this, &EventSystem::eventFired, wrapper,
                [event, handler](const QString& name, const QVariant& data) {
            if (name == event) handler(data);
        });
        
        return QMetaObject::Connection{};
    }
    
    // Publish event
    void publish(const QString& event, const QVariant& data = {}) {
        emit eventFired(event, data);
    }
    
signals:
    void eventFired(const QString& event, const QVariant& data);
};

// Usage
void useEvents() {
    auto* events = EventSystem::instance();
    
    QObject lifetime;
    
    events->subscribe("user.login", &lifetime, [](const QVariant& data) {
        qDebug() << "User logged in:" << data.toString();
    });
    
    events->subscribe("user.logout", &lifetime, [](const QVariant&) {
        qDebug() << "User logged out";
    });
    
    events->subscribe("cart.updated", &lifetime, [](const QVariant& data) {
        qDebug() << "Cart updated, items:" << data.toInt();
    });
    
    // Publish
    events->publish("user.login", "สมชาย");
    events->publish("cart.updated", 5);
    events->publish("user.logout");
}
```

---

## ขั้นตอนที่ 283: Command Pattern - Undo/Redo

```cpp
#include <QUndoStack>
#include <QUndoCommand>
#include <QTextEdit>

// === QUndoCommand ===
class ChangeColorCommand : public QUndoCommand {
private:
    QWidget* widget;
    QColor oldColor;
    QColor newColor;
    
public:
    ChangeColorCommand(QWidget* w, const QColor& color, QUndoCommand* parent = nullptr)
        : QUndoCommand("Change Color", parent), widget(w), newColor(color) {
        oldColor = w->palette().color(QPalette::Window);
    }
    
    void redo() override {
        QPalette p = widget->palette();
        p.setColor(QPalette::Window, newColor);
        widget->setPalette(p);
        widget->setAutoFillBackground(true);
    }
    
    void undo() override {
        QPalette p = widget->palette();
        p.setColor(QPalette::Window, oldColor);
        widget->setPalette(p);
        widget->setAutoFillBackground(true);
    }
    
    bool mergeWith(const QUndoCommand* other) override {
        auto* cmd = static_cast<const ChangeColorCommand*>(other);
        if (cmd->widget != widget) return false;
        newColor = cmd->newColor;
        return true;
    }
    
    int id() const override { return 1; }
};

class MoveCommand : public QUndoCommand {
private:
    QWidget* widget;
    QPoint oldPos;
    QPoint newPos;
    
public:
    MoveCommand(QWidget* w, const QPoint& pos, QUndoCommand* parent = nullptr)
        : QUndoCommand("Move", parent), widget(w), newPos(pos) {
        oldPos = w->pos();
    }
    
    void redo() override { widget->move(newPos); }
    void undo() override { widget->move(oldPos); }
};

// Editor with Undo/Redo
class UndoEditor : public QMainWindow {
    Q_OBJECT
    
public:
    UndoEditor(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Undo/Redo Demo");
        setMinimumSize(500, 400);
        
        undoStack = new QUndoStack(this);
        
        textEdit = new QTextEdit(this);
        setCentralWidget(textEdit);
        
        // Create undo/redo actions
        auto* undoAct = undoStack->createUndoAction(this);
        undoAct->setShortcut(QKeySequence::Undo);
        
        auto* redoAct = undoStack->createRedoAction(this);
        redoAct->setShortcut(QKeySequence::Redo);
        
        // Menu
        auto* editMenu = menuBar()->addMenu("Edit");
        editMenu->addAction(undoAct);
        editMenu->addAction(redoAct);
        editMenu->addSeparator();
        
        auto* changeColorAct = new QAction("Change Color", this);
        connect(changeColorAct, &QAction::triggered, [this]() {
            QColor color = QColorDialog::getColor(Qt::white, this);
            if (color.isValid()) {
                auto* cmd = new ChangeColorCommand(centralWidget(), color);
                undoStack->push(cmd);
            }
        });
        editMenu->addAction(changeColorAct);
        
        // Status bar shows undo/redo state
        connect(undoStack, &QUndoStack::indexChanged, [this](int) {
            statusBar()->showMessage(
                QString("Undo: %1/%2").arg(undoStack->index()).arg(undoStack->count())
            );
        });
    }
    
private:
    QUndoStack* undoStack;
    QTextEdit* textEdit;
};
```

---

## ขั้นตอนที่ 284: Factory Pattern

```cpp
// === Abstract Widget Factory ===
class WidgetFactory {
public:
    virtual ~WidgetFactory() = default;
    virtual QPushButton* createButton(const QString& text) = 0;
    virtual QLineEdit* createInput(const QString& placeholder) = 0;
    virtual QLabel* createLabel(const QString& text) = 0;
    
    static std::unique_ptr<WidgetFactory> create(const QString& theme);
};

class LightFactory : public WidgetFactory {
public:
    QPushButton* createButton(const QString& text) override {
        auto* btn = new QPushButton(text);
        btn->setStyleSheet(R"(
            QPushButton {
                background: #3498db;
                color: white;
                border-radius: 6px;
                padding: 8px 16px;
            }
            QPushButton:hover { background: #2980b9; }
        )");
        return btn;
    }
    
    QLineEdit* createInput(const QString& placeholder) override {
        auto* edit = new QLineEdit();
        edit->setPlaceholderText(placeholder);
        edit->setStyleSheet(R"(
            QLineEdit {
                border: 2px solid #ddd;
                border-radius: 4px;
                padding: 6px;
                background: white;
            }
            QLineEdit:focus { border-color: #3498db; }
        )");
        return edit;
    }
    
    QLabel* createLabel(const QString& text) override {
        auto* lbl = new QLabel(text);
        lbl->setStyleSheet("color: #2c3e50; font-size: 14px;");
        return lbl;
    }
};

class DarkFactory : public WidgetFactory {
public:
    QPushButton* createButton(const QString& text) override {
        auto* btn = new QPushButton(text);
        btn->setStyleSheet(R"(
            QPushButton {
                background: #1a73e8;
                color: white;
                border-radius: 6px;
                padding: 8px 16px;
            }
            QPushButton:hover { background: #1557b0; }
        )");
        return btn;
    }
    
    QLineEdit* createInput(const QString& placeholder) override {
        auto* edit = new QLineEdit();
        edit->setPlaceholderText(placeholder);
        edit->setStyleSheet(R"(
            QLineEdit {
                border: 2px solid #444;
                border-radius: 4px;
                padding: 6px;
                background: #2d2d2d;
                color: white;
            }
            QLineEdit:focus { border-color: #1a73e8; }
        )");
        return edit;
    }
    
    QLabel* createLabel(const QString& text) override {
        auto* lbl = new QLabel(text);
        lbl->setStyleSheet("color: #e0e0e0; font-size: 14px;");
        return lbl;
    }
};

std::unique_ptr<WidgetFactory> WidgetFactory::create(const QString& theme) {
    if (theme == "dark") return std::make_unique<DarkFactory>();
    return std::make_unique<LightFactory>();
}
```

---

## ขั้นตอนที่ 285: Strategy Pattern

```cpp
// === Sort Strategy ===
class SortStrategy {
public:
    virtual ~SortStrategy() = default;
    virtual void sort(QList<int>& data) = 0;
    virtual QString name() const = 0;
};

class BubbleSort : public SortStrategy {
public:
    void sort(QList<int>& data) override {
        int n = data.size();
        for (int i = 0; i < n-1; i++) {
            for (int j = 0; j < n-i-1; j++) {
                if (data[j] > data[j+1]) {
                    qSwap(data[j], data[j+1]);
                }
            }
        }
    }
    QString name() const override { return "Bubble Sort"; }
};

class QuickSort : public SortStrategy {
    void quicksort(QList<int>& arr, int lo, int hi) {
        if (lo >= hi) return;
        int pivot = arr[hi];
        int i = lo - 1;
        for (int j = lo; j < hi; j++) {
            if (arr[j] <= pivot) {
                i++;
                qSwap(arr[i], arr[j]);
            }
        }
        qSwap(arr[i+1], arr[hi]);
        int p = i + 1;
        quicksort(arr, lo, p-1);
        quicksort(arr, p+1, hi);
    }
    
public:
    void sort(QList<int>& data) override {
        if (!data.isEmpty()) quicksort(data, 0, data.size()-1);
    }
    QString name() const override { return "Quick Sort"; }
};

class StdSort : public SortStrategy {
public:
    void sort(QList<int>& data) override {
        std::sort(data.begin(), data.end());
    }
    QString name() const override { return "std::sort"; }
};

class Sorter {
private:
    std::unique_ptr<SortStrategy> strategy;
    
public:
    void setStrategy(std::unique_ptr<SortStrategy> s) {
        strategy = std::move(s);
    }
    
    QList<int> sort(QList<int> data) {
        if (strategy) strategy->sort(data);
        return data;
    }
    
    QString strategyName() const {
        return strategy ? strategy->name() : "None";
    }
};

// Usage
void useSortStrategy() {
    Sorter sorter;
    QList<int> data = {5, 2, 8, 1, 9, 3, 7};
    
    sorter.setStrategy(std::make_unique<BubbleSort>());
    auto r1 = sorter.sort(data);
    
    sorter.setStrategy(std::make_unique<QuickSort>());
    auto r2 = sorter.sort(data);
    
    sorter.setStrategy(std::make_unique<StdSort>());
    auto r3 = sorter.sort(data);
}
```

---

## ขั้นตอนที่ 286-295: Decorator Pattern

```cpp
// === Widget Decorator ===
class WidgetDecorator {
protected:
    QWidget* widget;
    
public:
    explicit WidgetDecorator(QWidget* w) : widget(w) {}
    virtual ~WidgetDecorator() = default;
    virtual QWidget* decorate() { return widget; }
};

class ShadowDecorator : public WidgetDecorator {
    QColor shadowColor;
    int blurRadius;
    
public:
    ShadowDecorator(QWidget* w, const QColor& color = QColor(0,0,0,80), int blur = 10)
        : WidgetDecorator(w), shadowColor(color), blurRadius(blur) {}
    
    QWidget* decorate() override {
        auto* effect = new QGraphicsDropShadowEffect(widget);
        effect->setBlurRadius(blurRadius);
        effect->setColor(shadowColor);
        effect->setOffset(3, 3);
        widget->setGraphicsEffect(effect);
        return widget;
    }
};

class TooltipDecorator : public WidgetDecorator {
    QString tooltip;
    
public:
    TooltipDecorator(QWidget* w, const QString& tip)
        : WidgetDecorator(w), tooltip(tip) {}
    
    QWidget* decorate() override {
        widget->setToolTip(tooltip);
        return widget;
    }
};

class BadgeDecorator : public WidgetDecorator {
    int count;
    
public:
    BadgeDecorator(QWidget* w, int badgeCount)
        : WidgetDecorator(w), count(badgeCount) {}
    
    QWidget* decorate() override {
        // Wrap button in container with badge overlay
        auto* container = new QWidget(widget->parentWidget());
        auto* layout = new QVBoxLayout(container);
        layout->setContentsMargins(0, 0, 0, 0);
        layout->addWidget(widget);
        
        // Badge label
        auto* badge = new QLabel(QString::number(count), container);
        badge->setStyleSheet(R"(
            QLabel {
                background: red;
                color: white;
                border-radius: 10px;
                padding: 2px 6px;
                font-size: 10px;
                font-weight: bold;
            }
        )");
        badge->resize(20, 20);
        badge->move(widget->width() - 10, -5);
        badge->raise();
        
        return container;
    }
};
```

---

## สรุป Part 021

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Singleton Pattern (Thread-safe)
2. ✅ Observer Pattern ด้วย Signal/Slot
3. ✅ Command Pattern (QUndoStack)
4. ✅ Factory Pattern (Widget creation)
5. ✅ Strategy Pattern (Sort algorithms)
6. ✅ Decorator Pattern

---

⬅️ [Part 020](part020.md) | ➡️ [Part 022: Qt Internationalization](part022.md)
