# Part 013: Qt Styling & Custom Widgets

## ขั้นตอนที่ 161-175

---

## ขั้นตอนที่ 161: Qt Style Sheets

Qt Style Sheets คล้ายกับ CSS สำหรับ Web และใช้ syntax เดียวกัน:

```cpp
#include <QApplication>
#include <QWidget>
#include <QPushButton>
#include <QLabel>
#include <QLineEdit>
#include <QVBoxLayout>
#include <QProgressBar>
#include <QCheckBox>

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    // Global stylesheet
    app.setStyleSheet(R"(
        QWidget {
            font-family: 'Segoe UI', Arial, sans-serif;
            font-size: 13px;
        }
        
        QPushButton {
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 6px;
            padding: 8px 16px;
            min-width: 80px;
        }
        QPushButton:hover {
            background-color: #45a049;
        }
        QPushButton:pressed {
            background-color: #357a38;
        }
        QPushButton:disabled {
            background-color: #cccccc;
            color: #666666;
        }
        
        QPushButton#dangerBtn {
            background-color: #f44336;
        }
        QPushButton#dangerBtn:hover {
            background-color: #d32f2f;
        }
        
        QPushButton.secondary {
            background-color: transparent;
            color: #4CAF50;
            border: 2px solid #4CAF50;
        }
        
        QLineEdit {
            border: 2px solid #ddd;
            border-radius: 4px;
            padding: 6px 10px;
            background: white;
        }
        QLineEdit:focus {
            border-color: #4CAF50;
            outline: none;
        }
        QLineEdit:hover {
            border-color: #aaa;
        }
        
        QLabel {
            color: #333333;
        }
        QLabel#title {
            font-size: 20px;
            font-weight: bold;
            color: #2c3e50;
        }
        QLabel#subtitle {
            font-size: 14px;
            color: #666;
        }
        
        QProgressBar {
            border: none;
            border-radius: 4px;
            background-color: #e0e0e0;
            text-align: center;
            height: 20px;
        }
        QProgressBar::chunk {
            background-color: qlineargradient(x1:0, y1:0, x2:1, y2:0,
                stop:0 #4CAF50, stop:1 #8BC34A);
            border-radius: 4px;
        }
        
        QCheckBox {
            spacing: 8px;
        }
        QCheckBox::indicator {
            width: 18px;
            height: 18px;
            border-radius: 3px;
            border: 2px solid #ddd;
        }
        QCheckBox::indicator:checked {
            background-color: #4CAF50;
            border-color: #4CAF50;
            image: url(checkmark.png);
        }
    )");
    
    QWidget window;
    window.setWindowTitle("Qt Styling Demo");
    window.setFixedSize(400, 400);
    window.setStyleSheet("background-color: #f5f5f5;");
    
    auto* layout = new QVBoxLayout(&window);
    layout->setContentsMargins(20, 20, 20, 20);
    layout->setSpacing(12);
    
    // Title
    auto* title = new QLabel("สไตล์ Qt");
    title->setObjectName("title");
    title->setAlignment(Qt::AlignCenter);
    
    auto* subtitle = new QLabel("ตัวอย่างการใช้ Qt Style Sheets");
    subtitle->setObjectName("subtitle");
    subtitle->setAlignment(Qt::AlignCenter);
    
    // Inputs
    auto* nameInput = new QLineEdit();
    nameInput->setPlaceholderText("ชื่อผู้ใช้");
    
    auto* passInput = new QLineEdit();
    passInput->setEchoMode(QLineEdit::Password);
    passInput->setPlaceholderText("รหัสผ่าน");
    
    // Buttons
    auto* primaryBtn = new QPushButton("เข้าสู่ระบบ");
    
    auto* secondaryBtn = new QPushButton("สมัครสมาชิก");
    secondaryBtn->setProperty("class", "secondary");
    
    auto* dangerBtn = new QPushButton("ลบบัญชี");
    dangerBtn->setObjectName("dangerBtn");
    
    // Progress
    auto* progress = new QProgressBar();
    progress->setValue(65);
    
    auto* check = new QCheckBox("จำชื่อผู้ใช้");
    check->setChecked(true);
    
    layout->addWidget(title);
    layout->addWidget(subtitle);
    layout->addSpacing(10);
    layout->addWidget(nameInput);
    layout->addWidget(passInput);
    layout->addWidget(check);
    layout->addWidget(primaryBtn);
    layout->addWidget(secondaryBtn);
    layout->addWidget(dangerBtn);
    layout->addWidget(progress);
    
    window.show();
    return app.exec();
}
```

