# Part 051: World-Class Level — Packaging & Cross-Platform Deployment

## ขั้นตอนที่ 731-745

---

## ขั้นตอนที่ 731: Qt Deployment Overview

```
การ Deploy แอปพลิเคชัน Qt ข้ามแพลตฟอร์ม:

Windows:
  windeployqt.exe app.exe
  → copies Qt DLLs, plugins, translations

macOS:
  macdeployqt app.app
  → bundles frameworks, creates .dmg

Linux:
  linuxdeployqt app -appimage
  → creates AppImage (self-contained)

Tools:
  - CPack (CMake): สร้าง installer (NSIS, DEB, RPM, DMG)
  - Qt Installer Framework (QtIFW): custom installer with wizard
  - GitHub Actions: CI/CD cross-platform build matrix
```

---

## ขั้นตอนที่ 732: CMakeLists.txt สำหรับ Production

```cmake
cmake_minimum_required(VERSION 3.21)
project(MyQtApp VERSION 1.2.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

# Release optimizations
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    if(MSVC)
        add_compile_options(/O2 /GL /DNDEBUG)
        add_link_options(/LTCG)
    else()
        add_compile_options(-O3 -DNDEBUG -flto)
        add_link_options(-flto)
    endif()
endif()

find_package(Qt6 6.5 REQUIRED COMPONENTS
    Core Gui Widgets Network Sql Charts
)

# Source files
file(GLOB_RECURSE SOURCES src/*.cpp src/*.h)
file(GLOB_RECURSE QRC_FILES resources/*.qrc)

qt_add_executable(${PROJECT_NAME}
    ${SOURCES}
    ${QRC_FILES}
)

target_include_directories(${PROJECT_NAME} PRIVATE src)

target_link_libraries(${PROJECT_NAME} PRIVATE
    Qt6::Core Qt6::Gui Qt6::Widgets
    Qt6::Network Qt6::Sql Qt6::Charts
)

# Windows: no console window in release
if(WIN32)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        WIN32_EXECUTABLE $<CONFIG:Release>
    )
    # Application icon
    target_sources(${PROJECT_NAME} PRIVATE resources/app.rc)
endif()

# macOS: app bundle settings
if(APPLE)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        MACOSX_BUNDLE TRUE
        MACOSX_BUNDLE_GUI_IDENTIFIER "com.mycompany.myqtapp"
        MACOSX_BUNDLE_BUNDLE_NAME "My Qt App"
        MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
        MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION}
        MACOSX_BUNDLE_ICON_FILE AppIcon.icns
    )
    target_sources(${PROJECT_NAME} PRIVATE
        resources/AppIcon.icns
    )
    set_source_files_properties(resources/AppIcon.icns
        PROPERTIES MACOSX_PACKAGE_LOCATION "Resources"
    )
endif()

# Install rules
install(TARGETS ${PROJECT_NAME}
    BUNDLE  DESTINATION .
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# Qt deploy helper
qt_generate_deploy_app_script(
    TARGET ${PROJECT_NAME}
    OUTPUT_SCRIPT deploy_script
    NO_UNSUPPORTED_PLATFORM_ERROR
)
install(SCRIPT ${deploy_script})

# CPack configuration
set(CPACK_PACKAGE_NAME "MyQtApp")
set(CPACK_PACKAGE_VENDOR "My Company")
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION "A professional Qt application")
set(CPACK_PACKAGE_INSTALL_DIRECTORY "MyQtApp")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_SOURCE_DIR}/LICENSE.txt")

if(WIN32)
    set(CPACK_GENERATOR "NSIS;ZIP")
    set(CPACK_NSIS_DISPLAY_NAME "My Qt App")
    set(CPACK_NSIS_ENABLE_UNINSTALL_BEFORE_INSTALL ON)
    set(CPACK_NSIS_MUI_ICON "${CMAKE_SOURCE_DIR}/resources/app.ico")
elseif(APPLE)
    set(CPACK_GENERATOR "DragNDrop")
    set(CPACK_DMG_VOLUME_NAME "MyQtApp")
else()
    set(CPACK_GENERATOR "DEB;RPM;TGZ")
    set(CPACK_DEBIAN_PACKAGE_MAINTAINER "support@mycompany.com")
    set(CPACK_RPM_PACKAGE_LICENSE "MIT")
endif()

include(CPack)
```

