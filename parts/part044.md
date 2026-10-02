# Part 044: Real-World Project — File Manager

## ขั้นตอนที่ 626-640

---

## ขั้นตอนที่ 626: File Manager Overview

```
FileManager Features:
  - Dual-pane view (like Total Commander)
  - QFileSystemModel + QTreeView + QListView
  - Drag & Drop copy/move
  - Context menu operations (copy, move, rename, delete)
  - Quick search / filter
  - Breadcrumb navigation
  - File preview panel
  - Bookmark manager
```

---

## ขั้นตอนที่ 627: File System Model

```cpp
#include <QFileSystemModel>
#include <QTreeView>
#include <QListView>
#include <QFileInfo>
#include <QDirIterator>
#include <QMimeData>

// === Custom File System Model ===
class FileModel : public QFileSystemModel {
    Q_OBJECT
    
public:
    explicit FileModel(QObject* parent = nullptr) : QFileSystemModel(parent) {
        setReadOnly(false);
        setFilter(QDir::AllEntries | QDir::NoDot | QDir::Hidden);
    }
    
    // Extra column: file description
    int columnCount(const QModelIndex& parent) const override {
        return QFileSystemModel::columnCount(parent) + 1;
    }
    
    QVariant headerData(int section, Qt::Orientation orientation, int role) const override {
        if (orientation == Qt::Horizontal && role == Qt::DisplayRole) {
            if (section == 4) return "Type";
        }
        return QFileSystemModel::headerData(section, orientation, role);
    }
    
    QVariant data(const QModelIndex& index, int role) const override {
        if (index.column() == 4 && role == Qt::DisplayRole) {
            QFileInfo info = fileInfo(index.siblingAtColumn(0));
            return fileTypeDescription(info);
        }
        return QFileSystemModel::data(index, role);
    }
    
private:
    static QString fileTypeDescription(const QFileInfo& info) {
        if (info.isDir())          return "Folder";
        if (info.isSymLink())      return "Symlink";
        
        static const QMap<QString, QString> types = {
            {"cpp", "C++ Source"},    {"h", "C++ Header"},
            {"py", "Python Script"},  {"js", "JavaScript"},
            {"ts", "TypeScript"},     {"html", "HTML Document"},
            {"css", "Stylesheet"},    {"json", "JSON File"},
            {"xml", "XML File"},      {"yaml", "YAML File"},
            {"md", "Markdown"},       {"txt", "Text File"},
            {"pdf", "PDF Document"},  {"png", "PNG Image"},
            {"jpg", "JPEG Image"},    {"svg", "SVG Image"},
            {"zip", "ZIP Archive"},   {"tar", "TAR Archive"},
            {"gz", "GZ Archive"},     {"mp3", "MP3 Audio"},
            {"mp4", "MP4 Video"},     {"exe", "Executable"},
        };
        
        QString ext = info.suffix().toLower();
        return types.value(ext, ext.isEmpty() ? "File" : ext.toUpper() + " File");
    }
};
```

---

## ขั้นตอนที่ 628: File Pane Widget

