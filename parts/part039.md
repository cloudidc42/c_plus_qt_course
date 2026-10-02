# Part 039: Qt Settings & Configuration Management

## ขั้นตอนที่ 551-565

---

## ขั้นตอนที่ 551: QSettings พื้นฐาน

```cpp
#include <QSettings>

// === Basic QSettings usage ===
void saveSettings() {
    // Default: platform-specific location (Registry on Windows, plist on macOS, ini on Linux)
    QSettings settings("MyCompany", "MyApp");
    
    settings.setValue("window/geometry", QByteArray());
    settings.setValue("window/state", 0);
    settings.setValue("theme", "dark");
    settings.setValue("language", "th");
    settings.setValue("recent_files", QStringList{});
    settings.setValue("max_recent", 10);
    settings.setValue("auto_save", true);
    settings.setValue("auto_save_interval", 300); // seconds
}

void loadSettings() {
    QSettings settings("MyCompany", "MyApp");
    
    QString theme = settings.value("theme", "light").toString();
    QString lang  = settings.value("language", "en").toString();
    int maxRecent = settings.value("max_recent", 10).toInt();
    bool autoSave = settings.value("auto_save", false).toBool();
    
    qDebug() << "Theme:" << theme << "Lang:" << lang;
}

// === INI file settings ===
QSettings iniSettings(QDir::appDataLocation() + "/settings.ini",
                      QSettings::IniFormat);
```

---

## ขั้นตอนที่ 552: Type-Safe Settings

```cpp
class AppSettings : public QObject {
    Q_OBJECT
    
    // Macro for typed setting property
    #define SETTING(type, name, key, def) \
    private: \
        type m_##name = def; \
    public: \
        Q_PROPERTY(type name READ name WRITE set##name NOTIFY name##Changed) \
        type name() const { return m_##name; } \
        void set##name(const type& val) { \
            if (m_##name == val) return; \
            m_##name = val; \
            m_settings.setValue(key, val); \
            emit name##Changed(val); \
        } \
    signals: \
        void name##Changed(const type&);
    
public:
    static AppSettings& instance() {
        static AppSettings inst;
        return inst;
    }
    
    // Define settings
    SETTING(QString, theme,        "ui/theme",           "dark")
    SETTING(QString, language,     "ui/language",        "en")
    SETTING(bool,    autoSave,     "editor/auto_save",   true)
    SETTING(int,     autoInterval, "editor/interval",    300)
    SETTING(bool,    showToolbar,  "ui/show_toolbar",    true)
    SETTING(QFont,   editorFont,   "editor/font",        QFont("Monospace", 12))
    
    void load() {
        m_theme        = m_settings.value("ui/theme",           "dark").toString();
        m_language     = m_settings.value("ui/language",        "en").toString();
        m_autoSave     = m_settings.value("editor/auto_save",   true).toBool();
        m_autoInterval = m_settings.value("editor/interval",    300).toInt();
        m_showToolbar  = m_settings.value("ui/show_toolbar",    true).toBool();
        
        QFont defaultFont("Monospace", 12);
        m_editorFont = m_settings.value("editor/font", defaultFont).value<QFont>();
    }
    
    void save() { m_settings.sync(); }
    
    void reset() {
        m_settings.clear();
        load();
    }
    
    QStringList recentFiles() const {
        return m_settings.value("recent_files").toStringList();
    }
    
    void addRecentFile(const QString& path) {
        QStringList files = recentFiles();
        files.removeAll(path);
        files.prepend(path);
        while (files.size() > 10) files.removeLast();
        m_settings.setValue("recent_files", files);
        emit recentFilesChanged(files);
    }
    
    void removeRecentFile(const QString& path) {
        QStringList files = recentFiles();
        files.removeAll(path);
        m_settings.setValue("recent_files", files);
        emit recentFilesChanged(files);
    }
    
signals:
    void recentFilesChanged(const QStringList& files);
    
private:
    AppSettings() : m_settings("MyCompany", "MyApp") { load(); }
    QSettings m_settings;
};
```

---

## ขั้นตอนที่ 553: Main Window Persistence