---

## ขั้นตอนที่ 733: GitHub Actions CI/CD Matrix Build

```yaml
# .github/workflows/release.yml
name: Build & Release

on:
  push:
    tags:
      - 'v*'
  pull_request:
    branches: [ main ]

env:
  QT_VERSION: '6.6.0'
  CMAKE_BUILD_TYPE: Release

jobs:
  build:
    name: ${{ matrix.os }} / Qt ${{ env.QT_VERSION }}
    runs-on: ${{ matrix.os }}
    
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: windows-latest
            arch: win64_msvc2019_64
            generator: "Visual Studio 17 2022"
            package_suffix: windows-x64
            artifact_name: MyQtApp-windows-x64.zip
            
          - os: macos-latest
            arch: clang_64
            generator: "Ninja"
            package_suffix: macos
            artifact_name: MyQtApp-macos.dmg
            
          - os: ubuntu-22.04
            arch: gcc_64
            generator: "Ninja"
            package_suffix: linux
            artifact_name: MyQtApp-linux-x64.tar.gz
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Install Qt
      uses: jurplel/install-qt-action@v3
      with:
        version: ${{ env.QT_VERSION }}
        arch: ${{ matrix.arch }}
        cache: true
    
    - name: Install Linux dependencies
      if: runner.os == 'Linux'
      run: |
        sudo apt-get update
        sudo apt-get install -y \
          ninja-build libxcb-xinerama0 libxcb-cursor0 \
          libgl1-mesa-dev libglu1-mesa-dev
    
    - name: Install macOS dependencies
      if: runner.os == 'macOS'
      run: brew install ninja
    
    - name: Configure CMake
      run: |
        cmake -B build -G "${{ matrix.generator }}" \
          -DCMAKE_BUILD_TYPE=${{ env.CMAKE_BUILD_TYPE }} \
          -DCMAKE_INSTALL_PREFIX=install
    
    - name: Build
      run: cmake --build build --config ${{ env.CMAKE_BUILD_TYPE }} --parallel
    
    - name: Test
      run: ctest --test-dir build -C ${{ env.CMAKE_BUILD_TYPE }} --output-on-failure
    
    - name: Install (Deploy)
      run: cmake --install build --config ${{ env.CMAKE_BUILD_TYPE }}
    
    - name: Package (CPack)
      working-directory: build
      run: cpack -C ${{ env.CMAKE_BUILD_TYPE }} --verbose
    
    - name: Upload Artifact
      uses: actions/upload-artifact@v4
      with:
        name: ${{ matrix.artifact_name }}
        path: build/${{ matrix.artifact_name }}
        if-no-files-found: error

  publish-release:
    name: Publish GitHub Release
    needs: build
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    
    permissions:
      contents: write
    
    steps:
    - name: Download all artifacts
      uses: actions/download-artifact@v4
      with:
        path: dist/
    
    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        files: dist/**/*
        generate_release_notes: true
        draft: false
        prerelease: ${{ contains(github.ref, 'beta') || contains(github.ref, 'rc') }}
```

---

## ขั้นตอนที่ 734: Qt Installer Framework

```xml
<!-- installer/config/config.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<Installer>
    <Name>My Qt Application</Name>
    <Version>1.2.0</Version>
    <Title>My Qt Application Installer</Title>
    <Publisher>My Company Ltd.</Publisher>
    <StartMenuDir>My Qt Application</StartMenuDir>
    <TargetDir>@HomeDir@/MyQtApp</TargetDir>
    <InstallActionColumnVisible>false</InstallActionColumnVisible>
    <RunProgram>@TargetDir@/MyQtApp</RunProgram>
    <RunProgramDescription>Launch My Qt Application</RunProgramDescription>
    <WizardStyle>Aero</WizardStyle>
    <MaintenanceToolName>MaintenanceTool</MaintenanceToolName>
    <AllowNonAsciiCharacters>true</AllowNonAsciiCharacters>
</Installer>
```

