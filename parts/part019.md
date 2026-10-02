# Part 019: Qt Charts & Data Visualization

## ขั้นตอนที่ 251-265

---

## ขั้นตอนที่ 251: Qt Charts Module

**เพิ่มใน .pro:**
```qmake
QT += charts
```

```
Chart Types:
  ├── QLineSeries       - เส้นกราฟ
  ├── QSplineSeries     - เส้นโค้ง
  ├── QBarSeries        - แท่งกราฟ
  ├── QStackedBarSeries - แท่งซ้อน
  ├── QPieSeries        - วงกลม
  ├── QScatterSeries    - กระจาย
  ├── QAreaSeries       - พื้นที่
  └── QCandlestickSeries - หุ้น
```

---

## ขั้นตอนที่ 252: Line Chart

```cpp
#include <QChartView>
#include <QChart>
#include <QLineSeries>
#include <QSplineSeries>
#include <QValueAxis>
#include <QDateTimeAxis>
#include <QDateTime>

class LineChartDemo : public QWidget {
    Q_OBJECT
    
public:
    LineChartDemo(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("Line Chart");
        setMinimumSize(800, 500);
        
        // Create series
        auto* series1 = new QLineSeries();
        series1->setName("Revenue");
        
        auto* series2 = new QSplineSeries();
        series2->setName("Profit");
        
        // Add data points
        QList<QPair<double, double>> revenue = {
            {1, 120}, {2, 145}, {3, 132}, {4, 165}, {5, 180},
            {6, 155}, {7, 190}, {8, 220}, {9, 210}, {10, 240},
            {11, 235}, {12, 280}
        };
        
        for (const auto& [x, y] : revenue) {
            series1->append(x, y);
            series2->append(x, y * 0.3 + (qrand() % 20 - 10));  // Profit ~30%
        }
        
        // Style
        QPen pen1(QColor("#3498db"));
        pen1.setWidth(3);
        series1->setPen(pen1);
        
        QPen pen2(QColor("#2ecc71"));
        pen2.setWidth(3);
        series2->setPen(pen2);
        
        // Chart
        auto* chart = new QChart();
        chart->addSeries(series1);
        chart->addSeries(series2);
        chart->setTitle("Monthly Performance 2024");
        chart->setAnimationOptions(QChart::AllAnimations);
        chart->setTheme(QChart::ChartThemeBlueIcy);
        
        // Axes
        auto* axisX = new QValueAxis();
        axisX->setRange(1, 12);
        axisX->setTickCount(12);
        axisX->setTitleText("Month");
        axisX->setLabelFormat("%d");
        
        auto* axisY = new QValueAxis();
        axisY->setRange(0, 350);
        axisY->setTitleText("Amount (K฿)");
        
        chart->addAxis(axisX, Qt::AlignBottom);
        chart->addAxis(axisY, Qt::AlignLeft);
        series1->attachAxis(axisX);
        series1->attachAxis(axisY);
        series2->attachAxis(axisX);
        series2->attachAxis(axisY);
        
        // Chart view
        auto* chartView = new QChartView(chart);
        chartView->setRenderHint(QPainter::Antialiasing);
        
        // Allow mouse zoom and pan
        chart->setAcceptDrops(true);
        
        // Legend
        chart->legend()->setVisible(true);
        chart->legend()->setAlignment(Qt::AlignBottom);
        
        auto* layout = new QVBoxLayout(this);
        layout->addWidget(chartView);
        
        // Interactive zoom
        connect(series1, &QLineSeries::clicked, [](const QPointF& point) {
            qDebug() << "Clicked:" << point;
        });
    }
};
```

---

## ขั้นตอนที่ 253: Bar Chart

