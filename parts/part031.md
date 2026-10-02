# Part 031: Qt Plugin System & Dynamic Libraries

## ขั้นตอนที่ 431-445

---

## ขั้นตอนที่ 431: Qt Plugin Overview

```
Qt Plugin Architecture:
  ├── Static Plugins   - รวมเข้า executable
  ├── Dynamic Plugins  - โหลดตอน runtime (.so/.dll)
  └── Qt Built-in Plugins:
        ├── Image formats (PNG, JPEG, GIF)
        ├── Database drivers (SQLite, MySQL)
        ├── Platform (xcb, windows, cocoa)
        └── Style (fusion, windows, mac)

Plugin Types:
  ├── QPlugin (Qt's built-in plugin framework)
  └── QLibrary (low-level dynamic loading)

Use cases:
  - Application extensions
  - Modular architecture
  - Third-party add-ons
  - Hot-swappable features
```

---

## ขั้นตอนที่ 432: Define Plugin Interface

```cpp
// === shapes_interface.h (shared between app and plugins) ===
#pragma once
#include <QString>
#include <QWidget>
#include <QPainter>
#include <QRectF>

#define ShapePlugin_iid "com.example.ShapePlugin/1.0"

class ShapePlugin {
public:
    virtual ~ShapePlugin() = default;
    
    // Plugin metadata
    virtual QString name() const = 0;
    virtual QString description() const = 0;
    virtual QString version() const = 0;
    
    // Draw the shape
    virtual void draw(QPainter* painter, const QRectF& rect) = 0;
    
    // Get shape area
    virtual double area(const QRectF& rect) const = 0;
    
    // Get config widget (optional)
    virtual QWidget* configWidget(QWidget* parent = nullptr) { 
        Q_UNUSED(parent)
        return nullptr; 
    }
};

Q_DECLARE_INTERFACE(ShapePlugin, ShapePlugin_iid)
```

---

## ขั้นตอนที่ 433: Implement Plugin

```cpp
// === circle_plugin/circle_plugin.h ===
#pragma once
#include <QObject>
#include <QtPlugin>
#include "../shapes_interface.h"

class CirclePlugin : public QObject, public ShapePlugin {
    Q_OBJECT
    Q_INTERFACES(ShapePlugin)
    Q_PLUGIN_METADATA(IID ShapePlugin_iid FILE "circle_plugin.json")
    
public:
    QString name() const override { return "Circle"; }
    QString description() const override { return "Draws a circle inside the rect"; }
    QString version() const override { return "1.0.0"; }
    
    void draw(QPainter* painter, const QRectF& rect) override {
        painter->save();
        painter->setRenderHint(QPainter::Antialiasing);
        painter->setBrush(QBrush(m_color));
        painter->setPen(QPen(m_borderColor, m_borderWidth));
        painter->drawEllipse(rect);
        painter->restore();
    }
    
    double area(const QRectF& rect) const override {
        double r = std::min(rect.width(), rect.height()) / 2.0;
        return M_PI * r * r;
    }
    
    QWidget* configWidget(QWidget* parent) override;
    
    void setColor(const QColor& c) { m_color = c; }
    void setBorderColor(const QColor& c) { m_borderColor = c; }
    void setBorderWidth(int w) { m_borderWidth = w; }
    
private:
    QColor m_color{Qt::blue};
    QColor m_borderColor{Qt::black};
    int m_borderWidth = 2;
};

// === circle_plugin.cpp ===
#include "circle_plugin.h"
#include <QVBoxLayout>
#include <QFormLayout>
#include <QPushButton>
#include <QSpinBox>

QWidget* CirclePlugin::configWidget(QWidget* parent) {
    auto* widget = new QWidget(parent);
    auto* layout = new QFormLayout(widget);
    
    auto* colorBtn = new QPushButton("Choose Color");
    colorBtn->setStyleSheet(QString("background: %1").arg(m_color.name()));
    
    auto* borderSpin = new QSpinBox();
    borderSpin->setRange(0, 20);
    borderSpin->setValue(m_borderWidth);
    
    layout->addRow("Fill Color:", colorBtn);
    layout->addRow("Border Width:", borderSpin);
    
    QObject::connect(colorBtn, &QPushButton::clicked, [this, colorBtn]() {
        QColor c = QColorDialog::getColor(m_color);
        if (c.isValid()) {
            m_color = c;
            colorBtn->setStyleSheet(QString("background: %1").arg(c.name()));
        }
    });
    
    QObject::connect(borderSpin, qOverload<int>(&QSpinBox::valueChanged), 
                     [this](int val) { m_borderWidth = val; });
    
    return widget;
}

// === circle_plugin.json ===
// {
//   "IID": "com.example.ShapePlugin/1.0",
//   "MetaData": {
//     "name": "Circle Plugin",
//     "version": "1.0.0"
//   }
// }

// === circle_plugin/CMakeLists.txt ===
// add_library(circle_plugin SHARED
//     circle_plugin.cpp
//     circle_plugin.h
// )
// target_link_libraries(circle_plugin PRIVATE Qt6::Core Qt6::Gui Qt6::Widgets)
// set_target_properties(circle_plugin PROPERTIES LIBRARY_OUTPUT_DIRECTORY "${APP_PLUGINS_DIR}")
```

