# Part 046: Professional Level — Reactive Signals & Async Patterns

## ขั้นตอนที่ 656-670

---

## ขั้นตอนที่ 656: Modern Qt Async Patterns

```
Qt 6 Async Tools:
  - QFuture<T> + QPromise<T>: composable async values
  - QtConcurrent::run: background tasks
  - QFutureSynchronizer: wait for multiple futures
  - co_await QtFuture::connect (Qt 6.3+)
  - Q_ASYNC macro (concept)

Goal: avoid callback hell, keep UI responsive
```

---

## ขั้นตอนที่ 657: QPromise + QFuture

```cpp
#include <QFuture>
#include <QFutureWatcher>
#include <QPromise>
#include <QtConcurrent>

// === Basic QPromise usage ===
QFuture<QString> fetchDataAsync(const QString& url) {
    QPromise<QString> promise;
    QFuture<QString> future = promise.future();
    
    auto* watcher = new QNetworkAccessManager();
    
    QNetworkRequest req{QUrl(url)};
    auto* reply = watcher->get(req);
    
    QObject::connect(reply, &QNetworkReply::finished, [=, p = std::move(promise)]() mutable {
        if (reply->error() == QNetworkReply::NoError) {
            p.addResult(QString::fromUtf8(reply->readAll()));
        } else {
            p.setException(std::make_exception_ptr(
                std::runtime_error(reply->errorString().toStdString())));
        }
        p.finish();
        reply->deleteLater();
        watcher->deleteLater();
    });
    
    promise.start();
    return future;
}

// === QFutureWatcher: connect to UI ===
class DataLoader : public QObject {
    Q_OBJECT
    
public:
    explicit DataLoader(QObject* parent = nullptr) : QObject(parent) {
        watcher = new QFutureWatcher<QString>(this);
        
        connect(watcher, &QFutureWatcher<QString>::finished, [this]() {
            try {
                QString result = watcher->result();
                emit dataLoaded(result);
            } catch (const std::exception& e) {
                emit loadError(QString::fromStdString(e.what()));
            }
        });
        
        connect(watcher, &QFutureWatcher<QString>::progressValueChanged,
                this, &DataLoader::progressChanged);
    }
    
    void load(const QString& url) {
        QFuture<QString> future = fetchDataAsync(url);
        watcher->setFuture(future);
        emit loadStarted();
    }
    
signals:
    void loadStarted();
    void dataLoaded(const QString& data);
    void loadError(const QString& msg);
    void progressChanged(int percent);
    
private:
    QFutureWatcher<QString>* watcher;
};
```

---

## ขั้นตอนที่ 658: Promise Chain Pattern

```cpp
// === Promise chaining ===
template<typename T, typename Func>
auto then(QFuture<T> future, Func func) {
    using RetType = decltype(func(std::declval<T>()));
    
    QPromise<RetType> promise;
    QFuture<RetType> resultFuture = promise.future();
    promise.start();
    
    auto* watcher = new QFutureWatcher<T>();
    QObject::connect(watcher, &QFutureWatcher<T>::finished, [=, p = std::move(promise)]() mutable {
        try {
            if constexpr (std::is_void_v<RetType>) {
                func(watcher->result());
                p.finish();
            } else {
                p.addResult(func(watcher->result()));
                p.finish();
            }
        } catch (...) {
            p.setException(std::current_exception());
            p.finish();
        }
        watcher->deleteLater();
    });
    watcher->setFuture(future);
    
    return resultFuture;
}

// === Pipeline API ===
class AsyncPipeline {
public:
    // Load JSON from URL → Parse → Transform
    static void example() {
        auto loadFuture = QtConcurrent::run([]() -> QByteArray {
            // Simulate HTTP fetch
            QThread::msleep(100);
            return R"({"users":[{"id":1,"name":"Alice"},{"id":2,"name":"Bob"}]})";
        });
        
        auto parseFuture = then(loadFuture, [](const QByteArray& data) -> QJsonDocument {
            return QJsonDocument::fromJson(data);
        });
        
        auto extractFuture = then(parseFuture, [](const QJsonDocument& doc) -> QStringList {
            QStringList names;
            for (const auto& user : doc.object()["users"].toArray()) {
                names << user.toObject()["name"].toString();
            }
            return names;
        });
        
        auto* watcher = new QFutureWatcher<QStringList>();
        QObject::connect(watcher, &QFutureWatcher<QStringList>::finished, [watcher]() {
            qDebug() << "Users:" << watcher->result();
            watcher->deleteLater();
        });
        watcher->setFuture(extractFuture);
    }
};
```

---

## ขั้นตอนที่ 659: Reactive Property System

