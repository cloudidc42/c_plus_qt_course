# Part 063: Internationalization (i18n) & Accessibility

## ขั้นตอนที่ 911-925

---

## ขั้นตอนที่ 911: Qt Internationalization Fundamentals

```cpp
// Qt i18n ด้วย QTranslator + lupdate/lrelease workflow

// main.cpp
#include <QTranslator>
#include <QLocale>
#include <QLibraryInfo>

void installTranslations(QApplication& app) {
    QLocale locale = QLocale::system();
    
    // Install Qt base translations (dialogs, buttons)
    QTranslator qtTranslator;
    QString qtTransPath = QLibraryInfo::path(QLibraryInfo::TranslationsPath);
    if (qtTranslator.load(locale, "qt", "_", qtTransPath)) {
        app.installTranslator(&qtTranslator);
    }
    
    // Install application translations
    QTranslator appTranslator;
    // ไฟล์: translations/app_th.qm, app_en.qm, app_ja.qm
    if (appTranslator.load(locale, "app", "_", ":/translations")) {
        app.installTranslator(&appTranslator);
    }
}

// ในโค้ด — ใช้ tr() เสมอ
class CustomerDialog : public QDialog {
    Q_OBJECT
    
    void setupUi() {
        setWindowTitle(tr("แก้ไขข้อมูลลูกค้า"));
        
        auto* nameLabel = new QLabel(tr("ชื่อ-นามสกุล:"));
        auto* emailLabel = new QLabel(tr("อีเมล:"));
        
        // Plural forms
        int count = 5;
        statusBar()->showMessage(tr("%n รายการ", "", count));
        // → "5 รายการ" (ภาษาไทย), "5 items" (English)
        
        // Context disambiguation
        // สองข้อความเหมือนกันแต่ความหมายต่างกัน
        QString openFile  = QCoreApplication::translate("FileMenu", "Open");
        QString openOrder = QCoreApplication::translate("OrderMenu", "Open");
    }
};

// CMakeLists.txt — i18n setup
// qt_add_translations(ErpPro
//     TS_FILES
//         translations/app_th.ts
//         translations/app_en.ts
//         translations/app_ja.ts
//     LUPDATE_OPTIONS
//         -no-obsolete
// )
```

---

## ขั้นตอนที่ 912: LanguageManager สำหรับ Runtime Language Switch

