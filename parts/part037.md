# Part 037: Qt PDF, Printing & Document Generation

## ขั้นตอนที่ 521-535

---

## ขั้นตอนที่ 521: Qt PDF พื้นฐาน

```
Qt PDF Module:
  - อ่าน/แสดง PDF ด้วย QPdfDocument
  - Print ด้วย QPrinter + QPainter
  - Generate reports ด้วย HTML → PDF

CMakeLists.txt:
  find_package(Qt6 REQUIRED COMPONENTS PrintSupport Pdf PdfWidgets)
  target_link_libraries(app PRIVATE Qt6::PrintSupport Qt6::Pdf Qt6::PdfWidgets)
```

---

## ขั้นตอนที่ 522: PDF Viewer

```cpp
#include <QPdfDocument>
#include <QPdfView>
#include <QPdfPageNavigator>
#include <QPrinter>
#include <QPrintDialog>
#include <QPainter>

class PdfViewer : public QMainWindow {
    Q_OBJECT
    
public:
    PdfViewer(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("PDF Viewer");
        setMinimumSize(900, 700);
        
        doc = new QPdfDocument(this);
        
        setupUi();
        setupMenus();
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* layout = new QVBoxLayout(central);
        setCentralWidget(central);
        
        // Toolbar
        auto* toolbar = new QToolBar("Navigation");
        addToolBar(toolbar);
        
        auto* prevBtn = toolbar->addAction(QIcon::fromTheme("go-previous"), "Previous");
        auto* nextBtn = toolbar->addAction(QIcon::fromTheme("go-next"), "Next");
        toolbar->addSeparator();
        
        pageLabel = new QLabel("0 / 0");
        pageLabel->setMinimumWidth(80);
        pageLabel->setAlignment(Qt::AlignCenter);
        toolbar->addWidget(pageLabel);
        
        toolbar->addSeparator();
        
        auto* zoomInBtn = toolbar->addAction(QIcon::fromTheme("zoom-in"), "Zoom In");
        auto* zoomOutBtn = toolbar->addAction(QIcon::fromTheme("zoom-out"), "Zoom Out");
        auto* fitBtn = toolbar->addAction("Fit Width");
        
        // PDF View
        pdfView = new QPdfView(central);
        pdfView->setDocument(doc);
        pdfView->setPageMode(QPdfView::PageMode::MultiPage);
        pdfView->setZoomMode(QPdfView::ZoomMode::FitInView);
        
        layout->addWidget(pdfView);
        
        auto* navigator = pdfView->pageNavigator();
        
        connect(prevBtn, &QAction::triggered, [navigator]() {
            int page = navigator->currentPage();
            if (page > 0) navigator->jump(page - 1, {});
        });
        
        connect(nextBtn, &QAction::triggered, [this, navigator]() {
            int page = navigator->currentPage();
            if (page < doc->pageCount() - 1) navigator->jump(page + 1, {});
        });
        
        connect(navigator, &QPdfPageNavigator::currentPageChanged, [this](int page) {
            pageLabel->setText(QString("%1 / %2").arg(page + 1).arg(doc->pageCount()));
        });
        
        connect(zoomInBtn, &QAction::triggered, [this]() {
            pdfView->setZoomFactor(pdfView->zoomFactor() * 1.25);
        });
        
        connect(zoomOutBtn, &QAction::triggered, [this]() {
            pdfView->setZoomFactor(pdfView->zoomFactor() / 1.25);
        });
        
        connect(fitBtn, &QAction::triggered, [this]() {
            pdfView->setZoomMode(QPdfView::ZoomMode::FitInView);
        });
    }
    
    void setupMenus() {
        auto* fileMenu = menuBar()->addMenu("File");
        
        auto* openAct = fileMenu->addAction("Open...", QKeySequence::Open);
        fileMenu->addSeparator();
        auto* printAct = fileMenu->addAction("Print...", QKeySequence::Print);
        
        connect(openAct, &QAction::triggered, this, &PdfViewer::openFile);
        connect(printAct, &QAction::triggered, this, &PdfViewer::printDocument);
    }
    
    void openFile() {
        QString path = QFileDialog::getOpenFileName(this, "Open PDF", "", "PDF (*.pdf)");
        if (path.isEmpty()) return;
        
        if (doc->load(path) == QPdfDocument::Error::None) {
            setWindowTitle("PDF Viewer - " + QFileInfo(path).fileName());
            pageLabel->setText(QString("1 / %1").arg(doc->pageCount()));
        } else {
            QMessageBox::warning(this, "Error", "Cannot open file: " + path);
        }
    }
    
    void printDocument() {
        if (doc->pageCount() == 0) return;
        
        QPrinter printer(QPrinter::HighResolution);
        QPrintDialog dialog(&printer, this);
        
        if (dialog.exec() != QDialog::Accepted) return;
        
        QPainter painter(&printer);
        
        int startPage = 0, endPage = doc->pageCount() - 1;
        if (printer.printRange() == QPrinter::PageRange) {
            startPage = printer.fromPage() - 1;
            endPage = printer.toPage() - 1;
        }
        
        for (int page = startPage; page <= endPage; ++page) {
            if (page > startPage) printer.newPage();
            
            QSizeF pageSize = doc->pagePointSize(page);
            QRect rect = painter.viewport();
            
            // Scale to fit printer
            double scale = qMin(rect.width() / pageSize.width(),
                                rect.height() / pageSize.height());
            
            QImage img = doc->render(page, QSize(
                static_cast<int>(pageSize.width() * scale),
                static_cast<int>(pageSize.height() * scale)
            ));
            
            painter.drawImage(QPoint(0, 0), img);
        }
        
        painter.end();
    }
    
    QPdfDocument* doc;
    QPdfView* pdfView;
    QLabel* pageLabel;
};
```

