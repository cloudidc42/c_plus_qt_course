# Part 056: World-Class — Qt Plugin Architecture

## ขั้นตอนที่ 806-820

---

## ขั้นตอนที่ 806: Qt Plugin System Overview

```
Qt Plugin System:
  - QPluginLoader: โหลด .dll/.so ขณะ runtime
  - Q_DECLARE_INTERFACE: ประกาศ interface ID
  - Q_INTERFACES: บอก Qt ว่า class implement interface ไหน
  - Q_PLUGIN_METADATA: metadata JSON สำหรับ plugin

Use Cases:
  - Theme/skin plugins
  - Report generator plugins
  - Data source connectors
  - Widget extensions

Plugin Project Structure:
  main_app/
  ├── interfaces/          ← shared interface headers (no .cpp)
  │   ├── IPlugin.h
  │   ├── IReportPlugin.h
  │   └── IThemePlugin.h
  ├── app/                 ← main application
  └── plugins/
      ├── report_pdf/      ← PDF report plugin
      ├── report_excel/    ← Excel report plugin
      └── theme_dark/      ← Dark theme plugin
```

---

## ขั้นตอนที่ 807: Plugin Interface Definition

```cpp
// interfaces/IPlugin.h
#pragma once
#include <QObject>
#include <QString>
#include <QWidget>
#include <QIcon>

#define IPlugin_iid "com.mycompany.erp.IPlugin/1.0"

class IPlugin {
public:
    virtual ~IPlugin() = default;
    
    virtual QString id() const = 0;
    virtual QString name() const = 0;
    virtual QString version() const = 0;
    virtual QString description() const = 0;
    virtual QIcon icon() const { return {}; }
    
    virtual void initialize(QObject* appContext) = 0;
    virtual void shutdown() = 0;
    
    virtual bool isCompatible(const QString& appVersion) const {
        Q_UNUSED(appVersion)
        return true;
    }
};

Q_DECLARE_INTERFACE(IPlugin, IPlugin_iid)

// interfaces/IReportPlugin.h
#define IReportPlugin_iid "com.mycompany.erp.IReportPlugin/1.0"

class IReportPlugin : public IPlugin {
public:
    enum OutputFormat { PDF, HTML, Excel, CSV };
    
    virtual QList<OutputFormat> supportedFormats() const = 0;
    virtual QStringList availableReports() const = 0;
    
    struct ReportParams {
        QString reportName;
        QVariantMap filters;
        OutputFormat format;
        QString outputPath;
    };
    
    virtual bool generateReport(const ReportParams& params) = 0;
    virtual QWidget* configWidget(QWidget* parent = nullptr) { Q_UNUSED(parent) return nullptr; }
};

Q_DECLARE_INTERFACE(IReportPlugin, IReportPlugin_iid)

// interfaces/IThemePlugin.h
#define IThemePlugin_iid "com.mycompany.erp.IThemePlugin/1.0"

class IThemePlugin : public IPlugin {
public:
    virtual void applyTheme(QApplication* app) = 0;
    virtual void removeTheme(QApplication* app) = 0;
    virtual QString stylesheetPath() const = 0;
    virtual QMap<QString, QColor> colorPalette() const = 0;
};

Q_DECLARE_INTERFACE(IThemePlugin, IThemePlugin_iid)
```

---

## ขั้นตอนที่ 808: Implementing a Plugin

