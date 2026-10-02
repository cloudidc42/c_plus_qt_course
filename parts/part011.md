# Part 011: Qt Framework - Introduction

## ขั้นตอนที่ 131-145

---

## ขั้นตอนที่ 131: Qt Framework คืออะไร?

Qt Framework เป็น cross-platform framework สำหรับพัฒนา GUI Application ด้วย C++ และรองรับหลายแพลตฟอร์ม:

```
แพลตฟอร์มที่รองรับ:
  ├── Desktop: Windows, macOS, Linux
  ├── Mobile:  Android, iOS
  ├── Embedded: QNX, VxWorks, Integrity
  └── WebAssembly: Browser

โมดูลสำคัญ:
  ├── Qt Core        - Core non-GUI functionality
  ├── Qt GUI         - Base classes for GUI
  ├── Qt Widgets     - Classic Desktop UI
  ├── Qt Quick       - Modern declarative UI (QML)
  ├── Qt Network     - Networking
  ├── Qt SQL         - Database access
  ├── Qt Multimedia  - Audio/Video
  └── Qt WebEngine   - Chromium-based browser
```

### การติดตั้ง Qt

**Windows:**
```
1. ดาวน์โหลด Qt Online Installer จาก qt.io
2. เลือก Qt 6.x.x
3. เลือก MSVC หรือ MinGW compiler
4. ติดตั้ง Qt Creator IDE
```

**macOS:**
```
1. ดาวน์โหลด Qt Online Installer
2. หรือใช้ Homebrew: brew install qt
3. เลือก Clang compiler
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install qt6-base-dev qt6-tools-dev qt6-tools-dev-tools
sudo apt install qtcreator
# หรือ
sudo apt install qt5-default qtcreator
```

---

## ขั้นตอนที่ 132: Qt Project Structure

```
MyProject/
├── MyProject.pro         ← Qt Project File (qmake)
├── CMakeLists.txt        ← CMake build file (Qt6)
├── main.cpp              ← Entry point
├── mainwindow.h          ← Window header
├── mainwindow.cpp        ← Window implementation
├── mainwindow.ui         ← UI Designer file (XML)
└── resources.qrc         ← Resource file (icons, images)
```

### Qt Project File (.pro)
```qmake
QT += core gui widgets

greaterThan(QT_MAJOR_VERSION, 4): QT += widgets

CONFIG += c++17

TARGET = MyProject
TEMPLATE = app

SOURCES += \
    main.cpp \
    mainwindow.cpp

HEADERS += \
    mainwindow.h

FORMS += \
    mainwindow.ui

RESOURCES += resources.qrc
```

### CMakeLists.txt (Qt6)
```cmake
cmake_minimum_required(VERSION 3.16)
project(MyProject VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets)

add_executable(MyProject
    main.cpp
    mainwindow.cpp
    mainwindow.h
    mainwindow.ui
)

target_link_libraries(MyProject PRIVATE
    Qt6::Core
    Qt6::Gui
    Qt6::Widgets
)
```

---

## ขั้นตอนที่ 133: Hello World Qt

```cpp
// === main.cpp ===
#include <QApplication>
#include <QLabel>
#include <QPushButton>
#include <QVBoxLayout>
#include <QWidget>
#include <QMessageBox>

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    // สร้าง Window หลัก
    QWidget window;
    window.setWindowTitle("Hello Qt!");
    window.setFixedSize(300, 200);
    
    // Layout
    QVBoxLayout* layout = new QVBoxLayout(&window);
    
    // Label
    QLabel* label = new QLabel("สวัสดี Qt Framework!");
    label->setAlignment(Qt::AlignCenter);
    
    QFont font("Arial", 14, QFont::Bold);
    label->setFont(font);
    
    // Button
    QPushButton* button = new QPushButton("คลิกฉัน!");
    
    // Connect signal/slot (C++11 syntax)
    QObject::connect(button, &QPushButton::clicked, [&]() {
        QMessageBox::information(&window, "สวัสดี", "คุณคลิกปุ่มแล้ว!");
        label->setText("ถูกคลิก!");
    });
    
    layout->addWidget(label);
    layout->addWidget(button);
    
    window.show();
    
    return app.exec();
}
```