```xml
<!-- installer/packages/com.mycompany.myqtapp/meta/package.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<Package>
    <DisplayName>My Qt Application</DisplayName>
    <Description>Core application files</Description>
    <Version>1.2.0</Version>
    <ReleaseDate>2024-01-15</ReleaseDate>
    <Name>com.mycompany.myqtapp</Name>
    <Default>true</Default>
    <ForcedInstallation>true</ForcedInstallation>
    <Script>installscript.qs</Script>
    <Licenses>
        <License name="MIT License" file="LICENSE.txt" />
    </Licenses>
</Package>
```

```javascript
// installer/packages/com.mycompany.myqtapp/meta/installscript.qs
function Component() {
    // Custom constructor - called when installer starts
}

Component.prototype.createOperations = function() {
    component.createOperations();
    
    if (systemInfo.productType === "windows") {
        // Create Start Menu shortcut
        component.addOperation("CreateShortcut",
            "@TargetDir@/MyQtApp.exe",
            "@StartMenuDir@/My Qt Application.lnk",
            "workingDirectory=@TargetDir@",
            "description=Launch My Qt Application"
        );
        
        // Create Desktop shortcut
        component.addOperation("CreateShortcut",
            "@TargetDir@/MyQtApp.exe",
            "@DesktopDir@/My Qt Application.lnk",
            "workingDirectory=@TargetDir@"
        );
    }
    
    if (systemInfo.productType === "osx") {
        component.addOperation("CreateSymlink",
            "@TargetDir@/MyQtApp.app",
            "/Applications/MyQtApp.app"
        );
    }
};

// Called before uninstall
Component.prototype.createOperationsForArchive = function() {
    component.createOperationsForArchive();
};
```

---

## ขั้นตอนที่ 735: AppVersion & Auto-Update System

