# Part 012: Qt MainWindow, Menus & Toolbars

## ขั้นตอนที่ 146-160

---

## ขั้นตอนที่ 146: QMainWindow Structure

```
QMainWindow Layout:
┌─────────────────────────────────────────┐
│             Menu Bar                    │
├─────────────────────────────────────────┤
│          Tool Bar(s)                    │
├────┬────────────────────────────┬───────┤
│ D  │                            │   D   │
│ o  │       Central Widget       │   o   │
│ c  │                            │   c   │
│ k  │                            │   k   │
├────┴────────────────────────────┴───────┤
│             Status Bar                  │
└─────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 147: สร้าง MainWindow

```cpp
// === mainwindow.h ===
#pragma once
#include <QMainWindow>
#include <QTextEdit>
#include <QLabel>

class MainWindow : public QMainWindow {
    Q_OBJECT
    
public:
    explicit MainWindow(QWidget* parent = nullptr);
    ~MainWindow() = default;
    
private slots:
    void newFile();
    void openFile();
    void saveFile();
    void saveFileAs();
    void printFile();
    void exit();
    
    void undo();
    void redo();
    void cut();
    void copy();
    void paste();
    void selectAll();
    void find();
    
    void zoomIn();
    void zoomOut();
    void toggleFullscreen();
    void showStatusBar(bool show);
    
    void about();
    void aboutQt();
    
    void documentModified();
    void cursorPositionChanged();
    
private:
    void createActions();
    void createMenus();
    void createToolBars();
    void createStatusBar();
    void createCentralWidget();
    
    void loadFile(const QString& filename);
    bool saveFile(const QString& filename);
    void setCurrentFile(const QString& filename);
    bool maybeSave();
    
    // Widgets
    QTextEdit* textEdit;
    QLabel* statusLabel;
    QLabel* positionLabel;
    QLabel* encodingLabel;
    
    // Actions
    QAction* newAct;
    QAction* openAct;
    QAction* saveAct;
    QAction* saveAsAct;
    QAction* printAct;
    QAction* exitAct;
    
    QAction* undoAct;
    QAction* redoAct;
    QAction* cutAct;
    QAction* copyAct;
    QAction* pasteAct;
    QAction* selectAllAct;
    QAction* findAct;
    
    QAction* zoomInAct;
    QAction* zoomOutAct;
    QAction* toggleFullscreenAct;
    QAction* showStatusBarAct;
    
    QAction* aboutAct;
    QAction* aboutQtAct;
    
    // Data
    QString currentFile;
    bool isModified;
};
```

```cpp
// === mainwindow.cpp ===
#include "mainwindow.h"
#include <QApplication>
#include <QMenuBar>
#include <QToolBar>
#include <QStatusBar>
#include <QFileDialog>
#include <QMessageBox>
#include <QTextStream>
#include <QCloseEvent>
#include <QPrintDialog>
#include <QPrinter>
#include <QInputDialog>
#include <QFont>
#include <QKeySequence>
#include <QIcon>
#include <QShortcut>

MainWindow::MainWindow(QWidget* parent) 
    : QMainWindow(parent), isModified(false) {
    
    setWindowTitle("Qt Text Editor");
    setMinimumSize(800, 600);
    
    createCentralWidget();
    createActions();
    createMenus();
    createToolBars();
    createStatusBar();
    
    // Connect TextEdit signals
    connect(textEdit, &QTextEdit::textChanged, this, &MainWindow::documentModified);
    connect(textEdit, &QTextEdit::cursorPositionChanged, this, &MainWindow::cursorPositionChanged);
    
    setCurrentFile("");
}

void MainWindow::createCentralWidget() {
    textEdit = new QTextEdit(this);
    textEdit->setFont(QFont("Consolas", 11));
    setCentralWidget(textEdit);
}