---

## ขั้นตอนที่ 134: Signals & Slots

Signals & Slots คือกลไกหลักของ Qt สำหรับ event handling และ communication ระหว่าง objects:

```cpp
// === counter.h ===
#pragma once
#include <QObject>

class Counter : public QObject {
    Q_OBJECT  // ← Macro สำคัญ! ต้องมีใน class ที่ใช้ Signal/Slot
    
private:
    int m_count;
    int m_max;
    
public:
    explicit Counter(int maxVal = 10, QObject* parent = nullptr)
        : QObject(parent), m_count(0), m_max(maxVal) {}
    
    int count() const { return m_count; }
    
public slots:
    void increment() {
        if (m_count < m_max) {
            m_count++;
            emit countChanged(m_count);
            
            if (m_count == m_max) {
                emit reachedMax();
            }
        }
    }
    
    void decrement() {
        if (m_count > 0) {
            m_count--;
            emit countChanged(m_count);
        }
    }
    
    void reset() {
        m_count = 0;
        emit countChanged(m_count);
    }
    
signals:
    void countChanged(int newCount);
    void reachedMax();
};
```

```cpp
// === display.h ===
#pragma once
#include <QObject>
#include <QLabel>

class Display : public QObject {
    Q_OBJECT
    
private:
    QLabel* label;
    
public:
    explicit Display(QLabel* lbl, QObject* parent = nullptr)
        : QObject(parent), label(lbl) {}
    
public slots:
    void onCountChanged(int count) {
        label->setText(QString("Count: %1").arg(count));
    }
    
    void onReachedMax() {
        label->setStyleSheet("color: red; font-weight: bold;");
        label->setText("ถึงค่าสูงสุดแล้ว!");
    }
};
```

```cpp
// === main.cpp ===
#include <QApplication>
#include <QWidget>
#include <QVBoxLayout>
#include <QLabel>
#include <QPushButton>
#include <QSpinBox>
#include "counter.h"
#include "display.h"

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    QWidget window;
    window.setWindowTitle("Signals & Slots Demo");
    window.setFixedSize(300, 250);
    
    QVBoxLayout* layout = new QVBoxLayout(&window);
    
    QLabel* countLabel = new QLabel("Count: 0");
    countLabel->setAlignment(Qt::AlignCenter);
    countLabel->setFont(QFont("Arial", 16));
    
    QPushButton* incBtn = new QPushButton("+1");
    QPushButton* decBtn = new QPushButton("-1");
    QPushButton* resetBtn = new QPushButton("Reset");
    
    // สร้าง Counter และ Display
    Counter* counter = new Counter(5, &window);
    Display* display = new Display(countLabel, &window);
    
    // เชื่อมต่อ Signals กับ Slots
    QObject::connect(counter, &Counter::countChanged, display, &Display::onCountChanged);
    QObject::connect(counter, &Counter::reachedMax, display, &Display::onReachedMax);
    
    // Button connections
    QObject::connect(incBtn, &QPushButton::clicked, counter, &Counter::increment);
    QObject::connect(decBtn, &QPushButton::clicked, counter, &Counter::decrement);
    QObject::connect(resetBtn, &QPushButton::clicked, counter, &Counter::reset);
    QObject::connect(resetBtn, &QPushButton::clicked, [&countLabel]() {
        countLabel->setStyleSheet("");
    });
    
    layout->addWidget(countLabel);
    layout->addWidget(incBtn);
    layout->addWidget(decBtn);
    layout->addWidget(resetBtn);
    
    window.show();
    return app.exec();
}
```

---

## ขั้นตอนที่ 135: QObject และ Object Tree

