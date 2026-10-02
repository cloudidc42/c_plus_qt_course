# Part 018: Qt Quick & QML

## ขั้นตอนที่ 236-250

---

## ขั้นตอนที่ 236: QML คืออะไร?

QML (Qt Modeling Language) คือภาษา declarative สำหรับสร้าง UI ที่ทันสมัย:

```
Qt Quick vs Qt Widgets:
  Qt Widgets:
    ✓ Desktop applications
    ✓ Native look & feel
    ✓ Mature and stable
    ✗ Less flexible animations

  Qt Quick (QML):
    ✓ Fluid animations
    ✓ Touch-friendly
    ✓ Modern UI design
    ✓ Mobile-ready
    ✓ JavaScript integration
```

**เพิ่มใน .pro:**
```qmake
QT += qml quick quickcontrols2
```

---

## ขั้นตอนที่ 237: Hello QML

```qml
// === main.qml ===
import QtQuick
import QtQuick.Controls

ApplicationWindow {
    id: root
    width: 400
    height: 300
    title: "Hello QML"
    visible: true
    
    // Background gradient
    Rectangle {
        anchors.fill: parent
        gradient: Gradient {
            GradientStop { position: 0.0; color: "#2c3e50" }
            GradientStop { position: 1.0; color: "#3498db" }
        }
    }
    
    Column {
        anchors.centerIn: parent
        spacing: 20
        
        Text {
            id: titleText
            text: "สวัสดี QML!"
            font.pixelSize: 32
            font.bold: true
            color: "white"
            anchors.horizontalCenter: parent.horizontalCenter
            
            // Animation
            SequentialAnimation on opacity {
                loops: Animation.Infinite
                NumberAnimation { to: 0.3; duration: 1000 }
                NumberAnimation { to: 1.0; duration: 1000 }
            }
        }
        
        Button {
            text: "คลิกฉัน"
            anchors.horizontalCenter: parent.horizontalCenter
            
            background: Rectangle {
                color: parent.pressed ? "#e74c3c" : 
                       parent.hovered ? "#c0392b" : "#e74c3c"
                radius: 6
                
                Behavior on color {
                    ColorAnimation { duration: 200 }
                }
            }
            
            contentItem: Text {
                text: parent.text
                color: "white"
                font.pixelSize: 16
                horizontalAlignment: Text.AlignHCenter
                verticalAlignment: Text.AlignVCenter
            }
            
            onClicked: {
                titleText.text = "ถูกคลิกแล้ว!"
            }
        }
    }
}
```

```cpp
// === main.cpp ===
#include <QApplication>
#include <QQmlApplicationEngine>
#include <QQmlContext>

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    QQmlApplicationEngine engine;
    
    // Load QML
    engine.load(QUrl("qrc:/main.qml"));
    
    if (engine.rootObjects().isEmpty()) {
        return -1;
    }
    
    return app.exec();
}
```

---

## ขั้นตอนที่ 238: QML Fundamentals

```qml
// === types.qml - QML Types พื้นฐาน ===
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

Item {
    width: 600
    height: 700
    
    // === Properties ===
    property string myString: "Hello"
    property int myInt: 42
    property real myReal: 3.14
    property bool myBool: true
    property color myColor: "#ff0000"
    property var myList: [1, 2, 3, 4, 5]
    
    // Computed property
    property int doubled: myInt * 2
    
    // === Anchors ===
    // anchors.fill: parent      - เต็ม parent
    // anchors.centerIn: parent  - กลาง parent
    // anchors.left: parent.left - ชิดซ้าย parent
    // anchors.margins: 10       - margin รอบด้าน
    
    // === Basic Items ===
    Column {
        anchors.fill: parent
        anchors.margins: 20
        spacing: 10
        
        // Rectangle
        Rectangle {
            width: 200; height: 50
            color: "#3498db"
            radius: 8
            
            Text {
                anchors.centerIn: parent
                text: "Rectangle"
                color: "white"
                font.pixelSize: 16
            }
        }
        
        // Image
        Image {
            width: 100; height: 100
            source: "https://via.placeholder.com/100"
            fillMode: Image.PreserveAspectFit
        }
        
        // Text types
        Text {
            text: "Plain text"
            font.pixelSize: 16
            color: "#2c3e50"
        }
        
        TextInput {
            width: 200; height: 36
            text: "TextInput"
            font.pixelSize: 14
            color: "#2c3e50"
            
            Rectangle {
                anchors.fill: parent
                color: "transparent"
                border.color: parent.activeFocus ? "#3498db" : "#bdc3c7"
                border.width: 2
                radius: 4
                z: -1
            }
        }
        
        // MouseArea
        Rectangle {
            width: 150; height: 50
            color: mouseArea.pressed ? "#e74c3c" : 
                   mouseArea.containsMouse ? "#c0392b" : "#e74c3c"
            radius: 6
            
            Text {
                anchors.centerIn: parent
                text: "Click Me"
                color: "white"
            }
            
            MouseArea {
                id: mouseArea
                anchors.fill: parent
                hoverEnabled: true
                onClicked: console.log("Clicked!")
            }
        }
    }
}
```