```cpp
// appversion.h
#pragma once
#include <QString>
#include <QVersionNumber>
#include <QJsonObject>
#include <QNetworkAccessManager>

class AppVersion : public QObject {
    Q_OBJECT
    
public:
    static AppVersion& instance() {
        static AppVersion inst;
        return inst;
    }
    
    QVersionNumber current() const {
        return QVersionNumber(APP_VERSION_MAJOR, APP_VERSION_MINOR, APP_VERSION_PATCH);
    }
    
    QString currentString() const {
        return QString("%1.%2.%3")
            .arg(APP_VERSION_MAJOR)
            .arg(APP_VERSION_MINOR)
            .arg(APP_VERSION_PATCH);
    }
    
    void checkForUpdates(const QUrl& updateFeedUrl) {
        QNetworkRequest req(updateFeedUrl);
        req.setHeader(QNetworkRequest::UserAgentHeader,
                      "MyQtApp/" + currentString());
        
        auto* reply = m_nam.get(req);
        connect(reply, &QNetworkReply::finished, this, [=] {
            reply->deleteLater();
            if (reply->error() != QNetworkReply::NoError) {
                emit updateCheckFailed(reply->errorString());
                return;
            }
            
            QJsonDocument doc = QJsonDocument::fromJson(reply->readAll());
            parseUpdateResponse(doc.object());
        });
    }
    
signals:
    void updateAvailable(const QVersionNumber& latestVersion,
                         const QString& releaseNotes,
                         const QUrl& downloadUrl);
    void noUpdateAvailable();
    void updateCheckFailed(const QString& error);
    
private:
    void parseUpdateResponse(const QJsonObject& obj) {
        QString latestStr = obj["version"].toString();
        QVersionNumber latest = QVersionNumber::fromString(latestStr);
        
        if (latest > current()) {
            emit updateAvailable(
                latest,
                obj["releaseNotes"].toString(),
                QUrl(obj["downloadUrl"].toString())
            );
        } else {
            emit noUpdateAvailable();
        }
    }
    
    QNetworkAccessManager m_nam{this};
    
    static constexpr int APP_VERSION_MAJOR = 1;
    static constexpr int APP_VERSION_MINOR = 2;
    static constexpr int APP_VERSION_PATCH = 0;
};

// Update notification dialog
class UpdateDialog : public QDialog {
    Q_OBJECT
    
public:
    UpdateDialog(const QVersionNumber& version,
                 const QString& notes,
                 const QUrl& downloadUrl,
                 QWidget* parent = nullptr)
        : QDialog(parent)
    {
        setWindowTitle("อัปเดตพร้อมใช้งาน");
        setMinimumWidth(480);
        
        auto* layout = new QVBoxLayout(this);
        
        auto* titleLabel = new QLabel(
            QString("เวอร์ชัน %1 พร้อมให้ดาวน์โหลด")
                .arg(version.toString()),
            this
        );
        titleLabel->setStyleSheet("font-size: 16px; font-weight: bold;");
        
        auto* notesEdit = new QTextBrowser(this);
        notesEdit->setMarkdown(notes);
        notesEdit->setMinimumHeight(200);
        
        auto* buttonBox = new QDialogButtonBox(this);
        auto* downloadBtn = buttonBox->addButton(
            "ดาวน์โหลดอัปเดต", QDialogButtonBox::AcceptRole);
        auto* laterBtn = buttonBox->addButton(
            "ข้ามครั้งนี้", QDialogButtonBox::RejectRole);
        
        layout->addWidget(titleLabel);
        layout->addWidget(new QLabel("สิ่งที่ใหม่ในเวอร์ชันนี้:", this));
        layout->addWidget(notesEdit);
        layout->addWidget(buttonBox);
        
        connect(downloadBtn, &QPushButton::clicked, this, [=] {
            QDesktopServices::openUrl(downloadUrl);
            accept();
        });
        connect(laterBtn, &QPushButton::clicked, this, &QDialog::reject);
    }
};
```

---

## ขั้นตอนที่ 736: Application Crash Reporter