---

## ขั้นตอนที่ 162: Custom Painting (QPainter)

```cpp
#include <QWidget>
#include <QPainter>
#include <QPen>
#include <QBrush>
#include <QFont>
#include <QColor>
#include <QLinearGradient>
#include <QRadialGradient>
#include <QConicalGradient>
#include <cmath>

class PainterDemo : public QWidget {
    Q_OBJECT
    
public:
    PainterDemo(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("QPainter Demo");
        setFixedSize(800, 600);
    }
    
protected:
    void paintEvent(QPaintEvent*) override {
        QPainter p(this);
        p.setRenderHint(QPainter::Antialiasing);
        
        // Background
        p.fillRect(rect(), QColor(30, 30, 45));
        
        // === Lines ===
        p.save();
        p.translate(50, 50);
        
        // Simple line
        p.setPen(QPen(Qt::white, 2));
        p.drawLine(0, 0, 100, 0);
        
        // Styled lines
        QPen dashPen(QColor(255, 200, 0), 2, Qt::DashLine);
        p.setPen(dashPen);
        p.drawLine(0, 20, 100, 20);
        
        QPen dotPen(QColor(0, 200, 255), 2, Qt::DotLine);
        p.setPen(dotPen);
        p.drawLine(0, 40, 100, 40);
        
        p.restore();
        
        // === Shapes ===
        p.save();
        p.translate(200, 30);
        
        // Rectangle
        p.setPen(QPen(Qt::cyan, 2));
        p.setBrush(QBrush(QColor(0, 100, 200, 100)));
        p.drawRect(0, 0, 80, 50);
        
        // Rounded rect
        p.setPen(QPen(QColor(200, 100, 0), 2));
        p.setBrush(QBrush(QColor(200, 100, 0, 80)));
        p.drawRoundedRect(100, 0, 80, 50, 10, 10);
        
        // Ellipse
        p.setPen(QPen(QColor(100, 255, 100), 2));
        p.setBrush(QBrush(QColor(0, 200, 0, 80)));
        p.drawEllipse(200, 0, 80, 50);
        
        p.restore();
        
        // === Gradients ===
        p.save();
        p.translate(50, 130);
        
        // Linear Gradient
        QLinearGradient linGrad(0, 0, 150, 0);
        linGrad.setColorAt(0, Qt::red);
        linGrad.setColorAt(0.5, Qt::green);
        linGrad.setColorAt(1, Qt::blue);
        p.fillRect(0, 0, 150, 60, linGrad);
        p.setPen(Qt::white);
        p.drawText(0, 0, 150, 60, Qt::AlignCenter, "Linear");
        
        // Radial Gradient
        QRadialGradient radGrad(230, 30, 60);
        radGrad.setColorAt(0, Qt::yellow);
        radGrad.setColorAt(1, Qt::transparent);
        p.fillRect(170, 0, 120, 60, radGrad);
        p.drawText(170, 0, 120, 60, Qt::AlignCenter, "Radial");
        
        p.restore();
        
        // === Text Rendering ===
        p.save();
        p.translate(50, 230);
        
        p.setPen(Qt::white);
        p.setFont(QFont("Arial", 18, QFont::Bold));
        p.drawText(0, 0, "Hello Qt!");
        
        p.setFont(QFont("Arial", 12, QFont::Normal, true));
        p.drawText(0, 30, "Italic Text");
        
        // Text with background
        p.setPen(Qt::black);
        p.setFont(QFont("Arial", 14, QFont::Bold));
        QRect textRect(0, 50, 200, 30);
        p.fillRect(textRect, QColor(255, 220, 0));
        p.drawText(textRect, Qt::AlignCenter, "ข้อความ");
        
        p.restore();
        
        // === Polygon ===
        p.save();
        p.translate(400, 30);
        
        // Star
        QPolygonF star;
        int points = 5;
        double outerR = 50, innerR = 25;
        for (int i = 0; i < points * 2; i++) {
            double angle = i * M_PI / points - M_PI / 2;
            double r = (i % 2 == 0) ? outerR : innerR;
            star << QPointF(r * cos(angle), r * sin(angle));
        }
        star.translate(60, 60);
        
        p.setPen(QPen(Qt::yellow, 2));
        p.setBrush(QBrush(QColor(255, 200, 0, 150)));
        p.drawPolygon(star);
        
        p.restore();
        
        // === Transform ===
        p.save();
        p.translate(400, 180);
        
        for (int i = 0; i < 8; i++) {
            p.save();
            p.rotate(i * 45);
            p.setPen(QPen(QColor(255, i * 30, 200), 2));
            p.drawRect(20, -5, 80, 10);
            p.restore();
        }
        
        p.restore();
        
        // === Path ===
        p.save();
        p.translate(50, 380);
        
        QPainterPath path;
        path.moveTo(0, 80);
        path.cubicTo(50, -20, 100, 120, 150, 20);
        path.cubicTo(200, -40, 250, 100, 300, 40);
        
        p.setPen(QPen(QColor(0, 200, 255), 3));
        p.drawPath(path);
        
        p.restore();
    }
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    PainterDemo demo;
    demo.show();
    return app.exec();
}
```