```cpp
class FilePaneWidget : public QWidget {
    Q_OBJECT
    Q_PROPERTY(QString currentPath READ currentPath WRITE setCurrentPath NOTIFY pathChanged)
    
public:
    enum ViewMode { List, Detail, Icon };
    
    explicit FilePaneWidget(QWidget* parent = nullptr) : QWidget(parent) {
        model = new FileModel(this);
        model->setRootPath(QDir::homePath());
        
        setupUi();
        setCurrentPath(QDir::homePath());
    }
    
    QString currentPath() const { return m_currentPath; }
    
    void setCurrentPath(const QString& path) {
        if (!QDir(path).exists()) return;
        
        m_currentPath = path;
        QModelIndex idx = model->setRootPath(path);
        
        if (viewMode == Detail) {
            detailView->setRootIndex(idx);
        } else {
            iconView->setRootIndex(idx);
        }
        
        breadcrumb->setPath(path);
        emit pathChanged(path);
    }
    
    QStringList selectedPaths() const {
        QStringList result;
        QAbstractItemView* view = currentView();
        
        for (const QModelIndex& idx : view->selectionModel()->selectedIndexes()) {
            if (idx.column() == 0) {
                result << model->filePath(idx);
            }
        }
        return result;
    }
    
signals:
    void pathChanged(const QString& path);
    void fileActivated(const QString& path);
    void selectionChanged(const QStringList& paths);
    
private:
    void setupUi() {
        auto* layout = new QVBoxLayout(this);
        layout->setSpacing(0);
        layout->setContentsMargins(0, 0, 0, 0);
        
        // Toolbar
        auto* toolbar = new QHBoxLayout();
        toolbar->setContentsMargins(4, 4, 4, 4);
        
        backBtn = new QToolButton();
        backBtn->setIcon(QIcon::fromTheme("go-previous"));
        backBtn->setToolTip("Back");
        
        forwardBtn = new QToolButton();
        forwardBtn->setIcon(QIcon::fromTheme("go-next"));
        forwardBtn->setToolTip("Forward");
        
        upBtn = new QToolButton();
        upBtn->setIcon(QIcon::fromTheme("go-up"));
        upBtn->setToolTip("Up");
        
        homeBtn = new QToolButton();
        homeBtn->setIcon(QIcon::fromTheme("go-home"));
        homeBtn->setToolTip("Home");
        
        filterEdit = new QLineEdit();
        filterEdit->setPlaceholderText("Filter...");
        filterEdit->setMaximumWidth(180);
        
        toolbar->addWidget(backBtn);
        toolbar->addWidget(forwardBtn);
        toolbar->addWidget(upBtn);
        toolbar->addWidget(homeBtn);
        toolbar->addStretch();
        toolbar->addWidget(filterEdit);
        
        // Breadcrumb
        breadcrumb = new BreadcrumbBar(this);
        
        // Views
        detailView = new QTreeView();
        detailView->setModel(model);
        detailView->setSelectionMode(QAbstractItemView::ExtendedSelection);
        detailView->setDragEnabled(true);
        detailView->setAcceptDrops(true);
        detailView->setDropIndicatorShown(true);
        detailView->setDragDropMode(QAbstractItemView::DragDrop);
        detailView->setSortingEnabled(true);
        detailView->hideColumn(2); // Hide type column from model
        detailView->header()->setSectionResizeMode(0, QHeaderView::Stretch);
        detailView->setAlternatingRowColors(true);
        
        iconView = new QListView();
        iconView->setModel(model);
        iconView->setViewMode(QListView::IconMode);
        iconView->setIconSize(QSize(64, 64));
        iconView->setGridSize(QSize(96, 96));
        iconView->setWordWrap(true);
        iconView->setSelectionMode(QAbstractItemView::ExtendedSelection);
        iconView->setDragEnabled(true);
        iconView->setAcceptDrops(true);
        iconView->hide();
        
        // Status bar
        statusLabel = new QLabel();
        statusLabel->setStyleSheet("padding: 4px; font-size: 12px; color: #666;");
        
        layout->addLayout(toolbar);
        layout->addWidget(breadcrumb);
        layout->addWidget(detailView, 1);
        layout->addWidget(iconView, 1);
        layout->addWidget(statusLabel);
        
        // Connections
        connect(backBtn, &QToolButton::clicked, [this]() {
            if (!m_history.isEmpty() && m_historyIdx > 0) {
                setCurrentPath(m_history[--m_historyIdx]);
            }
        });
        
        connect(upBtn, &QToolButton::clicked, [this]() {
            QDir dir(m_currentPath);
            if (dir.cdUp()) setCurrentPath(dir.absolutePath());
        });
        
        connect(homeBtn, &QToolButton::clicked, [this]() {
            setCurrentPath(QDir::homePath());
        });
        
        connect(breadcrumb, &BreadcrumbBar::pathClicked, this, &FilePaneWidget::setCurrentPath);
        
        connect(filterEdit, &QLineEdit::textChanged, [this](const QString& text) {
            model->setNameFilters(text.isEmpty() ? QStringList() : QStringList{"*" + text + "*"});
            model->setNameFilterDisables(false);
        });
        
        auto activateSlot = [this](const QModelIndex& idx) {
            QString path = model->filePath(idx);
            if (model->isDir(idx)) {
                navigate(path);
            } else {
                emit fileActivated(path);
            }
        };
        
        connect(detailView, &QTreeView::activated, activateSlot);
        connect(iconView,   &QListView::activated, activateSlot);
        
        connect(detailView->selectionModel(), &QItemSelectionModel::selectionChanged,
                [this]() { updateStatus(); emit selectionChanged(selectedPaths()); });
    }
    
    void navigate(const QString& path) {
        // Update history
        if (m_historyIdx < m_history.size() - 1) {
            m_history.erase(m_history.begin() + m_historyIdx + 1, m_history.end());
        }
        
        if (m_history.isEmpty() || m_history.last() != path) {
            m_history.append(path);
            m_historyIdx = m_history.size() - 1;
        }
        
        setCurrentPath(path);
        backBtn->setEnabled(m_historyIdx > 0);
        forwardBtn->setEnabled(m_historyIdx < m_history.size() - 1);
    }
    
    void updateStatus() {
        QStringList sel = selectedPaths();
        if (sel.isEmpty()) {
            int count = model->rowCount(model->index(m_currentPath));
            statusLabel->setText(QString("%1 items").arg(count));
        } else {
            qint64 total = 0;
            for (const QString& p : sel) total += QFileInfo(p).size();
            statusLabel->setText(QString("%1 selected | %2")
                .arg(sel.size()).arg(formatSize(total)));
        }
    }
    
    static QString formatSize(qint64 bytes) {
        if (bytes < 1024) return QString("%1 B").arg(bytes);
        if (bytes < 1024*1024) return QString("%1 KB").arg(bytes/1024.0, 0, 'f', 1);
        if (bytes < 1024LL*1024*1024) return QString("%1 MB").arg(bytes/1024.0/1024.0, 0, 'f', 1);
        return QString("%1 GB").arg(bytes/1024.0/1024.0/1024.0, 0, 'f', 2);
    }
    
    QAbstractItemView* currentView() const {
        return (viewMode == Detail) ? static_cast<QAbstractItemView*>(detailView)
                                    : static_cast<QAbstractItemView*>(iconView);
    }
    
    FileModel* model;
    QTreeView* detailView;
    QListView* iconView;
    QToolButton* backBtn;
    QToolButton* forwardBtn;
    QToolButton* upBtn;
    QToolButton* homeBtn;
    QLineEdit* filterEdit;
    BreadcrumbBar* breadcrumb;
    QLabel* statusLabel;
    
    QString m_currentPath;
    QStringList m_history;
    int m_historyIdx = -1;
    ViewMode viewMode = Detail;
};
```