---

## ขั้นตอนที่ 523: Report Generator

```cpp
class ReportGenerator : public QObject {
    Q_OBJECT
    
public:
    struct ReportData {
        QString title;
        QString subtitle;
        QString company;
        QDate date;
        QList<QStringList> tableData;
        QStringList tableHeaders;
    };
    
    explicit ReportGenerator(QObject* parent = nullptr) : QObject(parent) {}
    
    bool generateHtmlReport(const ReportData& data, const QString& outputPath) {
        QFile file(outputPath);
        if (!file.open(QIODevice::WriteOnly | QIODevice::Text)) return false;
        
        QTextStream out(&file);
        out.setEncoding(QStringConverter::Utf8);
        
        out << buildHtml(data);
        return true;
    }
    
    bool printToPdf(const ReportData& data, const QString& pdfPath) {
        QString html = buildHtml(data);
        
        QPrinter printer;
        printer.setOutputFormat(QPrinter::PdfFormat);
        printer.setOutputFileName(pdfPath);
        printer.setPageSize(QPageSize::A4);
        printer.setPageOrientation(QPageLayout::Portrait);
        printer.setPageMargins(QMarginsF(20, 20, 20, 20), QPageLayout::Millimeter);
        
        QTextDocument doc;
        doc.setDefaultStyleSheet(defaultCss());
        doc.setHtml(html);
        doc.print(&printer);
        
        return QFile::exists(pdfPath);
    }
    
private:
    QString buildHtml(const ReportData& data) {
        QString html = QString(R"(
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<style>
  body { font-family: 'Segoe UI', Arial, sans-serif; margin: 0; color: #333; }
  .header { background: #2c3e50; color: white; padding: 24px; }
  .header h1 { margin: 0; font-size: 24px; }
  .header p { margin: 4px 0 0; color: #bdc3c7; }
  .meta { display: flex; justify-content: space-between; padding: 12px 24px; 
          background: #ecf0f1; font-size: 13px; color: #666; }
  .content { padding: 24px; }
  h2 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 8px; }
  table { width: 100%%; border-collapse: collapse; margin-top: 16px; }
  th { background: #3498db; color: white; padding: 10px 12px; text-align: left; font-weight: 600; }
  td { padding: 9px 12px; border-bottom: 1px solid #e8e8e8; }
  tr:nth-child(even) td { background: #f8f9fa; }
  tr:hover td { background: #e3f2fd; }
  .footer { margin-top: 32px; text-align: center; font-size: 11px; color: #999; 
            border-top: 1px solid #eee; padding-top: 12px; }
  .badge { display: inline-block; padding: 2px 8px; border-radius: 12px; font-size: 12px; }
  .badge-green { background: #d5f5e3; color: #1e8449; }
  .badge-red { background: #fadbd8; color: #922b21; }
  .badge-blue { background: #d6eaf8; color: #1a5276; }
</style>
</head>
<body>
<div class="header">
  <h1>%1</h1>
  <p>%2</p>
</div>
<div class="meta">
  <span><b>บริษัท:</b> %3</span>
  <span><b>วันที่:</b> %4</span>
  <span><b>จำนวนรายการ:</b> %5 รายการ</span>
</div>
<div class="content">
  <h2>รายงานข้อมูล</h2>
  <table>
    <thead><tr>%6</tr></thead>
    <tbody>%7</tbody>
  </table>
</div>
<div class="footer">
  รายงานนี้สร้างอัตโนมัติโดยระบบ · %8
</div>
</body></html>)");
        
        // Headers
        QString headers;
        for (const QString& h : data.tableHeaders) {
            headers += "<th>" + h.toHtmlEscaped() + "</th>";
        }
        
        // Rows
        QString rows;
        for (const QStringList& row : data.tableData) {
            rows += "<tr>";
            for (const QString& cell : row) {
                rows += "<td>" + cell.toHtmlEscaped() + "</td>";
            }
            rows += "</tr>";
        }
        
        return html.arg(
            data.title.toHtmlEscaped(),
            data.subtitle.toHtmlEscaped(),
            data.company.toHtmlEscaped(),
            QLocale(QLocale::Thai).toString(data.date, QLocale::LongFormat),
            QString::number(data.tableData.size()),
            headers,
            rows,
            QDateTime::currentDateTime().toString("dd/MM/yyyy hh:mm:ss")
        );
    }
    
    static QString defaultCss() { return ""; }
};
```