```cpp
// === Reactive property with automatic update ===
template<typename T>
class Observable {
public:
    using Callback = std::function<void(const T&)>;
    
    Observable() = default;
    explicit Observable(const T& value) : m_value(value) {}
    
    const T& get() const { return m_value; }
    
    void set(const T& value) {
        if (m_value == value) return;
        m_value = value;
        notifyAll();
    }
    
    Observable& operator=(const T& v) { set(v); return *this; }
    operator const T&() const { return m_value; }
    
    int subscribe(Callback cb) {
        int id = m_nextId++;
        m_callbacks[id] = std::move(cb);
        return id;
    }
    
    void unsubscribe(int id) {
        m_callbacks.erase(id);
    }
    
    // Derive a computed observable
    template<typename Func>
    auto map(Func func) const {
        using U = decltype(func(m_value));
        auto derived = std::make_shared<Observable<U>>(func(m_value));
        
        subscribe([derived, func](const T& val) {
            derived->set(func(val));
        });
        
        return derived;
    }
    
private:
    void notifyAll() {
        for (auto& [id, cb] : m_callbacks) cb(m_value);
    }
    
    T m_value{};
    std::map<int, Callback> m_callbacks;
    int m_nextId = 0;
};

// === Usage with Qt Widgets ===
class ReactiveForm : public QWidget {
    Q_OBJECT
    
public:
    ReactiveForm(QWidget* parent = nullptr) : QWidget(parent) {
        auto* layout = new QFormLayout(this);
        
        // Observable state
        auto username = std::make_shared<Observable<QString>>("");
        auto email    = std::make_shared<Observable<QString>>("");
        auto isValid  = username->map([](const QString& u) { return u.length() >= 3; });
        
        auto* nameEdit = new QLineEdit();
        auto* emailEdit = new QLineEdit();
        auto* submitBtn = new QPushButton("Submit");
        auto* preview = new QLabel("---");
        
        layout->addRow("Username:", nameEdit);
        layout->addRow("Email:", emailEdit);
        layout->addRow("Preview:", preview);
        layout->addRow(submitBtn);
        
        // Bind QLineEdit → Observable
        connect(nameEdit, &QLineEdit::textChanged, [username](const QString& t) {
            username->set(t);
        });
        
        connect(emailEdit, &QLineEdit::textChanged, [email](const QString& t) {
            email->set(t);
        });
        
        // Bind Observable → Widget
        username->subscribe([preview, email](const QString& name) {
            // Note: captures email by shared_ptr
        });
        
        isValid->subscribe([submitBtn](bool valid) {
            submitBtn->setEnabled(valid);
            submitBtn->setStyleSheet(valid
                ? "background: #27ae60; color: white; padding: 8px 24px;"
                : "background: #ccc; color: white; padding: 8px 24px;");
        });
        
        // Trigger initial state
        submitBtn->setEnabled(false);
    }
};
```

---

## ขั้นตอนที่ 660: Async Queue Worker

```cpp
// === Thread-safe task queue with worker threads ===
template<typename Task>
class AsyncTaskQueue : public QObject {
    Q_OBJECT
    
public:
    using TaskFunc = std::function<Task()>;
    using ResultCallback = std::function<void(const Task&)>;
    
    AsyncTaskQueue(int numWorkers = QThread::idealThreadCount(), QObject* parent = nullptr)
        : QObject(parent) {
        
        for (int i = 0; i < numWorkers; ++i) {
            auto* worker = QThread::create([this]() { workerLoop(); });
            worker->setParent(this);
            m_workers.append(worker);
            worker->start();
        }
    }
    
    ~AsyncTaskQueue() {
        {
            QMutexLocker lock(&m_mutex);
            m_stopping = true;
        }
        m_cond.wakeAll();
        for (auto* w : m_workers) w->wait();
    }
    
    void submit(TaskFunc func, ResultCallback callback = nullptr) {
        QMutexLocker lock(&m_mutex);
        m_queue.push_back({std::move(func), callback});
        m_cond.wakeOne();
    }
    
    int pending() const {
        QMutexLocker lock(&m_mutex);
        return m_queue.size();
    }
    
signals:
    void taskCompleted(int completedCount);
    void allTasksDone();
    
private:
    void workerLoop() {
        while (true) {
            Entry entry;
            
            {
                QMutexLocker lock(&m_mutex);
                while (m_queue.empty() && !m_stopping) {
                    m_cond.wait(&m_mutex);
                }
                
                if (m_stopping && m_queue.empty()) break;
                
                entry = std::move(m_queue.front());
                m_queue.pop_front();
            }
            
            try {
                Task result = entry.func();
                
                if (entry.callback) {
                    QMetaObject::invokeMethod(this, [=]() {
                        entry.callback(result);
                    }, Qt::QueuedConnection);
                }
                
                int completed = ++m_completed;
                
                QMetaObject::invokeMethod(this, [this, completed]() {
                    emit taskCompleted(completed);
                    QMutexLocker lock(&m_mutex);
                    if (m_queue.empty()) emit allTasksDone();
                }, Qt::QueuedConnection);
                
            } catch (const std::exception& e) {
                qWarning() << "Task error:" << e.what();
            }
        }
    }
    
    struct Entry {
        TaskFunc func;
        ResultCallback callback;
    };
    
    mutable QMutex m_mutex;
    QWaitCondition m_cond;
    std::deque<Entry> m_queue;
    QList<QThread*> m_workers;
    std::atomic<int> m_completed{0};
    bool m_stopping = false;
};

// Usage:
// AsyncTaskQueue<QString> queue(4);
// queue.submit([]() -> QString {
//     QThread::msleep(100);
//     return "result";
// }, [](const QString& result) {
//     qDebug() << "Got:" << result;
// });
```