```cpp
#include <QBarSeries>
#include <QBarSet>
#include <QGroupedBarSeries>
#include <QStackedBarSeries>
#include <QBarCategoryAxis>

class BarChartDemo : public QWidget {
    Q_OBJECT
    
public:
    BarChartDemo(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("Bar Chart");
        setMinimumSize(800, 500);
        
        auto* tabs = new QTabWidget(this);
        
        // === Grouped Bar ===
        {
            auto* q1 = new QBarSet("Q1");
            auto* q2 = new QBarSet("Q2");
            auto* q3 = new QBarSet("Q3");
            auto* q4 = new QBarSet("Q4");
            
            *q1 << 45 << 38 << 52 << 61 << 29;
            *q2 << 58 << 47 << 63 << 74 << 35;
            *q3 << 53 << 42 << 58 << 69 << 31;
            *q4 << 70 << 55 << 75 << 88 << 45;
            
            q1->setColor(QColor("#e74c3c"));
            q2->setColor(QColor("#f39c12"));
            q3->setColor(QColor("#2ecc71"));
            q4->setColor(QColor("#3498db"));
            
            auto* series = new QGroupedBarSeries();
            series->append(q1);
            series->append(q2);
            series->append(q3);
            series->append(q4);
            
            auto* chart = new QChart();
            chart->addSeries(series);
            chart->setTitle("Quarterly Sales by Product");
            chart->setAnimationOptions(QChart::AllAnimations);
            
            QStringList categories = {"A", "B", "C", "D", "E"};
            auto* axisX = new QBarCategoryAxis();
            axisX->append(categories);
            axisX->setTitleText("Product");
            
            auto* axisY = new QValueAxis();
            axisY->setRange(0, 100);
            axisY->setTitleText("Sales (K)");
            
            chart->addAxis(axisX, Qt::AlignBottom);
            chart->addAxis(axisY, Qt::AlignLeft);
            series->attachAxis(axisX);
            series->attachAxis(axisY);
            
            chart->legend()->setAlignment(Qt::AlignBottom);
            
            auto* view = new QChartView(chart);
            view->setRenderHint(QPainter::Antialiasing);
            tabs->addTab(view, "Grouped");
        }
        
        // === Stacked Bar ===
        {
            auto* online = new QBarSet("Online");
            auto* store = new QBarSet("Store");
            auto* wholesale = new QBarSet("Wholesale");
            
            *online    << 120 << 145 << 160 << 180 << 200 << 220;
            *store     <<  80 <<  75 <<  90 <<  85 << 100 << 110;
            *wholesale <<  50 <<  60 <<  55 <<  70 <<  65 <<  80;
            
            online->setColor(QColor("#3498db"));
            store->setColor(QColor("#2ecc71"));
            wholesale->setColor(QColor("#e74c3c"));
            
            auto* series = new QStackedBarSeries();
            series->append(online);
            series->append(store);
            series->append(wholesale);
            
            auto* chart = new QChart();
            chart->addSeries(series);
            chart->setTitle("Sales by Channel (H1 2024)");
            chart->setAnimationOptions(QChart::AllAnimations);
            
            QStringList months = {"Jan", "Feb", "Mar", "Apr", "May", "Jun"};
            auto* axisX = new QBarCategoryAxis();
            axisX->append(months);
            
            auto* axisY = new QValueAxis();
            axisY->setRange(0, 450);
            axisY->setTitleText("Revenue (K฿)");
            
            chart->addAxis(axisX, Qt::AlignBottom);
            chart->addAxis(axisY, Qt::AlignLeft);
            series->attachAxis(axisX);
            series->attachAxis(axisY);
            
            chart->legend()->setAlignment(Qt::AlignBottom);
            
            auto* view = new QChartView(chart);
            view->setRenderHint(QPainter::Antialiasing);
            tabs->addTab(view, "Stacked");
        }
        
        auto* layout = new QVBoxLayout(this);
        layout->addWidget(tabs);
    }
};
```

---

## ขั้นตอนที่ 254: Pie Chart

