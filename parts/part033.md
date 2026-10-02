# Part 033: QML Advanced

## ขั้นตอนที่ 461-475

---

## ขั้นตอนที่ 461: QML Components & Reusability

```qml
// === CustomButton.qml ===
import QtQuick
import QtQuick.Controls

Button {
    id: root
    
    // Custom properties
    property color bgColor: "#3498db"
    property color hoverColor: Qt.darker(bgColor, 1.2)
    property color textColor: "white"
    property int radius: 8
    property bool loading: false
    
    // Custom signals
    signal longPressed()
    
    implicitWidth: 120
    implicitHeight: 40
    
    contentItem: Row {
        spacing: 8
        anchors.centerIn: parent
        
        BusyIndicator {
            visible: root.loading
            running: root.loading
            width: 20; height: 20
        }
        
        Text {
            text: root.text
            color: root.textColor
            font.pixelSize: 14
            font.weight: Font.Medium
            verticalAlignment: Text.AlignVCenter
        }
    }
    
    background: Rectangle {
        color: root.pressed ? Qt.darker(root.bgColor, 1.4)
             : root.hovered ? root.hoverColor
             : root.bgColor
        radius: root.radius
        
        Behavior on color {
            ColorAnimation { duration: 150 }
        }
    }
    
    // Long press detection
    Timer {
        id: longPressTimer
        interval: 800
        onTriggered: root.longPressed()
    }
    
    onPressed: longPressTimer.start()
    onReleased: longPressTimer.stop()
    onCanceled: longPressTimer.stop()
}

// === Card.qml ===
import QtQuick
import QtQuick.Layouts

Rectangle {
    id: root
    
    property string title: ""
    property string subtitle: ""
    property color accentColor: "#3498db"
    default property alias content: contentItem.children
    
    radius: 12
    color: "white"
    
    layer.enabled: true
    layer.effect: MultiEffect {
        shadowEnabled: true
        shadowColor: "#20000000"
        shadowBlur: 0.8
        shadowVerticalOffset: 4
    }
    
    ColumnLayout {
        anchors.fill: parent
        anchors.margins: 16
        spacing: 8
        
        // Title area
        RowLayout {
            visible: title.length > 0
            
            Rectangle {
                width: 4
                height: 20
                radius: 2
                color: root.accentColor
            }
            
            ColumnLayout {
                spacing: 2
                
                Text {
                    text: root.title
                    font.pixelSize: 16
                    font.weight: Font.Bold
                    color: "#2c3e50"
                }
                
                Text {
                    text: root.subtitle
                    font.pixelSize: 12
                    color: "#7f8c8d"
                    visible: subtitle.length > 0
                }
            }
        }
        
        Rectangle {
            height: 1
            color: "#ecf0f1"
            Layout.fillWidth: true
            visible: title.length > 0
        }
        
        // Content area
        Item {
            id: contentItem
            Layout.fillWidth: true
            Layout.fillHeight: true
        }
    }
}

// === Using components ===
import QtQuick
import QtQuick.Layouts

ApplicationWindow {
    visible: true
    width: 800
    height: 600
    title: "QML Components Demo"
    
    ColumnLayout {
        anchors.fill: parent
        anchors.margins: 20
        spacing: 16
        
        Card {
            Layout.fillWidth: true
            height: 150
            title: "User Profile"
            subtitle: "Edit your information"
            accentColor: "#9b59b6"
            
            RowLayout {
                anchors.fill: parent
                spacing: 16
                
                Rectangle {
                    width: 64; height: 64
                    radius: 32
                    color: "#9b59b6"
                    
                    Text {
                        anchors.centerIn: parent
                        text: "JD"
                        color: "white"
                        font.pixelSize: 24
                        font.bold: true
                    }
                }
                
                ColumnLayout {
                    Layout.fillWidth: true
                    
                    Text { text: "John Doe"; font.pixelSize: 18; font.bold: true }
                    Text { text: "john.doe@example.com"; color: "#7f8c8d" }
                    Text { text: "Senior Developer"; color: "#3498db" }
                }
                
                CustomButton {
                    text: "Edit"
                    bgColor: "#9b59b6"
                    onClicked: console.log("Edit profile")
                }
            }
        }
        
        Card {
            Layout.fillWidth: true
            height: 100
            title: "Actions"
            
            RowLayout {
                anchors.fill: parent
                spacing: 12
                
                CustomButton { text: "Save"; bgColor: "#27ae60" }
                CustomButton { text: "Share"; bgColor: "#3498db" }
                CustomButton { text: "Delete"; bgColor: "#e74c3c" }
                CustomButton { 
                    text: "Process"
                    bgColor: "#e67e22"
                    loading: true
                }
                
                Item { Layout.fillWidth: true }
                
                CustomButton {
                    text: "Hold Me"
                    bgColor: "#8e44ad"
                    onLongPressed: console.log("Long pressed!")
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 462: QML Animations & Transitions

```qml
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

