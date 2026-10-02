# Part 025: Qt Concurrency & Multithreading (Advanced)

## ขั้นตอนที่ 341-355

---

## ขั้นตอนที่ 341: QThread พื้นฐาน

```cpp
#include <QThread>
#include <QMutex>
#include <QWaitCondition>
#include <QAtomicInt>
#include <QtConcurrent>
#include <QFuture>
#include <QFutureWatcher>
#include <QThreadPool>
#include <QRunnable>

// === Worker Thread ===
class DataProcessor : public QThread {
    Q_OBJECT
    
public:
    void setData(const QList<int>& data) { m_data = data; }
    void stop() { m_stop = true; }
    
protected:
    void run() override {
        emit started();
        
        QList<int> results;
        for (int i = 0; i < m_data.size() && !m_stop; i++) {
            // Heavy computation
            int val = m_data[i];
            for (int j = 0; j < 100000; j++) {
                val = (val * 31 + 17) % 1000000;
            }
            results.append(val);
            
            int progress = (i + 1) * 100 / m_data.size();
            emit progressChanged(progress);
            
            msleep(10); // Simulate work
        }
        
        emit finished(results);
    }
    
signals:
    void started();
    void progressChanged(int percent);
    void finished(const QList<int>& results);
    
private:
    QList<int> m_data;
    bool m_stop = false;
};

// === QObject Worker (recommended pattern) ===
class Worker : public QObject {
    Q_OBJECT
    
public slots:
    void process(const QList<int>& data) {
        QList<int> results;
        
        for (int i = 0; i < data.size(); i++) {
            if (m_abort) break;
            
            results.append(data[i] * data[i]); // Square each element
            
            int pct = (i + 1) * 100 / data.size();
            emit progress(pct);
        }
        
        emit done(results);
    }
    
    void abort() { m_abort = true; }
    
signals:
    void progress(int percent);
    void done(const QList<int>& results);
    
private:
    bool m_abort = false;
};

class MainApp : public QMainWindow {
    Q_OBJECT
    
public:
    MainApp() {
        auto* worker = new Worker();
        auto* thread = new QThread(this);
        worker->moveToThread(thread);
        
        // When thread starts, it just runs an event loop
        thread->start();
        
        // Connect signals across threads (Qt::QueuedConnection is auto)
        connect(this, &MainApp::startWork, worker, &Worker::process);
        connect(worker, &Worker::progress, this, &MainApp::onProgress);
        connect(worker, &Worker::done, this, &MainApp::onDone);
        
        // Cleanup
        connect(thread, &QThread::finished, worker, &QObject::deleteLater);
        connect(this, &MainApp::destroyed, [thread, worker]() {
            worker->abort();
            thread->quit();
            thread->wait();
        });
    }
    
    void doWork() {
        QList<int> data;
        for (int i = 0; i < 1000; i++) data << i;
        emit startWork(data);
    }
    
signals:
    void startWork(const QList<int>& data);
    
private slots:
    void onProgress(int pct) { statusBar()->showMessage(QString("Processing: %1%").arg(pct)); }
    void onDone(const QList<int>& r) { qDebug() << "Done, results:" << r.size(); }
};
```

---

## ขั้นตอนที่ 342: QtConcurrent