```cpp
#include <QPieSeries>
#include <QPieSlice>

class PieChartDemo : public QWidget {
    Q_OBJECT
    
public:
    PieChartDemo(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("Pie Chart");
        setMinimumSize(700, 500);
        
        auto* series = new QPieSeries();
        series->setHoleSize(0.35);  // Donut chart
        
        QList<QPair<QString, double>> data = {
            {"C++",        25},
            {"Python",     30},
            {"JavaScript", 20},
            {"Java",       15},
            {"Go",          7},
            {"Other",       3}
        };
        
        QList<QColor> colors = {
            "#3498db", "#e74c3c", "#f39c12",
            "#2ecc71", "#9b59b6", "#95a5a6"
        };
        
        for (int i = 0; i < data.size(); i++) {
            auto* slice = series->append(data[i].first, data[i].second);
            slice->setColor(colors[i]);
            slice->setLabelVisible(true);
            slice->setLabel(QString("%1\n%2%")
                .arg(data[i].first)
                .arg(data[i].second, 0, 'f', 0));
            slice->setLabelColor(Qt::white);
            slice->setLabelPosition(QPieSlice::LabelInsideHorizontal);
        }
        
        // Explode largest slice
        auto* largest = series->slices().at(1);  // Python
        largest->setExploded(true);
        largest->setExplodeDistanceFactor(0.1);
        
        auto* chart = new QChart();
        chart->addSeries(series);
        chart->setTitle("Programming Languages 2024");
        chart->setAnimationOptions(QChart::AllAnimations);
        chart->legend()->setAlignment(Qt::AlignRight);
        
        // Center text (for donut)
        // Custom drawing can be done here
        
        auto* view = new QChartView(chart);
        view->setRenderHint(QPainter::Antialiasing);
        
        // Detail label
        detailLabel = new QLabel("คลิกที่ slice เพื่อดูรายละเอียด");
        detailLabel->setAlignment(Qt::AlignCenter);
        
        auto* layout = new QVBoxLayout(this);
        layout->addWidget(view);
        layout->addWidget(detailLabel);
        
        // Click interaction
        for (auto* slice : series->slices()) {
            connect(slice, &QPieSlice::clicked, [=]() {
                for (auto* s : series->slices()) s->setExploded(false);
                slice->setExploded(true);
                
                double percent = slice->percentage() * 100;
                detailLabel->setText(
                    QString("<b>%1:</b> %2 (%.1f%%)")
                    .arg(slice->label())
                    .arg(slice->value())
                    .arg(percent)
                );
            });
            
            connect(slice, &QPieSlice::hovered, [=](bool hovered) {
                slice->setExplodeDistanceFactor(hovered ? 0.1 : 0.0);
                if (!slice->isExploded()) slice->setExploded(hovered);
                slice->setLabelVisible(hovered);
            });
        }
    }
    
private:
    QLabel* detailLabel;
};
```

---

## ขั้นตอนที่ 255-265: Custom Dashboard