void MainWindow::createActions() {
    // File Actions
    newAct = new QAction(QIcon::fromTheme("document-new"), "&New", this);
    newAct->setShortcut(QKeySequence::New);
    newAct->setStatusTip("สร้างไฟล์ใหม่");
    connect(newAct, &QAction::triggered, this, &MainWindow::newFile);
    
    openAct = new QAction(QIcon::fromTheme("document-open"), "&Open...", this);
    openAct->setShortcut(QKeySequence::Open);
    openAct->setStatusTip("เปิดไฟล์");
    connect(openAct, &QAction::triggered, this, &MainWindow::openFile);
    
    saveAct = new QAction(QIcon::fromTheme("document-save"), "&Save", this);
    saveAct->setShortcut(QKeySequence::Save);
    saveAct->setStatusTip("บันทึกไฟล์");
    connect(saveAct, &QAction::triggered, this, &MainWindow::saveFile);
    
    saveAsAct = new QAction("Save &As...", this);
    saveAsAct->setShortcut(QKeySequence::SaveAs);
    connect(saveAsAct, &QAction::triggered, this, &MainWindow::saveFileAs);
    
    exitAct = new QAction("E&xit", this);
    exitAct->setShortcut(QKeySequence::Quit);
    connect(exitAct, &QAction::triggered, this, &MainWindow::exit);
    
    // Edit Actions
    undoAct = new QAction(QIcon::fromTheme("edit-undo"), "&Undo", this);
    undoAct->setShortcut(QKeySequence::Undo);
    connect(undoAct, &QAction::triggered, textEdit, &QTextEdit::undo);
    
    redoAct = new QAction(QIcon::fromTheme("edit-redo"), "&Redo", this);
    redoAct->setShortcut(QKeySequence::Redo);
    connect(redoAct, &QAction::triggered, textEdit, &QTextEdit::redo);
    
    cutAct = new QAction(QIcon::fromTheme("edit-cut"), "Cu&t", this);
    cutAct->setShortcut(QKeySequence::Cut);
    connect(cutAct, &QAction::triggered, textEdit, &QTextEdit::cut);
    
    copyAct = new QAction(QIcon::fromTheme("edit-copy"), "&Copy", this);
    copyAct->setShortcut(QKeySequence::Copy);
    connect(copyAct, &QAction::triggered, textEdit, &QTextEdit::copy);
    
    pasteAct = new QAction(QIcon::fromTheme("edit-paste"), "&Paste", this);
    pasteAct->setShortcut(QKeySequence::Paste);
    connect(pasteAct, &QAction::triggered, textEdit, &QTextEdit::paste);
    
    selectAllAct = new QAction("Select &All", this);
    selectAllAct->setShortcut(QKeySequence::SelectAll);
    connect(selectAllAct, &QAction::triggered, textEdit, &QTextEdit::selectAll);
    
    findAct = new QAction("&Find...", this);
    findAct->setShortcut(QKeySequence::Find);
    connect(findAct, &QAction::triggered, this, &MainWindow::find);
    
    // View Actions
    zoomInAct = new QAction("Zoom &In", this);
    zoomInAct->setShortcut(QKeySequence::ZoomIn);
    connect(zoomInAct, &QAction::triggered, this, &MainWindow::zoomIn);
    
    zoomOutAct = new QAction("Zoom &Out", this);
    zoomOutAct->setShortcut(QKeySequence::ZoomOut);
    connect(zoomOutAct, &QAction::triggered, this, &MainWindow::zoomOut);
    
    toggleFullscreenAct = new QAction("&Fullscreen", this);
    toggleFullscreenAct->setShortcut(Qt::Key_F11);
    toggleFullscreenAct->setCheckable(true);
    connect(toggleFullscreenAct, &QAction::triggered, this, &MainWindow::toggleFullscreen);
    
    showStatusBarAct = new QAction("Show &Status Bar", this);
    showStatusBarAct->setCheckable(true);
    showStatusBarAct->setChecked(true);
    connect(showStatusBarAct, &QAction::toggled, this, &MainWindow::showStatusBar);
    
    // Help Actions
    aboutAct = new QAction("&About", this);
    connect(aboutAct, &QAction::triggered, this, &MainWindow::about);
    
    aboutQtAct = new QAction("About &Qt", this);
    connect(aboutQtAct, &QAction::triggered, qApp, &QApplication::aboutQt);
}