---

## ขั้นตอนที่ 434: Rectangle Plugin

```cpp
// === rect_plugin.h ===
#pragma once
#include <QObject>
#include <QtPlugin>
#include "../shapes_interface.h"

class RectPlugin : public QObject, public ShapePlugin {
    Q_OBJECT
    Q_INTERFACES(ShapePlugin)
    Q_PLUGIN_METADATA(IID ShapePlugin_iid FILE "rect_plugin.json")
    
public:
    QString name() const override { return "Rectangle"; }
    QString description() const override { return "Draws a rounded rectangle"; }
    QString version() const override { return "1.0.0"; }
    
    void draw(QPainter* painter, const QRectF& rect) override {
        painter->save();
        painter->setRenderHint(QPainter::Antialiasing);
        painter->setBrush(m_gradient);
        painter->setPen(QPen(Qt::darkGray, 2));
        painter->drawRoundedRect(rect, m_radius, m_radius);
        painter->restore();
    }
    
    double area(const QRectF& rect) const override {
        return rect.width() * rect.height();
    }
    
    void setRadius(int r) {
        m_radius = r;
        updateGradient();
    }
    
    void setColors(const QColor& top, const QColor& bottom) {
        m_topColor = top;
        m_bottomColor = bottom;
        updateGradient();
    }
    
private:
    void updateGradient() {
        m_gradient = QLinearGradient(0, 0, 0, 1);
        m_gradient.setCoordinateMode(QGradient::ObjectMode);
        m_gradient.setColorAt(0, m_topColor);
        m_gradient.setColorAt(1, m_bottomColor);
    }
    
    int m_radius = 10;
    QColor m_topColor{Qt::cyan};
    QColor m_bottomColor{Qt::blue};
    QLinearGradient m_gradient;
};
```

---

## ขั้นตอนที่ 435: Plugin Loader in App

```cpp
// === plugin_manager.h ===
#pragma once
#include <QMap>
#include <QList>
#include <QDir>
#include <QPluginLoader>
#include "../shapes_interface.h"

class PluginManager : public QObject {
    Q_OBJECT
    
public:
    explicit PluginManager(QObject* parent = nullptr) : QObject(parent) {}
    
    // Load all plugins from directory
    int loadPluginsFromDir(const QString& dirPath) {
        QDir dir(dirPath);
        if (!dir.exists()) {
            qWarning() << "Plugin dir not found:" << dirPath;
            return 0;
        }
        
        int loaded = 0;
        
        // Filter shared libraries
        QStringList filters;
#ifdef Q_OS_WIN
        filters << "*.dll";
#elif defined(Q_OS_MAC)
        filters << "*.dylib";
#else
        filters << "*.so";
#endif
        
        for (const QString& fileName : dir.entryList(filters, QDir::Files)) {
            QString filePath = dir.absoluteFilePath(fileName);
            
            if (loadPlugin(filePath)) {
                loaded++;
                qDebug() << "Loaded plugin:" << fileName;
            } else {
                qWarning() << "Failed to load:" << fileName;
            }
        }
        
        emit pluginsReloaded();
        return loaded;
    }
    
    bool loadPlugin(const QString& filePath) {
        auto* loader = new QPluginLoader(filePath, this);
        
        QJsonObject meta = loader->metaData();
        if (!meta.contains("IID")) {
            delete loader;
            return false;
        }
        
        QObject* obj = loader->instance();
        if (!obj) {
            qWarning() << "Plugin load error:" << loader->errorString();
            delete loader;
            return false;
        }
        
        auto* plugin = qobject_cast<ShapePlugin*>(obj);
        if (!plugin) {
            loader->unload();
            delete loader;
            return false;
        }
        
        m_plugins[plugin->name()] = plugin;
        m_loaders[plugin->name()] = loader;
        
        emit pluginLoaded(plugin->name());
        return true;
    }
    
    void unloadPlugin(const QString& name) {
        if (!m_loaders.contains(name)) return;
        
        m_plugins.remove(name);
        
        auto* loader = m_loaders.take(name);
        loader->unload();
        delete loader;
        
        emit pluginUnloaded(name);
    }
    
    QStringList availablePlugins() const {
        return m_plugins.keys();
    }
    
    ShapePlugin* plugin(const QString& name) const {
        return m_plugins.value(name, nullptr);
    }
    
    QList<ShapePlugin*> allPlugins() const {
        return m_plugins.values();
    }
    
signals:
    void pluginLoaded(const QString& name);
    void pluginUnloaded(const QString& name);
    void pluginsReloaded();
    
private:
    QMap<QString, ShapePlugin*> m_plugins;
    QMap<QString, QPluginLoader*> m_loaders;
};
```

---

## ขั้นตอนที่ 436-445: Plugin Demo Application

