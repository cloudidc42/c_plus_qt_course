# Part 060: Capstone Final — Complete ERP Integration & main.cpp

## ขั้นตอนที่ 866-880

---

## ขั้นตอนที่ 866: Application Bootstrap

```cpp
// main.cpp — Complete ERP Application Bootstrap
#include <QApplication>
#include <QSplashScreen>
#include <QPixmap>
#include <QTimer>
#include <QTranslator>
#include <QDir>
#include <QStandardPaths>
#include <QSslSocket>

#include "core/database/database.h"
#include "security/authservice.h"
#include "ui/auth/logindialog.h"
#include "ui/main/erpmainwindow.h"
#include "app/pluginmanager.h"

// Set up application metadata
void configureApplication(QApplication& app) {
    app.setApplicationName("ERP Pro");
    app.setApplicationDisplayName("ERP Pro — ระบบจัดการธุรกิจ");
    app.setApplicationVersion("1.2.0");
    app.setOrganizationName("My Company Ltd.");
    app.setOrganizationDomain("mycompany.co.th");
    
    // High DPI support
    app.setAttribute(Qt::AA_UseHighDpiPixmaps);
    
    // Font
    QFont defaultFont("Segoe UI", 10);
#ifdef Q_OS_MAC
    defaultFont.setFamily("SF Pro Display");
    defaultFont.setPointSize(13);
#elif defined(Q_OS_LINUX)
    defaultFont.setFamily("Ubuntu");
#endif
    app.setFont(defaultFont);
}

// Apply application stylesheet
void applyStylesheet(QApplication& app) {
    app.setStyleSheet(R"(
        QMainWindow, QDialog {
            background: #F5F6FA;
        }
        QTableView {
            gridline-color: #E0E0E0;
            selection-background-color: #3498DB;
            selection-color: white;
        }
        QTableView::item:hover {
            background: #EBF5FB;
        }
        QHeaderView::section {
            background: #2C3E50;
            color: white;
            padding: 6px;
            border: none;
            font-weight: bold;
        }
        QToolBar {
            background: white;
            border-bottom: 1px solid #E0E0E0;
            spacing: 4px;
        }
        QToolButton {
            border-radius: 4px;
            padding: 4px 8px;
        }
        QToolButton:hover { background: #EBF5FB; }
        QToolButton:pressed { background: #D6EAF8; }
        QLineEdit, QComboBox, QSpinBox, QDoubleSpinBox {
            border: 1px solid #BDC3C7;
            border-radius: 4px;
            padding: 6px 8px;
            background: white;
        }
        QLineEdit:focus, QComboBox:focus {
            border-color: #3498DB;
        }
        QPushButton {
            border-radius: 4px;
            padding: 8px 16px;
            background: white;
            border: 1px solid #BDC3C7;
        }
        QPushButton:hover { background: #EBF5FB; border-color: #3498DB; }
        QPushButton:pressed { background: #D6EAF8; }
        QPushButton[flat="true"] { border: none; background: transparent; }
        QGroupBox {
            font-weight: bold;
            border: 1px solid #E0E0E0;
            border-radius: 6px;
            margin-top: 12px;
            padding-top: 8px;
        }
        QGroupBox::title {
            subcontrol-origin: margin;
            left: 12px;
            padding: 0 4px;
        }
        QStatusBar { background: white; border-top: 1px solid #E0E0E0; }
        QScrollBar:vertical {
            width: 8px;
            background: transparent;
        }
        QScrollBar::handle:vertical {
            background: #BDC3C7;
            border-radius: 4px;
            min-height: 20px;
        }
        QScrollBar::handle:vertical:hover { background: #95A5A6; }
        QScrollBar::add-line:vertical, QScrollBar::sub-line:vertical { height: 0px; }
    )");
}

// Progress messages for splash screen
void runStartupSequence(QSplashScreen* splash,
                        std::function<void()> onComplete)
{
    struct Step { int delay; QString msg; };
    
    QList<Step> steps = {
        {200, "กำลังตรวจสอบระบบ..."},
        {400, "กำลังเชื่อมต่อฐานข้อมูล..."},
        {600, "กำลังโหลดการตั้งค่า..."},
        {800, "กำลังโหลด Plugin..."},
        {1000, "เสร็จสิ้น"}
    };
    
    for (const auto& step : steps) {
        QTimer::singleShot(step.delay, splash, [=] {
            splash->showMessage(step.msg,
                Qt::AlignBottom | Qt::AlignHCenter,
                Qt::white);
        });
    }
    
    QTimer::singleShot(1200, splash, [=] {
        onComplete();
    });
}

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    configureApplication(app);
    
    // Check SSL availability
    if (!QSslSocket::supportsSsl()) {
        qWarning() << "SSL not available. Some features may not work.";
    }
    
    // Show splash screen
    QSplashScreen* splash = nullptr;
    
    QPixmap splashPix(600, 400);
    splashPix.fill(QColor("#2C3E50"));
    
    {
        QPainter p(&splashPix);
        p.setRenderHint(QPainter::Antialiasing);
        
        p.setPen(Qt::white);
        p.setFont(QFont("Segoe UI", 36, QFont::Bold));
        p.drawText(splashPix.rect(), Qt::AlignCenter, "ERP Pro");
        
        p.setFont(QFont("Segoe UI", 14));
        p.setPen(QColor(189, 195, 199));
        p.drawText(QRect(0, 240, 600, 40), Qt::AlignCenter, "ระบบจัดการธุรกิจ");
        
        p.setFont(QFont("Segoe UI", 10));
        p.setPen(QColor(127, 140, 141));
        p.drawText(QRect(0, 340, 600, 30), Qt::AlignCenter,
                   QString("v%1  |  © 2024 My Company Ltd.").arg(app.applicationVersion()));
    }
    
    splash = new QSplashScreen(splashPix, Qt::WindowStaysOnTopHint);
    splash->show();
    app.processEvents();
    
    // Run startup
    runStartupSequence(splash, [&] {
        // Initialize database
        QString dbPath = QStandardPaths::writableLocation(
            QStandardPaths::AppDataLocation) + "/erp.db";
        QDir().mkpath(QFileInfo(dbPath).path());
        
        if (!Database::instance().connect(dbPath)) {
            QMessageBox::critical(nullptr, "ข้อผิดพลาด",
                "ไม่สามารถเชื่อมต่อฐานข้อมูลได้\n" + dbPath);
            app.quit();
            return;
        }
        
        // Apply logging
        AppLogger::instance().initialize(
            QStandardPaths::writableLocation(QStandardPaths::AppDataLocation) + "/logs");
        
        // Load plugins
        QString pluginDir = QCoreApplication::applicationDirPath() + "/plugins";
        if (QDir(pluginDir).exists()) {
            PluginManager::instance().loadPlugins(pluginDir);
        }
        
        // Install theme (load from settings or default)
        applyStylesheet(app);
        
        // Translation support
        QTranslator translator;
        QString lang = QSettings().value("app/language", "th").toString();
        if (translator.load(":/translations/app_" + lang)) {
            app.installTranslator(&translator);
        }
        
        // Close splash
        splash->close();
        splash->deleteLater();
        
        // Try auto-login
        bool autoLoggedIn = AuthService::instance().tryAutoLogin();
        
        if (!autoLoggedIn) {
            LoginDialog login;
            if (login.exec() != QDialog::Accepted) {
                app.quit();
                return;
            }
        }
        
        // Show main window
        auto* mainWin = new ErpMainWindow;
        mainWin->show();
        
        // Handle auto-logout on token expiry
        QTimer* tokenChecker = new QTimer(mainWin);
        tokenChecker->setInterval(60000);
        QObject::connect(tokenChecker, &QTimer::timeout, mainWin, [mainWin] {
            if (!AuthService::instance().isLoggedIn()) {
                mainWin->hide();
                
                LoginDialog login(mainWin);
                if (login.exec() == QDialog::Accepted) {
                    mainWin->show();
                } else {
                    qApp->quit();
                }
            }
        });
        tokenChecker->start();
    });
    
    return app.exec();
}
```

