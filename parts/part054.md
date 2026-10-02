# Part 054: Capstone — ERP Dashboard & Reports

## ขั้นตอนที่ 776-790

---

## ขั้นตอนที่ 776: Dashboard KPI Cards

```cpp
// ui/dashboard/kpicard.h
#pragma once
#include <QWidget>
#include <QPainter>
#include <QPropertyAnimation>

class KpiCard : public QWidget {
    Q_OBJECT
    Q_PROPERTY(double value READ value WRITE setValue NOTIFY valueChanged)
    
public:
    struct Config {
        QString title;
        QString icon;          // Material icon name
        QColor accentColor;
        QString prefix;        // e.g. "฿"
        QString suffix;        // e.g. " รายการ"
        int decimals = 0;
    };
    
    explicit KpiCard(const Config& config, QWidget* parent = nullptr)
        : QWidget(parent), m_config(config)
    {
        setMinimumSize(200, 120);
        setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Fixed);
        
        m_anim = new QPropertyAnimation(this, "value", this);
        m_anim->setDuration(800);
        m_anim->setEasingCurve(QEasingCurve::OutCubic);
    }
    
    double value() const { return m_value; }
    
    void setValue(double v) {
        if (qFuzzyCompare(m_value, v)) return;
        m_value = v;
        update();
        emit valueChanged(v);
    }
    
    void animateTo(double target) {
        m_anim->stop();
        m_anim->setStartValue(m_value);
        m_anim->setEndValue(target);
        m_anim->start();
    }
    
signals:
    void valueChanged(double v);
    
protected:
    void paintEvent(QPaintEvent*) override {
        QPainter p(this);
        p.setRenderHint(QPainter::Antialiasing);
        
        QRectF r(1, 1, width() - 2, height() - 2);
        
        // Card background
        p.setPen(Qt::NoPen);
        p.setBrush(palette().window());
        p.drawRoundedRect(r, 12, 12);
        
        // Left accent bar
        p.setBrush(m_config.accentColor);
        p.drawRoundedRect(QRectF(1, 1, 6, height() - 2), 3, 3);
        
        // Title
        p.setPen(QColor(150, 150, 150));
        p.setFont(QFont("Segoe UI", 10));
        p.drawText(QRectF(20, 12, width() - 30, 20),
                   Qt::AlignLeft | Qt::AlignVCenter, m_config.title);
        
        // Value
        p.setPen(m_config.accentColor);
        
        QString displayValue = m_config.prefix +
            QString("%L1").arg(m_value, 0, 'f', m_config.decimals) +
            m_config.suffix;
        
        // Auto-scale font
        QFont valueFont("Segoe UI", 22, QFont::Bold);
        p.setFont(valueFont);
        
        QFontMetrics fm(valueFont);
        while (fm.horizontalAdvance(displayValue) > width() - 40 && valueFont.pointSize() > 12) {
            valueFont.setPointSize(valueFont.pointSize() - 1);
            p.setFont(valueFont);
            fm = QFontMetrics(valueFont);
        }
        
        p.drawText(QRectF(20, 35, width() - 30, height() - 50),
                   Qt::AlignLeft | Qt::AlignVCenter, displayValue);
        
        // Border
        p.setPen(QPen(m_config.accentColor.lighter(150), 1));
        p.setBrush(Qt::NoBrush);
        p.drawRoundedRect(r, 12, 12);
    }
    
private:
    Config m_config;
    double m_value = 0;
    QPropertyAnimation* m_anim;
};
```

---

## ขั้นตอนที่ 777: Sales Chart Widget