---

## ขั้นตอนที่ 239: QML Layouts

```qml
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

ApplicationWindow {
    width: 600; height: 500
    visible: true
    title: "QML Layouts"
    
    ColumnLayout {
        anchors.fill: parent
        anchors.margins: 10
        spacing: 10
        
        // === Row Layout ===
        GroupBox {
            title: "RowLayout"
            Layout.fillWidth: true
            
            RowLayout {
                anchors.fill: parent
                spacing: 8
                
                Button { text: "Left"; Layout.preferredWidth: 80 }
                Button { text: "Fill"; Layout.fillWidth: true }
                Button { text: "Right"; Layout.preferredWidth: 80 }
            }
        }
        
        // === Grid Layout ===
        GroupBox {
            title: "GridLayout"
            Layout.fillWidth: true
            
            GridLayout {
                anchors.fill: parent
                columns: 3
                rowSpacing: 6
                columnSpacing: 6
                
                Button { text: "A"; Layout.fillWidth: true }
                Button { text: "B"; Layout.fillWidth: true; Layout.columnSpan: 2 }
                Button { text: "C"; Layout.fillWidth: true; Layout.rowSpan: 2 }
                Button { text: "D"; Layout.fillWidth: true }
                Button { text: "E"; Layout.fillWidth: true }
            }
        }
        
        // === Stack Layout ===
        GroupBox {
            title: "StackLayout"
            Layout.fillWidth: true
            Layout.fillHeight: true
            
            ColumnLayout {
                anchors.fill: parent
                
                RowLayout {
                    Repeater {
                        model: ["Tab 1", "Tab 2", "Tab 3"]
                        Button {
                            text: modelData
                            onClicked: stack.currentIndex = index
                            background: Rectangle {
                                color: stack.currentIndex === index ? "#3498db" : "#ecf0f1"
                                radius: 4
                            }
                            contentItem: Text {
                                text: parent.text
                                color: stack.currentIndex === parent.index ? "white" : "black"
                                horizontalAlignment: Text.AlignHCenter
                                verticalAlignment: Text.AlignVCenter
                            }
                        }
                    }
                }
                
                StackLayout {
                    id: stack
                    Layout.fillWidth: true
                    Layout.fillHeight: true
                    
                    Rectangle { color: "#d5e8d4"; 
                        Text { anchors.centerIn: parent; text: "เนื้อหา Tab 1" } }
                    Rectangle { color: "#dae8fc";
                        Text { anchors.centerIn: parent; text: "เนื้อหา Tab 2" } }
                    Rectangle { color: "#ffe6cc";
                        Text { anchors.centerIn: parent; text: "เนื้อหา Tab 3" } }
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 240: QML Animations

```qml
import QtQuick
import QtQuick.Controls