```cpp
#include <QtConcurrent/QtConcurrent>

// === map: แปลงทุก element ===
void concurrentMap() {
    QList<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // map + ได้ future กลับมา
    QFuture<int> future = QtConcurrent::mapped(numbers, [](int n) {
        return n * n;
    });
    
    // Wait and get results
    future.waitForFinished();
    QList<int> squares = future.results();
    qDebug() << squares;  // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    
    // mapInPlace: แก้ไข in-place
    QtConcurrent::blockingMap(numbers, [](int& n) { n *= 2; });
    qDebug() << numbers; // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
}

// === filter: กรอง elements ===
void concurrentFilter() {
    QList<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Filter even numbers
    QFuture<int> future = QtConcurrent::filtered(numbers, [](int n) {
        return n % 2 == 0;
    });
    
    QList<int> evens = future.results(); // [2, 4, 6, 8, 10]
    qDebug() << evens;
}

// === reduce: รวม elements ===
void concurrentReduce() {
    QList<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // Sum all using reduce
    QFuture<int> future = QtConcurrent::mappedReduced(
        numbers,
        [](int n) { return n; },              // map
        [](int& result, int val) { result += val; }  // reduce
    );
    
    int sum = future.result(); // 55
    qDebug() << "Sum:" << sum;
}

// === run: รัน function ใน thread ===
void concurrentRun() {
    // Run a function in thread pool
    QFuture<QString> future = QtConcurrent::run([](const QString& msg, int count) {
        QString result;
        for (int i = 0; i < count; i++) {
            result += msg + "\n";
            QThread::msleep(100);
        }
        return result;
    }, QString("Hello"), 5);
    
    // Non-blocking wait with watcher
    auto* watcher = new QFutureWatcher<QString>();
    QObject::connect(watcher, &QFutureWatcher<QString>::finished, [watcher]() {
        qDebug() << "Result:" << watcher->result();
        watcher->deleteLater();
    });
    watcher->setFuture(future);
}
```

---

## ขั้นตอนที่ 343: QFutureWatcher & Progress

```cpp
class ImageProcessor : public QObject {
    Q_OBJECT
    
public:
    void processImages(const QStringList& paths) {
        // Process images concurrently
        QFuture<QImage> future = QtConcurrent::mapped(paths, [](const QString& path) {
            QImage img(path);
            // Apply grayscale filter
            for (int y = 0; y < img.height(); y++) {
                for (int x = 0; x < img.width(); x++) {
                    QColor c = img.pixelColor(x, y);
                    int gray = (c.red() + c.green() + c.blue()) / 3;
                    img.setPixelColor(x, y, QColor(gray, gray, gray));
                }
            }
            return img;
        });
        
        watcher = new QFutureWatcher<QImage>(this);
        
        connect(watcher, &QFutureWatcher<QImage>::progressValueChanged,
                this, &ImageProcessor::onProgress);
        
        connect(watcher, &QFutureWatcher<QImage>::resultReadyAt, [this](int idx) {
            QImage img = watcher->resultAt(idx);
            emit imageReady(idx, img);
        });
        
        connect(watcher, &QFutureWatcher<QImage>::finished, [this]() {
            emit allDone();
            watcher->deleteLater();
        });
        
        watcher->setFuture(future);
    }
    
    void cancel() {
        if (watcher) watcher->cancel();
    }
    
signals:
    void onProgress(int value);
    void imageReady(int idx, const QImage& img);
    void allDone();
    
private:
    QFutureWatcher<QImage>* watcher = nullptr;
};
```

---

## ขั้นตอนที่ 344: Thread-Safe Queue

```cpp
#include <QMutex>
#include <QWaitCondition>
#include <QQueue>

template<typename T>
class ThreadSafeQueue {
public:
    void enqueue(const T& item) {
        QMutexLocker lock(&m_mutex);
        m_queue.enqueue(item);
        m_condition.wakeOne();
    }
    
    T dequeue(int timeoutMs = -1) {
        QMutexLocker lock(&m_mutex);
        
        if (m_queue.isEmpty()) {
            if (timeoutMs < 0) {
                m_condition.wait(&m_mutex);
            } else if (!m_condition.wait(&m_mutex, timeoutMs)) {
                throw std::runtime_error("Queue timeout");
            }
        }
        
        return m_queue.dequeue();
    }
    
    bool tryDequeue(T& item) {
        QMutexLocker lock(&m_mutex);
        if (m_queue.isEmpty()) return false;
        item = m_queue.dequeue();
        return true;
    }
    
    int size() const {
        QMutexLocker lock(&m_mutex);
        return m_queue.size();
    }
    
    bool isEmpty() const {
        QMutexLocker lock(&m_mutex);
        return m_queue.isEmpty();
    }
    
    void close() {
        QMutexLocker lock(&m_mutex);
        m_closed = true;
        m_condition.wakeAll();
    }
    
private:
    mutable QMutex m_mutex;
    QWaitCondition m_condition;
    QQueue<T> m_queue;
    bool m_closed = false;
};

// Producer-Consumer Pattern
class ProducerConsumer : public QObject {
    Q_OBJECT
    
public:
    void start() {
        // Producer thread
        QtConcurrent::run([this]() {
            for (int i = 0; i < 100; i++) {
                QString task = QString("Task %1").arg(i);
                queue.enqueue(task);
                qDebug() << "Produced:" << task;
                QThread::msleep(50);
            }
            queue.close();
        });
        
        // Multiple consumers
        for (int i = 0; i < 3; i++) {
            int consumerId = i;
            QtConcurrent::run([this, consumerId]() {
                while (true) {
                    try {
                        QString task = queue.dequeue(1000);
                        qDebug() << "Consumer" << consumerId << "processing:" << task;
                        QThread::msleep(150); // Simulate work
                    } catch (...) {
                        break; // Queue closed
                    }
                }
            });
        }
    }
    
private:
    ThreadSafeQueue<QString> queue;
};
```

