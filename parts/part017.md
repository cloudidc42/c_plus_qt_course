# Part 017: Qt File System & I/O

## ขั้นตอนที่ 221-235

---

## ขั้นตอนที่ 221: QFile & QDir

```cpp
#include <QFile>
#include <QDir>
#include <QFileInfo>
#include <QTextStream>
#include <QDataStream>
#include <QSettings>
#include <QTemporaryFile>
#include <QStandardPaths>

// === File Read/Write ===
class FileOperations {
public:
    // เขียนข้อความ
    static bool writeText(const QString& path, const QString& content) {
        QFile file(path);
        if (!file.open(QIODevice::WriteOnly | QIODevice::Text)) {
            qWarning() << "Cannot write:" << path << file.errorString();
            return false;
        }
        QTextStream out(&file);
        out.setEncoding(QStringConverter::Utf8);
        out << content;
        return true;
    }
    
    // อ่านข้อความทั้งไฟล์
    static QString readText(const QString& path) {
        QFile file(path);
        if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
            qWarning() << "Cannot read:" << path;
            return {};
        }
        QTextStream in(&file);
        in.setEncoding(QStringConverter::Utf8);
        return in.readAll();
    }
    
    // อ่านทีละบรรทัด
    static QStringList readLines(const QString& path) {
        QStringList lines;
        QFile file(path);
        if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) return lines;
        
        QTextStream in(&file);
        in.setEncoding(QStringConverter::Utf8);
        while (!in.atEnd()) {
            lines << in.readLine();
        }
        return lines;
    }
    
    // append ต่อท้าย
    static bool appendText(const QString& path, const QString& text) {
        QFile file(path);
        if (!file.open(QIODevice::Append | QIODevice::Text)) return false;
        QTextStream out(&file);
        out.setEncoding(QStringConverter::Utf8);
        out << text;
        return true;
    }
    
    // Copy/Move/Delete
    static bool copyFile(const QString& src, const QString& dst) {
        if (QFile::exists(dst)) QFile::remove(dst);
        return QFile::copy(src, dst);
    }
    
    static bool moveFile(const QString& src, const QString& dst) {
        if (QFile::exists(dst)) QFile::remove(dst);
        return QFile::rename(src, dst);
    }
    
    static bool deleteFile(const QString& path) {
        return QFile::remove(path);
    }
    
    // Binary read/write
    static bool writeBinary(const QString& path, const QByteArray& data) {
        QFile file(path);
        if (!file.open(QIODevice::WriteOnly)) return false;
        return file.write(data) == data.size();
    }
    
    static QByteArray readBinary(const QString& path) {
        QFile file(path);
        if (!file.open(QIODevice::ReadOnly)) return {};
        return file.readAll();
    }
};

// === QDir Operations ===
class DirectoryOperations {
public:
    static bool createDir(const QString& path) {
        return QDir().mkpath(path);
    }
    
    static bool removeDir(const QString& path) {
        QDir dir(path);
        return dir.removeRecursively();
    }
    
    static QStringList listFiles(const QString& path, 
                                  const QStringList& filters = {"*"}) {
        QDir dir(path);
        dir.setNameFilters(filters);
        dir.setFilter(QDir::Files | QDir::NoDotAndDotDot);
        return dir.entryList();
    }
    
    static QStringList listAll(const QString& path) {
        QDir dir(path);
        dir.setFilter(QDir::AllEntries | QDir::NoDotAndDotDot);
        return dir.entryList(QDir::DirsFirst | QDir::Name);
    }
    
    static void traverseTree(const QString& path, int depth = 0) {
        QDir dir(path);
        auto entries = dir.entryInfoList(QDir::AllEntries | QDir::NoDotAndDotDot);
        
        for (const QFileInfo& info : entries) {
            QString indent(depth * 2, ' ');
            if (info.isDir()) {
                qDebug() << indent + "📁 " + info.fileName();
                traverseTree(info.filePath(), depth + 1);
            } else {
                QString size = formatSize(info.size());
                qDebug() << indent + "📄 " + info.fileName() + " (" + size + ")";
            }
        }
    }
    
    static QString formatSize(qint64 bytes) {
        if (bytes < 1024) return QString("%1 B").arg(bytes);
        if (bytes < 1024*1024) return QString("%1 KB").arg(bytes/1024.0, 0, 'f', 1);
        if (bytes < 1024*1024*1024) return QString("%1 MB").arg(bytes/(1024.0*1024), 0, 'f', 1);
        return QString("%1 GB").arg(bytes/(1024.0*1024*1024), 0, 'f', 1);
    }
};
```

---

## ขั้นตอนที่ 222: QDataStream - Serialization