ApplicationWindow {
    width: 700; height: 500
    visible: true
    title: "QML Animations"
    
    // === NumberAnimation ===
    Rectangle {
        id: ball
        x: 50; y: 50
        width: 60; height: 60
        radius: 30
        color: "#e74c3c"
        
        // Behavior - animation on property change
        Behavior on x { NumberAnimation { duration: 500; easing.type: Easing.OutBounce } }
        Behavior on y { NumberAnimation { duration: 500; easing.type: Easing.OutBounce } }
        Behavior on color { ColorAnimation { duration: 300 } }
        
        Text {
            anchors.centerIn: parent
            text: "Ball"
            color: "white"
        }
        
        MouseArea {
            anchors.fill: parent
            onClicked: {
                ball.x = Math.random() * 600
                ball.y = Math.random() * 400
                ball.color = Qt.rgba(Math.random(), Math.random(), Math.random(), 1)
            }
        }
    }
    
    // === SequentialAnimation ===
    Rectangle {
        id: box1
        x: 100; y: 200
        width: 80; height: 80
        color: "#3498db"
        
        Text { anchors.centerIn: parent; text: "Seq"; color: "white" }
        
        SequentialAnimation {
            id: seqAnim
            loops: Animation.Infinite
            
            NumberAnimation { target: box1; property: "x"; to: 500; duration: 1000 }
            RotationAnimation { target: box1; to: 360; duration: 500 }
            NumberAnimation { target: box1; property: "x"; to: 100; duration: 1000 }
            RotationAnimation { target: box1; to: 0; duration: 500 }
        }
        
        Component.onCompleted: seqAnim.start()
    }
    
    // === ParallelAnimation ===
    Rectangle {
        id: box2
        x: 300; y: 350
        width: 70; height: 70
        color: "#27ae60"
        
        Text { anchors.centerIn: parent; text: "Para"; color: "white" }
        
        ParallelAnimation {
            loops: Animation.Infinite
            
            NumberAnimation { target: box2; property: "y"; 
                              from: 350; to: 300; duration: 800;
                              easing.type: Easing.InOutSine }
            ScaleAnimator { target: box2; from: 1; to: 1.3; duration: 800 }
        }
    }
    
    // === PathAnimation ===
    Rectangle {
        id: pathItem
        width: 50; height: 50
        color: "#e67e22"
        radius: 25
        
        PathAnimation {
            target: pathItem
            duration: 3000
            loops: Animation.Infinite
            
            path: Path {
                startX: 100; startY: 400
                PathCurve { x: 300; y: 200 }
                PathCurve { x: 500; y: 400 }
                PathCurve { x: 300; y: 450 }
                PathCurve { x: 100; y: 400 }
            }
        }
    }
    
    // === Transition ===
    states: [
        State {
            name: "expanded"
            PropertyChanges { target: expandRect; width: 300; height: 200 }
        }
    ]
    
    transitions: [
        Transition {
            NumberAnimation { properties: "width, height"; duration: 400; 
                              easing.type: Easing.OutExpo }
        }
    ]
    
    Rectangle {
        id: expandRect
        x: 10; y: 10
        width: 100; height: 60
        color: "#9b59b6"
        
        Text { anchors.centerIn: parent; text: "Click"; color: "white" }
        
        MouseArea {
            anchors.fill: parent
            onClicked: {
                if (root.state === "expanded") root.state = ""
                else root.state = "expanded"
            }
        }
    }
}
```

---

## ขั้นตอนที่ 241-250: C++ ↔ QML Integration

```cpp
// === Backend class ที่ expose ให้ QML ===
// backend.h
#pragma once
#include <QObject>
#include <QString>
#include <QVariantList>

class Backend : public QObject {
    Q_OBJECT
    
    // Q_PROPERTY ให้ QML อ่าน/เขียนได้
    Q_PROPERTY(QString name READ name WRITE setName NOTIFY nameChanged)
    Q_PROPERTY(int count READ count NOTIFY countChanged)
    Q_PROPERTY(QVariantList items READ items NOTIFY itemsChanged)
    
private:
    QString m_name;
    int m_count = 0;
    QStringList m_items;
    
public:
    explicit Backend(QObject* parent = nullptr) : QObject(parent) {}
    
    QString name() const { return m_name; }
    int count() const { return m_count; }
    QVariantList items() const {
        QVariantList list;
        for (const QString& s : m_items) list << s;
        return list;
    }
    
    void setName(const QString& n) {
        if (m_name != n) {
            m_name = n;
            emit nameChanged(n);
        }
    }
    
    // Q_INVOKABLE ให้ QML เรียกได้
    Q_INVOKABLE void increment() {
        m_count++;
        emit countChanged(m_count);
    }
    
