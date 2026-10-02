# Part 022: Qt Internationalization (i18n)

## ขั้นตอนที่ 296-310

---

## ขั้นตอนที่ 296: Qt i18n Overview

```
Qt i18n Workflow:
  1. ทำเครื่องหมายข้อความด้วย tr()
  2. สร้างไฟล์ .ts ด้วย lupdate
  3. แปลด้วย Qt Linguist
  4. คอมไพล์เป็น .qm ด้วย lrelease
  5. โหลด QTranslator ตอน runtime

Tools:
  lupdate  - สแกนโค้ด ดึง tr() strings
  lrelease - คอมไพล์ .ts -> .qm
  Qt Linguist - GUI แปล
```

---

## ขั้นตอนที่ 297: ทำเครื่องหมายข้อความ

```cpp
// === mainwindow.cpp ===
#include <QCoreApplication>

class MainWindow : public QMainWindow {
    Q_OBJECT
    
public:
    MainWindow() {
        setWindowTitle(tr("My Application"));
        
        // Menu
        auto* fileMenu = menuBar()->addMenu(tr("&File"));
        fileMenu->addAction(tr("&New"), this, &MainWindow::newFile);
        fileMenu->addAction(tr("&Open..."), this, &MainWindow::openFile);
        fileMenu->addAction(tr("&Save"), this, &MainWindow::saveFile);
        fileMenu->addSeparator();
        fileMenu->addAction(tr("E&xit"), qApp, &QApplication::quit);
        
        // Status
        statusBar()->showMessage(tr("Ready"));
        
        // With context
        // tr("text", "comment for translator")
        // tr("text", "comment", n) - plurals
    }
    
    void showMessage(int count) {
        QString msg = tr("%n item(s) selected", "", count);
        // "1 item selected" or "5 items selected"
        statusBar()->showMessage(msg);
    }
    
    void showError(const QString& detail) {
        QString msg = tr("Error: %1\nPlease try again.").arg(detail);
        QMessageBox::critical(this, tr("Error"), msg);
    }
};
```

---

## ขั้นตอนที่ 298: QTranslator

```cpp
// === main.cpp ===
#include <QApplication>
#include <QTranslator>
#include <QLocale>
#include <QLibraryInfo>

class LanguageManager : public QObject {
    Q_OBJECT
    
private:
    QTranslator appTranslator;
    QTranslator qtTranslator;
    QApplication* app;
    
public:
    LanguageManager(QApplication* a) : QObject(a), app(a) {}
    
    QStringList availableLanguages() const {
        return {"th", "en", "ja", "zh_CN", "ko"};
    }
    
    QString languageName(const QString& code) const {
        static QMap<QString, QString> names = {
            {"th", "ภาษาไทย"},
            {"en", "English"},
            {"ja", "日本語"},
            {"zh_CN", "中文"},
            {"ko", "한국어"}
        };
        return names.value(code, code);
    }
    
    bool loadLanguage(const QString& langCode) {
        // Remove old translators
        app->removeTranslator(&appTranslator);
        app->removeTranslator(&qtTranslator);
        
        // Load Qt's own translations
        if (qtTranslator.load("qt_" + langCode,
            QLibraryInfo::path(QLibraryInfo::TranslationsPath))) {
            app->installTranslator(&qtTranslator);
        }
        
        // Load app translations
        QString qmPath = ":/translations/myapp_" + langCode + ".qm";
        if (appTranslator.load(qmPath)) {
            app->installTranslator(&appTranslator);
            emit languageChanged(langCode);
            return true;
        }
        
        // Fallback to English
        if (langCode != "en") {
            return loadLanguage("en");
        }
        
        return false;
    }
    
    QString currentLanguage() const {
        return QLocale().language() == QLocale::Thai ? "th" : "en";
    }
    
signals:
    void languageChanged(const QString& code);
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    // Auto-detect system language
    QLocale locale = QLocale::system();
    
    LanguageManager langMgr(&app);
    
    // Try to load from settings
    QSettings settings;
    QString savedLang = settings.value("language", locale.name().left(2)).toString();
    langMgr.loadLanguage(savedLang);
    
    MainWindow window;
    window.show();
    
    return app.exec();
}
```

---

## ขั้นตอนที่ 299: Language Switcher UI