---

## ขั้นตอนที่ 867: CMakeLists.txt Final

```cmake
cmake_minimum_required(VERSION 3.21)
project(ErpPro VERSION 1.2.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

# Find Qt components
find_package(Qt6 6.5 REQUIRED COMPONENTS
    Core Gui Widgets Network NetworkAuth
    Sql Charts WebSockets Concurrent
    OpenGL OpenGLWidgets PrintSupport Pdf
    Quick QuickControls2 Qml
)

# Collect sources
file(GLOB_RECURSE SOURCES
    src/core/*.cpp
    src/security/*.cpp
    src/network/*.cpp
    src/modules/**/*.cpp
    src/ui/**/*.cpp
    src/app/*.cpp
)

# Resources
qt_add_resources(QRC_SOURCES
    resources/resources.qrc
)

# QML module
qt_add_qml_module(erp_qml
    URI "com.erppro"
    VERSION 1.0
    QML_FILES
        qml/main.qml
        qml/DashboardPage.qml
        qml/ContactsPage.qml
        qml/ContactDelegate.qml
)

# Main executable
qt_add_executable(ErpPro
    src/main.cpp
    ${SOURCES}
    ${QRC_SOURCES}
)

target_include_directories(ErpPro PRIVATE src)

target_link_libraries(ErpPro PRIVATE
    Qt6::Core Qt6::Gui Qt6::Widgets
    Qt6::Network Qt6::NetworkAuth
    Qt6::Sql Qt6::Charts
    Qt6::WebSockets Qt6::Concurrent
    Qt6::OpenGL Qt6::OpenGLWidgets
    Qt6::PrintSupport Qt6::Pdf
    Qt6::Quick Qt6::QuickControls2 Qt6::Qml
    erp_qml
)

# Plugin subdirectories
add_subdirectory(plugins/report_pdf)
add_subdirectory(plugins/theme_dark)

# Tests
enable_testing()
add_subdirectory(tests)

# Install + CPack
qt_generate_deploy_app_script(
    TARGET ErpPro
    OUTPUT_SCRIPT deploy_script
)
install(SCRIPT ${deploy_script})

include(CPack)
```