```cpp
// === objecttree.h ===
#pragma once
#include <QObject>
#include <QString>

class Animal : public QObject {
    Q_OBJECT
    
    Q_PROPERTY(QString name READ name WRITE setName NOTIFY nameChanged)
    Q_PROPERTY(int age READ age WRITE setAge NOTIFY ageChanged)
    
private:
    QString m_name;
    int m_age;
    
public:
    explicit Animal(const QString& name, int age, QObject* parent = nullptr)
        : QObject(parent), m_name(name), m_age(age) {
        setObjectName(name);  // ตั้งชื่อ object
    }
    
    QString name() const { return m_name; }
    int age() const { return m_age; }
    
    void setName(const QString& name) {
        if (m_name != name) {
            m_name = name;
            emit nameChanged(name);
        }
    }
    
    void setAge(int age) {
        if (m_age != age) {
            m_age = age;
            emit ageChanged(age);
        }
    }
    
    QString toString() const {
        return QString("%1 (age=%2)").arg(m_name).arg(m_age);
    }
    
signals:
    void nameChanged(const QString& name);
    void ageChanged(int age);
};

// สาธิต Object Tree
void demonstrateObjectTree() {
    // Parent-child relationship
    QObject* zoo = new QObject();
    zoo->setObjectName("Zoo");
    
    Animal* lion = new Animal("Lion", 5, zoo);    // zoo เป็น parent
    Animal* tiger = new Animal("Tiger", 3, zoo);  // zoo เป็น parent
    Animal* bear = new Animal("Bear", 7, zoo);    // zoo เป็น parent
    
    // ลูกของ lion
    Animal* cub1 = new Animal("Cub1", 1, lion);
    Animal* cub2 = new Animal("Cub2", 1, lion);
    
    // ดู children
    qDebug() << "Zoo children:" << zoo->children().size();
    for (auto* child : zoo->children()) {
        auto* a = qobject_cast<Animal*>(child);
        if (a) qDebug() << " -" << a->toString();
    }
    
    // find child by name
    auto* foundTiger = zoo->findChild<Animal*>("Tiger");
    if (foundTiger) {
        qDebug() << "Found:" << foundTiger->toString();
    }
    
    // delete parent = delete ทุก children
    delete zoo;
    // lion, tiger, bear, cub1, cub2 ถูก delete ไปด้วยหมด!
}
```

---

## ขั้นตอนที่ 136: Qt Widgets พื้นฐาน