---

## ขั้นตอนที่ 661-670: Async Download Manager

```cpp
class DownloadManager : public QObject {
    Q_OBJECT
    
public:
    struct Download {
        int id;
        QString url;
        QString savePath;
        qint64 total = 0;
        qint64 received = 0;
        bool finished = false;
        bool error = false;
        QString errorString;
    };
    
    explicit DownloadManager(QObject* parent = nullptr) : QObject(parent) {
        nam = new QNetworkAccessManager(this);
    }
    
    int add(const QString& url, const QString& savePath) {
        int id = ++m_nextId;
        Download dl;
        dl.id = id;
        dl.url = url;
        dl.savePath = savePath;
        m_downloads[id] = dl;
        
        return id;
    }
    
    void start(int id) {
        auto it = m_downloads.find(id);
        if (it == m_downloads.end()) return;
        
        Download& dl = it.value();
        
        QNetworkRequest req{QUrl(dl.url)};
        req.setAttribute(QNetworkRequest::RedirectPolicyAttribute,
                         QNetworkRequest::NoLessSafeRedirectPolicy);
        
        auto* reply = nam->get(req);
        m_replies[id] = reply;
        
        // Progress
        connect(reply, &QNetworkReply::downloadProgress,
                [this, id](qint64 received, qint64 total) {
            auto& d = m_downloads[id];
            d.received = received;
            d.total = total;
            int pct = (total > 0) ? static_cast<int>(received * 100 / total) : 0;
            emit downloadProgress(id, pct, received, total);
        });
        
        // Write to file
        auto* file = new QFile(dl.savePath, reply);
        file->open(QIODevice::WriteOnly);
        
        connect(reply, &QNetworkReply::readyRead, [reply, file]() {
            file->write(reply->readAll());
        });
        
        // Finish
        connect(reply, &QNetworkReply::finished, [this, id, reply, file]() {
            file->flush();
            file->close();
            
            auto& d = m_downloads[id];
            d.finished = true;
            
            if (reply->error() != QNetworkReply::NoError) {
                d.error = true;
                d.errorString = reply->errorString();
                emit downloadFailed(id, d.errorString);
            } else {
                emit downloadFinished(id, d.savePath);
            }
            
            m_replies.remove(id);
            reply->deleteLater();
        });
    }
    
    void startAll() {
        for (auto id : m_downloads.keys()) {
            if (!m_replies.contains(id)) start(id);
        }
    }
    
    void cancel(int id) {
        if (auto* reply = m_replies.value(id)) {
            reply->abort();
        }
    }
    
    void cancelAll() {
        for (auto* reply : m_replies) reply->abort();
    }
    
    const Download& downloadInfo(int id) const {
        return m_downloads[id];
    }
    
    QList<int> allIds() const { return m_downloads.keys(); }
    
signals:
    void downloadProgress(int id, int percent, qint64 received, qint64 total);
    void downloadFinished(int id, const QString& path);
    void downloadFailed(int id, const QString& error);
    
private:
    QNetworkAccessManager* nam;
    QHash<int, Download> m_downloads;
    QHash<int, QNetworkReply*> m_replies;
    int m_nextId = 0;
};

// === Download Manager UI ===
class DownloadManagerWindow : public QMainWindow {
    Q_OBJECT
    
public:
    DownloadManagerWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Download Manager");
        setMinimumSize(700, 500);
        
        manager = new DownloadManager(this);
        
        setupUi();
        
        connect(manager, &DownloadManager::downloadProgress,
                this, &DownloadManagerWindow::onProgress);
        connect(manager, &DownloadManager::downloadFinished,
                this, &DownloadManagerWindow::onFinished);
        connect(manager, &DownloadManager::downloadFailed,
                this, &DownloadManagerWindow::onFailed);
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* layout = new QVBoxLayout(central);
        setCentralWidget(central);
        
        // Add URL bar
        auto* addBar = new QHBoxLayout();
        urlEdit = new QLineEdit();
        urlEdit->setPlaceholderText("https://example.com/file.zip");
        auto* addBtn = new QPushButton("Add Download");
        addBtn->setStyleSheet("background: #3498db; color: white; padding: 8px 16px;");
        addBar->addWidget(urlEdit, 1);
        addBar->addWidget(addBtn);
        
        // Downloads list
        downloadList = new QTreeWidget();
        downloadList->setColumnCount(4);
        downloadList->setHeaderLabels({"File", "Size", "Progress", "Status"});
        downloadList->header()->setSectionResizeMode(0, QHeaderView::Stretch);
        downloadList->setAlternatingRowColors(true);
        
        // Controls
        auto* ctrlBar = new QHBoxLayout();
        auto* startAllBtn = new QPushButton("Start All");
        auto* cancelAllBtn = new QPushButton("Cancel All");
        startAllBtn->setStyleSheet("background: #27ae60; color: white; padding: 6px 16px;");
        cancelAllBtn->setStyleSheet("background: #e74c3c; color: white; padding: 6px 16px;");
        ctrlBar->addWidget(startAllBtn);
        ctrlBar->addWidget(cancelAllBtn);
        ctrlBar->addStretch();
        
        layout->addLayout(addBar);
        layout->addWidget(downloadList, 1);
        layout->addLayout(ctrlBar);
        
        connect(addBtn, &QPushButton::clicked, this, &DownloadManagerWindow::addDownload);
        connect(startAllBtn, &QPushButton::clicked, manager, &DownloadManager::startAll);
        connect(cancelAllBtn, &QPushButton::clicked, manager, &DownloadManager::cancelAll);
    }
    
    void addDownload() {
        QString url = urlEdit->text().trimmed();
        if (url.isEmpty()) return;
        
        QString fileName = QUrl(url).fileName();
        if (fileName.isEmpty()) fileName = "download_" + QString::number(QDateTime::currentMSecsSinceEpoch());
        
        QString savePath = QFileDialog::getSaveFileName(this, "Save As", 
            QDir::downloadPath() + "/" + fileName);
        if (savePath.isEmpty()) return;
        
        int id = manager->add(url, savePath);
        
        auto* item = new QTreeWidgetItem(downloadList);
        item->setText(0, fileName);
        item->setText(1, "...");
        item->setText(3, "Waiting");
        item->setData(0, Qt::UserRole, id);
        
        // Add progress bar
        auto* bar = new QProgressBar();
        bar->setRange(0, 100);
        downloadList->setItemWidget(item, 2, bar);
        
        m_items[id] = item;
        
        manager->start(id);
        urlEdit->clear();
    }
    
    void onProgress(int id, int percent, qint64 received, qint64 total) {
        auto* item = m_items.value(id);
        if (!item) return;
        
        auto* bar = qobject_cast<QProgressBar*>(downloadList->itemWidget(item, 2));
        if (bar) bar->setValue(percent);
        
        item->setText(1, formatSize(total));
        item->setText(3, QString("Downloading (%1%)").arg(percent));
    }
    
    void onFinished(int id, const QString& path) {
        auto* item = m_items.value(id);
        if (!item) return;
        
        item->setText(3, "Done ✓");
        item->setForeground(3, QBrush(QColor("#27ae60")));
        
        auto* bar = qobject_cast<QProgressBar*>(downloadList->itemWidget(item, 2));
        if (bar) bar->setValue(100);
    }
    
    void onFailed(int id, const QString& error) {
        auto* item = m_items.value(id);
        if (!item) return;
        
        item->setText(3, "Error: " + error);
        item->setForeground(3, QBrush(QColor("#e74c3c")));
    }
    
    static QString formatSize(qint64 bytes) {
        if (bytes < 0) return "?";
        if (bytes < 1024) return QString("%1 B").arg(bytes);
        if (bytes < 1024*1024) return QString("%1 KB").arg(bytes/1024.0, 0, 'f', 1);
        return QString("%1 MB").arg(bytes/1024.0/1024, 0, 'f', 1);
    }
    
    DownloadManager* manager;
    QTreeWidget* downloadList;
    QLineEdit* urlEdit;
    QHash<int, QTreeWidgetItem*> m_items;
};
```

---

## สรุป Part 046

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QPromise<T> + QFuture<T> สำหรับ async operations
2. ✅ Promise chaining pattern ด้วย `then()`
3. ✅ Reactive Observable<T> system ด้วย templates
4. ✅ AsyncTaskQueue ด้วย multiple worker threads
5. ✅ Download Manager ด้วย QNetworkReply + progress

---

⬅️ [Part 045](part045.md) | ➡️ [Part 047: Professional Level — Qt Testing & CI](part047.md)