```cpp
// crashreporter.h
#pragma once
#include <QObject>
#include <QString>
#include <QFile>
#include <QTextStream>
#include <QDateTime>
#include <csignal>
#include <execinfo.h>   // Linux only; use DbgHelp on Windows

class CrashReporter : public QObject {
    Q_OBJECT
    
public:
    static void install() {
        std::signal(SIGSEGV, CrashReporter::signalHandler);
        std::signal(SIGABRT, CrashReporter::signalHandler);
        std::signal(SIGFPE,  CrashReporter::signalHandler);
        std::signal(SIGILL,  CrashReporter::signalHandler);
    }
    
    static QString logDirectory() {
        return QStandardPaths::writableLocation(
            QStandardPaths::AppDataLocation) + "/crash_logs";
    }
    
private:
    static void signalHandler(int signum) {
        QDir().mkpath(logDirectory());
        
        QString timestamp = QDateTime::currentDateTime()
            .toString("yyyyMMdd_HHmmss");
        QString logPath = logDirectory() + "/crash_" + timestamp + ".log";
        
        QFile logFile(logPath);
        if (logFile.open(QIODevice::WriteOnly | QIODevice::Text)) {
            QTextStream out(&logFile);
            out << "=== CRASH REPORT ===\n";
            out << "Time: " << QDateTime::currentDateTime().toString(Qt::ISODate) << "\n";
            out << "Signal: " << signum << " (" << signalName(signum) << ")\n";
            out << "Application: " << QCoreApplication::applicationName() << "\n";
            out << "Version: " << QCoreApplication::applicationVersion() << "\n";
            out << "\n=== STACK TRACE ===\n";
            
#ifndef Q_OS_WIN
            void* stack[64];
            int count = backtrace(stack, 64);
            char** symbols = backtrace_symbols(stack, count);
            
            for (int i = 0; i < count; ++i) {
                out << "[" << i << "] " << symbols[i] << "\n";
            }
            
            free(symbols);
#else
            out << "(Stack trace not available on this platform)\n";
#endif
        }
        
        // Try to launch crash reporter GUI
        QString reporter = QCoreApplication::applicationDirPath()
            + "/CrashReporter";
#ifdef Q_OS_WIN
        reporter += ".exe";
#endif
        
        if (QFile::exists(reporter)) {
            QProcess::startDetached(reporter, {logPath});
        }
        
        std::signal(signum, SIG_DFL);
        std::raise(signum);
    }
    
    static const char* signalName(int signum) {
        switch (signum) {
            case SIGSEGV: return "SIGSEGV (Segmentation fault)";
            case SIGABRT: return "SIGABRT (Abort)";
            case SIGFPE:  return "SIGFPE (Floating point exception)";
            case SIGILL:  return "SIGILL (Illegal instruction)";
            default:      return "Unknown";
        }
    }
};

// Crash Reporter GUI (separate process)
class CrashReporterWindow : public QMainWindow {
    Q_OBJECT
    
public:
    explicit CrashReporterWindow(const QString& logPath, QWidget* parent = nullptr)
        : QMainWindow(parent)
    {
        setWindowTitle("แอปพลิเคชันหยุดทำงาน");
        setMinimumSize(600, 400);
        
        auto* central = new QWidget(this);
        setCentralWidget(central);
        
        auto* layout = new QVBoxLayout(central);
        
        auto* icon = new QLabel(this);
        icon->setPixmap(QIcon::fromTheme("dialog-error")
            .pixmap(64, 64));
        icon->setAlignment(Qt::AlignCenter);
        
        auto* title = new QLabel(
            "เกิดข้อผิดพลาดที่ไม่คาดคิด\nกรุณาส่งรายงานเพื่อช่วยแก้ไขปัญหา",
            this);
        title->setAlignment(Qt::AlignCenter);
        title->setStyleSheet("font-size: 14px;");
        
        QFile logFile(logPath);
        QString logContent;
        if (logFile.open(QIODevice::ReadOnly)) {
            logContent = logFile.readAll();
        }
        
        auto* logEdit = new QTextEdit(this);
        logEdit->setReadOnly(true);
        logEdit->setPlainText(logContent);
        logEdit->setFont(QFont("Courier New", 9));
        
        auto* commentEdit = new QTextEdit(this);
        commentEdit->setPlaceholderText(
            "อธิบายสิ่งที่คุณกำลังทำก่อนเกิดข้อผิดพลาด (ไม่บังคับ)");
        commentEdit->setMaximumHeight(80);
        
        auto* buttonBox = new QDialogButtonBox(this);
        auto* sendBtn = buttonBox->addButton("ส่งรายงาน", QDialogButtonBox::AcceptRole);
        auto* closeBtn = buttonBox->addButton("ปิด", QDialogButtonBox::RejectRole);
        
        layout->addWidget(icon);
        layout->addWidget(title);
        layout->addWidget(new QLabel("รายละเอียด:", this));
        layout->addWidget(logEdit, 1);
        layout->addWidget(new QLabel("ความคิดเห็นเพิ่มเติม:", this));
        layout->addWidget(commentEdit);
        layout->addWidget(buttonBox);
        
        connect(sendBtn, &QPushButton::clicked, this, [=] {
            // Send crash report to server
            submitReport(logContent, commentEdit->toPlainText());
        });
        connect(closeBtn, &QPushButton::clicked, this, &QMainWindow::close);
    }
    
private:
    void submitReport(const QString& log, const QString& comment) {
        QNetworkAccessManager* nam = new QNetworkAccessManager(this);
        
        QJsonObject payload;
        payload["log"] = log;
        payload["comment"] = comment;
        payload["app_version"] = QCoreApplication::applicationVersion();
        
        QNetworkRequest req(QUrl("https://api.mycompany.com/crash-report"));
        req.setHeader(QNetworkRequest::ContentTypeHeader, "application/json");
        
        nam->post(req, QJsonDocument(payload).toJson());
        
        QMessageBox::information(this, "ขอบคุณ",
            "ส่งรายงานเรียบร้อยแล้ว ขอบคุณที่ช่วยปรับปรุงแอปพลิเคชัน");
        close();
    }
};
```