```cpp
// ui/dashboard/saleschart.h
#pragma once
#include <QtCharts>
#include <QDateEdit>

class SalesChart : public QWidget {
    Q_OBJECT
    
public:
    explicit SalesChart(SalesOrderRepository* repo, QWidget* parent = nullptr)
        : QWidget(parent), m_repo(repo)
    {
        auto* layout = new QVBoxLayout(this);
        
        // Period selector
        auto* toolbar = new QHBoxLayout;
        
        m_periodCombo = new QComboBox(this);
        m_periodCombo->addItems({"สัปดาห์นี้", "เดือนนี้", "ไตรมาสนี้", "ปีนี้"});
        m_periodCombo->setCurrentIndex(1);
        
        auto* refreshBtn = new QPushButton(QIcon::fromTheme("view-refresh"), "", this);
        refreshBtn->setToolTip("รีเฟรช");
        
        toolbar->addStretch();
        toolbar->addWidget(new QLabel("ช่วงเวลา:", this));
        toolbar->addWidget(m_periodCombo);
        toolbar->addWidget(refreshBtn);
        
        // Chart
        m_chart = new QChart;
        m_chart->setTitle("ยอดขายตามช่วงเวลา");
        m_chart->setTheme(QChart::ChartThemeDark);
        m_chart->legend()->setVisible(true);
        m_chart->legend()->setAlignment(Qt::AlignBottom);
        m_chart->setAnimationOptions(QChart::SeriesAnimations);
        
        m_chartView = new QChartView(m_chart, this);
        m_chartView->setRenderHint(QPainter::Antialiasing);
        m_chartView->setMinimumHeight(280);
        
        layout->addLayout(toolbar);
        layout->addWidget(m_chartView);
        
        connect(m_periodCombo, &QComboBox::currentIndexChanged, this, &SalesChart::reload);
        connect(refreshBtn, &QPushButton::clicked, this, &SalesChart::reload);
        
        reload();
    }
    
public slots:
    void reload() {
        m_chart->removeAllSeries();
        
        // Clear old axes
        for (auto* axis : m_chart->axes()) {
            m_chart->removeAxis(axis);
            delete axis;
        }
        
        auto [fromDate, toDate, groupBy] = getPeriod();
        
        // Load data from DB
        auto data = loadSalesData(fromDate, toDate, groupBy);
        
        auto* barSet = new QBarSet("ยอดขาย (บาท)");
        barSet->setColor(QColor("#3498DB"));
        
        QStringList categories;
        
        for (const auto& [label, amount] : data) {
            *barSet << amount;
            categories << label;
        }
        
        auto* series = new QBarSeries;
        series->append(barSet);
        m_chart->addSeries(series);
        
        auto* axisX = new QBarCategoryAxis;
        axisX->append(categories);
        axisX->setLabelsAngle(-30);
        m_chart->addAxis(axisX, Qt::AlignBottom);
        series->attachAxis(axisX);
        
        auto* axisY = new QValueAxis;
        axisY->setTitleText("บาท");
        axisY->setLabelFormat("%',.0f");
        m_chart->addAxis(axisY, Qt::AlignLeft);
        series->attachAxis(axisY);
        
        // Tooltip on hover
        connect(series, &QBarSeries::hovered,
                this, [=](bool status, int index, QBarSet* barSet) {
                    if (status && index < categories.size()) {
                        m_chartView->setToolTip(
                            QString("%1\n฿%L2")
                                .arg(categories[index])
                                .arg(barSet->at(index), 0, 'f', 2));
                    }
                });
    }
    
private:
    struct Period {
        QDate from, to;
        QString groupBy;  // "day", "week", "month"
    };
    
    Period getPeriod() {
        QDate today = QDate::currentDate();
        
        switch (m_periodCombo->currentIndex()) {
            case 0: return {today.addDays(-6), today, "day"};
            case 1: {
                QDate from(today.year(), today.month(), 1);
                return {from, today, "day"};
            }
            case 2: {
                int qStart = ((today.month() - 1) / 3) * 3 + 1;
                QDate from(today.year(), qStart, 1);
                return {from, today, "week"};
            }
            case 3: return {QDate(today.year(), 1, 1), today, "month"};
            default: return {today.addDays(-30), today, "day"};
        }
    }
    
    QList<QPair<QString, double>> loadSalesData(
        const QDate& from, const QDate& to, const QString& groupBy)
    {
        QString dateFormat;
        if (groupBy == "day")   dateFormat = "%Y-%m-%d";
        else if (groupBy == "week") dateFormat = "%Y-W%W";
        else                    dateFormat = "%Y-%m";
        
        auto rows = Database::instance().fetchAll(
            QString("SELECT strftime('%1', order_date) AS period,"
                    " SUM(total) AS total"
                    " FROM sales_orders"
                    " WHERE status NOT IN ('cancelled','draft')"
                    " AND order_date BETWEEN ? AND ?"
                    " GROUP BY period ORDER BY period")
                .arg(dateFormat),
            {from.toString(Qt::ISODate), to.toString(Qt::ISODate)}
        );
        
        QList<QPair<QString, double>> result;
        for (const auto& row : rows) {
            result.append({row["period"].toString(), row["total"].toDouble()});
        }
        return result;
    }
    
    SalesOrderRepository* m_repo;
    QChart* m_chart;
    QChartView* m_chartView;
    QComboBox* m_periodCombo;
};
```

---

## ขั้นตอนที่ 778: Main ERP Dashboard