```cpp
// === Plugin Demo Main Window ===
class PluginDemoWindow : public QMainWindow {
    Q_OBJECT
    
public:
    PluginDemoWindow() {
        setWindowTitle("Shape Plugin Demo");
        setMinimumSize(900, 600);
        
        manager = new PluginManager(this);
        
        setupUi();
        loadPlugins();
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* mainLayout = new QHBoxLayout(central);
        setCentralWidget(central);
        
        // Left panel: plugin list
        auto* leftPanel = new QWidget();
        leftPanel->setMaximumWidth(250);
        auto* leftLayout = new QVBoxLayout(leftPanel);
        
        leftLayout->addWidget(new QLabel("Available Plugins:"));
        pluginList = new QListWidget();
        leftLayout->addWidget(pluginList);
        
        auto* reloadBtn = new QPushButton("Reload Plugins");
        leftLayout->addWidget(reloadBtn);
        
        // Config panel
        configGroup = new QGroupBox("Configuration");
        configLayout = new QVBoxLayout(configGroup);
        leftLayout->addWidget(configGroup);
        
        // Right panel: canvas
        canvas = new ShapeCanvas(this);
        
        mainLayout->addWidget(leftPanel);
        mainLayout->addWidget(canvas, 1);
        
        // Toolbar
        auto* toolbar = addToolBar("Actions");
        auto* loadDirBtn = toolbar->addAction("Load from Dir");
        
        toolbar->addSeparator();
        areaLabel = new QLabel("Area: -");
        toolbar->addWidget(areaLabel);
        
        // Connections
        connect(pluginList, &QListWidget::currentTextChanged, 
                this, &PluginDemoWindow::onPluginSelected);
        
        connect(reloadBtn, &QPushButton::clicked, this, &PluginDemoWindow::loadPlugins);
        
        connect(loadDirBtn, &QAction::triggered, [this]() {
            QString dir = QFileDialog::getExistingDirectory(this, "Plugin Directory");
            if (!dir.isEmpty()) {
                int n = manager->loadPluginsFromDir(dir);
                QMessageBox::information(this, "Plugins", 
                    QString("Loaded %1 plugins").arg(n));
                refreshPluginList();
            }
        });
        
        connect(manager, &PluginManager::pluginLoaded, [this](const QString& name) {
            refreshPluginList();
        });
    }
    
    void loadPlugins() {
        // Load from app's plugins directory
        QString pluginDir = QApplication::applicationDirPath() + "/plugins";
        manager->loadPluginsFromDir(pluginDir);
        refreshPluginList();
    }
    
    void refreshPluginList() {
        pluginList->clear();
        for (const QString& name : manager->availablePlugins()) {
            pluginList->addItem(name);
        }
    }
    
    void onPluginSelected(const QString& name) {
        auto* plugin = manager->plugin(name);
        if (!plugin) return;
        
        canvas->setPlugin(plugin);
        
        // Show config widget
        QLayoutItem* item;
        while ((item = configLayout->takeAt(0)) != nullptr) {
            delete item->widget();
            delete item;
        }
        
        QWidget* config = plugin->configWidget(this);
        if (config) {
            configGroup->setTitle(name + " Settings");
            configLayout->addWidget(config);
        }
        
        // Update area label
        QRectF r(0, 0, 200, 150);
        areaLabel->setText(QString("Area: %1").arg(plugin->area(r), 0, 'f', 1));
    }
    
    PluginManager* manager;
    QListWidget* pluginList;
    QGroupBox* configGroup;
    QVBoxLayout* configLayout;
    QLabel* areaLabel;
    ShapeCanvas* canvas;
};

// === Shape Canvas ===
class ShapeCanvas : public QWidget {
    Q_OBJECT
    
public:
    ShapeCanvas(QWidget* parent = nullptr) : QWidget(parent) {
        setMinimumSize(300, 300);
        setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Expanding);
        setStyleSheet("background: white; border: 1px solid #ccc;");
    }
    
    void setPlugin(ShapePlugin* plugin) {
        m_plugin = plugin;
        update();
    }
    
protected:
    void paintEvent(QPaintEvent*) override {
        QPainter painter(this);
        painter.fillRect(rect(), Qt::white);
        
        if (!m_plugin) {
            painter.drawText(rect(), Qt::AlignCenter, "Select a plugin");
            return;
        }
        
        // Draw shape centered with margin
        int margin = 40;
        QRectF shapeRect(margin, margin, 
                         width() - 2*margin, height() - 2*margin);
        
        m_plugin->draw(&painter, shapeRect);
        
        // Draw info
        painter.setPen(Qt::black);
        painter.drawText(10, 20, m_plugin->name() + " v" + m_plugin->version());
    }
    
private:
    ShapePlugin* m_plugin = nullptr;
};
```

---

## สรุป Part 031

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt Plugin architecture overview
2. ✅ กำหนด Plugin Interface ด้วย Q_DECLARE_INTERFACE
3. ✅ Implement plugins ด้วย Q_PLUGIN_METADATA
4. ✅ PluginManager สำหรับ load/unload plugins
5. ✅ Plugin demo application

---

⬅️ [Part 030](part030.md) | ➡️ [Part 032: Qt Network Advanced](part032.md)