```cpp
// plugins/report_pdf/reportpdfplugin.h
#pragma once
#include <QObject>
#include "../../interfaces/IReportPlugin.h"

class ReportPdfPlugin : public QObject, public IReportPlugin {
    Q_OBJECT
    Q_INTERFACES(IPlugin IReportPlugin)
    Q_PLUGIN_METADATA(IID IReportPlugin_iid FILE "metadata.json")
    
public:
    // IPlugin
    QString id()          const override { return "report_pdf"; }
    QString name()        const override { return "PDF Report Generator"; }
    QString version()     const override { return "1.0.0"; }
    QString description() const override { return "Generates reports in PDF format"; }
    
    void initialize(QObject* appContext) override {
        m_appContext = appContext;
    }
    
    void shutdown() override {
        m_appContext = nullptr;
    }
    
    // IReportPlugin
    QList<OutputFormat> supportedFormats() const override {
        return {PDF};
    }
    
    QStringList availableReports() const override {
        return {
            "sales_summary",
            "customer_list",
            "product_stock",
            "invoice"
        };
    }
    
    bool generateReport(const ReportParams& params) override {
        if (params.reportName == "sales_summary") {
            return generateSalesSummary(params);
        } else if (params.reportName == "invoice") {
            return generateInvoice(params);
        }
        return false;
    }
    
    QWidget* configWidget(QWidget* parent) override {
        auto* w = new QWidget(parent);
        auto* lay = new QFormLayout(w);
        
        auto* pageSize = new QComboBox(w);
        pageSize->addItems({"A4", "Letter", "A3"});
        
        auto* orientation = new QComboBox(w);
        orientation->addItems({"แนวตั้ง", "แนวนอน"});
        
        auto* dpi = new QSpinBox(w);
        dpi->setRange(72, 600);
        dpi->setValue(300);
        
        lay->addRow("ขนาดกระดาษ:", pageSize);
        lay->addRow("การวางแนว:", orientation);
        lay->addRow("ความละเอียด (DPI):", dpi);
        
        return w;
    }
    
private:
    bool generateSalesSummary(const ReportParams& params) {
        if (params.outputPath.isEmpty()) return false;
        
        // Get data from context
        QDate from = params.filters["dateFrom"].toDate();
        QDate to   = params.filters["dateTo"].toDate();
        
        auto rows = Database::instance().fetchAll(R"(
            SELECT so.order_no, c.name AS customer, so.order_date, so.total, so.status
            FROM sales_orders so
            LEFT JOIN customers c ON c.id = so.customer_id
            WHERE so.order_date BETWEEN ? AND ?
            ORDER BY so.order_date DESC
        )", {from.toString(Qt::ISODate), to.toString(Qt::ISODate)});
        
        // Build HTML
        QString rowsHtml;
        double grandTotal = 0;
        
        for (const auto& row : rows) {
            double total = row["total"].toDouble();
            grandTotal += total;
            rowsHtml += QString("<tr><td>%1</td><td>%2</td><td>%3</td>"
                               "<td style='text-align:right'>%L4</td><td>%5</td></tr>")
                .arg(row["order_no"].toString())
                .arg(row["customer"].toString())
                .arg(QDate::fromString(row["order_date"].toString(), Qt::ISODate)
                     .toString("dd/MM/yyyy"))
                .arg(total, 0, 'f', 2)
                .arg(row["status"].toString());
        }
        
        QString html = QString(R"(
            <html><body>
            <h2>สรุปยอดขาย %1 — %2</h2>
            <table border="1" cellpadding="5" width="100%%">
                <tr bgcolor="#2C3E50" style="color:white">
                    <th>เลขที่</th><th>ลูกค้า</th><th>วันที่</th>
                    <th>ยอดรวม</th><th>สถานะ</th>
                </tr>
                %3
                <tr bgcolor="#ECF0F1">
                    <td colspan="3"><b>รวมทั้งสิ้น</b></td>
                    <td style='text-align:right'><b>%L4</b></td><td></td>
                </tr>
            </table>
            </body></html>
        )").arg(from.toString("dd/MM/yyyy"))
           .arg(to.toString("dd/MM/yyyy"))
           .arg(rowsHtml)
           .arg(grandTotal, 0, 'f', 2);
        
        QTextDocument doc;
        doc.setHtml(html);
        
        QPrinter printer(QPrinter::HighResolution);
        printer.setOutputFormat(QPrinter::PdfFormat);
        printer.setOutputFileName(params.outputPath);
        printer.setPageSize(QPageSize(QPageSize::A4));
        
        doc.print(&printer);
        return true;
    }
    
    bool generateInvoice(const ReportParams& params) {
        int orderId = params.filters["orderId"].toInt();
        SalesOrderRepository repo;
        SalesOrder order = repo.findById(orderId);
        return InvoiceGenerator::generatePdf(order, params.outputPath);
    }
    
    QObject* m_appContext = nullptr;
};
```