```cpp
// === widgetdemo.cpp ===
#include <QApplication>
#include <QMainWindow>
#include <QWidget>
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QGridLayout>
#include <QLabel>
#include <QPushButton>
#include <QLineEdit>
#include <QTextEdit>
#include <QComboBox>
#include <QSpinBox>
#include <QCheckBox>
#include <QRadioButton>
#include <QSlider>
#include <QProgressBar>
#include <QGroupBox>
#include <QScrollArea>

class WidgetGallery : public QWidget {
    Q_OBJECT
    
public:
    explicit WidgetGallery(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("Qt Widget Gallery");
        setMinimumSize(500, 600);
        
        auto* mainLayout = new QVBoxLayout(this);
        mainLayout->setSpacing(10);
        mainLayout->setContentsMargins(15, 15, 15, 15);
        
        // === Buttons Group ===
        auto* btnGroup = new QGroupBox("Buttons");
        auto* btnLayout = new QHBoxLayout(btnGroup);
        
        auto* normalBtn = new QPushButton("Normal");
        auto* toggleBtn = new QPushButton("Toggle");
        toggleBtn->setCheckable(true);
        auto* iconBtn = new QPushButton("Icon Button");
        iconBtn->setIcon(QIcon::fromTheme("document-open"));
        
        btnLayout->addWidget(normalBtn);
        btnLayout->addWidget(toggleBtn);
        btnLayout->addWidget(iconBtn);
        
        // === Input Group ===
        auto* inputGroup = new QGroupBox("Input");
        auto* inputLayout = new QGridLayout(inputGroup);
        
        inputLayout->addWidget(new QLabel("Name:"), 0, 0);
        auto* nameEdit = new QLineEdit();
        nameEdit->setPlaceholderText("ใส่ชื่อของคุณ...");
        inputLayout->addWidget(nameEdit, 0, 1);
        
        inputLayout->addWidget(new QLabel("Password:"), 1, 0);
        auto* passEdit = new QLineEdit();
        passEdit->setEchoMode(QLineEdit::Password);
        passEdit->setPlaceholderText("ใส่รหัสผ่าน...");
        inputLayout->addWidget(passEdit, 1, 1);
        
        inputLayout->addWidget(new QLabel("Age:"), 2, 0);
        auto* ageSpin = new QSpinBox();
        ageSpin->setRange(1, 120);
        ageSpin->setValue(25);
        inputLayout->addWidget(ageSpin, 2, 1);
        
        // === Selection Group ===
        auto* selectGroup = new QGroupBox("Selection");
        auto* selectLayout = new QVBoxLayout(selectGroup);
        
        auto* combo = new QComboBox();
        combo->addItems({"C++", "Python", "Java", "JavaScript", "Go", "Rust"});
        selectLayout->addWidget(combo);
        
        auto* chk1 = new QCheckBox("รับข่าวสาร");
        auto* chk2 = new QCheckBox("ยอมรับเงื่อนไข");
        chk2->setChecked(true);
        selectLayout->addWidget(chk1);
        selectLayout->addWidget(chk2);
        
        auto* rb1 = new QRadioButton("Male");
        auto* rb2 = new QRadioButton("Female");
        auto* rb3 = new QRadioButton("Other");
        rb1->setChecked(true);
        auto* rbLayout = new QHBoxLayout();
        rbLayout->addWidget(rb1);
        rbLayout->addWidget(rb2);
        rbLayout->addWidget(rb3);
        selectLayout->addLayout(rbLayout);
        
        // === Slider & Progress ===
        auto* progressGroup = new QGroupBox("Slider & Progress");
        auto* progressLayout = new QVBoxLayout(progressGroup);
        
        auto* slider = new QSlider(Qt::Horizontal);
        slider->setRange(0, 100);
        slider->setValue(50);
        
        auto* progress = new QProgressBar();
        progress->setRange(0, 100);
        progress->setValue(50);
        
        connect(slider, &QSlider::valueChanged, progress, &QProgressBar::setValue);
        
        progressLayout->addWidget(slider);
        progressLayout->addWidget(progress);
        
        // === Text Area ===
        auto* textGroup = new QGroupBox("Text");
        auto* textLayout = new QVBoxLayout(textGroup);
        
        auto* textEdit = new QTextEdit();
        textEdit->setPlaceholderText("พิมพ์ข้อความที่นี่...");
        textEdit->setMaximumHeight(100);
        textLayout->addWidget(textEdit);
        
        // Status Label
        auto* statusLabel = new QLabel("พร้อมใช้งาน");
        statusLabel->setStyleSheet("color: green; font-weight: bold;");
        
        // Connect some events
        connect(normalBtn, &QPushButton::clicked, [=]() {
            statusLabel->setText("คลิกปุ่ม Normal: " + nameEdit->text());
        });
        
        connect(combo, &QComboBox::currentTextChanged, [=](const QString& text) {
            statusLabel->setText("เลือก: " + text);
        });
        
        // Add all groups
        mainLayout->addWidget(btnGroup);
        mainLayout->addWidget(inputGroup);
        mainLayout->addWidget(selectGroup);
        mainLayout->addWidget(progressGroup);
        mainLayout->addWidget(textGroup);
        mainLayout->addWidget(statusLabel);
    }
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    WidgetGallery gallery;
    gallery.show();
    
    return app.exec();
}

#include "widgetdemo.moc"
```

---

## ขั้นตอนที่ 137-145: Qt Layouts