void MainWindow::createMenus() {
    // File Menu
    QMenu* fileMenu = menuBar()->addMenu("&File");
    fileMenu->addAction(newAct);
    fileMenu->addAction(openAct);
    fileMenu->addAction(saveAct);
    fileMenu->addAction(saveAsAct);
    fileMenu->addSeparator();
    fileMenu->addAction(exitAct);
    
    // Edit Menu
    QMenu* editMenu = menuBar()->addMenu("&Edit");
    editMenu->addAction(undoAct);
    editMenu->addAction(redoAct);
    editMenu->addSeparator();
    editMenu->addAction(cutAct);
    editMenu->addAction(copyAct);
    editMenu->addAction(pasteAct);
    editMenu->addSeparator();
    editMenu->addAction(selectAllAct);
    editMenu->addSeparator();
    editMenu->addAction(findAct);
    
    // View Menu
    QMenu* viewMenu = menuBar()->addMenu("&View");
    viewMenu->addAction(zoomInAct);
    viewMenu->addAction(zoomOutAct);
    viewMenu->addSeparator();
    viewMenu->addAction(toggleFullscreenAct);
    viewMenu->addAction(showStatusBarAct);
    
    // Help Menu
    QMenu* helpMenu = menuBar()->addMenu("&Help");
    helpMenu->addAction(aboutAct);
    helpMenu->addAction(aboutQtAct);
}

void MainWindow::createToolBars() {
    // File Toolbar
    QToolBar* fileToolBar = addToolBar("File");
    fileToolBar->addAction(newAct);
    fileToolBar->addAction(openAct);
    fileToolBar->addAction(saveAct);
    
    // Edit Toolbar
    QToolBar* editToolBar = addToolBar("Edit");
    editToolBar->addAction(undoAct);
    editToolBar->addAction(redoAct);
    editToolBar->addSeparator();
    editToolBar->addAction(cutAct);
    editToolBar->addAction(copyAct);
    editToolBar->addAction(pasteAct);
}

void MainWindow::createStatusBar() {
    statusLabel = new QLabel("พร้อมใช้งาน");
    positionLabel = new QLabel("Ln 1, Col 1");
    encodingLabel = new QLabel("UTF-8");
    
    statusBar()->addWidget(statusLabel, 1);
    statusBar()->addPermanentWidget(positionLabel);
    statusBar()->addPermanentWidget(encodingLabel);
}

void MainWindow::newFile() {
    if (maybeSave()) {
        textEdit->clear();
        setCurrentFile("");
    }
}

void MainWindow::openFile() {
    if (maybeSave()) {
        QString filename = QFileDialog::getOpenFileName(
            this, "Open File", "",
            "Text Files (*.txt);;All Files (*)"
        );
        if (!filename.isEmpty()) {
            loadFile(filename);
        }
    }
}

void MainWindow::saveFile() {
    if (currentFile.isEmpty()) {
        saveFileAs();
    } else {
        saveFile(currentFile);
    }
}

void MainWindow::saveFileAs() {
    QString filename = QFileDialog::getSaveFileName(
        this, "Save File", "",
        "Text Files (*.txt);;All Files (*)"
    );
    if (!filename.isEmpty()) {
        saveFile(filename);
    }
}

bool MainWindow::saveFile(const QString& filename) {
    QFile file(filename);
    if (!file.open(QFile::WriteOnly | QFile::Text)) {
        QMessageBox::warning(this, "Error",
            QString("Cannot save file %1:\n%2").arg(filename, file.errorString()));
        return false;
    }
    
    QTextStream out(&file);
    out << textEdit->toPlainText();
    
    setCurrentFile(filename);
    statusBar()->showMessage("บันทึกแล้ว: " + filename, 3000);
    return true;
}

void MainWindow::loadFile(const QString& filename) {
    QFile file(filename);
    if (!file.open(QFile::ReadOnly | QFile::Text)) {
        QMessageBox::warning(this, "Error",
            QString("Cannot read file %1:\n%2").arg(filename, file.errorString()));
        return;
    }
    
    QTextStream in(&file);
    textEdit->setPlainText(in.readAll());
    
    setCurrentFile(filename);
    statusBar()->showMessage("เปิดไฟล์: " + filename, 3000);
}

void MainWindow::exit() {
    close();
}

void MainWindow::find() {
    bool ok;
    QString text = QInputDialog::getText(
        this, "Find", "ค้นหา:", QLineEdit::Normal, "", &ok
    );
    if (ok && !text.isEmpty()) {
        if (!textEdit->find(text)) {
            QMessageBox::information(this, "Find", "ไม่พบ: " + text);
        }
    }
}

void MainWindow::zoomIn() {
    QFont f = textEdit->font();
    f.setPointSize(f.pointSize() + 1);
    textEdit->setFont(f);
}

void MainWindow::zoomOut() {
    QFont f = textEdit->font();
    if (f.pointSize() > 6) {
        f.setPointSize(f.pointSize() - 1);
        textEdit->setFont(f);
    }
}