---

## ขั้นตอนที่ 524: Print Preview

```cpp
class PrintPreviewWindow : public QDialog {
    Q_OBJECT
    
public:
    explicit PrintPreviewWindow(QWidget* parent = nullptr) : QDialog(parent) {
        setWindowTitle("Print Preview");
        setMinimumSize(800, 600);
        
        auto* layout = new QVBoxLayout(this);
        
        // Toolbar
        auto* toolbar = new QHBoxLayout();
        
        auto* printBtn = new QPushButton("Print");
        printBtn->setStyleSheet("background: #3498db; color: white; padding: 6px 16px;");
        
        auto* zoomInBtn = new QPushButton("+");
        auto* zoomOutBtn = new QPushButton("-");
        
        zoomLabel = new QLabel("100%");
        zoomLabel->setMinimumWidth(50);
        zoomLabel->setAlignment(Qt::AlignCenter);
        
        toolbar->addWidget(printBtn);
        toolbar->addStretch();
        toolbar->addWidget(zoomOutBtn);
        toolbar->addWidget(zoomLabel);
        toolbar->addWidget(zoomInBtn);
        
        // Preview
        preview = new QPrintPreviewWidget(&printer, this);
        connect(preview, &QPrintPreviewWidget::paintRequested,
                this, &PrintPreviewWindow::renderPage);
        
        layout->addLayout(toolbar);
        layout->addWidget(preview);
        
        connect(printBtn, &QPushButton::clicked, [this]() {
            QPrintDialog dlg(&printer, this);
            if (dlg.exec() == QDialog::Accepted) {
                preview->print();
            }
        });
        
        double zoom = 1.0;
        connect(zoomInBtn, &QPushButton::clicked, [this, zoom]() mutable {
            zoom = qMin(zoom * 1.25, 4.0);
            preview->setZoomFactor(zoom);
            zoomLabel->setText(QString("%1%").arg(static_cast<int>(zoom * 100)));
        });
        
        connect(zoomOutBtn, &QPushButton::clicked, [this, zoom]() mutable {
            zoom = qMax(zoom / 1.25, 0.25);
            preview->setZoomFactor(zoom);
            zoomLabel->setText(QString("%1%").arg(static_cast<int>(zoom * 100)));
        });
    }
    
    void setHtmlContent(const QString& html) {
        m_html = html;
        preview->updatePreview();
    }
    
private:
    void renderPage(QPrinter* p) {
        QPainter painter(p);
        QTextDocument doc;
        doc.setHtml(m_html);
        
        QRect body = painter.viewport();
        doc.setPageSize(QSizeF(body.width(), body.height()));
        doc.drawContents(&painter);
    }
    
    QPrinter printer;
    QPrintPreviewWidget* preview;
    QLabel* zoomLabel;
    QString m_html;
};
```