```cpp
#include <QDataStream>

struct Student {
    QString name;
    int age;
    double gpa;
    QStringList subjects;
    
    // Serialize
    friend QDataStream& operator<<(QDataStream& out, const Student& s) {
        out << s.name << s.age << s.gpa << s.subjects;
        return out;
    }
    
    // Deserialize
    friend QDataStream& operator>>(QDataStream& in, Student& s) {
        in >> s.name >> s.age >> s.gpa >> s.subjects;
        return in;
    }
    
    QString toString() const {
        return QString("Student{%1, age=%2, gpa=%3, subjects=%4}")
            .arg(name).arg(age).arg(gpa, 0, 'f', 2)
            .arg(subjects.join(", "));
    }
};

void serializationDemo() {
    // === Write ===
    QString path = "students.dat";
    
    QList<Student> students = {
        {"สมชาย", 20, 3.5, {"Math", "Physics", "CS"}},
        {"สมหญิง", 22, 3.8, {"Biology", "Chemistry"}},
        {"อนุชา", 21, 3.2, {"History", "Thai", "English"}}
    };
    
    {
        QFile file(path);
        file.open(QIODevice::WriteOnly);
        QDataStream out(&file);
        out.setVersion(QDataStream::Qt_6_0);
        
        // Write magic number and version
        out << (quint32)0xA0B0C0D0;
        out << (quint16)1;  // version
        
        // Write data
        out << students;
    }
    
    qDebug() << "Saved" << students.size() << "students";
    
    // === Read ===
    {
        QFile file(path);
        file.open(QIODevice::ReadOnly);
        QDataStream in(&file);
        in.setVersion(QDataStream::Qt_6_0);
        
        // Verify magic number
        quint32 magic;
        quint16 version;
        in >> magic >> version;
        
        if (magic != 0xA0B0C0D0) {
            qCritical() << "Invalid file format";
            return;
        }
        
        // Read data
        QList<Student> loaded;
        in >> loaded;
        
        qDebug() << "Loaded" << loaded.size() << "students:";
        for (const Student& s : loaded) {
            qDebug() << " -" << s.toString();
        }
    }
}
```

---

## ขั้นตอนที่ 223: QSettings - Application Settings

```cpp
#include <QSettings>

class AppSettings {
private:
    QSettings settings;
    
public:
    AppSettings() : settings("MyCompany", "MyApp") {}
    
    // === Window settings ===
    void saveWindowGeometry(QWidget* window) {
        settings.beginGroup("Window");
        settings.setValue("geometry", window->saveGeometry());
        settings.setValue("state", qobject_cast<QMainWindow*>(window) ?
            qobject_cast<QMainWindow*>(window)->saveState() : QByteArray());
        settings.endGroup();
    }
    
    void restoreWindowGeometry(QWidget* window) {
        settings.beginGroup("Window");
        window->restoreGeometry(settings.value("geometry").toByteArray());
        if (auto* mw = qobject_cast<QMainWindow*>(window)) {
            mw->restoreState(settings.value("state").toByteArray());
        }
        settings.endGroup();
    }
    
    // === Application preferences ===
    void setTheme(const QString& theme) {
        settings.setValue("Appearance/theme", theme);
    }
    
    QString theme() const {
        return settings.value("Appearance/theme", "light").toString();
    }
    
    void setFontSize(int size) {
        settings.setValue("Appearance/fontSize", size);
    }
    
    int fontSize() const {
        return settings.value("Appearance/fontSize", 12).toInt();
    }
    
    void setLanguage(const QString& lang) {
        settings.setValue("General/language", lang);
    }
    
    QString language() const {
        return settings.value("General/language", "th").toString();
    }
    
    void setAutoSave(bool enabled) {
        settings.setValue("Editor/autoSave", enabled);
    }
    
    bool autoSave() const {
        return settings.value("Editor/autoSave", true).toBool();
    }
    
    // === Recent files ===
    void addRecentFile(const QString& path) {
        QStringList recent = recentFiles();
        recent.removeAll(path);
        recent.prepend(path);
        while (recent.size() > 10) recent.removeLast();
        settings.setValue("RecentFiles/list", recent);
    }
    
    QStringList recentFiles() const {
        return settings.value("RecentFiles/list").toStringList();
    }
    
    void clearRecentFiles() {
        settings.remove("RecentFiles/list");
    }
    
    // === Shortcuts ===
    void setShortcut(const QString& action, const QKeySequence& key) {
        settings.setValue("Shortcuts/" + action, key.toString());
    }
    
    QKeySequence shortcut(const QString& action, const QKeySequence& defaultKey = {}) const {
        return QKeySequence(settings.value("Shortcuts/" + action, 
                                          defaultKey.toString()).toString());
    }
    
    // === Export/Import ===
    bool exportToFile(const QString& path) const {
        QFile file(path);
        if (!file.open(QIODevice::WriteOnly | QIODevice::Text)) return false;
        
        QTextStream out(&file);
        for (const QString& key : settings.allKeys()) {
            out << key << "=" << settings.value(key).toString() << "\n";
        }
        return true;
    }
};
```