```cpp
class Dashboard : public QMainWindow {
    Q_OBJECT
    
public:
    Dashboard(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Sales Dashboard");
        setMinimumSize(1000, 700);
        
        auto* central = new QWidget(this);
        setCentralWidget(central);
        
        auto* mainLayout = new QVBoxLayout(central);
        mainLayout->setSpacing(10);
        mainLayout->setContentsMargins(10, 10, 10, 10);
        
        // === Header KPIs ===
        auto* kpiBar = new QHBoxLayout();
        
        auto addKpi = [&](const QString& title, const QString& value, 
                          const QString& change, const QString& color) {
            auto* card = new QFrame();
            card->setStyleSheet(QString(R"(
                QFrame {
                    background: white;
                    border-radius: 8px;
                    padding: 12px;
                    border-left: 4px solid %1;
                }
            )").arg(color));
            
            auto* cl = new QVBoxLayout(card);
            auto* titleLbl = new QLabel(title);
            titleLbl->setStyleSheet("color: #666; font-size: 12px;");
            
            auto* valueLbl = new QLabel(value);
            valueLbl->setStyleSheet("font-size: 24px; font-weight: bold; color: #2c3e50;");
            
            auto* changeLbl = new QLabel(change);
            bool isPos = change.startsWith("+");
            changeLbl->setStyleSheet(QString("color: %1; font-size: 11px;")
                .arg(isPos ? "#27ae60" : "#e74c3c"));
            
            cl->addWidget(titleLbl);
            cl->addWidget(valueLbl);
            cl->addWidget(changeLbl);
            
            kpiBar->addWidget(card);
        };
        
        addKpi("รายได้รวม",  "฿2.45M",  "+12.5% vs last month", "#3498db");
        addKpi("ออเดอร์",    "1,247",    "+8.3%",                "#2ecc71");
        addKpi("ลูกค้าใหม่", "384",      "+15.2%",               "#e74c3c");
        addKpi("Conversion", "3.82%",    "-0.4%",                "#f39c12");
        
        mainLayout->addLayout(kpiBar);
        
        // === Charts Grid ===
        auto* chartGrid = new QGridLayout();
        chartGrid->setSpacing(10);
        
        // Revenue line chart
        chartGrid->addWidget(createRevenueChart(), 0, 0);
        
        // Category pie chart
        chartGrid->addWidget(createCategoryChart(), 0, 1);
        
        // Monthly bar chart
        chartGrid->addWidget(createMonthlyChart(), 1, 0, 1, 2);
        
        mainLayout->addLayout(chartGrid);
    }
    
private:
    QChartView* createRevenueChart() {
        auto* series = new QLineSeries();
        series->setName("Daily Revenue");
        
        // Simulate 30 days of revenue
        qsrand(42);
        for (int d = 1; d <= 30; d++) {
            double revenue = 80000 + qrand() % 50000;
            series->append(d, revenue);
        }
        
        QPen pen(QColor("#3498db"));
        pen.setWidth(2);
        series->setPen(pen);
        
        QLinearGradient gradient(0, 0, 0, 300);
        gradient.setColorAt(0.0, QColor("#3498db").lighter(150));
        gradient.setColorAt(1.0, Qt::transparent);
        
        auto* area = new QAreaSeries(series);
        area->setBrush(gradient);
        area->setPen(Qt::NoPen);
        
        auto* chart = new QChart();
        chart->addSeries(series);
        chart->addSeries(area);
        chart->setTitle("30-Day Revenue");
        chart->setAnimationOptions(QChart::AllAnimations);
        chart->legend()->hide();
        chart->createDefaultAxes();
        
        auto* view = new QChartView(chart);
        view->setRenderHint(QPainter::Antialiasing);
        view->setMinimumHeight(250);
        return view;
    }
    
    QChartView* createCategoryChart() {
        auto* series = new QPieSeries();
        series->setHoleSize(0.4);
        
        series->append("Electronics", 35)->setColor("#3498db");
        series->append("Clothing",    25)->setColor("#e74c3c");
        series->append("Food",        20)->setColor("#2ecc71");
        series->append("Books",       12)->setColor("#f39c12");
        series->append("Other",        8)->setColor("#9b59b6");
        
        for (auto* slice : series->slices()) {
            slice->setLabelVisible(true);
            slice->setLabel(QString("%1\n%2%")
                .arg(slice->label().split('\n').first())
                .arg(slice->percentage() * 100, 0, 'f', 0));
        }
        
        auto* chart = new QChart();
        chart->addSeries(series);
        chart->setTitle("Sales by Category");
        chart->setAnimationOptions(QChart::AllAnimations);
        chart->legend()->hide();
        
        auto* view = new QChartView(chart);
        view->setRenderHint(QPainter::Antialiasing);
        view->setMinimumHeight(250);
        return view;
    }
    
    QChartView* createMonthlyChart() {
        auto* actual = new QBarSet("Actual");
        auto* target = new QBarSet("Target");
        
        actual->setColor(QColor("#3498db"));
        target->setColor(QColor("#95a5a6"));
        
        *actual << 245 << 290 << 312 << 278 << 345 << 380
                << 410 << 395 << 425 << 460 << 490 << 520;
        *target << 280 << 280 << 300 << 300 << 320 << 320
                << 380 << 380 << 400 << 420 << 450 << 480;
        
        auto* series = new QGroupedBarSeries();
        series->append(actual);
        series->append(target);
        
        auto* chart = new QChart();
        chart->addSeries(series);
        chart->setTitle("Monthly Actual vs Target (K฿)");
        chart->setAnimationOptions(QChart::AllAnimations);
        
        QStringList months = {"Jan","Feb","Mar","Apr","May","Jun",
                              "Jul","Aug","Sep","Oct","Nov","Dec"};
        auto* axisX = new QBarCategoryAxis();
        axisX->append(months);
        
        auto* axisY = new QValueAxis();
        axisY->setRange(0, 600);
        axisY->setTitleText("K฿");
        
        chart->addAxis(axisX, Qt::AlignBottom);
        chart->addAxis(axisY, Qt::AlignLeft);
        series->attachAxis(axisX);
        series->attachAxis(axisY);
        
        chart->legend()->setAlignment(Qt::AlignBottom);
        
        auto* view = new QChartView(chart);
        view->setRenderHint(QPainter::Antialiasing);
        view->setMinimumHeight(250);
        return view;
    }
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    app.setStyleSheet("QMainWindow { background: #f0f2f5; }");
    
    Dashboard dashboard;
    dashboard.show();
    
    return app.exec();
}
```

---

## สรุป Part 019

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt Charts Module
2. ✅ Line Chart & Spline Chart
3. ✅ Bar Chart (Grouped & Stacked)
4. ✅ Pie Chart & Donut Chart
5. ✅ Area Series
6. ✅ Full Dashboard Application

---

⬅️ [Part 018](part018.md) | ➡️ [Part 020: Qt Testing](part020.md)