---

## ขั้นตอนที่ 525: ใช้งาน Report Generator

```cpp
class SalesReportApp : public QMainWindow {
    Q_OBJECT
    
public:
    SalesReportApp(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Sales Report System");
        setMinimumSize(1000, 700);
        
        generator = new ReportGenerator(this);
        
        setupUi();
        loadSampleData();
    }
    
private:
    void setupUi() {
        auto* toolbar = addToolBar("Actions");
        
        toolbar->addAction("Generate PDF", this, &SalesReportApp::generatePdf);
        toolbar->addAction("Export HTML", this, &SalesReportApp::exportHtml);
        toolbar->addAction("Print Preview", this, &SalesReportApp::showPreview);
        
        // Table
        table = new QTableWidget();
        table->setColumnCount(5);
        table->setHorizontalHeaderLabels({"รหัส", "สินค้า", "จำนวน", "ราคา/ชิ้น", "รวม"});
        table->horizontalHeader()->setStretchLastSection(true);
        table->setAlternatingRowColors(true);
        table->setSelectionBehavior(QAbstractItemView::SelectRows);
        
        auto* central = new QWidget();
        auto* layout = new QVBoxLayout(central);
        
        auto* titleLabel = new QLabel("รายงานการขาย");
        titleLabel->setStyleSheet("font-size: 20px; font-weight: bold; color: #2c3e50;");
        titleLabel->setAlignment(Qt::AlignCenter);
        
        layout->addWidget(titleLabel);
        layout->addWidget(table);
        
        // Summary
        summaryLabel = new QLabel();
        summaryLabel->setStyleSheet("font-size: 14px; padding: 8px; background: #ecf0f1;");
        layout->addWidget(summaryLabel);
        
        setCentralWidget(central);
    }
    
    void loadSampleData() {
        struct SaleItem {
            QString id, product;
            int qty;
            double price;
        };
        
        QList<SaleItem> items = {
            {"P001", "Qt Framework License", 5, 12500.0},
            {"P002", "C++ Development Kit", 10, 8900.0},
            {"P003", "Training Manual", 50, 350.0},
            {"P004", "Support Package (1yr)", 3, 25000.0},
            {"P005", "Plugin SDK", 8, 4500.0},
            {"P006", "Mobile Deploy License", 2, 18000.0},
        };
        
        table->setRowCount(items.size());
        double total = 0;
        
        for (int i = 0; i < items.size(); ++i) {
            const auto& item = items[i];
            double subtotal = item.qty * item.price;
            total += subtotal;
            
            table->setItem(i, 0, new QTableWidgetItem(item.id));
            table->setItem(i, 1, new QTableWidgetItem(item.product));
            
            auto* qtyItem = new QTableWidgetItem(QString::number(item.qty));
            qtyItem->setTextAlignment(Qt::AlignCenter);
            table->setItem(i, 2, qtyItem);
            
            auto* priceItem = new QTableWidgetItem(
                QLocale(QLocale::Thai).toCurrencyString(item.price));
            priceItem->setTextAlignment(Qt::AlignRight | Qt::AlignVCenter);
            table->setItem(i, 3, priceItem);
            
            auto* totalItem = new QTableWidgetItem(
                QLocale(QLocale::Thai).toCurrencyString(subtotal));
            totalItem->setTextAlignment(Qt::AlignRight | Qt::AlignVCenter);
            table->setItem(i, 4, totalItem);
        }
        
        m_items = items;
        m_total = total;
        summaryLabel->setText(QString("รวมทั้งสิ้น: %1 บาท | จำนวน %2 รายการ")
            .arg(QLocale(QLocale::Thai).toCurrencyString(total))
            .arg(items.size()));
    }
    
    ReportGenerator::ReportData buildReportData() {
        ReportGenerator::ReportData data;
        data.title = "รายงานการขาย";
        data.subtitle = "สรุปยอดขายประจำเดือน";
        data.company = "บริษัท Qt Solutions Thailand จำกัด";
        data.date = QDate::currentDate();
        data.tableHeaders = {"รหัส", "สินค้า", "จำนวน", "ราคา/ชิ้น", "รวม"};
        
        for (int i = 0; i < table->rowCount(); ++i) {
            QStringList row;
            for (int j = 0; j < table->columnCount(); ++j) {
                row << (table->item(i, j) ? table->item(i, j)->text() : "");
            }
            data.tableData.append(row);
        }
        
        return data;
    }
    
    void generatePdf() {
        QString path = QFileDialog::getSaveFileName(this, "Save PDF", 
            "sales_report.pdf", "PDF (*.pdf)");
        if (path.isEmpty()) return;
        
        if (generator->printToPdf(buildReportData(), path)) {
            QMessageBox::information(this, "Success", "PDF saved to: " + path);
            
            if (QMessageBox::question(this, "Open", "Open PDF now?") == QMessageBox::Yes) {
                QDesktopServices::openUrl(QUrl::fromLocalFile(path));
            }
        }
    }
    
    void exportHtml() {
        QString path = QFileDialog::getSaveFileName(this, "Save HTML",
            "sales_report.html", "HTML (*.html)");
        if (path.isEmpty()) return;
        
        if (generator->generateHtmlReport(buildReportData(), path)) {
            QMessageBox::information(this, "Success", "HTML saved to: " + path);
        }
    }
    
    void showPreview() {
        ReportGenerator::ReportData data = buildReportData();
        
        QFile tempHtml(QDir::temp().filePath("preview_report.html"));
        generator->generateHtmlReport(data, tempHtml.fileName());
        
        tempHtml.open(QIODevice::ReadOnly | QIODevice::Text);
        QString html = QString::fromUtf8(tempHtml.readAll());
        
        auto* previewDlg = new PrintPreviewWindow(this);
        previewDlg->setHtmlContent(html);
        previewDlg->exec();
    }
    
    struct SaleItem {
        QString id, product;
        int qty;
        double price;
    };
    
    ReportGenerator* generator;
    QTableWidget* table;
    QLabel* summaryLabel;
    QList<SaleItem> m_items;
    double m_total = 0;
};
```

---

## สรุป Part 037

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QPdfDocument + QPdfView สำหรับแสดง PDF
2. ✅ การ navigate หน้า PDF
3. ✅ ReportGenerator สร้าง HTML + PDF จาก template
4. ✅ Print Preview ด้วย QPrintPreviewWidget
5. ✅ Sales Report App ครบวงจร

---

⬅️ [Part 036](part036.md) | ➡️ [Part 038: Qt Internationalization Advanced](part038.md)