---

## ขั้นตอนที่ 163: Custom Widget

```cpp
// === thermometer.h - Custom Widget ===
#pragma once
#include <QWidget>
#include <QTimer>

class Thermometer : public QWidget {
    Q_OBJECT
    Q_PROPERTY(double temperature READ temperature WRITE setTemperature NOTIFY temperatureChanged)
    
private:
    double m_temperature;
    double m_minTemp;
    double m_maxTemp;
    QString m_unit;
    
    QTimer* animTimer;
    double targetTemp;
    
public:
    explicit Thermometer(QWidget* parent = nullptr);
    
    double temperature() const { return m_temperature; }
    void setRange(double min, double max) { m_minTemp = min; m_maxTemp = max; update(); }
    void setUnit(const QString& unit) { m_unit = unit; update(); }
    
    QSize sizeHint() const override { return QSize(80, 200); }
    QSize minimumSizeHint() const override { return QSize(60, 150); }
    
public slots:
    void setTemperature(double temp);
    void animateTo(double temp);
    
signals:
    void temperatureChanged(double temp);
    void highTemperatureAlert(double temp);
    void lowTemperatureAlert(double temp);
    
protected:
    void paintEvent(QPaintEvent*) override;
    
private slots:
    void animate();
};
```

```cpp
// === thermometer.cpp ===
#include "thermometer.h"
#include <QPainter>
#include <QPainterPath>
#include <QLinearGradient>
#include <cmath>

Thermometer::Thermometer(QWidget* parent)
    : QWidget(parent), m_temperature(20), m_minTemp(-20), m_maxTemp(60), m_unit("°C") {
    
    animTimer = new QTimer(this);
    animTimer->setInterval(16);  // ~60fps
    connect(animTimer, &QTimer::timeout, this, &Thermometer::animate);
    
    targetTemp = m_temperature;
    setMinimumSize(60, 150);
}

void Thermometer::setTemperature(double temp) {
    temp = qBound(m_minTemp, temp, m_maxTemp);
    if (m_temperature != temp) {
        m_temperature = temp;
        emit temperatureChanged(temp);
        
        if (temp > m_maxTemp * 0.8) emit highTemperatureAlert(temp);
        if (temp < m_minTemp + (m_maxTemp - m_minTemp) * 0.2) emit lowTemperatureAlert(temp);
        
        update();
    }
}

void Thermometer::animateTo(double temp) {
    targetTemp = qBound(m_minTemp, temp, m_maxTemp);
    animTimer->start();
}

void Thermometer::animate() {
    double diff = targetTemp - m_temperature;
    if (qAbs(diff) < 0.1) {
        setTemperature(targetTemp);
        animTimer->stop();
    } else {
        setTemperature(m_temperature + diff * 0.1);
    }
}

void Thermometer::paintEvent(QPaintEvent*) {
    QPainter p(this);
    p.setRenderHint(QPainter::Antialiasing);
    
    int w = width(), h = height();
    int tubeW = w / 4;
    int bulbR = tubeW;
    int tubeTop = 20;
    int tubeBottom = h - bulbR - 15;
    int tubeH = tubeBottom - tubeTop;
    
    // Proportion of temperature
    double ratio = (m_temperature - m_minTemp) / (m_maxTemp - m_minTemp);
    ratio = qBound(0.0, ratio, 1.0);
    
    // Color: blue -> green -> red
    QColor tempColor;
    if (ratio < 0.5) {
        int g = (int)(ratio * 2 * 255);
        tempColor = QColor(0, g, 255 - g);
    } else {
        int r = (int)((ratio - 0.5) * 2 * 255);
        tempColor = QColor(r, 255 - r, 0);
    }
    
    // === Draw tube ===
    int tubeX = (w - tubeW) / 2;
    
    // Tube background
    p.setPen(QPen(QColor(200, 200, 200), 2));
    p.setBrush(QColor(240, 240, 255));
    p.drawRoundedRect(tubeX, tubeTop, tubeW, tubeH, tubeW/2, tubeW/2);
    
    // Fill
    int fillH = (int)(tubeH * ratio);
    QLinearGradient grad(tubeX, 0, tubeX + tubeW, 0);
    grad.setColorAt(0, tempColor.darker(120));
    grad.setColorAt(0.5, tempColor.lighter(120));
    grad.setColorAt(1, tempColor.darker(120));
    
    p.setPen(Qt::NoPen);
    p.setBrush(grad);
    
    QPainterPath fillPath;
    fillPath.addRoundedRect(
        tubeX + 3, tubeBottom - fillH,
        tubeW - 6, fillH,
        (tubeW-6)/2, (tubeW-6)/2
    );
    p.drawPath(fillPath);
    
    // === Draw bulb ===
    int bulbX = (w - bulbR * 2) / 2;
    int bulbY = tubeBottom - 5;
    
    p.setBrush(tempColor);
    p.setPen(QPen(tempColor.darker(130), 2));
    p.drawEllipse(bulbX, bulbY, bulbR * 2, bulbR * 2);
    
    // Bulb highlight
    p.setBrush(QColor(255, 255, 255, 80));
    p.setPen(Qt::NoPen);
    p.drawEllipse(bulbX + bulbR/4, bulbY + bulbR/4, bulbR/2, bulbR/2);
    
    // === Draw scale ===
    p.setPen(QPen(QColor(80, 80, 80), 1));
    p.setFont(QFont("Arial", 7));
    
    int numTicks = 10;
    for (int i = 0; i <= numTicks; i++) {
        double t = m_minTemp + (m_maxTemp - m_minTemp) * i / numTicks;
        int y = tubeBottom - (int)(tubeH * i / numTicks);
        
        if (i % 2 == 0) {
            p.drawLine(tubeX - 10, y, tubeX, y);
            p.drawText(QRect(0, y - 7, tubeX - 12, 14), Qt::AlignRight,
                       QString::number((int)t));
        } else {
            p.drawLine(tubeX - 5, y, tubeX, y);
        }
    }
    
    // === Temperature label ===
    p.setPen(Qt::black);
    p.setFont(QFont("Arial", 9, QFont::Bold));
    QString tempStr = QString("%1%2").arg(m_temperature, 0, 'f', 1).arg(m_unit);
    p.drawText(QRect(0, h - 15, w, 15), Qt::AlignCenter, tempStr);
}
```