ApplicationWindow {
    visible: true
    width: 800
    height: 600
    title: "QML Animations"
    
    // StackView with animated transitions
    StackView {
        id: stack
        anchors.fill: parent
        
        pushEnter: Transition {
            XAnimator {
                from: stack.width; to: 0
                duration: 300
                easing.type: Easing.OutCubic
            }
            OpacityAnimator { from: 0; to: 1; duration: 300 }
        }
        
        pushExit: Transition {
            XAnimator {
                from: 0; to: -stack.width / 3
                duration: 300
                easing.type: Easing.OutCubic
            }
            OpacityAnimator { from: 1; to: 0.7; duration: 300 }
        }
        
        popEnter: Transition {
            XAnimator {
                from: -stack.width / 3; to: 0
                duration: 300
                easing.type: Easing.OutCubic
            }
            OpacityAnimator { from: 0.7; to: 1; duration: 300 }
        }
        
        popExit: Transition {
            XAnimator {
                from: 0; to: stack.width
                duration: 300
                easing.type: Easing.OutCubic
            }
            OpacityAnimator { from: 1; to: 0; duration: 300 }
        }
        
        initialItem: Page1 {}
    }
}

// === Page1.qml ===
Rectangle {
    color: "#f0f2f5"
    
    Column {
        anchors.centerIn: parent
        spacing: 20
        
        Text {
            text: "Page 1"
            font.pixelSize: 32
            font.bold: true
        }
        
        Button {
            text: "Go to Page 2"
            onClicked: StackView.view.push("Page2.qml")
        }
    }
    
    // Staggered entrance animation
    Column {
        id: itemList
        anchors { left: parent.left; right: parent.right; bottom: parent.bottom }
        anchors.margins: 20
        spacing: 8
        
        Repeater {
            model: ["Item 1", "Item 2", "Item 3", "Item 4"]
            
            Rectangle {
                width: parent.width
                height: 50
                radius: 8
                color: "white"
                opacity: 0
                x: -50
                
                Text {
                    anchors.centerIn: parent
                    text: modelData
                    font.pixelSize: 16
                }
                
                // Staggered animation
                Component.onCompleted: {
                    delay.start()
                }
                
                Timer {
                    id: delay
                    interval: index * 100
                    onTriggered: {
                        opAnim.start()
                        xAnim.start()
                    }
                }
                
                NumberAnimation on opacity { id: opAnim; to: 1; duration: 400; easing.type: Easing.OutQuad }
                NumberAnimation on x { id: xAnim; to: 0; duration: 400; easing.type: Easing.OutBack }
            }
        }
    }
}

// === Particle-like animation ===
Rectangle {
    width: 400; height: 400
    color: "#1a1a2e"
    
    Repeater {
        model: 20
        
        Rectangle {
            property real startX: Math.random() * parent.width
            property real startY: Math.random() * parent.height
            property real speed: 1000 + Math.random() * 2000
            property real size: 2 + Math.random() * 6
            
            x: startX
            y: startY
            width: size
            height: size
            radius: size / 2
            color: Qt.hsla(Math.random(), 0.8, 0.6, 0.8)
            
            SequentialAnimation on y {
                loops: Animation.Infinite
                NumberAnimation {
                    from: startY
                    to: -20
                    duration: speed
                    easing.type: Easing.Linear
                }
                NumberAnimation {
                    from: parent.height + 20
                    to: startY
                    duration: 0
                }
            }
            
            NumberAnimation on opacity {
                from: 0.8; to: 0
                duration: speed
                loops: Animation.Infinite
            }
        }
    }
}
```

---

## ขั้นตอนที่ 463: QML ListView & Models

```qml
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

// === ListModel ===
ListModel {
    id: taskModel
    
    ListElement { title: "Setup project"; done: false; priority: "high" }
    ListElement { title: "Write tests"; done: false; priority: "medium" }
    ListElement { title: "Deploy app"; done: false; priority: "high" }
    ListElement { title: "Review code"; done: true; priority: "low" }
}