```json
// plugins/report_pdf/metadata.json
{
    "IID": "com.mycompany.erp.IReportPlugin/1.0",
    "className": "ReportPdfPlugin",
    "version": "1.0.0",
    "name": "PDF Report Generator",
    "dependencies": []
}
```

---

## ขั้นตอนที่ 809: Plugin Manager

```cpp
// app/pluginmanager.h
#pragma once
#include <QObject>
#include <QPluginLoader>
#include <QDir>
#include "../../interfaces/IPlugin.h"
#include "../../interfaces/IReportPlugin.h"
#include "../../interfaces/IThemePlugin.h"

class PluginManager : public QObject {
    Q_OBJECT
    
public:
    static PluginManager& instance() {
        static PluginManager mgr;
        return mgr;
    }
    
    void loadPlugins(const QString& pluginDir) {
        QDir dir(pluginDir);
        
        // Platform-specific library suffix
        QString filter;
#ifdef Q_OS_WIN
        filter = "*.dll";
#elif defined(Q_OS_MAC)
        filter = "*.dylib";
#else
        filter = "*.so";
#endif
        
        for (const QString& fileName : dir.entryList({filter}, QDir::Files)) {
            loadPlugin(dir.absoluteFilePath(fileName));
        }
        
        qInfo() << "Loaded" << m_plugins.size() << "plugins";
    }
    
    bool loadPlugin(const QString& filePath) {
        auto* loader = new QPluginLoader(filePath, this);
        
        QObject* instance = loader->instance();
        if (!instance) {
            qWarning() << "Cannot load plugin:" << filePath << loader->errorString();
            loader->deleteLater();
            return false;
        }
        
        auto* plugin = qobject_cast<IPlugin*>(instance);
        if (!plugin) {
            qWarning() << "Not a valid IPlugin:" << filePath;
            loader->unload();
            loader->deleteLater();
            return false;
        }
        
        if (!plugin->isCompatible(qApp->applicationVersion())) {
            qWarning() << "Incompatible plugin:" << plugin->name();
            loader->unload();
            loader->deleteLater();
            return false;
        }
        
        plugin->initialize(qApp);
        
        m_plugins[plugin->id()] = plugin;
        m_loaders[plugin->id()] = loader;
        
        emit pluginLoaded(plugin->id(), plugin->name());
        qInfo() << "Loaded plugin:" << plugin->name() << plugin->version();
        return true;
    }
    
    void unloadPlugin(const QString& id) {
        if (auto* plugin = m_plugins.value(id)) {
            plugin->shutdown();
            m_plugins.remove(id);
        }
        
        if (auto* loader = m_loaders.value(id)) {
            loader->unload();
            m_loaders.remove(id);
        }
        
        emit pluginUnloaded(id);
    }
    
    void unloadAll() {
        for (const auto& id : m_plugins.keys()) {
            unloadPlugin(id);
        }
    }
    
    IPlugin* plugin(const QString& id) const {
        return m_plugins.value(id);
    }
    
    QList<IPlugin*> allPlugins() const {
        return m_plugins.values();
    }
    
    template<typename T>
    QList<T*> pluginsOfType() const {
        QList<T*> result;
        for (auto* p : m_plugins.values()) {
            if (auto* typed = qobject_cast<T*>(p->asQObject())) {
                result.append(typed);
            }
        }
        return result;
    }
    
    QList<IReportPlugin*> reportPlugins() const {
        QList<IReportPlugin*> result;
        for (auto* p : m_plugins.values()) {
            if (auto* rp = dynamic_cast<IReportPlugin*>(p)) {
                result.append(rp);
            }
        }
        return result;
    }
    
signals:
    void pluginLoaded(const QString& id, const QString& name);
    void pluginUnloaded(const QString& id);
    
private:
    PluginManager() = default;
    
    QMap<QString, IPlugin*>       m_plugins;
    QMap<QString, QPluginLoader*> m_loaders;
};

// Plugin settings page
class PluginSettingsPage : public QWidget {
    Q_OBJECT
    
public:
    explicit PluginSettingsPage(QWidget* parent = nullptr) : QWidget(parent) {
        setupUi();
        refreshList();
        
        connect(&PluginManager::instance(), &PluginManager::pluginLoaded,
                this, &PluginSettingsPage::refreshList);
        connect(&PluginManager::instance(), &PluginManager::pluginUnloaded,
                this, &PluginSettingsPage::refreshList);
    }
    
private:
    void setupUi() {
        auto* lay = new QVBoxLayout(this);
        
        auto* header = new QLabel("จัดการปลั๊กอิน", this);
        header->setStyleSheet("font-size: 18px; font-weight: bold;");
        
        auto* toolbar = new QHBoxLayout;
        auto* loadBtn = new QPushButton(QIcon::fromTheme("list-add"), "โหลดปลั๊กอิน", this);
        toolbar->addWidget(loadBtn);
        toolbar->addStretch();
        
        m_table = new QTableWidget(0, 4, this);
        m_table->setHorizontalHeaderLabels({"ชื่อ", "เวอร์ชัน", "คำอธิบาย", "สถานะ"});
        m_table->horizontalHeader()->setSectionResizeMode(2, QHeaderView::Stretch);
        m_table->setSelectionBehavior(QAbstractItemView::SelectRows);
        m_table->setEditTriggers(QAbstractItemView::NoEditTriggers);
        
        lay->addWidget(header);
        lay->addLayout(toolbar);
        lay->addWidget(m_table);
        
        connect(loadBtn, &QPushButton::clicked, this, [=] {
            QString path = QFileDialog::getOpenFileName(
                this, "เลือกปลั๊กอิน", {},
#ifdef Q_OS_WIN
                "Plugin Files (*.dll)"
#elif defined(Q_OS_MAC)
                "Plugin Files (*.dylib)"
#else
                "Plugin Files (*.so)"
#endif
            );
            
            if (!path.isEmpty()) {
                PluginManager::instance().loadPlugin(path);
            }
        });
    }
    
    void refreshList() {
        m_table->setRowCount(0);
        
        for (const auto* plugin : PluginManager::instance().allPlugins()) {
            int row = m_table->rowCount();
            m_table->insertRow(row);
            
            m_table->setItem(row, 0, new QTableWidgetItem(plugin->name()));
            m_table->setItem(row, 1, new QTableWidgetItem(plugin->version()));
            m_table->setItem(row, 2, new QTableWidgetItem(plugin->description()));
            
            auto* statusItem = new QTableWidgetItem("โหลดแล้ว");
            statusItem->setForeground(QColor("#2ECC71"));
            m_table->setItem(row, 3, statusItem);
        }
    }
    
    QTableWidget* m_table;
};
```

---

## สรุป Part 056

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt Plugin ด้วย Q_DECLARE_INTERFACE + Q_PLUGIN_METADATA
2. ✅ IPlugin / IReportPlugin / IThemePlugin interfaces
3. ✅ Plugin implementation ที่สมบูรณ์ (ReportPdfPlugin)
4. ✅ PluginManager ด้วย QPluginLoader + lifecycle management
5. ✅ Plugin settings page ด้วย dynamic loading

---

⬅️ [Part 055](part055.md) | ➡️ [Part 057: World-Class — Advanced QML & Animations](part057.md)