    Q_INVOKABLE void decrement() {
        if (m_count > 0) {
            m_count--;
            emit countChanged(m_count);
        }
    }
    
    Q_INVOKABLE void addItem(const QString& item) {
        m_items.append(item);
        emit itemsChanged();
    }
    
    Q_INVOKABLE void removeItem(int index) {
        if (index >= 0 && index < m_items.size()) {
            m_items.removeAt(index);
            emit itemsChanged();
        }
    }
    
    Q_INVOKABLE QString formatGreeting(const QString& lang) const {
        if (lang == "th") return "สวัสดี " + m_name + "!";
        return "Hello " + m_name + "!";
    }
    
signals:
    void nameChanged(const QString& name);
    void countChanged(int count);
    void itemsChanged();
    void messageReceived(const QString& msg);
};
```

```cpp
// === main.cpp ===
int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    QQmlApplicationEngine engine;
    
    // สร้าง Backend
    Backend backend;
    
    // Expose ให้ QML ใช้ได้
    engine.rootContext()->setContextProperty("backend", &backend);
    
    // หรือ register เป็น type
    // qmlRegisterType<Backend>("com.myapp", 1, 0, "Backend");
    
    engine.load(QUrl("qrc:/main.qml"));
    return app.exec();
}
```

```qml
// === main.qml - ใช้ C++ Backend ===
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

ApplicationWindow {
    width: 450; height: 500
    visible: true
    title: "C++ + QML"
    
    ColumnLayout {
        anchors.fill: parent
        anchors.margins: 20
        spacing: 10
        
        // === Property binding ===
        TextField {
            Layout.fillWidth: true
            placeholderText: "ชื่อ..."
            text: backend.name
            onTextChanged: backend.name = text
        }
        
        Text {
            text: backend.formatGreeting("th")
            font.pixelSize: 18
            color: "#2c3e50"
        }
        
        // === Counter ===
        RowLayout {
            Button { text: "-"; onClicked: backend.decrement() }
            
            Text {
                text: backend.count
                font.pixelSize: 24
                font.bold: true
                Layout.fillWidth: true
                horizontalAlignment: Text.AlignHCenter
            }
            
            Button { text: "+"; onClicked: backend.increment() }
        }
        
        // === List ===
        GroupBox {
            title: "รายการ"
            Layout.fillWidth: true
            Layout.fillHeight: true
            
            ColumnLayout {
                anchors.fill: parent
                
                RowLayout {
                    TextField {
                        id: itemInput
                        Layout.fillWidth: true
                        placeholderText: "เพิ่มรายการ..."
                    }
                    Button {
                        text: "เพิ่ม"
                        onClicked: {
                            if (itemInput.text.length > 0) {
                                backend.addItem(itemInput.text)
                                itemInput.clear()
                            }
                        }
                    }
                }
                
                ListView {
                    Layout.fillWidth: true
                    Layout.fillHeight: true
                    model: backend.items
                    
                    delegate: ItemDelegate {
                        width: ListView.view.width
                        text: modelData
                        
                        contentItem: RowLayout {
                            Text {
                                Layout.fillWidth: true
                                text: modelData
                            }
                            Button {
                                text: "✕"
                                flat: true
                                onClicked: backend.removeItem(index)
                            }
                        }
                    }
                    
                    ScrollIndicator.vertical: ScrollIndicator {}
                }
            }
        }
    }
    
    // Listen to signals from C++
    Connections {
        target: backend
        function onMessageReceived(msg) {
            messagePopup.message = msg
            messagePopup.open()
        }
    }
    
    Dialog {
        id: messagePopup
        property string message: ""
        title: "ข้อความ"
        Label { text: messagePopup.message }
        standardButtons: Dialog.Ok
    }
}
```

---

## สรุป Part 018

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QML Basics
2. ✅ QML Types (Rectangle, Text, Image, MouseArea)
3. ✅ QML Layouts (Row, Column, Grid, Stack)
4. ✅ QML Animations (Number, Sequential, Parallel, Path)
5. ✅ C++ ↔ QML Integration ด้วย Q_PROPERTY และ Q_INVOKABLE

---

⬅️ [Part 017](part017.md) | ➡️ [Part 019: Qt Charts & Data Visualization](part019.md)