---

## ขั้นตอนที่ 345: QRunnable & QThreadPool

```cpp
class HeavyTask : public QRunnable {
public:
    HeavyTask(int id, QObject* receiver) : m_id(id), m_receiver(receiver) {
        setAutoDelete(true);
    }
    
    void run() override {
        // Simulate heavy computation
        long long result = 0;
        for (long long i = 0; i < 1000000; i++) {
            result += i * m_id;
        }
        
        // Send result back to main thread
        QMetaObject::invokeMethod(m_receiver, "onTaskComplete",
            Qt::QueuedConnection,
            Q_ARG(int, m_id),
            Q_ARG(long long, result));
    }
    
private:
    int m_id;
    QObject* m_receiver;
};

class TaskManager : public QObject {
    Q_OBJECT
    
public:
    TaskManager() {
        QThreadPool::globalInstance()->setMaxThreadCount(4);
    }
    
    void runTasks(int count) {
        m_completed = 0;
        m_total = count;
        
        for (int i = 0; i < count; i++) {
            auto* task = new HeavyTask(i, this);
            QThreadPool::globalInstance()->start(task);
        }
    }
    
    Q_INVOKABLE void onTaskComplete(int id, long long result) {
        m_completed++;
        qDebug() << "Task" << id << "done, result:" << result;
        
        int pct = m_completed * 100 / m_total;
        emit progress(pct);
        
        if (m_completed == m_total) {
            emit allTasksDone();
        }
    }
    
signals:
    void progress(int percent);
    void allTasksDone();
    
private:
    int m_completed = 0;
    int m_total = 0;
};
```

---

## ขั้นตอนที่ 346-355: Progress Dialog with Concurrent Operations