```cpp
class PersistentMainWindow : public QMainWindow {
    Q_OBJECT
    
public:
    explicit PersistentMainWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setupUi();
        restoreWindowState();
        
        // Auto-save timer
        m_autoSaveTimer = new QTimer(this);
        connect(m_autoSaveTimer, &QTimer::timeout, this, &PersistentMainWindow::autoSave);
        
        auto& cfg = AppSettings::instance();
        if (cfg.autoSave()) {
            m_autoSaveTimer->start(cfg.autoInterval() * 1000);
        }
        
        connect(&cfg, &AppSettings::autoSaveChanged, [this, &cfg](bool enabled) {
            if (enabled) m_autoSaveTimer->start(cfg.autoInterval() * 1000);
            else m_autoSaveTimer->stop();
        });
    }
    
protected:
    void closeEvent(QCloseEvent* event) override {
        saveWindowState();
        QMainWindow::closeEvent(event);
    }
    
private:
    void saveWindowState() {
        QSettings s("MyCompany", "MyApp");
        s.setValue("window/geometry", saveGeometry());
        s.setValue("window/state", saveState());
        s.setValue("window/maximized", isMaximized());
        
        if (auto* splitter = findChild<QSplitter*>("mainSplitter")) {
            s.setValue("window/splitter", splitter->saveState());
        }
    }
    
    void restoreWindowState() {
        QSettings s("MyCompany", "MyApp");
        
        QByteArray geometry = s.value("window/geometry").toByteArray();
        if (!geometry.isEmpty()) restoreGeometry(geometry);
        else resize(1200, 800);
        
        QByteArray state = s.value("window/state").toByteArray();
        if (!state.isEmpty()) restoreState(state);
        
        if (s.value("window/maximized").toBool()) showMaximized();
        
        if (auto* splitter = findChild<QSplitter*>("mainSplitter")) {
            QByteArray splitterState = s.value("window/splitter").toByteArray();
            if (!splitterState.isEmpty()) splitter->restoreState(splitterState);
        }
    }
    
    void setupUi() {
        auto* splitter = new QSplitter(Qt::Horizontal);
        splitter->setObjectName("mainSplitter");
        
        auto* sidebar = new QListWidget();
        sidebar->setMaximumWidth(220);
        sidebar->addItems({"Dashboard", "Files", "Settings", "About"});
        
        auto* content = new QWidget();
        auto* contentLayout = new QVBoxLayout(content);
        
        m_editor = new QPlainTextEdit();
        m_editor->setFont(AppSettings::instance().editorFont());
        contentLayout->addWidget(m_editor);
        
        splitter->addWidget(sidebar);
        splitter->addWidget(content);
        splitter->setSizes({220, 980});
        
        setCentralWidget(splitter);
        setMinimumSize(800, 600);
    }
    
    void autoSave() {
        qDebug() << "Auto-saving...";
        // Actual save logic here
    }
    
    QTimer* m_autoSaveTimer;
    QPlainTextEdit* m_editor;
};
```

---

## ขั้นตอนที่ 554: Settings Dialog

```cpp
class SettingsDialog : public QDialog {
    Q_OBJECT
    
public:
    explicit SettingsDialog(QWidget* parent = nullptr) : QDialog(parent) {
        setWindowTitle("Preferences");
        setMinimumSize(600, 450);
        
        setupUi();
        loadCurrentValues();
    }
    
private:
    void setupUi() {
        auto* layout = new QVBoxLayout(this);
        
        auto* tabs = new QTabWidget();
        
        // === General tab ===
        auto* generalTab = new QWidget();
        auto* gLayout = new QFormLayout(generalTab);
        
        themeCombo = new QComboBox();
        themeCombo->addItems({"Light", "Dark", "System"});
        
        langCombo = new QComboBox();
        langCombo->addItem("English", "en");
        langCombo->addItem("ภาษาไทย", "th");
        langCombo->addItem("日本語", "ja");
        langCombo->addItem("中文", "zh");
        
        toolbarCheck = new QCheckBox("Show toolbar");
        
        gLayout->addRow("Theme:", themeCombo);
        gLayout->addRow("Language:", langCombo);
        gLayout->addRow("", toolbarCheck);
        tabs->addTab(generalTab, "General");
        
        // === Editor tab ===
        auto* editorTab = new QWidget();
        auto* eLayout = new QFormLayout(editorTab);
        
        fontBtn = new QPushButton();
        fontBtn->setFlat(true);
        fontBtn->setStyleSheet("text-align: left; padding: 4px;");
        
        autoSaveCheck = new QCheckBox("Enable auto-save");
        
        intervalSpin = new QSpinBox();
        intervalSpin->setRange(30, 3600);
        intervalSpin->setSuffix(" seconds");
        
        eLayout->addRow("Editor Font:", fontBtn);
        eLayout->addRow("Auto-save:", autoSaveCheck);
        eLayout->addRow("Interval:", intervalSpin);
        tabs->addTab(editorTab, "Editor");
        
        // Buttons
        auto* btnBox = new QDialogButtonBox(
            QDialogButtonBox::Ok | QDialogButtonBox::Cancel | QDialogButtonBox::RestoreDefaults);
        
        layout->addWidget(tabs);
        layout->addWidget(btnBox);
        
        // Connections
        connect(fontBtn, &QPushButton::clicked, this, &SettingsDialog::chooseFont);
        connect(autoSaveCheck, &QCheckBox::toggled, intervalSpin, &QSpinBox::setEnabled);
        
        connect(btnBox->button(QDialogButtonBox::Ok), &QPushButton::clicked,
                this, &SettingsDialog::applyAndClose);
        connect(btnBox->button(QDialogButtonBox::Cancel), &QPushButton::clicked,
                this, &QDialog::reject);
        connect(btnBox->button(QDialogButtonBox::RestoreDefaults), &QPushButton::clicked,
                this, &SettingsDialog::restoreDefaults);
    }
    
    void loadCurrentValues() {
        auto& cfg = AppSettings::instance();
        
        QString theme = cfg.theme();
        themeCombo->setCurrentText(theme.at(0).toUpper() + theme.mid(1));
        
        int langIdx = langCombo->findData(cfg.language());
        if (langIdx >= 0) langCombo->setCurrentIndex(langIdx);
        
        toolbarCheck->setChecked(cfg.showToolbar());
        
        m_currentFont = cfg.editorFont();
        fontBtn->setText(QString("%1, %2pt").arg(m_currentFont.family()).arg(m_currentFont.pointSize()));
        fontBtn->setFont(m_currentFont);
        
        autoSaveCheck->setChecked(cfg.autoSave());
        intervalSpin->setValue(cfg.autoInterval());
        intervalSpin->setEnabled(cfg.autoSave());
    }
    
    void chooseFont() {
        bool ok;
        QFont font = QFontDialog::getFont(&ok, m_currentFont, this, "Choose Editor Font");
        if (ok) {
            m_currentFont = font;
            fontBtn->setText(QString("%1, %2pt").arg(font.family()).arg(font.pointSize()));
            fontBtn->setFont(font);
        }
    }
    
    void applyAndClose() {
        auto& cfg = AppSettings::instance();
        
        cfg.setTheme(themeCombo->currentText().toLower());
        cfg.setLanguage(langCombo->currentData().toString());
        cfg.setShowToolbar(toolbarCheck->isChecked());
        cfg.setEditorFont(m_currentFont);
        cfg.setAutoSave(autoSaveCheck->isChecked());
        cfg.setAutoInterval(intervalSpin->value());
        
        accept();
    }
    
    void restoreDefaults() {
        if (QMessageBox::question(this, "Reset",
            "Reset all settings to defaults?") == QMessageBox::Yes) {
            AppSettings::instance().reset();
            loadCurrentValues();
        }
    }
    
    QComboBox* themeCombo;
    QComboBox* langCombo;
    QCheckBox* toolbarCheck;
    QPushButton* fontBtn;
    QCheckBox* autoSaveCheck;
    QSpinBox* intervalSpin;
    QFont m_currentFont;
};
```