```cpp
// i18n/languagemanager.h
#pragma once
#include <QObject>
#include <QTranslator>
#include <QMap>

struct Language {
    QString code;        // "th", "en", "ja"
    QString name;        // "ภาษาไทย", "English", "日本語"
    QString flagResource; // ":/flags/th.png"
    Qt::LayoutDirection direction; // Qt::LeftToRight or Qt::RightToLeft (Arabic)
};

class LanguageManager : public QObject {
    Q_OBJECT
    Q_PROPERTY(QString currentLanguage READ currentLanguage NOTIFY languageChanged)
    
public:
    static LanguageManager& instance() {
        static LanguageManager inst;
        return inst;
    }
    
    void initialize() {
        m_languages = {
            {"th", {.code="th", .name="ภาษาไทย", .flagResource=":/flags/th.png",
                    .direction=Qt::LeftToRight}},
            {"en", {.code="en", .name="English",  .flagResource=":/flags/en.png",
                    .direction=Qt::LeftToRight}},
            {"ja", {.code="ja", .name="日本語",    .flagResource=":/flags/ja.png",
                    .direction=Qt::LeftToRight}},
            {"ar", {.code="ar", .name="العربية",   .flagResource=":/flags/ar.png",
                    .direction=Qt::RightToLeft}},
        };
        
        QString savedLang = QSettings().value("app/language", "th").toString();
        switchLanguage(savedLang, false);
    }
    
    bool switchLanguage(const QString& code, bool saveToSettings = true) {
        if (!m_languages.contains(code)) return false;
        if (code == m_currentCode) return true;
        
        // Remove old translator
        if (m_appTranslator) {
            QCoreApplication::removeTranslator(m_appTranslator.get());
        }
        
        auto translator = std::make_unique<QTranslator>();
        if (!translator->load(QString(":/translations/app_%1").arg(code))) {
            qWarning() << "Failed to load translation for" << code;
            return false;
        }
        
        QCoreApplication::installTranslator(translator.get());
        m_appTranslator = std::move(translator);
        m_currentCode = code;
        
        // Apply layout direction for RTL languages
        const auto& lang = m_languages[code];
        QApplication::setLayoutDirection(lang.direction);
        
        if (saveToSettings) {
            QSettings().setValue("app/language", code);
        }
        
        emit languageChanged(code);
        return true;
    }
    
    QString currentLanguage() const { return m_currentCode; }
    QList<Language> availableLanguages() const { return m_languages.values(); }
    const Language& language(const QString& code) const { return m_languages[code]; }
    
signals:
    void languageChanged(const QString& code);
    
private:
    LanguageManager() = default;
    
    QMap<QString, Language> m_languages;
    QString m_currentCode;
    std::unique_ptr<QTranslator> m_appTranslator;
};

// LanguageSwitcherWidget — ปุ่มเลือกภาษาใน toolbar
class LanguageSwitcherWidget : public QWidget {
    Q_OBJECT
    
public:
    explicit LanguageSwitcherWidget(QWidget* parent = nullptr)
        : QWidget(parent)
    {
        auto* layout = new QHBoxLayout(this);
        layout->setContentsMargins(0, 0, 0, 0);
        
        m_combo = new QComboBox(this);
        m_combo->setMinimumWidth(140);
        
        for (const auto& lang : LanguageManager::instance().availableLanguages()) {
            m_combo->addItem(QIcon(lang.flagResource), lang.name, lang.code);
        }
        
        // Set current
        QString current = LanguageManager::instance().currentLanguage();
        int idx = m_combo->findData(current);
        if (idx >= 0) m_combo->setCurrentIndex(idx);
        
        connect(m_combo, &QComboBox::currentIndexChanged, this, [this](int idx) {
            QString code = m_combo->itemData(idx).toString();
            LanguageManager::instance().switchLanguage(code);
        });
        
        layout->addWidget(new QLabel(tr("ภาษา:")));
        layout->addWidget(m_combo);
    }
    
private:
    QComboBox* m_combo;
};
```

---

## ขั้นตอนที่ 913: Number, Date & Currency Formatting

```cpp
// i18n/formatter.h — locale-aware formatting
#pragma once
#include <QLocale>
#include <QDateTime>

class Formatter {
public:
    static Formatter& instance() {
        static Formatter inst;
        return inst;
    }
    
    void setLocale(const QLocale& locale) { m_locale = locale; }
    
    // ===== Currency =====
    QString currency(double amount, const QString& currencyCode = "THB") const {
        // ไทยบาท: ฿1,234.56
        // US Dollar: $1,234.56
        // Japanese Yen: ¥1,235
        
        static QMap<QString, QPair<QString, int>> currencies = {
            {"THB", {"฿", 2}},
            {"USD", {"$", 2}},
            {"EUR", {"€", 2}},
            {"JPY", {"¥", 0}},
            {"GBP", {"£", 2}},
        };
        
        auto it = currencies.find(currencyCode);
        if (it == currencies.end()) {
            return m_locale.toString(amount, 'f', 2);
        }
        
        const auto& [symbol, decimals] = it.value();
        QString num = m_locale.toString(amount, 'f', decimals);
        return symbol + num;
    }
    
    // ===== Date & Time =====
    QString date(const QDate& d, QLocale::FormatType fmt = QLocale::ShortFormat) const {
        return m_locale.toString(d, fmt);
    }
    
    QString dateTime(const QDateTime& dt, QLocale::FormatType fmt = QLocale::ShortFormat) const {
        return m_locale.toString(dt, fmt);
    }
    
    QString relativeTime(const QDateTime& dt) const {
        qint64 secs = dt.secsTo(QDateTime::currentDateTime());
        
        if (secs < 60)       return tr("เมื่อสักครู่");
        if (secs < 3600)     return tr("%n นาทีที่แล้ว", "", (int)(secs / 60));
        if (secs < 86400)    return tr("%n ชั่วโมงที่แล้ว", "", (int)(secs / 3600));
        if (secs < 604800)   return tr("%n วันที่แล้ว", "", (int)(secs / 86400));
        if (secs < 2592000)  return tr("%n สัปดาห์ที่แล้ว", "", (int)(secs / 604800));
        
        return date(dt.date());
    }
    
    // ===== Numbers =====
    QString number(double n, int decimals = 2) const {
        return m_locale.toString(n, 'f', decimals);
    }
    
    QString percent(double ratio, int decimals = 1) const {
        return m_locale.toString(ratio * 100.0, 'f', decimals) + "%";
    }
    
    QString fileSize(qint64 bytes) const {
        if (bytes < 1024)       return tr("%1 B").arg(bytes);
        if (bytes < 1048576)    return tr("%1 KB").arg(bytes / 1024.0, 0, 'f', 1);
        if (bytes < 1073741824) return tr("%1 MB").arg(bytes / 1048576.0, 0, 'f', 1);
        return tr("%1 GB").arg(bytes / 1073741824.0, 0, 'f', 2);
    }
    
private:
    Formatter() : m_locale(QLocale::system()) {}
    QLocale m_locale;
};

// Convenience macros
#define FMT_CURRENCY(x)  Formatter::instance().currency(x)
#define FMT_DATE(x)      Formatter::instance().date(x)
#define FMT_RELATIVE(x)  Formatter::instance().relativeTime(x)
```