```cpp
class ProgressWindow : public QDialog {
    Q_OBJECT
    
public:
    ProgressWindow(QWidget* parent = nullptr) : QDialog(parent) {
        setWindowTitle("Processing...");
        setFixedSize(400, 200);
        setWindowFlags(windowFlags() & ~Qt::WindowCloseButtonHint);
        
        auto* layout = new QVBoxLayout(this);
        
        titleLabel = new QLabel("Processing files...");
        titleLabel->setAlignment(Qt::AlignCenter);
        titleLabel->setStyleSheet("font-size: 16px; font-weight: bold;");
        
        progressBar = new QProgressBar();
        progressBar->setRange(0, 100);
        progressBar->setStyleSheet(R"(
            QProgressBar {
                border: 2px solid #bbb;
                border-radius: 5px;
                text-align: center;
                height: 25px;
            }
            QProgressBar::chunk {
                background: qlineargradient(x1:0, y1:0, x2:1, y2:0,
                    stop:0 #3498db, stop:1 #2ecc71);
                border-radius: 3px;
            }
        )");
        
        statusLabel = new QLabel("Starting...");
        statusLabel->setAlignment(Qt::AlignCenter);
        
        cancelBtn = new QPushButton("ยกเลิก");
        cancelBtn->setStyleSheet("background: #e74c3c; color: white; padding: 6px 20px;");
        
        layout->addWidget(titleLabel);
        layout->addSpacing(10);
        layout->addWidget(progressBar);
        layout->addWidget(statusLabel);
        layout->addStretch();
        layout->addWidget(cancelBtn, 0, Qt::AlignCenter);
        
        connect(cancelBtn, &QPushButton::clicked, this, &ProgressWindow::cancelled);
    }
    
    void setTitle(const QString& t) { titleLabel->setText(t); }
    
public slots:
    void setProgress(int pct) { progressBar->setValue(pct); }
    void setStatus(const QString& s) { statusLabel->setText(s); }
    
signals:
    void cancelled();
    
private:
    QLabel* titleLabel;
    QProgressBar* progressBar;
    QLabel* statusLabel;
    QPushButton* cancelBtn;
};

// === Concurrent file processor ===
class FileBatchProcessor : public QObject {
    Q_OBJECT
    
public:
    struct FileResult {
        QString path;
        qint64 size;
        int lineCount;
        bool success;
        QString error;
    };
    
    void process(const QStringList& files) {
        m_watcher = new QFutureWatcher<FileResult>(this);
        
        QFuture<FileResult> future = QtConcurrent::mapped(files, 
            [](const QString& path) -> FileResult {
                FileResult result;
                result.path = path;
                result.success = false;
                
                QFile file(path);
                if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
                    result.error = "Cannot open: " + file.errorString();
                    return result;
                }
                
                result.size = file.size();
                result.lineCount = 0;
                
                QTextStream stream(&file);
                while (!stream.atEnd()) {
                    stream.readLine();
                    result.lineCount++;
                }
                
                result.success = true;
                return result;
            }
        );
        
        connect(m_watcher, &QFutureWatcher<FileResult>::progressValueChanged,
                this, &FileBatchProcessor::onProgress);
        connect(m_watcher, &QFutureWatcher<FileResult>::finished,
                this, &FileBatchProcessor::onFinished);
        
        m_watcher->setFuture(future);
        m_progressDlg = new ProgressWindow();
        m_progressDlg->setTitle("กำลังประมวลผลไฟล์...");
        m_progressDlg->setProgressRange(0, files.size());
        
        connect(m_progressDlg, &ProgressWindow::cancelled, [this]() {
            m_watcher->cancel();
        });
        
        m_progressDlg->show();
    }
    
signals:
    void finished(const QList<FileResult>& results);
    
private slots:
    void onProgress(int val) {
        if (m_progressDlg) {
            m_progressDlg->setProgress(val);
            m_progressDlg->setStatus(QString("ประมวลผลแล้ว %1/%2 ไฟล์")
                .arg(val).arg(m_watcher->progressMaximum()));
        }
    }
    
    void onFinished() {
        if (m_progressDlg) {
            m_progressDlg->hide();
            m_progressDlg->deleteLater();
            m_progressDlg = nullptr;
        }
        
        QList<FileResult> results;
        for (int i = 0; i < m_watcher->resultCount(); i++) {
            results.append(m_watcher->resultAt(i));
        }
        
        emit finished(results);
        m_watcher->deleteLater();
        m_watcher = nullptr;
    }
    
private:
    QFutureWatcher<FileResult>* m_watcher = nullptr;
    ProgressWindow* m_progressDlg = nullptr;
};
```

---

## สรุป Part 025

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QThread (subclass และ worker patterns)
2. ✅ QtConcurrent::mapped/filtered/reduced/run
3. ✅ QFutureWatcher สำหรับ progress tracking
4. ✅ Thread-safe queue ด้วย QMutex + QWaitCondition
5. ✅ QRunnable + QThreadPool
6. ✅ Progress dialog + concurrent file processing

---

⬅️ [Part 024](part024.md) | ➡️ [Part 026: Qt WebEngine](part026.md)