```cpp
class LanguageDialog : public QDialog {
    Q_OBJECT
    
public:
    LanguageDialog(LanguageManager* mgr, QWidget* parent = nullptr)
        : QDialog(parent), langMgr(mgr) {
        
        setWindowTitle(tr("Language Settings"));
        setFixedSize(300, 250);
        
        auto* layout = new QVBoxLayout(this);
        
        auto* label = new QLabel(tr("Select language:"));
        layout->addWidget(label);
        
        listWidget = new QListWidget();
        for (const QString& code : mgr->availableLanguages()) {
            auto* item = new QListWidgetItem(mgr->languageName(code));
            item->setData(Qt::UserRole, code);
            
            // Flag icon
            QIcon flag(QString(":/flags/%1.png").arg(code));
            item->setIcon(flag);
            
            listWidget->addItem(item);
        }
        layout->addWidget(listWidget);
        
        auto* buttons = new QDialogButtonBox(
            QDialogButtonBox::Ok | QDialogButtonBox::Cancel);
        layout->addWidget(buttons);
        
        connect(buttons, &QDialogButtonBox::accepted, this, &QDialog::accept);
        connect(buttons, &QDialogButtonBox::rejected, this, &QDialog::reject);
        
        // Live preview
        connect(listWidget, &QListWidget::currentItemChanged, [=](QListWidgetItem* item) {
            if (item) {
                QString code = item->data(Qt::UserRole).toString();
                mgr->loadLanguage(code);
            }
        });
    }
    
    QString selectedLanguage() const {
        auto* item = listWidget->currentItem();
        return item ? item->data(Qt::UserRole).toString() : "en";
    }
    
private:
    LanguageManager* langMgr;
    QListWidget* listWidget;
};
```

---

## ขั้นตอนที่ 300-310: Date, Time & Number Formatting

```cpp
#include <QLocale>
#include <QDate>
#include <QTime>
#include <QDateTime>

class LocaleDemo : public QWidget {
    Q_OBJECT
    
public:
    LocaleDemo() {
        setWindowTitle("Locale Demo");
        setMinimumSize(600, 400);
        
        auto* layout = new QVBoxLayout(this);
        auto* text = new QTextEdit();
        text->setReadOnly(true);
        layout->addWidget(text);
        
        QString output;
        
        // === Locale comparisons ===
        QList<QLocale> locales = {
            QLocale(QLocale::Thai, QLocale::Thailand),
            QLocale(QLocale::English, QLocale::UnitedStates),
            QLocale(QLocale::Japanese, QLocale::Japan),
            QLocale(QLocale::Chinese, QLocale::China),
            QLocale(QLocale::German, QLocale::Germany)
        };
        
        QDate today = QDate::currentDate();
        QTime now = QTime::currentTime();
        double number = 1234567.89;
        double currency = 9999.99;
        
        output += "<h3>Date Formatting</h3><table border=1 cellpadding=5>";
        output += "<tr><th>Locale</th><th>Date</th><th>Time</th></tr>";
        
        for (const QLocale& locale : locales) {
            output += QString("<tr><td>%1</td><td>%2</td><td>%3</td></tr>")
                .arg(locale.name())
                .arg(locale.toString(today, QLocale::LongFormat))
                .arg(locale.toString(now, QLocale::ShortFormat));
        }
        output += "</table>";
        
        output += "<h3>Number Formatting</h3><table border=1 cellpadding=5>";
        output += "<tr><th>Locale</th><th>Number</th><th>Percent</th></tr>";
        
        for (const QLocale& locale : locales) {
            output += QString("<tr><td>%1</td><td>%2</td><td>%3</td></tr>")
                .arg(locale.name())
                .arg(locale.toString(number, 'f', 2))
                .arg(locale.toString(0.85, 'f', 1) + "%");
        }
        output += "</table>";
        
        output += "<h3>Calendar</h3>";
        
        // Thai Buddhist Era
        QLocale thLocale(QLocale::Thai, QLocale::Thailand);
        output += "<p>Thai date: " + thLocale.toString(today, "dd MMMM yyyy") + "</p>";
        output += "<p>Thai BE year: " + QString::number(today.year() + 543) + "</p>";
        
        text->setHtml(output);
    }
};
```

---

## สรุป Part 022

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt i18n workflow
2. ✅ tr() สำหรับ mark ข้อความ
3. ✅ QTranslator การโหลด .qm
4. ✅ Language switching UI
5. ✅ QLocale สำหรับ format date/number

---

⬅️ [Part 021](part021.md) | ➡️ [Part 023: Qt Multimedia](part023.md)