---

## ขั้นตอนที่ 914: Qt Accessibility

```cpp
// accessibility/accessibilityhelper.h
#pragma once
#include <QWidget>
#include <QAccessible>
#include <QAccessibleWidget>

// ตั้งค่า accessibility properties สำหรับ widgets
class AccessibilityHelper {
public:
    // กำหนด accessible name และ description
    static void setup(QWidget* widget, const QString& name,
                      const QString& description = QString())
    {
        widget->setAccessibleName(name);
        if (!description.isEmpty()) {
            widget->setAccessibleDescription(description);
        }
    }
    
    // Keyboard shortcut label ด้วย buddy
    static QLabel* labelFor(const QString& text, QWidget* buddy, QWidget* parent) {
        auto* label = new QLabel(text, parent);
        label->setBuddy(buddy);  // Click label → focus buddy; screen reader reads label
        return label;
    }
    
    // Group ด้วย focus chain ที่ถูกต้อง
    static void setTabOrder(QWidget* parent, QList<QWidget*> widgets) {
        for (int i = 0; i + 1 < widgets.size(); ++i) {
            QWidget::setTabOrder(widgets[i], widgets[i + 1]);
        }
    }
};

// Custom accessible interface สำหรับ custom widget
class KpiCardAccessible : public QAccessibleWidget {
public:
    explicit KpiCardAccessible(KpiCard* card)
        : QAccessibleWidget(card, QAccessible::StaticText)
        , m_card(card)
    {}
    
    QString text(QAccessible::Text t) const override {
        switch (t) {
        case QAccessible::Name:
            return m_card->title();
        case QAccessible::Value:
            return QString::number(m_card->value(), 'f', 0);
        case QAccessible::Description:
            return QString("%1: %2 %3").arg(
                m_card->title(),
                QString::number(m_card->value()),
                m_card->unit());
        default:
            return QAccessibleWidget::text(t);
        }
    }
    
    QAccessible::State state() const override {
        auto s = QAccessibleWidget::state();
        s.readOnly = true;
        return s;
    }
    
private:
    KpiCard* m_card;
};

// Register accessible interface
// QAccessible::installFactory([](const QString& key, QObject* o) -> QAccessibleInterface* {
//     if (key == "KpiCard") return new KpiCardAccessible(qobject_cast<KpiCard*>(o));
//     return nullptr;
// });

// High Contrast Theme support
class HighContrastTheme {
public:
    static bool isEnabled() {
        // Windows: check system high contrast setting
#ifdef Q_OS_WIN
        HIGHCONTRAST hc;
        hc.cbSize = sizeof(HIGHCONTRAST);
        SystemParametersInfo(SPI_GETHIGHCONTRAST, 0, &hc, 0);
        return (hc.dwFlags & HCF_HIGHCONTRASTON) != 0;
#else
        // Linux/macOS: check environment
        return qEnvironmentVariable("QT_ACCESSIBILITY_HIGHCONTRAST") == "1";
#endif
    }
    
    static QString stylesheet() {
        return R"(
            * { background: #000000; color: #FFFFFF; }
            QToolTip { background: #FFFF00; color: #000000; border: 2px solid #FFFFFF; }
            QPushButton { border: 2px solid #FFFFFF; padding: 6px 12px; }
            QPushButton:focus { border-color: #FFFF00; outline: 2px solid #FFFF00; }
            QLineEdit { border: 2px solid #FFFFFF; background: #000000; color: #FFFFFF; }
            QLineEdit:focus { border-color: #FFFF00; }
            QTableView::item:selected { background: #0000FF; color: #FFFFFF; }
        )";
    }
};
```

