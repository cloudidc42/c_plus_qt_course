# Part 040: Modern C++ Application Architecture

## ขั้นตอนที่ 566-580

---

## ขั้นตอนที่ 566: Application Bootstrap Pattern

```cpp
// === main.cpp bootstrap pattern ===
#include <QApplication>
#include <QSplashScreen>
#include <QStyleFactory>

int main(int argc, char* argv[]) {
    QApplication::setHighDpiScaleFactorRoundingPolicy(
        Qt::HighDpiScaleFactorRoundingPolicy::PassThrough);
    
    QApplication app(argc, argv);
    app.setApplicationName("ProApp");
    app.setApplicationVersion("1.0.0");
    app.setOrganizationName("MyCompany");
    app.setOrganizationDomain("mycompany.com");
    
    // Splash screen
    QSplashScreen splash(QPixmap(":/images/splash.png"));
    splash.show();
    app.processEvents();
    
    // Initialize subsystems with progress
    auto showProgress = [&splash](const QString& msg, int progress) {
        splash.showMessage(
            QString("%1 (%2%)").arg(msg).arg(progress),
            Qt::AlignBottom | Qt::AlignHCenter,
            Qt::white
        );
        QApplication::processEvents();
        QThread::msleep(50);
    };
    
    showProgress("Loading configuration...", 20);
    AppSettings::instance(); // Initialize settings
    
    showProgress("Opening database...", 40);
    DatabaseManager::instance().open(
        QDir::appDataLocation() + "/app.sqlite");
    
    showProgress("Loading plugins...", 60);
    // PluginManager::instance().loadAll();
    
    showProgress("Starting network...", 80);
    // NetworkManager::instance().start();
    
    showProgress("Ready", 100);
    QThread::msleep(300);
    
    MainWindow window;
    window.show();
    splash.finish(&window);
    
    return app.exec();
}
```

---

## ขั้นตอนที่ 567: Service Locator Pattern

```cpp
#include <functional>
#include <typeindex>

class ServiceLocator {
public:
    static ServiceLocator& instance() {
        static ServiceLocator sl;
        return sl;
    }
    
    template<typename T>
    void registerService(std::shared_ptr<T> service) {
        m_services[std::type_index(typeid(T))] = service;
    }
    
    template<typename T, typename... Args>
    std::shared_ptr<T> create(Args&&... args) {
        auto service = std::make_shared<T>(std::forward<Args>(args)...);
        registerService<T>(service);
        return service;
    }
    
    template<typename T>
    std::shared_ptr<T> get() {
        auto it = m_services.find(std::type_index(typeid(T)));
        if (it == m_services.end()) {
            qWarning() << "Service not registered:" << typeid(T).name();
            return nullptr;
        }
        return std::static_pointer_cast<T>(it->second);
    }
    
    template<typename T>
    bool has() const {
        return m_services.count(std::type_index(typeid(T))) > 0;
    }
    
    template<typename T>
    void remove() {
        m_services.erase(std::type_index(typeid(T)));
    }
    
    void clear() { m_services.clear(); }
    
private:
    ServiceLocator() = default;
    std::unordered_map<std::type_index, std::shared_ptr<void>> m_services;
};

// --- Interfaces ---
class IUserService {
public:
    virtual ~IUserService() = default;
    virtual QList<User> getUsers() = 0;
    virtual std::optional<User> getUser(int id) = 0;
    virtual int createUser(const QString& name, const QString& email) = 0;
    virtual bool updateUser(const User& user) = 0;
    virtual bool deleteUser(int id) = 0;
};

class INotificationService {
public:
    virtual ~INotificationService() = default;
    virtual void notify(const QString& title, const QString& message) = 0;
    virtual void notifyError(const QString& message) = 0;
};

// --- Concrete implementations ---
class UserService : public IUserService {
    UserRepository m_repo;
public:
    QList<User> getUsers() override { return m_repo.all(); }
    
    std::optional<User> getUser(int id) override { return m_repo.findById(id); }
    
    int createUser(const QString& name, const QString& email) override {
        if (name.isEmpty() || email.isEmpty()) {
            throw std::invalid_argument("Name and email required");
        }
        
        if (m_repo.findByEmail(email)) {
            throw std::runtime_error("Email already exists");
        }
        
        return m_repo.create(name, email);
    }
    
    bool updateUser(const User& user) override { return m_repo.update(user); }
    bool deleteUser(int id) override { return m_repo.remove(id); }
};

class ToastNotificationService : public INotificationService {
    QWidget* m_parent;
public:
    explicit ToastNotificationService(QWidget* parent) : m_parent(parent) {}
    
    void notify(const QString& title, const QString& message) override {
        showToast(title, message, "#2c3e50");
    }
    
    void notifyError(const QString& message) override {
        showToast("Error", message, "#e74c3c");
    }
    
private:
    void showToast(const QString& title, const QString& msg, const QString& color) {
        auto* toast = new QLabel(m_parent);
        toast->setText(QString("<b>%1</b><br>%2").arg(title, msg));
        toast->setStyleSheet(QString(
            "background: %1; color: white; padding: 12px 16px; "
            "border-radius: 6px; font-size: 13px;").arg(color));
        toast->setWordWrap(true);
        toast->setMaximumWidth(280);
        toast->adjustSize();
        
        // Position bottom-right
        QPoint pos(m_parent->width() - toast->width() - 16,
                   m_parent->height() - toast->height() - 16);
        toast->move(pos);
        toast->show();
        toast->raise();
        
        // Auto-dismiss
        QTimer::singleShot(3000, toast, [toast]() {
            auto* anim = new QPropertyAnimation(toast, "windowOpacity");
            anim->setDuration(400);
            anim->setStartValue(1.0);
            anim->setEndValue(0.0);
            connect(anim, &QAbstractAnimation::finished, toast, &QLabel::deleteLater);
            anim->start(QAbstractAnimation::DeleteWhenStopped);
        });
    }
};

// Usage:
// auto& sl = ServiceLocator::instance();
// sl.create<UserService>();
// sl.create<ToastNotificationService>(mainWindow);
//
// auto users = sl.get<IUserService>()->getUsers();
// sl.get<INotificationService>()->notify("Done", "Users loaded");
```