```cpp
// Layout Types ใน Qt:
// 1. QVBoxLayout - จัดเรียงแนวตั้ง
// 2. QHBoxLayout - จัดเรียงแนวนอน
// 3. QGridLayout - จัดเรียงแบบตาราง
// 4. QFormLayout - จัดเรียงแบบ Form (label + input)
// 5. QStackedLayout - ซ้อน widgets

#include <QApplication>
#include <QWidget>
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QGridLayout>
#include <QFormLayout>
#include <QStackedLayout>
#include <QLabel>
#include <QPushButton>
#include <QLineEdit>
#include <QComboBox>
#include <QGroupBox>
#include <QFrame>

class LayoutDemo : public QWidget {
    Q_OBJECT
    
public:
    LayoutDemo(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("Qt Layout Demo");
        setMinimumSize(600, 500);
        
        auto* main = new QVBoxLayout(this);
        
        // === HBox ===
        auto* hboxGroup = new QGroupBox("QHBoxLayout");
        auto* hbox = new QHBoxLayout(hboxGroup);
        hbox->addWidget(new QPushButton("Left"));
        hbox->addStretch();  // ดึงพื้นที่ระหว่าง
        hbox->addWidget(new QPushButton("Center-Right"));
        hbox->addWidget(new QPushButton("Right"));
        main->addWidget(hboxGroup);
        
        // === Grid ===
        auto* gridGroup = new QGroupBox("QGridLayout");
        auto* grid = new QGridLayout(gridGroup);
        
        // span rows/columns
        grid->addWidget(new QPushButton("(0,0)"), 0, 0);
        grid->addWidget(new QPushButton("(0,1)-(0,2) span"), 0, 1, 1, 2);
        grid->addWidget(new QPushButton("(1,0)"), 1, 0);
        grid->addWidget(new QPushButton("(1,1)"), 1, 1);
        grid->addWidget(new QPushButton("(1,2)"), 1, 2);
        grid->addWidget(new QPushButton("(2,0)-(3,0) span"), 2, 0, 2, 1);
        grid->addWidget(new QPushButton("(2,1)"), 2, 1);
        grid->addWidget(new QPushButton("(2,2)"), 2, 2);
        grid->addWidget(new QPushButton("(3,1)-(3,2) span"), 3, 1, 1, 2);
        
        main->addWidget(gridGroup);
        
        // === Form ===
        auto* formGroup = new QGroupBox("QFormLayout");
        auto* form = new QFormLayout(formGroup);
        form->addRow("ชื่อ:", new QLineEdit());
        form->addRow("อีเมล:", new QLineEdit());
        form->addRow("ประเทศ:", new QComboBox());
        form->addRow("ที่อยู่:", new QLineEdit());
        main->addWidget(formGroup);
        
        // === Nested Layouts ===
        auto* nestedGroup = new QGroupBox("Nested Layouts");
        auto* nestedMain = new QHBoxLayout(nestedGroup);
        
        // Left column
        auto* leftCol = new QVBoxLayout();
        leftCol->addWidget(new QLabel("Left Column"));
        leftCol->addWidget(new QPushButton("A"));
        leftCol->addWidget(new QPushButton("B"));
        leftCol->addStretch();
        
        // Separator
        auto* sep = new QFrame();
        sep->setFrameShape(QFrame::VLine);
        
        // Right column
        auto* rightCol = new QVBoxLayout();
        rightCol->addWidget(new QLabel("Right Column"));
        auto* rightGrid = new QGridLayout();
        for (int r = 0; r < 2; r++) {
            for (int c = 0; c < 2; c++) {
                rightGrid->addWidget(
                    new QPushButton(QString("%1,%2").arg(r).arg(c)), r, c
                );
            }
        }
        rightCol->addLayout(rightGrid);
        rightCol->addStretch();
        
        nestedMain->addLayout(leftCol);
        nestedMain->addWidget(sep);
        nestedMain->addLayout(rightCol);
        
        main->addWidget(nestedGroup);
    }
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    LayoutDemo demo;
    demo.show();
    
    return app.exec();
}

#include "layoutdemo.moc"
```

---

## สรุป Part 011

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt Framework overview และการติดตั้ง
2. ✅ Qt Project Structure (.pro และ CMakeLists.txt)
3. ✅ Hello World Qt Application
4. ✅ Signals & Slots mechanism
5. ✅ QObject และ Object Tree
6. ✅ Qt Widgets พื้นฐาน (Button, Label, Input, etc.)
7. ✅ Qt Layouts (HBox, VBox, Grid, Form, Nested)

---

⬅️ [Part 010](part010.md) | ➡️ [Part 012: Qt MainWindow & Menus](part012.md)