---

## ขั้นตอนที่ 915: RTL Layout Support (Arabic/Hebrew)

```cpp
// RTL (Right-to-Left) layout support
// Qt จัดการ RTL โดยอัตโนมัติเมื่อ setLayoutDirection(Qt::RightToLeft)

class RtlAwareWidget : public QWidget {
    Q_OBJECT
    
public:
    explicit RtlAwareWidget(QWidget* parent = nullptr) : QWidget(parent) {
        // Listen for layout direction changes
        connect(qApp, &QApplication::layoutDirectionChanged,
                this, &RtlAwareWidget::onDirectionChanged);
    }
    
protected:
    void onDirectionChanged(Qt::LayoutDirection dir) {
        // Mirror custom-painted elements
        update();
    }
    
    void paintEvent(QPaintEvent*) override {
        QPainter p(this);
        
        if (layoutDirection() == Qt::RightToLeft) {
            // Mirror the painter
            p.translate(width(), 0);
            p.scale(-1, 1);
        }
        
        // Draw content — now auto-mirrored for RTL
        p.drawPixmap(10, 10, QPixmap(":/icons/arrow_right.png"));
    }
};

// Custom widget that adapts to RTL
class BreadcrumbBar : public QWidget {
    Q_OBJECT
    
public:
    void setPath(const QStringList& parts) {
        qDeleteAll(m_labels);
        m_labels.clear();
        
        auto* layout = qobject_cast<QHBoxLayout*>(this->layout());
        
        // Qt handles mirroring automatically for QHBoxLayout in RTL
        QString separator = (layoutDirection() == Qt::LeftToRight) ? " › " : " ‹ ";
        
        for (int i = 0; i < parts.size(); ++i) {
            if (i > 0) {
                auto* sep = new QLabel(separator, this);
                sep->setObjectName("breadcrumb-separator");
                layout->addWidget(sep);
                m_labels.append(sep);
            }
            
            auto* label = new QLabel(parts[i], this);
            if (i == parts.size() - 1) {
                label->setObjectName("breadcrumb-current");
                label->setFont(QFont(label->font().family(), -1, QFont::Bold));
            } else {
                label->setCursor(Qt::PointingHandCursor);
                label->setObjectName("breadcrumb-link");
                int idx = i;
                label->installEventFilter(this);
                label->setProperty("pathIndex", idx);
            }
            
            layout->addWidget(label);
            m_labels.append(label);
        }
        
        layout->addStretch();
    }
    
signals:
    void navigateTo(int index);
    
private:
    QList<QLabel*> m_labels;
};
```

---

## ขั้นตอนที่ 916: Translation Workflow

```bash
# CMakeLists.txt snippet
# qt_add_translations(ErpPro
#     TS_FILES
#         translations/app_th.ts
#         translations/app_en.ts
# )

# 1. Generate/update .ts files from source
lupdate src/ -ts translations/app_th.ts translations/app_en.ts

# 2. Edit translations with Qt Linguist
linguist translations/app_th.ts

# 3. Compile .ts → .qm
lrelease translations/app_th.ts  # → translations/app_th.qm
lrelease translations/app_en.ts  # → translations/app_en.qm

# 4. Embed .qm in resources (translations.qrc)
# <qresource prefix="/translations">
#     <file>app_th.qm</file>
#     <file>app_en.qm</file>
# </qresource>
```