---

## ขั้นตอนที่ 164-175: Dark Theme

```cpp
// Dark Theme Application
void applyDarkTheme(QApplication& app) {
    QPalette darkPalette;
    
    darkPalette.setColor(QPalette::Window,          QColor(53, 53, 53));
    darkPalette.setColor(QPalette::WindowText,      Qt::white);
    darkPalette.setColor(QPalette::Base,            QColor(35, 35, 35));
    darkPalette.setColor(QPalette::AlternateBase,   QColor(53, 53, 53));
    darkPalette.setColor(QPalette::ToolTipBase,     Qt::white);
    darkPalette.setColor(QPalette::ToolTipText,     Qt::white);
    darkPalette.setColor(QPalette::Text,            Qt::white);
    darkPalette.setColor(QPalette::Button,          QColor(53, 53, 53));
    darkPalette.setColor(QPalette::ButtonText,      Qt::white);
    darkPalette.setColor(QPalette::BrightText,      Qt::red);
    darkPalette.setColor(QPalette::Link,            QColor(42, 130, 218));
    darkPalette.setColor(QPalette::Highlight,       QColor(42, 130, 218));
    darkPalette.setColor(QPalette::HighlightedText, Qt::black);
    
    darkPalette.setColor(QPalette::Disabled, QPalette::WindowText, QColor(127, 127, 127));
    darkPalette.setColor(QPalette::Disabled, QPalette::Text,       QColor(127, 127, 127));
    darkPalette.setColor(QPalette::Disabled, QPalette::ButtonText, QColor(127, 127, 127));
    
    app.setPalette(darkPalette);
    app.setStyleSheet("QToolTip { color: #ffffff; background-color: #2a82da; border: 1px solid white; }");
}

// Theme Manager
class ThemeManager : public QObject {
    Q_OBJECT
    
public:
    enum Theme { Light, Dark, Custom };
    
    static ThemeManager* instance() {
        static ThemeManager inst;
        return &inst;
    }
    
    void applyTheme(Theme theme) {
        currentTheme = theme;
        switch (theme) {
            case Light:  applyLightTheme(); break;
            case Dark:   applyDarkTheme(); break;
            case Custom: applyCustomTheme(); break;
        }
        emit themeChanged(theme);
    }
    
    Theme current() const { return currentTheme; }
    
signals:
    void themeChanged(Theme);
    
private:
    Theme currentTheme = Light;
    
    void applyLightTheme() {
        qApp->setPalette(QPalette());  // Default
        qApp->setStyleSheet("");
    }
    
    void applyDarkTheme() {
        // (code from above)
    }
    
    void applyCustomTheme() {
        qApp->setStyleSheet(R"(
            * { background-color: #1a1a2e; color: #e0e0e0; }
            QPushButton { 
                background: #16213e; 
                border: 1px solid #0f3460; 
                border-radius: 4px; 
                padding: 6px 12px;
            }
            QPushButton:hover { background: #0f3460; }
        )");
    }
};
```

---

## สรุป Part 013

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt Style Sheets (คล้าย CSS)
2. ✅ QPainter - Custom Drawing
3. ✅ Custom Widgets
4. ✅ Dark Theme

---

⬅️ [Part 012](part012.md) | ➡️ [Part 014: Qt Model/View](part014.md)