// === ListView with custom delegate ===
ListView {
    id: taskList
    anchors.fill: parent
    model: taskModel
    spacing: 8
    clip: true
    
    header: Rectangle {
        width: parent.width
        height: 60
        color: "#3498db"
        
        Text {
            anchors.centerIn: parent
            text: "Tasks (" + taskModel.count + ")"
            color: "white"
            font.pixelSize: 20
            font.bold: true
        }
    }
    
    delegate: Rectangle {
        width: ListView.view.width
        height: 70
        radius: 8
        color: "white"
        
        // Priority color indicator
        Rectangle {
            width: 4
            height: parent.height * 0.6
            anchors.verticalCenter: parent.verticalCenter
            anchors.left: parent.left
            anchors.leftMargin: 8
            radius: 2
            color: priority === "high" ? "#e74c3c" 
                 : priority === "medium" ? "#f39c12"
                 : "#27ae60"
        }
        
        RowLayout {
            anchors.fill: parent
            anchors.margins: 12
            anchors.leftMargin: 20
            spacing: 12
            
            // Checkbox
            CheckBox {
                checked: done
                onCheckedChanged: taskModel.setProperty(index, "done", checked)
            }
            
            // Text
            ColumnLayout {
                Layout.fillWidth: true
                spacing: 2
                
                Text {
                    text: title
                    font.pixelSize: 16
                    font.strikeout: done
                    color: done ? "#bdc3c7" : "#2c3e50"
                }
                
                Text {
                    text: priority.charAt(0).toUpperCase() + priority.slice(1) + " Priority"
                    font.pixelSize: 12
                    color: priority === "high" ? "#e74c3c"
                         : priority === "medium" ? "#f39c12"
                         : "#27ae60"
                }
            }
            
            // Delete button
            Button {
                text: "✕"
                width: 30; height: 30
                background: Rectangle { color: "transparent" }
                contentItem: Text { 
                    text: parent.text
                    color: "#bdc3c7"
                    font.pixelSize: 16
                    horizontalAlignment: Text.AlignHCenter
                }
                onClicked: taskModel.remove(index)
                
                hoverEnabled: true
                onHoveredChanged: contentItem.color = hovered ? "#e74c3c" : "#bdc3c7"
            }
        }
        
        // Swipe to delete (touch-friendly)
        SwipeDelegate {
            anchors.fill: parent
            swipe.right: Rectangle {
                width: 80; height: parent.height
                color: "#e74c3c"
                
                Text {
                    anchors.centerIn: parent
                    text: "Delete"
                    color: "white"
                }
                
                MouseArea {
                    anchors.fill: parent
                    onClicked: taskModel.remove(index)
                }
            }
        }
    }
    
    // Add item
    footer: Button {
        width: parent.width
        height: 50
        text: "+ Add Task"
        
        background: Rectangle {
            color: hovered ? "#ecf0f1" : "transparent"
            border.color: "#bdc3c7"
            border.width: 1
            radius: 8
        }
        
        onClicked: {
            taskModel.append({
                title: "New Task " + (taskModel.count + 1),
                done: false,
                priority: "medium"
            })
        }
    }
    
    // Scroll indicator
    ScrollBar.vertical: ScrollBar {
        policy: ScrollBar.AsNeeded
    }
    
    // Add/remove animations
    add: Transition {
        NumberAnimation { property: "opacity"; from: 0; to: 1; duration: 300 }
        NumberAnimation { property: "x"; from: 50; to: 0; duration: 300; easing.type: Easing.OutBack }
    }
    
    remove: Transition {
        NumberAnimation { property: "opacity"; to: 0; duration: 200 }
        NumberAnimation { property: "x"; to: -parent.width; duration: 200 }
    }
    
    displaced: Transition {
        NumberAnimation { property: "y"; duration: 200; easing.type: Easing.OutCubic }
    }
}
```

---

## ขั้นตอนที่ 464-475: QML C++ Integration (Signals from C++)

```qml
// === main.qml ===
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