```cpp
// ui/dashboard/erpdashboard.h
#pragma once
#include <QScrollArea>
#include <QGridLayout>
#include "kpicard.h"
#include "saleschart.h"

class ErpDashboard : public QWidget {
    Q_OBJECT
    
public:
    ErpDashboard(
        CustomerRepository* customers,
        ProductRepository* products,
        SalesOrderRepository* orders,
        QWidget* parent = nullptr)
        : QWidget(parent)
        , m_customers(customers)
        , m_products(products)
        , m_orders(orders)
    {
        setupUi();
        
        // Auto-refresh every minute
        auto* refreshTimer = new QTimer(this);
        connect(refreshTimer, &QTimer::timeout, this, &ErpDashboard::refreshKpis);
        refreshTimer->start(60000);
        
        refreshKpis();
    }
    
public slots:
    void refreshKpis() {
        // Sales KPIs
        m_salesCard->animateTo(m_orders->totalSalesThisMonth());
        m_ordersCard->animateTo(m_orders->orderCountByStatus("confirmed")
                               + m_orders->orderCountByStatus("shipped"));
        m_customersCard->animateTo(m_customers->count(true));
        m_lowStockCard->animateTo(m_products->lowStockProducts().size());
        
        m_salesChart->reload();
        refreshTopProducts();
        refreshRecentOrders();
    }
    
private:
    void setupUi() {
        auto* scroll = new QScrollArea(this);
        scroll->setWidgetResizable(true);
        scroll->setFrameStyle(QFrame::NoFrame);
        
        auto* content = new QWidget;
        scroll->setWidget(content);
        
        auto* mainLayout = new QVBoxLayout(this);
        mainLayout->addWidget(scroll);
        mainLayout->setContentsMargins(0, 0, 0, 0);
        
        auto* vlay = new QVBoxLayout(content);
        vlay->setSpacing(16);
        vlay->setContentsMargins(16, 16, 16, 16);
        
        // Header
        auto* header = new QLabel("แดชบอร์ด", content);
        header->setStyleSheet("font-size: 24px; font-weight: bold;");
        vlay->addWidget(header);
        
        // KPI Cards row
        auto* kpiRow = new QHBoxLayout;
        
        m_salesCard = new KpiCard({
            "ยอดขายเดือนนี้", "attach_money",
            QColor("#3498DB"), "฿", "", 2
        }, content);
        
        m_ordersCard = new KpiCard({
            "คำสั่งซื้อที่รอดำเนินการ", "shopping_cart",
            QColor("#2ECC71"), "", " รายการ", 0
        }, content);
        
        m_customersCard = new KpiCard({
            "ลูกค้าทั้งหมด", "people",
            QColor("#9B59B6"), "", " ราย", 0
        }, content);
        
        m_lowStockCard = new KpiCard({
            "สินค้าใกล้หมดสต็อก", "warning",
            QColor("#E74C3C"), "", " รายการ", 0
        }, content);
        
        kpiRow->addWidget(m_salesCard);
        kpiRow->addWidget(m_ordersCard);
        kpiRow->addWidget(m_customersCard);
        kpiRow->addWidget(m_lowStockCard);
        vlay->addLayout(kpiRow);
        
        // Sales Chart
        m_salesChart = new SalesChart(m_orders, content);
        vlay->addWidget(m_salesChart);
        
        // Bottom row: Top Products + Recent Orders
        auto* bottomRow = new QHBoxLayout;
        
        // Top products
        auto* topProductsGroup = new QGroupBox("สินค้าขายดี", content);
        auto* tpLayout = new QVBoxLayout(topProductsGroup);
        m_topProductsTable = new QTableWidget(0, 3, topProductsGroup);
        m_topProductsTable->setHorizontalHeaderLabels({"สินค้า", "จำนวน", "ยอดรวม"});
        m_topProductsTable->horizontalHeader()->setSectionResizeMode(0, QHeaderView::Stretch);
        m_topProductsTable->setEditTriggers(QAbstractItemView::NoEditTriggers);
        m_topProductsTable->setMaximumHeight(200);
        tpLayout->addWidget(m_topProductsTable);
        
        // Recent orders
        auto* recentOrdersGroup = new QGroupBox("คำสั่งซื้อล่าสุด", content);
        auto* roLayout = new QVBoxLayout(recentOrdersGroup);
        m_recentOrdersTable = new QTableWidget(0, 4, recentOrdersGroup);
        m_recentOrdersTable->setHorizontalHeaderLabels({"เลขที่", "ลูกค้า", "ยอดรวม", "สถานะ"});
        m_recentOrdersTable->horizontalHeader()->setSectionResizeMode(1, QHeaderView::Stretch);
        m_recentOrdersTable->setEditTriggers(QAbstractItemView::NoEditTriggers);
        m_recentOrdersTable->setMaximumHeight(200);
        roLayout->addWidget(m_recentOrdersTable);
        
        bottomRow->addWidget(topProductsGroup);
        bottomRow->addWidget(recentOrdersGroup);
        vlay->addLayout(bottomRow);
        vlay->addStretch();
    }
    
    void refreshTopProducts() {
        auto rows = Database::instance().fetchAll(R"(
            SELECT p.name, SUM(soi.qty) AS total_qty, SUM(soi.line_total) AS total_amount
            FROM sales_order_items soi
            JOIN products p ON p.id = soi.product_id
            JOIN sales_orders so ON so.id = soi.order_id
            WHERE so.status NOT IN ('cancelled','draft')
              AND so.order_date >= date('now', '-30 days')
            GROUP BY p.id
            ORDER BY total_amount DESC
            LIMIT 10
        )");
        
        m_topProductsTable->setRowCount(0);
        for (const auto& row : rows) {
            int r = m_topProductsTable->rowCount();
            m_topProductsTable->insertRow(r);
            m_topProductsTable->setItem(r, 0, new QTableWidgetItem(row["name"].toString()));
            m_topProductsTable->setItem(r, 1, new QTableWidgetItem(
                QString::number(row["total_qty"].toDouble(), 'f', 0)));
            m_topProductsTable->setItem(r, 2, new QTableWidgetItem(
                QString("฿%L1").arg(row["total_amount"].toDouble(), 0, 'f', 2)));
        }
    }
    
    void refreshRecentOrders() {
        auto orders = m_orders->all({}, 10);
        
        m_recentOrdersTable->setRowCount(0);
        for (const auto& order : orders) {
            int r = m_recentOrdersTable->rowCount();
            m_recentOrdersTable->insertRow(r);
            m_recentOrdersTable->setItem(r, 0, new QTableWidgetItem(order.orderNo));
            m_recentOrdersTable->setItem(r, 1, new QTableWidgetItem(order.customerName));
            m_recentOrdersTable->setItem(r, 2, new QTableWidgetItem(
                QString("฿%L1").arg(order.total, 0, 'f', 2)));
            
            auto* statusItem = new QTableWidgetItem(order.statusLabel());
            statusItem->setForeground(order.status == "paid" ? QColor("#2ECC71")
                                     : order.status == "cancelled" ? QColor("#E74C3C")
                                     : QColor("#F39C12"));
            m_recentOrdersTable->setItem(r, 3, statusItem);
        }
    }
    
    CustomerRepository* m_customers;
    ProductRepository* m_products;
    SalesOrderRepository* m_orders;
    
    KpiCard* m_salesCard;
    KpiCard* m_ordersCard;
    KpiCard* m_customersCard;
    KpiCard* m_lowStockCard;
    SalesChart* m_salesChart;
    QTableWidget* m_topProductsTable;
    QTableWidget* m_recentOrdersTable;
};
```