---

## ขั้นตอนที่ 568: Event Bus (Publish/Subscribe)

```cpp
class EventBus : public QObject {
    Q_OBJECT
    
public:
    static EventBus& instance() {
        static EventBus bus;
        return bus;
    }
    
    using Handler = std::function<void(const QVariant&)>;
    
    void subscribe(const QString& event, QObject* owner, Handler handler) {
        m_handlers[event].append({owner, handler});
        
        // Auto-cleanup on owner destroy
        connect(owner, &QObject::destroyed, [this, event, owner]() {
            auto& handlers = m_handlers[event];
            handlers.erase(std::remove_if(handlers.begin(), handlers.end(),
                [owner](const Subscription& s) { return s.owner == owner; }),
                handlers.end());
        });
    }
    
    void unsubscribe(const QString& event, QObject* owner) {
        if (!m_handlers.contains(event)) return;
        auto& handlers = m_handlers[event];
        handlers.erase(std::remove_if(handlers.begin(), handlers.end(),
            [owner](const Subscription& s) { return s.owner == owner; }),
            handlers.end());
    }
    
    void publish(const QString& event, const QVariant& data = {}) {
        if (!m_handlers.contains(event)) return;
        
        // Copy list to allow modification during iteration
        auto handlers = m_handlers[event];
        for (const auto& sub : handlers) {
            if (sub.owner) sub.handler(data);
        }
    }
    
    // Convenience: subscribe with QObject slot
    template<typename Obj, typename Slot>
    void on(const QString& event, Obj* obj, Slot slot) {
        subscribe(event, obj, [obj, slot](const QVariant& data) {
            (obj->*slot)(data);
        });
    }
    
private:
    EventBus() = default;
    
    struct Subscription {
        QObject* owner;
        Handler handler;
    };
    
    QHash<QString, QList<Subscription>> m_handlers;
};

// --- Events ---
namespace Events {
    const QString UserCreated   = "user.created";
    const QString UserUpdated   = "user.updated";
    const QString UserDeleted   = "user.deleted";
    const QString ThemeChanged  = "ui.theme_changed";
    const QString LangChanged   = "ui.lang_changed";
    const QString NetworkError  = "network.error";
    const QString DataLoaded    = "data.loaded";
}

// Usage:
// EventBus::instance().subscribe(Events::UserCreated, this, [this](const QVariant& data) {
//     auto user = data.value<User>();
//     refreshTable();
// });
//
// EventBus::instance().publish(Events::UserCreated, QVariant::fromValue(newUser));
```

---

## ขั้นตอนที่ 569: Command Pattern + Undo/Redo Stack