```xml
<!-- translations/app_en.ts (excerpt) -->
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE TS>
<TS version="2.1" language="en_US">
<context>
    <name>CustomerDialog</name>
    <message>
        <location filename="../src/ui/customerdialog.cpp" line="15"/>
        <source>แก้ไขข้อมูลลูกค้า</source>
        <translation>Edit Customer</translation>
    </message>
    <message>
        <location filename="../src/ui/customerdialog.cpp" line="20"/>
        <source>ชื่อ-นามสกุล:</source>
        <translation>Full Name:</translation>
    </message>
    <message numerus="yes">
        <location filename="../src/ui/customerdialog.cpp" line="30"/>
        <source>%n รายการ</source>
        <translation>
            <numerusform>%n item</numerusform>
            <numerusform>%n items</numerusform>
        </translation>
    </message>
</context>
</TS>
```

---

## ขั้นตอนที่ 917: Font & Text Rendering

```cpp
// การจัดการ font สำหรับหลายภาษา
class FontManager {
public:
    static void initialize() {
        // Load custom fonts from resources
        QFontDatabase::addApplicationFont(":/fonts/NotoSansThai-Regular.ttf");
        QFontDatabase::addApplicationFont(":/fonts/NotoSansThai-Bold.ttf");
        QFontDatabase::addApplicationFont(":/fonts/NotoSansArabic-Regular.ttf");
        QFontDatabase::addApplicationFont(":/fonts/NotoSansJP-Regular.otf");
    }
    
    static QFont fontForLocale(const QLocale& locale, int pointSize = 10) {
        switch (locale.script()) {
        case QLocale::ThaiScript:
            return QFont("Noto Sans Thai", pointSize);
        case QLocale::ArabicScript:
            return QFont("Noto Sans Arabic", pointSize);
        case QLocale::JapaneseScript:
        case QLocale::ChineseSimplifiedScript:
            return QFont("Noto Sans JP", pointSize);
        default:
            return QFont("Segoe UI", pointSize);
        }
    }
    
    static void applyToApplication(const QLocale& locale) {
        qApp->setFont(fontForLocale(locale));
    }
};

// QTextEdit ที่รองรับ bidirectional text
class BiDiTextEdit : public QTextEdit {
    Q_OBJECT
    
public:
    explicit BiDiTextEdit(QWidget* parent = nullptr) : QTextEdit(parent) {
        // เปิดใช้ complex text layout
        document()->setDefaultTextOption(
            QTextOption(Qt::AlignLeft | Qt::AlignAbsolute));
    }
    
    void insertText(const QString& text) {
        QTextCursor cursor = textCursor();
        
        // Detect text direction
        bool isRtl = false;
        for (QChar ch : text) {
            if (ch.direction() == QChar::DirAL || ch.direction() == QChar::DirR) {
                isRtl = true;
                break;
            }
        }
        
        QTextCharFormat fmt = cursor.charFormat();
        
        if (isRtl) {
            QTextBlockFormat blockFmt = cursor.blockFormat();
            blockFmt.setLayoutDirection(Qt::RightToLeft);
            cursor.setBlockFormat(blockFmt);
        }
        
        cursor.insertText(text, fmt);
    }
};
```

---

## สรุป Part 063

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QTranslator + lupdate/lrelease workflow สำหรับ i18n
2. ✅ LanguageManager สำหรับ runtime language switching
3. ✅ Locale-aware formatting: currency, date, number, file size
4. ✅ Qt Accessibility: accessible names, custom interfaces, high contrast
5. ✅ RTL layout support สำหรับภาษาอาหรับ/ฮีบรู
6. ✅ Font management สำหรับ multilingual text

---

⬅️ [Part 062](part062.md) | ➡️ [Part 064: Advanced Testing & Code Quality](part064.md)