---

## ขั้นตอนที่ 779: PDF Invoice Generator

```cpp
// modules/sales/invoicegenerator.h
#pragma once
#include <QPrinter>
#include <QPainter>
#include <QTextDocument>
#include <QPageSize>

class InvoiceGenerator {
public:
    static bool generatePdf(const SalesOrder& order, const QString& outputPath) {
        QString html = buildHtml(order);
        
        QTextDocument doc;
        doc.setDefaultStyleSheet(R"(
            body { font-family: 'TH Sarabun New', 'Segoe UI'; font-size: 13pt; }
            .header { text-align: center; margin-bottom: 20px; }
            .company-name { font-size: 20pt; font-weight: bold; color: #2C3E50; }
            .invoice-title { font-size: 16pt; color: #3498DB; }
            table { width: 100%; border-collapse: collapse; }
            th { background: #2C3E50; color: white; padding: 8px; text-align: left; }
            td { padding: 6px 8px; border-bottom: 1px solid #ddd; }
            .amount { text-align: right; }
            .total-row { font-weight: bold; background: #ECF0F1; }
            .footer { margin-top: 30px; font-size: 11pt; color: #666; }
        )");
        doc.setHtml(html);
        
        QPrinter printer(QPrinter::HighResolution);
        printer.setOutputFormat(QPrinter::PdfFormat);
        printer.setOutputFileName(outputPath);
        printer.setPageSize(QPageSize(QPageSize::A4));
        printer.setPageMargins(QMarginsF(15, 15, 15, 15), QPageLayout::Millimeter);
        
        doc.print(&printer);
        return QFile::exists(outputPath);
    }
    
    static QString buildHtml(const SalesOrder& order) {
        QString itemRows;
        int no = 1;
        for (const auto& item : order.items) {
            itemRows += QString(R"(
                <tr>
                    <td>%1</td>
                    <td>%2</td>
                    <td class="amount">%3</td>
                    <td class="amount">%4</td>
                    <td class="amount">%5%%</td>
                    <td class="amount">%6</td>
                </tr>
            )").arg(no++)
               .arg(item.productName)
               .arg(item.unit)
               .arg(QString::number(item.qty, 'f', 2))
               .arg(QString("%L1").arg(item.unitPrice, 0, 'f', 2))
               .arg(item.discount)
               .arg(QString("%L1").arg(item.lineTotal, 0, 'f', 2));
        }
        
        double discountAmt = order.subtotal * order.discountPct / 100.0;
        double taxAmt = (order.subtotal - discountAmt) * order.taxPct / 100.0;
        
        return QString(R"html(
            <html><body>
            <div class="header">
                <div class="company-name">บริษัท ตัวอย่าง จำกัด</div>
                <div>123 ถนนสุขุมวิท กรุงเทพฯ 10110</div>
                <div>โทร: 02-xxx-xxxx | อีเมล: info@example.co.th</div>
                <div>เลขประจำตัวผู้เสียภาษี: 0-1234-56789-01-2</div>
                <br>
                <div class="invoice-title">ใบเสนอราคา / QUOTATION</div>
            </div>
            
            <table style="margin-bottom:15px">
                <tr>
                    <td width="50%"><b>ลูกค้า:</b> %1</td>
                    <td><b>เลขที่:</b> %2</td>
                </tr>
                <tr>
                    <td></td>
                    <td><b>วันที่:</b> %3</td>
                </tr>
                <tr>
                    <td></td>
                    <td><b>วันครบกำหนด:</b> %4</td>
                </tr>
            </table>
            
            <table>
                <thead>
                    <tr>
                        <th width="5%%">ลำดับ</th>
                        <th>รายการ</th>
                        <th width="8%%">หน่วย</th>
                        <th width="10%%">จำนวน</th>
                        <th width="12%%">ราคา/หน่วย</th>
                        <th width="8%%">ส่วนลด</th>
                        <th width="12%%">จำนวนเงิน</th>
                    </tr>
                </thead>
                <tbody>
                    %5
                </tbody>
            </table>
            
            <table style="margin-top:15px; width:40%%; margin-left:60%%">
                <tr><td>ยอดรวม:</td><td class="amount">%L6</td></tr>
                <tr><td>ส่วนลด (%7%%):</td><td class="amount">-%L8</td></tr>
                <tr><td>ภาษีมูลค่าเพิ่ม (%9%%):</td><td class="amount">%L10</td></tr>
                <tr class="total-row">
                    <td>ยอดสุทธิ:</td>
                    <td class="amount">฿%L11</td>
                </tr>
            </table>
            
            <div class="footer">
                หมายเหตุ: %12<br><br>
                ลงชื่อ _________________________ ผู้เสนอราคา<br>
                วันที่ _________________________
            </div>
            </body></html>
        )html")
            .arg(order.customerName, order.orderNo)
            .arg(order.orderDate.toString("dd/MM/yyyy"))
            .arg(order.dueDate.toString("dd/MM/yyyy"))
            .arg(itemRows)
            .arg(order.subtotal, 0, 'f', 2)
            .arg(order.discountPct)
            .arg(discountAmt, 0, 'f', 2)
            .arg(order.taxPct)
            .arg(taxAmt, 0, 'f', 2)
            .arg(order.total, 0, 'f', 2)
            .arg(order.notes);
    }
};
```

---

## สรุป Part 054

ใน Part นี้คุณได้เรียนรู้:

1. ✅ KpiCard ด้วย QPainter + QPropertyAnimation (animateTo)
2. ✅ SalesChart ด้วย QBarSeries + QBarCategoryAxis + tooltips
3. ✅ ErpDashboard with scroll area, KPIs, chart, top products, recent orders
4. ✅ InvoiceGenerator ที่สร้าง PDF ด้วย QTextDocument + QPrinter
5. ✅ Auto-refresh dashboard ทุก 60 วินาที

---

⬅️ [Part 053](part053.md) | ➡️ [Part 055: Capstone — ERP Main Window & Navigation](part055.md)