---

## ขั้นตอนที่ 629: Breadcrumb Navigation Bar

```cpp
class BreadcrumbBar : public QWidget {
    Q_OBJECT
    
public:
    explicit BreadcrumbBar(QWidget* parent = nullptr) : QWidget(parent) {
        m_layout = new QHBoxLayout(this);
        m_layout->setContentsMargins(4, 2, 4, 2);
        m_layout->setSpacing(0);
        setStyleSheet("background: white; border-bottom: 1px solid #ddd;");
    }
    
    void setPath(const QString& path) {
        // Clear
        while (m_layout->count() > 0) {
            auto* item = m_layout->takeAt(0);
            delete item->widget();
            delete item;
        }
        
        QStringList parts;
        
#ifdef Q_OS_WIN
        // Windows: C:\Users\...
        QDir dir(path);
        parts << dir.absolutePath().split('/', Qt::SkipEmptyParts);
#else
        // Unix: /home/user/...
        parts << "/";
        parts << path.split('/', Qt::SkipEmptyParts);
#endif
        
        QString accumulated = "";
        for (int i = 0; i < parts.size(); ++i) {
            if (i > 0) {
                auto* sep = new QLabel("›");
                sep->setStyleSheet("color: #aaa; padding: 0 4px;");
                m_layout->addWidget(sep);
            }
            
#ifdef Q_OS_WIN
            accumulated += (i == 0 ? "" : "/") + parts[i];
#else
            accumulated = (parts[i] == "/") ? "/" : accumulated + "/" + parts[i];
#endif
            
            auto* btn = new QPushButton(parts[i]);
            btn->setFlat(true);
            btn->setStyleSheet(
                "QPushButton { border: none; padding: 4px 6px; color: #2c3e50; border-radius: 3px; }"
                "QPushButton:hover { background: #e8f4fd; color: #2980b9; }");
            
            QString crumbPath = accumulated;
            connect(btn, &QPushButton::clicked, [this, crumbPath]() {
                emit pathClicked(crumbPath);
            });
            
            m_layout->addWidget(btn);
        }
        
        m_layout->addStretch();
    }
    
signals:
    void pathClicked(const QString& path);
    
private:
    QHBoxLayout* m_layout;
};
```

---

## ขั้นตอนที่ 630-640: File Manager Main Window