---

## ขั้นตอนที่ 555: Configuration File (JSON)

```cpp
class JsonConfig {
public:
    bool load(const QString& path) {
        m_path = path;
        QFile file(path);
        
        if (!file.exists()) {
            m_data = defaultConfig();
            return save();
        }
        
        if (!file.open(QIODevice::ReadOnly)) return false;
        
        QJsonParseError err;
        m_data = QJsonDocument::fromJson(file.readAll(), &err).object();
        
        if (err.error != QJsonParseError::NoError) {
            qWarning() << "Config parse error:" << err.errorString();
            return false;
        }
        return true;
    }
    
    bool save() {
        QFile file(m_path);
        if (!file.open(QIODevice::WriteOnly)) return false;
        file.write(QJsonDocument(m_data).toJson(QJsonDocument::Indented));
        return true;
    }
    
    QJsonValue get(const QString& key, const QJsonValue& defaultVal = {}) const {
        QStringList parts = key.split('.');
        QJsonObject obj = m_data;
        
        for (int i = 0; i < parts.size() - 1; ++i) {
            if (!obj.contains(parts[i]) || !obj[parts[i]].isObject()) return defaultVal;
            obj = obj[parts[i]].toObject();
        }
        
        return obj.value(parts.last(), defaultVal);
    }
    
    void set(const QString& key, const QJsonValue& value) {
        QStringList parts = key.split('.');
        setNested(m_data, parts, value);
    }
    
    QJsonObject toObject() const { return m_data; }
    
private:
    void setNested(QJsonObject& obj, const QStringList& path, const QJsonValue& value) {
        if (path.size() == 1) {
            obj[path.first()] = value;
            return;
        }
        
        QJsonObject child = obj[path.first()].toObject();
        setNested(child, path.mid(1), value);
        obj[path.first()] = child;
    }
    
    QJsonObject defaultConfig() {
        return QJsonObject{
            {"version", "1.0"},
            {"ui", QJsonObject{
                {"theme", "dark"},
                {"language", "en"},
                {"font_size", 12},
                {"toolbar", true}
            }},
            {"editor", QJsonObject{
                {"auto_save", true},
                {"interval", 300},
                {"tab_size", 4},
                {"show_line_numbers", true}
            }},
            {"network", QJsonObject{
                {"timeout", 30},
                {"proxy", QJsonObject{{"enabled", false}, {"host", ""}, {"port", 8080}}}
            }}
        };
    }
    
    QString m_path;
    QJsonObject m_data;
};

// Usage:
// JsonConfig cfg;
// cfg.load(QDir::home().filePath(".myapp/config.json"));
// QString theme = cfg.get("ui.theme", "dark").toString();
// cfg.set("ui.theme", "light");
// cfg.save();
```

---

## สรุป Part 039

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QSettings พื้นฐาน (platform-specific, INI)
2. ✅ Type-safe AppSettings singleton ด้วย Q_PROPERTY macro
3. ✅ Window state persistence (geometry, splitter)
4. ✅ Settings Dialog พร้อม apply + restore defaults
5. ✅ JSON Configuration system แบบ nested key-value

---

⬅️ [Part 038](part038.md) | ➡️ [Part 040: Modern C++ Application Architecture](part040.md)