void MainWindow::toggleFullscreen() {
    if (isFullScreen()) {
        showNormal();
        toggleFullscreenAct->setChecked(false);
    } else {
        showFullScreen();
        toggleFullscreenAct->setChecked(true);
    }
}

void MainWindow::showStatusBar(bool show) {
    statusBar()->setVisible(show);
}

void MainWindow::about() {
    QMessageBox::about(this, "About Qt Text Editor",
        "<h3>Qt Text Editor</h3>"
        "<p>โปรแกรมแก้ไขข้อความสร้างด้วย Qt Framework</p>"
        "<p>Version 1.0</p>"
    );
}

void MainWindow::documentModified() {
    isModified = true;
    setWindowModified(true);
    statusLabel->setText("แก้ไขแล้ว");
}

void MainWindow::cursorPositionChanged() {
    QTextCursor cursor = textEdit->textCursor();
    int line = cursor.blockNumber() + 1;
    int col = cursor.columnNumber() + 1;
    positionLabel->setText(QString("Ln %1, Col %2").arg(line).arg(col));
}

bool MainWindow::maybeSave() {
    if (!isModified) return true;
    
    QMessageBox::StandardButton ret = QMessageBox::warning(
        this, "Text Editor",
        "ข้อความถูกแก้ไข\nต้องการบันทึกก่อนหรือไม่?",
        QMessageBox::Save | QMessageBox::Discard | QMessageBox::Cancel
    );
    
    if (ret == QMessageBox::Save) return saveFile(), true;
    if (ret == QMessageBox::Cancel) return false;
    return true;
}

void MainWindow::setCurrentFile(const QString& filename) {
    currentFile = filename;
    isModified = false;
    setWindowModified(false);
    
    QString title = "Qt Text Editor";
    if (!filename.isEmpty()) {
        title = QFileInfo(filename).fileName() + " - " + title;
    }
    setWindowTitle(title + "[*]");
}

void MainWindow::closeEvent(QCloseEvent* event) {
    if (maybeSave()) {
        event->accept();
    } else {
        event->ignore();
    }
}

// === main.cpp ===
int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    app.setApplicationName("Qt Text Editor");
    app.setApplicationVersion("1.0");
    
    MainWindow window;
    window.show();
    
    return app.exec();
}
```

---

## ขั้นตอนที่ 148-160: Context Menu

```cpp
// Context Menu (Right-click Menu)
class EditorWithContextMenu : public QTextEdit {
    Q_OBJECT
    
protected:
    void contextMenuEvent(QContextMenuEvent* event) override {
        QMenu* menu = createStandardContextMenu();  // Default menu
        
        menu->addSeparator();
        
        // Custom actions
        QAction* uppercaseAct = menu->addAction("UPPERCASE");
        QAction* lowercaseAct = menu->addAction("lowercase");
        QAction* wordCountAct = menu->addAction("Word Count");
        
        connect(uppercaseAct, &QAction::triggered, [this]() {
            QTextCursor cursor = textCursor();
            if (cursor.hasSelection()) {
                cursor.insertText(cursor.selectedText().toUpper());
            }
        });
        
        connect(lowercaseAct, &QAction::triggered, [this]() {
            QTextCursor cursor = textCursor();
            if (cursor.hasSelection()) {
                cursor.insertText(cursor.selectedText().toLower());
            }
        });
        
        connect(wordCountAct, &QAction::triggered, [this]() {
            QString text = toPlainText();
            int words = text.split(QRegularExpression("\\s+"), Qt::SkipEmptyParts).size();
            int chars = text.length();
            int lines = text.split('\n').size();
            QMessageBox::information(this, "Word Count",
                QString("บรรทัด: %1\nคำ: %2\nตัวอักษร: %3").arg(lines).arg(words).arg(chars));
        });
        
        menu->exec(event->globalPos());
        delete menu;
    }
};
```

---

## สรุป Part 012

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QMainWindow structure
2. ✅ QAction, สร้าง Menus, Toolbars
3. ✅ QFileDialog, QMessageBox, QInputDialog
4. ✅ StatusBar
5. ✅ Context Menu
6. ✅ Close event handling

---

⬅️ [Part 011](part011.md) | ➡️ [Part 013: Qt Styling & Custom Widgets](part013.md)