```cpp
class FileManager : public QMainWindow {
    Q_OBJECT
    
public:
    FileManager(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("File Manager");
        setMinimumSize(1200, 700);
        
        setupUi();
        setupMenus();
        setupContextMenus();
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* layout = new QVBoxLayout(central);
        layout->setContentsMargins(0, 0, 0, 0);
        layout->setSpacing(0);
        setCentralWidget(central);
        
        // Toolbar
        auto* tb = addToolBar("Main");
        
        tb->addAction(QIcon::fromTheme("folder-new"), "New Folder", [this]() {
            createFolder(currentPane()->currentPath());
        });
        tb->addAction(QIcon::fromTheme("document-new"), "New File", [this]() {
            createFile(currentPane()->currentPath());
        });
        tb->addSeparator();
        tb->addAction(QIcon::fromTheme("edit-copy"), "Copy", [this]() {
            copySelected();
        });
        tb->addAction(QIcon::fromTheme("edit-cut"), "Move", [this]() {
            moveSelected();
        });
        tb->addAction(QIcon::fromTheme("edit-delete"), "Delete", [this]() {
            deleteSelected();
        });
        tb->addSeparator();
        
        auto* swapBtn = tb->addAction("⇄ Swap Panes");
        connect(swapBtn, &QAction::triggered, [this]() {
            QString p1 = leftPane->currentPath();
            QString p2 = rightPane->currentPath();
            leftPane->setCurrentPath(p2);
            rightPane->setCurrentPath(p1);
        });
        
        // Dual pane
        splitter = new QSplitter(Qt::Horizontal);
        
        leftPane = new FilePaneWidget();
        rightPane = new FilePaneWidget();
        rightPane->setCurrentPath(QDir::homePath() + "/Documents");
        
        leftFrame = new QGroupBox("Left");
        leftFrame->setFlat(true);
        auto* lf = new QVBoxLayout(leftFrame);
        lf->setContentsMargins(0, 0, 0, 0);
        lf->addWidget(leftPane);
        
        rightFrame = new QGroupBox("Right");
        rightFrame->setFlat(true);
        auto* rf = new QVBoxLayout(rightFrame);
        rf->setContentsMargins(0, 0, 0, 0);
        rf->addWidget(rightPane);
        
        splitter->addWidget(leftFrame);
        splitter->addWidget(rightFrame);
        splitter->setSizes({600, 600});
        
        // Preview panel
        previewPanel = new QWidget();
        previewPanel->setMaximumHeight(200);
        previewPanel->setStyleSheet("background: #f5f5f5; border-top: 1px solid #ddd;");
        auto* previewLayout = new QHBoxLayout(previewPanel);
        
        previewIcon = new QLabel();
        previewIcon->setFixedSize(64, 64);
        previewInfo = new QLabel("Select a file to preview");
        previewInfo->setWordWrap(true);
        
        previewLayout->addWidget(previewIcon);
        previewLayout->addWidget(previewInfo, 1);
        
        layout->addWidget(splitter, 1);
        layout->addWidget(previewPanel);
        
        // Focus tracking
        connect(leftPane, &FilePaneWidget::selectionChanged, [this](const QStringList& paths) {
            m_activePane = leftPane;
            leftFrame->setTitle("Left (Active)");
            rightFrame->setTitle("Right");
            if (!paths.isEmpty()) showPreview(paths.first());
        });
        
        connect(rightPane, &FilePaneWidget::selectionChanged, [this](const QStringList& paths) {
            m_activePane = rightPane;
            rightFrame->setTitle("Right (Active)");
            leftFrame->setTitle("Left");
            if (!paths.isEmpty()) showPreview(paths.first());
        });
        
        connect(leftPane, &FilePaneWidget::fileActivated, [this](const QString& path) {
            QDesktopServices::openUrl(QUrl::fromLocalFile(path));
        });
        
        connect(rightPane, &FilePaneWidget::fileActivated, [this](const QString& path) {
            QDesktopServices::openUrl(QUrl::fromLocalFile(path));
        });
    }
    
    void setupMenus() {
        auto* fileMenu = menuBar()->addMenu("File");
        fileMenu->addAction("New Folder", QKeySequence("Ctrl+Shift+N"), [this]() {
            createFolder(currentPane()->currentPath());
        });
        fileMenu->addAction("New File", QKeySequence("Ctrl+N"), [this]() {
            createFile(currentPane()->currentPath());
        });
        fileMenu->addSeparator();
        fileMenu->addAction("Copy", QKeySequence::Copy, this, &FileManager::copySelected);
        fileMenu->addAction("Move", QKeySequence::Cut, this, &FileManager::moveSelected);
        fileMenu->addAction("Delete", QKeySequence::Delete, this, &FileManager::deleteSelected);
    }
    
    void setupContextMenus() {
        // Same as toolbar actions but attached to pane context menu
    }
    
    void showPreview(const QString& path) {
        QFileInfo info(path);
        
        QIcon icon = QFileIconProvider().icon(info);
        previewIcon->setPixmap(icon.pixmap(64, 64));
        
        QString details = QString(
            "<b>%1</b><br>"
            "Size: %2<br>"
            "Modified: %3<br>"
            "Permissions: %4")
            .arg(info.fileName())
            .arg(formatSize(info.size()))
            .arg(info.lastModified().toString("dd/MM/yyyy hh:mm"))
            .arg(info.isReadable() ? "R" : "")
            + (info.isWritable() ? "W" : "")
            + (info.isExecutable() ? "X" : "");
        
        previewInfo->setText(details);
    }
    
    FilePaneWidget* currentPane() const {
        return m_activePane ? m_activePane : leftPane;
    }
    
    void createFolder(const QString& parent) {
        bool ok;
        QString name = QInputDialog::getText(this, "New Folder", "Folder name:", 
            QLineEdit::Normal, "New Folder", &ok);
        if (!ok || name.isEmpty()) return;
        
        QDir dir(parent);
        if (!dir.mkdir(name)) {
            QMessageBox::warning(this, "Error", "Cannot create folder: " + name);
        }
    }
    
    void createFile(const QString& parent) {
        bool ok;
        QString name = QInputDialog::getText(this, "New File", "File name:",
            QLineEdit::Normal, "new_file.txt", &ok);
        if (!ok || name.isEmpty()) return;
        
        QFile f(parent + "/" + name);
        if (!f.open(QIODevice::WriteOnly)) {
            QMessageBox::warning(this, "Error", "Cannot create file: " + name);
        }
    }
    
    void copySelected() {
        QStringList src = currentPane()->selectedPaths();
        if (src.isEmpty()) return;
        
        FilePaneWidget* dest = (currentPane() == leftPane) ? rightPane : leftPane;
        
        for (const QString& path : src) {
            QFileInfo info(path);
            QString destPath = dest->currentPath() + "/" + info.fileName();
            
            if (info.isDir()) {
                copyDirectory(path, destPath);
            } else {
                QFile::copy(path, destPath);
            }
        }
    }
    
    void moveSelected() {
        QStringList src = currentPane()->selectedPaths();
        if (src.isEmpty()) return;
        
        FilePaneWidget* dest = (currentPane() == leftPane) ? rightPane : leftPane;
        
        for (const QString& path : src) {
            QFileInfo info(path);
            QString destPath = dest->currentPath() + "/" + info.fileName();
            QFile::rename(path, destPath);
        }
    }
    
    void deleteSelected() {
        QStringList paths = currentPane()->selectedPaths();
        if (paths.isEmpty()) return;
        
        int result = QMessageBox::question(this, "Delete",
            QString("Delete %1 item(s)? This cannot be undone.").arg(paths.size()),
            QMessageBox::Yes | QMessageBox::No, QMessageBox::No);
        
        if (result != QMessageBox::Yes) return;
        
        for (const QString& path : paths) {
            QFileInfo info(path);
            if (info.isDir()) {
                QDir(path).removeRecursively();
            } else {
                QFile::remove(path);
            }
        }
    }
    
    void copyDirectory(const QString& src, const QString& dest) {
        QDir srcDir(src);
        QDir().mkpath(dest);
        
        for (const QFileInfo& entry : srcDir.entryInfoList(QDir::AllEntries | QDir::NoDotAndDotDot)) {
            if (entry.isDir()) {
                copyDirectory(entry.absoluteFilePath(), dest + "/" + entry.fileName());
            } else {
                QFile::copy(entry.absoluteFilePath(), dest + "/" + entry.fileName());
            }
        }
    }
    
    static QString formatSize(qint64 bytes) {
        if (bytes < 1024) return QString("%1 B").arg(bytes);
        if (bytes < 1024*1024) return QString("%1 KB").arg(bytes/1024.0, 0, 'f', 1);
        if (bytes < 1024LL*1024*1024) return QString("%1 MB").arg(bytes/1024.0/1024.0, 0, 'f', 1);
        return QString("%1 GB").arg(bytes/1024.0/1024.0/1024.0, 0, 'f', 2);
    }
    
    QSplitter* splitter;
    FilePaneWidget* leftPane;
    FilePaneWidget* rightPane;
    FilePaneWidget* m_activePane = nullptr;
    QGroupBox* leftFrame;
    QGroupBox* rightFrame;
    QWidget* previewPanel;
    QLabel* previewIcon;
    QLabel* previewInfo;
};
```

---

## สรุป Part 044

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Custom QFileSystemModel ด้วย column เพิ่มเติม
2. ✅ FilePaneWidget ด้วย history navigation
3. ✅ BreadcrumbBar widget สำหรับ path navigation
4. ✅ Dual-pane File Manager ครบด้วย copy/move/delete
5. ✅ File preview panel

---

⬅️ [Part 043](part043.md) | ➡️ [Part 045: Professional-Level — Custom Widget Engine](part045.md)