---

## ขั้นตอนที่ 737: Application Logging System

```cpp
// applogger.h
#pragma once
#include <QObject>
#include <QFile>
#include <QTextStream>
#include <QMutex>
#include <QElapsedTimer>

enum class LogLevel { Trace, Debug, Info, Warning, Error, Critical };

class AppLogger : public QObject {
    Q_OBJECT
    
public:
    static AppLogger& instance() {
        static AppLogger inst;
        return inst;
    }
    
    void initialize(const QString& logDir, LogLevel minLevel = LogLevel::Info) {
        m_minLevel = minLevel;
        
        QDir().mkpath(logDir);
        QString logPath = logDir + "/" +
            QDateTime::currentDateTime().toString("yyyy-MM-dd") + ".log";
        
        m_file = std::make_unique<QFile>(logPath);
        m_file->open(QIODevice::Append | QIODevice::Text);
        m_stream = std::make_unique<QTextStream>(m_file.get());
        
        // Install Qt message handler
        qInstallMessageHandler([](QtMsgType type, const QMessageLogContext& ctx,
                                   const QString& msg) {
            LogLevel level = LogLevel::Debug;
            switch (type) {
                case QtDebugMsg:    level = LogLevel::Debug;    break;
                case QtInfoMsg:     level = LogLevel::Info;     break;
                case QtWarningMsg:  level = LogLevel::Warning;  break;
                case QtCriticalMsg: level = LogLevel::Error;    break;
                case QtFatalMsg:    level = LogLevel::Critical; break;
            }
            AppLogger::instance().log(level, msg, ctx.file, ctx.line, ctx.function);
        });
        
        log(LogLevel::Info, "=== Application Started ===", __FILE__, __LINE__, __func__);
    }
    
    void log(LogLevel level, const QString& message,
             const char* file = nullptr, int line = 0,
             const char* func = nullptr)
    {
        if (level < m_minLevel) return;
        
        QMutexLocker lock(&m_mutex);
        
        QString timestamp = QDateTime::currentDateTime()
            .toString("yyyy-MM-dd HH:mm:ss.zzz");
        
        QString levelStr;
        switch (level) {
            case LogLevel::Trace:    levelStr = "TRACE"; break;
            case LogLevel::Debug:    levelStr = "DEBUG"; break;
            case LogLevel::Info:     levelStr = "INFO "; break;
            case LogLevel::Warning:  levelStr = "WARN "; break;
            case LogLevel::Error:    levelStr = "ERROR"; break;
            case LogLevel::Critical: levelStr = "CRIT "; break;
        }
        
        QString location;
        if (file) {
            location = QString(" [%1:%2]")
                .arg(QFileInfo(file).fileName())
                .arg(line);
        }
        
        QString entry = QString("[%1] %2%3 %4")
            .arg(timestamp, levelStr, location, message);
        
        if (m_stream) {
            *m_stream << entry << "\n";
            m_stream->flush();
        }
        
        // Also emit signal for UI log viewer
        emit logEntry(level, entry);
        
        if (level == LogLevel::Critical) {
            std::abort();
        }
    }
    
    // Convenience macros support
    void trace   (const QString& msg) { log(LogLevel::Trace,    msg); }
    void debug   (const QString& msg) { log(LogLevel::Debug,    msg); }
    void info    (const QString& msg) { log(LogLevel::Info,     msg); }
    void warning (const QString& msg) { log(LogLevel::Warning,  msg); }
    void error   (const QString& msg) { log(LogLevel::Error,    msg); }
    void critical(const QString& msg) { log(LogLevel::Critical, msg); }
    
signals:
    void logEntry(LogLevel level, const QString& formattedEntry);
    
private:
    AppLogger() = default;
    
    LogLevel m_minLevel = LogLevel::Info;
    std::unique_ptr<QFile> m_file;
    std::unique_ptr<QTextStream> m_stream;
    QMutex m_mutex;
};

// Convenience macros
#define LOG_TRACE(msg)   AppLogger::instance().log(LogLevel::Trace,   msg, __FILE__, __LINE__, __func__)
#define LOG_DEBUG(msg)   AppLogger::instance().log(LogLevel::Debug,   msg, __FILE__, __LINE__, __func__)
#define LOG_INFO(msg)    AppLogger::instance().log(LogLevel::Info,    msg, __FILE__, __LINE__, __func__)
#define LOG_WARNING(msg) AppLogger::instance().log(LogLevel::Warning, msg, __FILE__, __LINE__, __func__)
#define LOG_ERROR(msg)   AppLogger::instance().log(LogLevel::Error,   msg, __FILE__, __LINE__, __func__)

// Log Viewer Widget
class LogViewerWidget : public QWidget {
    Q_OBJECT
    
public:
    explicit LogViewerWidget(QWidget* parent = nullptr) : QWidget(parent) {
        auto* layout = new QVBoxLayout(this);
        
        // Filter bar
        auto* filterBar = new QHBoxLayout;
        auto* levelFilter = new QComboBox(this);
        levelFilter->addItems({"All", "Trace", "Debug", "Info", "Warning", "Error", "Critical"});
        
        auto* searchEdit = new QLineEdit(this);
        searchEdit->setPlaceholderText("ค้นหา...");
        
        auto* clearBtn = new QPushButton("ล้าง", this);
        
        filterBar->addWidget(new QLabel("Level:", this));
        filterBar->addWidget(levelFilter);
        filterBar->addWidget(searchEdit, 1);
        filterBar->addWidget(clearBtn);
        
        // Log display
        m_logEdit = new QPlainTextEdit(this);
        m_logEdit->setReadOnly(true);
        m_logEdit->setFont(QFont("Courier New", 9));
        m_logEdit->setMaximumBlockCount(10000); // Keep last 10k lines
        
        layout->addLayout(filterBar);
        layout->addWidget(m_logEdit);
        
        connect(&AppLogger::instance(), &AppLogger::logEntry,
                this, [=](LogLevel level, const QString& entry) {
                    QColor color = Qt::white;
                    switch (level) {
                        case LogLevel::Warning:  color = QColor("#FFA500"); break;
                        case LogLevel::Error:    color = QColor("#FF4444"); break;
                        case LogLevel::Critical: color = QColor("#FF0000"); break;
                        default: break;
                    }
                    
                    QTextCharFormat fmt;
                    fmt.setForeground(color);
                    
                    auto cursor = m_logEdit->textCursor();
                    cursor.movePosition(QTextCursor::End);
                    cursor.insertText(entry + "\n", fmt);
                    
                    m_logEdit->ensureCursorVisible();
                });
        
        connect(clearBtn, &QPushButton::clicked,
                m_logEdit, &QPlainTextEdit::clear);
    }
    
private:
    QPlainTextEdit* m_logEdit;
};
```

---

## สรุป Part 051

ใน Part นี้คุณได้เรียนรู้:

1. ✅ CMakeLists.txt สำหรับ Production (Windows/macOS/Linux)
2. ✅ GitHub Actions multi-platform release pipeline
3. ✅ Qt Installer Framework (config + package + installscript)
4. ✅ Auto-update system ด้วย QVersionNumber + QNetworkAccessManager
5. ✅ Crash reporter (signal handler + separate GUI process)
6. ✅ Structured logging system ด้วย AppLogger + LogViewerWidget

---

⬅️ [Part 050](part050.md) | ➡️ [Part 052: Capstone Project — Enterprise ERP System](part052.md)