ApplicationWindow {
    id: window
    visible: true
    width: 900
    height: 600
    title: "C++ + QML Integration"
    
    // Access C++ backend (set via setContextProperty)
    // backend is a C++ AppBackend exposed to QML
    
    // Listen to C++ signals
    Connections {
        target: backend
        
        function onDataUpdated(data) {
            dataText.text = "Data: " + JSON.stringify(data)
        }
        
        function onError(msg) {
            errorBanner.show(msg)
        }
        
        function onProgressChanged(pct) {
            progressBar.value = pct / 100.0
        }
    }
    
    ColumnLayout {
        anchors.fill: parent
        anchors.margins: 20
        spacing: 16
        
        // Error banner
        Rectangle {
            id: errorBanner
            Layout.fillWidth: true
            height: visible ? 50 : 0
            color: "#e74c3c"
            radius: 6
            visible: false
            clip: true
            
            function show(msg) {
                errorText.text = msg
                visible = true
                hideTimer.start()
            }
            
            Timer {
                id: hideTimer
                interval: 3000
                onTriggered: errorBanner.visible = false
            }
            
            Text {
                id: errorText
                anchors.centerIn: parent
                color: "white"
                font.pixelSize: 14
            }
            
            Behavior on height {
                NumberAnimation { duration: 200 }
            }
        }
        
        // Buttons row
        RowLayout {
            spacing: 12
            
            Button {
                text: "Get Data"
                onClicked: {
                    backend.fetchData()
                }
            }
            
            Button {
                text: "Save"
                onClicked: {
                    var result = backend.saveData(nameField.text, valueField.text)
                    if (result) saveAnim.restart()
                }
            }
            
            Button {
                text: "Process Async"
                onClicked: backend.processAsync()
            }
        }
        
        // Form
        GridLayout {
            columns: 2
            columnSpacing: 12
            rowSpacing: 8
            
            Label { text: "Name:" }
            TextField {
                id: nameField
                Layout.fillWidth: true
                placeholderText: "Enter name"
                
                // Call C++ to validate
                onTextChanged: {
                    validLabel.text = backend.validateName(text) ? "✓" : "✗"
                    validLabel.color = backend.validateName(text) ? "green" : "red"
                }
            }
            
            Label { text: "Value:" }
            TextField {
                id: valueField
                Layout.fillWidth: true
                placeholderText: "Enter value"
            }
            
            Label { text: "Valid:" }
            Label {
                id: validLabel
                text: ""
                font.pixelSize: 18
            }
        }
        
        // Progress
        ProgressBar {
            id: progressBar
            Layout.fillWidth: true
            from: 0; to: 1; value: 0
            
            SequentialAnimation on value {
                id: saveAnim
                running: false
                NumberAnimation { to: 1; duration: 500 }
                NumberAnimation { to: 0; duration: 200 }
            }
        }
        
        // Data display
        ScrollView {
            Layout.fillWidth: true
            Layout.fillHeight: true
            
            TextArea {
                id: dataText
                readOnly: true
                wrapMode: TextArea.Wrap
                text: "Click 'Get Data' to load"
                font.family: "Monospace"
                font.pixelSize: 12
            }
        }
    }
}
```

```cpp
// === AppBackend for QML ===
class QmlBackend : public QObject {
    Q_OBJECT
    Q_PROPERTY(int itemCount READ itemCount NOTIFY itemCountChanged)
    Q_PROPERTY(QString status READ status NOTIFY statusChanged)
    
public:
    explicit QmlBackend(QObject* parent = nullptr) : QObject(parent) {}
    
    int itemCount() const { return m_items.size(); }
    QString status() const { return m_status; }
    
    Q_INVOKABLE void fetchData() {
        // Simulate async fetch
        QTimer::singleShot(500, [this]() {
            QJsonObject data{
                {"timestamp", QDateTime::currentDateTime().toString(Qt::ISODate)},
                {"count", m_items.size()},
                {"items", QJsonArray::fromStringList(m_items)}
            };
            emit dataUpdated(data.toVariantMap());
        });
    }
    
    Q_INVOKABLE bool saveData(const QString& name, const QString& value) {
        if (name.isEmpty() || value.isEmpty()) {
            emit error("Name and value cannot be empty");
            return false;
        }
        
        m_items.append(name + "=" + value);
        emit itemCountChanged(m_items.size());
        return true;
    }
    
    Q_INVOKABLE bool validateName(const QString& name) {
        return name.length() >= 3 && !name.contains(' ');
    }
    
    Q_INVOKABLE void processAsync() {
        QtConcurrent::run([this]() {
            for (int i = 0; i <= 100; i += 5) {
                QThread::msleep(50);
                QMetaObject::invokeMethod(this, [this, i]() {
                    emit progressChanged(i);
                }, Qt::QueuedConnection);
            }
        });
    }
    
signals:
    void dataUpdated(const QVariant& data);
    void error(const QString& msg);
    void progressChanged(int percent);
    void itemCountChanged(int count);
    void statusChanged(const QString& status);
    
private:
    QStringList m_items;
    QString m_status = "Ready";
};

// === main.cpp ===
int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    QQmlApplicationEngine engine;
    
    QmlBackend backend;
    engine.rootContext()->setContextProperty("backend", &backend);
    
    engine.load(QUrl(QStringLiteral("qrc:/main.qml")));
    
    return app.exec();
}
```

---

## สรุป Part 033

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Custom QML Components (CustomButton, Card)
2. ✅ StackView with slide animations
3. ✅ Staggered entrance animations
4. ✅ ListView with custom delegate, swipe, add/remove animations
5. ✅ Advanced C++ ↔ QML integration
6. ✅ Error banner, progress, form validation ใน QML

---

⬅️ [Part 032](part032.md) | ➡️ [Part 034: Qt Quick Controls 2](part034.md)