---

## ขั้นตอนที่ 224-235: File Watcher

```cpp
#include <QFileSystemWatcher>
#include <QFileSystemModel>
#include <QTreeView>

class FileWatcher : public QObject {
    Q_OBJECT
    
private:
    QFileSystemWatcher* watcher;
    QMap<QString, QDateTime> lastModified;
    
public:
    FileWatcher(QObject* parent = nullptr) : QObject(parent) {
        watcher = new QFileSystemWatcher(this);
        
        connect(watcher, &QFileSystemWatcher::fileChanged, 
                this, &FileWatcher::onFileChanged);
        connect(watcher, &QFileSystemWatcher::directoryChanged,
                this, &FileWatcher::onDirectoryChanged);
    }
    
    void watchFile(const QString& path) {
        if (QFile::exists(path)) {
            watcher->addPath(path);
            lastModified[path] = QFileInfo(path).lastModified();
            qDebug() << "Watching file:" << path;
        }
    }
    
    void watchDirectory(const QString& path) {
        if (QDir(path).exists()) {
            watcher->addPath(path);
            qDebug() << "Watching dir:" << path;
        }
    }
    
    void unwatch(const QString& path) {
        watcher->removePath(path);
    }
    
    QStringList watchedFiles() const {
        return watcher->files();
    }
    
    QStringList watchedDirs() const {
        return watcher->directories();
    }
    
private slots:
    void onFileChanged(const QString& path) {
        QFileInfo info(path);
        
        if (!info.exists()) {
            emit fileDeleted(path);
            lastModified.remove(path);
            // Re-watch after recreation
            watcher->addPath(path);
        } else {
            QDateTime newTime = info.lastModified();
            if (newTime != lastModified.value(path)) {
                lastModified[path] = newTime;
                emit fileModified(path);
            }
        }
    }
    
    void onDirectoryChanged(const QString& path) {
        emit directoryChanged(path);
    }
    
signals:
    void fileModified(const QString& path);
    void fileDeleted(const QString& path);
    void directoryChanged(const QString& path);
};

// === File Explorer Widget ===
class FileExplorer : public QWidget {
    Q_OBJECT
    
public:
    FileExplorer(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("File Explorer");
        setMinimumSize(400, 500);
        
        model = new QFileSystemModel(this);
        model->setRootPath(QDir::homePath());
        model->setFilter(QDir::AllEntries | QDir::NoDotAndDotDot);
        
        treeView = new QTreeView();
        treeView->setModel(model);
        treeView->setRootIndex(model->index(QDir::homePath()));
        treeView->setColumnWidth(0, 200);
        treeView->setSortingEnabled(true);
        
        // Hide unnecessary columns
        treeView->hideColumn(2);  // Type
        
        auto* pathLabel = new QLabel(QDir::homePath());
        auto* upBtn = new QPushButton("↑ Up");
        auto* homeBtn = new QPushButton("Home");
        
        auto* navBar = new QHBoxLayout();
        navBar->addWidget(pathLabel, 1);
        navBar->addWidget(upBtn);
        navBar->addWidget(homeBtn);
        
        auto* layout = new QVBoxLayout(this);
        layout->addLayout(navBar);
        layout->addWidget(treeView);
        
        // Connections
        connect(treeView, &QTreeView::activated, [=](const QModelIndex& idx) {
            QFileInfo info = model->fileInfo(idx);
            pathLabel->setText(info.filePath());
            
            if (info.isDir()) {
                treeView->setRootIndex(idx);
                currentPath = info.filePath();
            } else {
                emit fileSelected(info.filePath());
            }
        });
        
        connect(upBtn, &QPushButton::clicked, [=]() {
            QDir dir(currentPath);
            if (dir.cdUp()) {
                currentPath = dir.absolutePath();
                treeView->setRootIndex(model->index(currentPath));
                pathLabel->setText(currentPath);
            }
        });
        
        connect(homeBtn, &QPushButton::clicked, [=]() {
            currentPath = QDir::homePath();
            treeView->setRootIndex(model->index(currentPath));
            pathLabel->setText(currentPath);
        });
        
        currentPath = QDir::homePath();
    }
    
private:
    QFileSystemModel* model;
    QTreeView* treeView;
    QString currentPath;
    
signals:
    void fileSelected(const QString& path);
};
```

---

## สรุป Part 017

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QFile - อ่าน/เขียนไฟล์
2. ✅ QDir - จัดการ directory
3. ✅ QDataStream - Binary serialization
4. ✅ QSettings - Application settings
5. ✅ QFileSystemWatcher - Monitor file changes
6. ✅ QFileSystemModel + QTreeView - File Explorer

---

⬅️ [Part 016](part016.md) | ➡️ [Part 018: Qt QML Introduction](part018.md)