```cpp
// === Generic Command Interface ===
class ICommand {
public:
    virtual ~ICommand() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
    virtual QString description() const = 0;
    virtual bool isReversible() const { return true; }
};

class CommandHistory {
public:
    static CommandHistory& instance() {
        static CommandHistory hist;
        return hist;
    }
    
    void execute(std::unique_ptr<ICommand> cmd) {
        cmd->execute();
        
        m_undoStack.push_back(std::move(cmd));
        m_redoStack.clear();
        
        if (m_undoStack.size() > m_maxSize) {
            m_undoStack.pop_front();
        }
        
        emit changed();
    }
    
    bool canUndo() const { return !m_undoStack.empty(); }
    bool canRedo() const { return !m_redoStack.empty(); }
    
    void undo() {
        if (!canUndo()) return;
        auto cmd = std::move(m_undoStack.back());
        m_undoStack.pop_back();
        cmd->undo();
        m_redoStack.push_back(std::move(cmd));
        emit changed();
    }
    
    void redo() {
        if (!canRedo()) return;
        auto cmd = std::move(m_redoStack.back());
        m_redoStack.pop_back();
        cmd->execute();
        m_undoStack.push_back(std::move(cmd));
        emit changed();
    }
    
    QString undoDescription() const {
        return canUndo() ? m_undoStack.back()->description() : "";
    }
    
    QString redoDescription() const {
        return canRedo() ? m_redoStack.back()->description() : "";
    }
    
    void clear() {
        m_undoStack.clear();
        m_redoStack.clear();
        emit changed();
    }
    
    QStringList history() const {
        QStringList result;
        for (const auto& cmd : m_undoStack) result << cmd->description();
        return result;
    }
    
    // Signal via lambdas
    std::function<void()> changed = [](){};
    
private:
    CommandHistory() = default;
    std::deque<std::unique_ptr<ICommand>> m_undoStack;
    std::deque<std::unique_ptr<ICommand>> m_redoStack;
    int m_maxSize = 100;
};

// === Concrete commands ===
class CreateUserCommand : public ICommand {
public:
    CreateUserCommand(const QString& name, const QString& email)
        : m_name(name), m_email(email) {}
    
    void execute() override {
        UserRepository repo;
        m_createdId = repo.create(m_name, m_email);
    }
    
    void undo() override {
        if (m_createdId > 0) {
            UserRepository repo;
            repo.remove(m_createdId);
        }
    }
    
    QString description() const override {
        return QString("Create user: %1").arg(m_name);
    }
    
private:
    QString m_name, m_email;
    int m_createdId = 0;
};

class DeleteUserCommand : public ICommand {
public:
    explicit DeleteUserCommand(int id) : m_id(id) {}
    
    void execute() override {
        UserRepository repo;
        auto user = repo.findById(m_id);
        if (user) {
            m_backup = *user;
            repo.remove(m_id);
        }
    }
    
    void undo() override {
        if (m_backup.id > 0) {
            UserRepository repo;
            repo.create(m_backup.name, m_backup.email);
        }
    }
    
    QString description() const override {
        return QString("Delete user: %1").arg(m_backup.name);
    }
    
private:
    int m_id;
    User m_backup;
};
```

---

## ขั้นตอนที่ 570: Dependency Injection Container

```cpp
class DependencyContainer {
public:
    template<typename I, typename T, typename... Deps>
    void bind() {
        m_factories[std::type_index(typeid(I))] = [this]() -> std::shared_ptr<void> {
            return std::make_shared<T>(resolve<Deps>()...);
        };
    }
    
    template<typename T>
    void bindInstance(std::shared_ptr<T> instance) {
        auto key = std::type_index(typeid(T));
        m_singletons[key] = instance;
    }
    
    template<typename T>
    void bindSingleton() {
        auto key = std::type_index(typeid(T));
        m_factories[key] = [this, key]() -> std::shared_ptr<void> {
            if (m_singletons.count(key) == 0) {
                m_singletons[key] = std::make_shared<T>();
            }
            return m_singletons.at(key);
        };
    }
    
    template<typename T>
    std::shared_ptr<T> resolve() {
        auto key = std::type_index(typeid(T));
        
        if (m_singletons.count(key)) {
            return std::static_pointer_cast<T>(m_singletons.at(key));
        }
        
        if (m_factories.count(key)) {
            auto obj = m_factories.at(key)();
            return std::static_pointer_cast<T>(obj);
        }
        
        qWarning() << "DI: Cannot resolve" << typeid(T).name();
        return nullptr;
    }
    
private:
    std::unordered_map<std::type_index, std::function<std::shared_ptr<void>()>> m_factories;
    std::unordered_map<std::type_index, std::shared_ptr<void>> m_singletons;
};

// === Setup ===
void configureDI(DependencyContainer& di) {
    di.bindSingleton<DatabaseManager>();
    di.bind<IUserService, UserService>();
    // di.bind<INotificationService, ToastNotificationService>();
}

// === Usage ===
// DependencyContainer di;
// configureDI(di);
// auto userService = di.resolve<IUserService>();
// userService->createUser("สมชาย", "somchai@example.com");
```

---

## สรุป Part 040

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Application Bootstrap pattern ด้วย QSplashScreen
2. ✅ Service Locator pattern สำหรับ loose coupling
3. ✅ Event Bus (Publish/Subscribe) สำหรับ decoupled communication
4. ✅ Command Pattern + History สำหรับ Undo/Redo
5. ✅ Dependency Injection Container แบบ type-safe

---

⬅️ [Part 039](part039.md) | ➡️ [Part 041: Real-World Project — Task Management App](part041.md)