---

## ขั้นตอนที่ 868: Complete Project Checklist

```
✅ ERP Pro — สิ่งที่ครอบคลุมในโปรเจกต์นี้:

Core:
  ✅ SQLite database ด้วย WAL + migrations
  ✅ Repository pattern (Customer, Product, Order)
  ✅ EventBus สำหรับ cross-module communication
  ✅ ServiceLocator + DI Container

Security:
  ✅ PBKDF2-HMAC-SHA256 password hashing
  ✅ JWT token authentication (HS256)
  ✅ Account locking (5 failed attempts = 15 min lockout)
  ✅ Auto-login ด้วย stored token
  ✅ Role-based access control

Network:
  ✅ HTTP client (GET/POST/PUT/PATCH/DELETE + file upload)
  ✅ WebSocket client ด้วย auto-reconnect + subscriptions
  ✅ Real-time notifications

UI:
  ✅ Professional login dialog ด้วย animations
  ✅ Side navigation bar
  ✅ Dashboard ด้วย KPI cards + charts
  ✅ Customer/Product/Order management pages
  ✅ QML mobile-ready interface
  ✅ Dark mode support

Business Logic:
  ✅ Sales order workflow (draft → confirmed → shipped → paid)
  ✅ Stock management
  ✅ PDF invoice generation
  ✅ CSV export

Architecture:
  ✅ Plugin system ด้วย QPluginLoader
  ✅ Crash reporter
  ✅ Application logger
  ✅ Auto-update checker
  ✅ CI/CD ด้วย GitHub Actions
```

---

## สรุป Part 060

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Application bootstrap ด้วย QSplashScreen + startup sequence
2. ✅ Global stylesheet สำหรับ professional UI
3. ✅ Auto-login ด้วย JWT token + re-login on expiry
4. ✅ Plugin loading ที่ startup
5. ✅ CMakeLists.txt ที่สมบูรณ์สำหรับโปรเจกต์ระดับ production
6. ✅ Project checklist — ทุกองค์ประกอบของ ERP ระดับ World-Class

---

⬅️ [Part 059](part059.md) | ➡️ [Part 061: Advanced C++ — SIMD Optimization](part061.md)
