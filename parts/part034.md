# Part 034: Qt Quick Controls 2 & Theming

## ขั้นตอนที่ 476-490

---

## ขั้นตอนที่ 476: Qt Quick Controls Overview

```qml
// === QML Theme System ===
// qtquickcontrols2.conf
// [Controls]
// Style=Material
//
// [Material]
// Theme=Dark
// Primary=#3498db
// Accent=#e74c3c
// Foreground=#2c3e50
// Background=#ecf0f1

// Available styles:
// - Basic (lightweight)
// - Fusion (cross-platform)
// - Material (Google Material Design)
// - Universal (Windows)
// - macOS / iOS
// - Imagine (image-based)
```

---

## ขั้นตอนที่ 477: Material Design Components

```qml
import QtQuick
import QtQuick.Controls.Material

ApplicationWindow {
    visible: true
    width: 800; height: 600
    title: "Material Design"
    
    // Material theme
    Material.theme: Material.Light
    Material.primary: Material.Blue
    Material.accent: Material.Orange
    
    // Bottom Navigation Bar (Mobile pattern)
    footer: TabBar {
        id: tabBar
        
        TabButton {
            text: "Home"
            icon.source: "qrc:/icons/home.png"
        }
        TabButton {
            text: "Search"
            icon.source: "qrc:/icons/search.png"
        }
        TabButton {
            text: "Profile"
            icon.source: "qrc:/icons/person.png"
        }
    }
    
    // Content
    StackLayout {
        anchors.fill: parent
        currentIndex: tabBar.currentIndex
        
        // Home page
        Page {
            header: ToolBar {
                Label {
                    anchors.centerIn: parent
                    text: "Home"
                    font.pixelSize: 20
                    color: "white"
                }
                
                ToolButton {
                    anchors.right: parent.right
                    icon.source: "qrc:/icons/more.png"
                    onClicked: contextMenu.open()
                    
                    Menu {
                        id: contextMenu
                        
                        MenuItem { text: "Settings" }
                        MenuItem { text: "About" }
                        MenuSeparator {}
                        MenuItem { text: "Logout" }
                    }
                }
            }
            
            ScrollView {
                anchors.fill: parent
                
                Column {
                    width: parent.width
                    padding: 16
                    spacing: 16
                    
                    // Search field
                    TextField {
                        width: parent.width - 32
                        placeholderText: "Search..."
                        Material.containerStyle: Material.Outlined
                        leftPadding: 40
                        
                        Image {
                            anchors.left: parent.left
                            anchors.leftMargin: 8
                            anchors.verticalCenter: parent.verticalCenter
                            source: "qrc:/icons/search.png"
                            width: 24; height: 24
                        }
                    }
                    
                    // Cards
                    Repeater {
                        model: 5
                        
                        Frame {
                            width: parent.width - 32
                            
                            Column {
                                spacing: 8
                                
                                Label {
                                    text: "Card Title " + (index + 1)
                                    font.pixelSize: 18
                                    font.weight: Font.Medium
                                }
                                
                                Label {
                                    text: "This is the card content for item " + (index + 1)
                                    wrapMode: Text.Wrap
                                    width: parent.width
                                    color: Material.hintTextColor
                                }
                                
                                Row {
                                    spacing: 8
                                    
                                    Button {
                                        text: "Action 1"
                                        flat: true
                                        Material.foreground: Material.accent
                                    }
                                    
                                    Button {
                                        text: "Action 2"
                                        flat: true
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        
        // Search page
        Page {
            Label {
                anchors.centerIn: parent
                text: "Search Page"
                font.pixelSize: 24
            }
        }
        
        // Profile page
        Page {
            Label {
                anchors.centerIn: parent
                text: "Profile Page"
                font.pixelSize: 24
            }
        }
    }
    
    // FAB (Floating Action Button)
    RoundButton {
        anchors.right: parent.right
        anchors.bottom: tabBar.top
        anchors.margins: 16
        text: "+"
        font.pixelSize: 24
        Material.background: Material.accent
        
        onClicked: newItemDialog.open()
    }
    
    // Dialog
    Dialog {
        id: newItemDialog
        title: "New Item"
        anchors.centerIn: parent
        width: 360
        
        standardButtons: Dialog.Ok | Dialog.Cancel
        
        Column {
            spacing: 16
            width: parent.width
            
            TextField {
                id: titleField
                width: parent.width
                placeholderText: "Title"
                Material.containerStyle: Material.Outlined
            }
            
            TextArea {
                id: descField
                width: parent.width
                placeholderText: "Description"
                Material.containerStyle: Material.Outlined
                height: 100
            }
        }
        
        onAccepted: {
            console.log("Create:", titleField.text)
            titleField.text = ""
            descField.text = ""
        }
    }
}
```

---

## ขั้นตอนที่ 478: Custom Theme Engine

```qml
// === Theme.qml (singleton) ===
pragma Singleton
import QtQuick

QtObject {
    id: root
    
    // Current theme
    property string name: "light"
    
    // Colors
    readonly property color background:    name === "dark" ? "#1a1a2e" : "#f0f2f5"
    readonly property color surface:       name === "dark" ? "#16213e" : "#ffffff"
    readonly property color surfaceVariant: name === "dark" ? "#0f3460" : "#ecf0f1"
    readonly property color primary:       "#3498db"
    readonly property color primaryDark:   "#2980b9"
    readonly property color accent:        "#e74c3c"
    readonly property color onPrimary:     "#ffffff"
    readonly property color onSurface:     name === "dark" ? "#e0e0e0" : "#2c3e50"
    readonly property color secondaryText: name === "dark" ? "#9e9e9e" : "#7f8c8d"
    readonly property color divider:       name === "dark" ? "#333355" : "#ecf0f1"
    readonly property color error:         "#e74c3c"
    readonly property color success:       "#27ae60"
    readonly property color warning:       "#f39c12"
    
    // Typography
    property int titleSize: 24
    property int headingSize: 18
    property int bodySize: 14
    property int captionSize: 12
    property string fontFamily: "Segoe UI"
    
    // Spacing & Sizing
    property int spacing: 8
    property int radius: 8
    property int cardRadius: 12
    
    // Shadows
    readonly property color shadow: name === "dark" ? "#00000060" : "#00000020"
    
    // Animation
    property int animationSpeed: 250
    property int animationFast: 150
    
    function toggle() {
        name = name === "dark" ? "light" : "dark"
    }
    
    // Signal for components to react
    signal themeChanged()
    onNameChanged: themeChanged()
}

// === Register as singleton in main.cpp ===
// qmlRegisterSingletonType(QUrl("qrc:/Theme.qml"), "App", 1, 0, "Theme");

// === Using in components ===
// import App 1.0
// Rectangle { color: Theme.surface }
```

---

## ขั้นตอนที่ 479: Responsive Layout

```qml
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

ApplicationWindow {
    id: root
    visible: true
    width: 1024; height: 768
    title: "Responsive Layout"
    
    // Breakpoints
    property bool isNarrow: width < 600
    property bool isMedium: width >= 600 && width < 1024
    property bool isWide: width >= 1024
    
    // Sidebar + Content layout
    RowLayout {
        anchors.fill: parent
        spacing: 0
        
        // Sidebar - hidden on narrow screens
        Rectangle {
            id: sidebar
            width: isWide ? 250 : isMedium ? 60 : 0
            Layout.fillHeight: true
            color: "#2c3e50"
            clip: true
            
            Behavior on width {
                NumberAnimation { duration: 250; easing.type: Easing.OutCubic }
            }
            
            Column {
                width: parent.width
                padding: 8
                spacing: 4
                
                // Logo
                Rectangle {
                    width: parent.width - 16
                    height: 60
                    color: "transparent"
                    
                    Row {
                        anchors.verticalCenter: parent.verticalCenter
                        anchors.left: parent.left
                        anchors.leftMargin: 8
                        spacing: 12
                        
                        Rectangle {
                            width: 36; height: 36
                            radius: 8
                            color: "#3498db"
                            
                            Text {
                                anchors.centerIn: parent
                                text: "A"
                                color: "white"
                                font.bold: true
                                font.pixelSize: 18
                            }
                        }
                        
                        Text {
                            visible: isWide
                            text: "MyApp"
                            color: "white"
                            font.pixelSize: 18
                            font.bold: true
                            anchors.verticalCenter: parent.verticalCenter
                        }
                    }
                }
                
                // Nav items
                Repeater {
                    model: [
                        {icon: "⌂", label: "Dashboard"},
                        {icon: "📊", label: "Analytics"},
                        {icon: "👤", label: "Users"},
                        {icon: "⚙", label: "Settings"}
                    ]
                    
                    Rectangle {
                        width: parent.width - 16
                        height: 46
                        radius: 6
                        color: index === 0 ? "#3498db" : "transparent"
                        
                        Row {
                            anchors.verticalCenter: parent.verticalCenter
                            anchors.left: parent.left
                            anchors.leftMargin: 12
                            spacing: 12
                            
                            Text {
                                text: modelData.icon
                                font.pixelSize: 18
                                color: "white"
                            }
                            
                            Text {
                                visible: isWide
                                text: modelData.label
                                color: "white"
                                font.pixelSize: 14
                                anchors.verticalCenter: parent.verticalCenter
                            }
                        }
                        
                        MouseArea {
                            anchors.fill: parent
                            hoverEnabled: true
                            onEntered: if (index !== 0) parent.color = "#34495e"
                            onExited: if (index !== 0) parent.color = "transparent"
                            onClicked: console.log("Nav:", modelData.label)
                        }
                    }
                }
            }
        }
        
        // Main content
        Rectangle {
            Layout.fillWidth: true
            Layout.fillHeight: true
            color: "#f0f2f5"
            
            // Responsive grid
            GridLayout {
                anchors.fill: parent
                anchors.margins: 16
                columns: isWide ? 3 : isMedium ? 2 : 1
                rowSpacing: 16
                columnSpacing: 16
                
                Repeater {
                    model: 6
                    
                    Rectangle {
                        Layout.fillWidth: true
                        height: 150
                        radius: 12
                        color: "white"
                        
                        ColumnLayout {
                            anchors.fill: parent
                            anchors.margins: 16
                            spacing: 8
                            
                            Text {
                                text: "Widget " + (index + 1)
                                font.pixelSize: 16
                                font.bold: true
                                color: "#2c3e50"
                            }
                            
                            Text {
                                text: "Content goes here"
                                color: "#7f8c8d"
                                font.pixelSize: 13
                            }
                            
                            Item { Layout.fillHeight: true }
                            
                            Row {
                                spacing: 8
                                Text {
                                    text: "●"
                                    color: "#27ae60"
                                    font.pixelSize: 10
                                }
                                Text {
                                    text: "Active"
                                    color: "#27ae60"
                                    font.pixelSize: 12
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    
    // Bottom navigation for mobile
    footer: TabBar {
        visible: isNarrow
        height: visible ? 60 : 0
        
        TabButton { text: "⌂\nHome" }
        TabButton { text: "📊\nStats" }
        TabButton { text: "👤\nProfile" }
        TabButton { text: "⚙\nSettings" }
    }
}
```

---

## ขั้นตอนที่ 480-490: Dark Mode Toggle

```qml
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

// Complete dark/light mode application
ApplicationWindow {
    id: window
    visible: true
    width: 800; height: 600
    title: "Dark Mode Demo"
    
    // Theme state
    property bool darkMode: false
    
    // Animated theme colors
    property color bgColor: darkMode ? "#1a1a2e" : "#f0f2f5"
    property color cardColor: darkMode ? "#16213e" : "#ffffff"
    property color textColor: darkMode ? "#e0e0e0" : "#2c3e50"
    property color mutedColor: darkMode ? "#9e9e9e" : "#7f8c8d"
    property color accentColor: "#3498db"
    
    color: bgColor
    
    Behavior on bgColor { ColorAnimation { duration: 300 } }
    Behavior on cardColor { ColorAnimation { duration: 300 } }
    Behavior on textColor { ColorAnimation { duration: 300 } }
    
    ColumnLayout {
        anchors.fill: parent
        anchors.margins: 20
        spacing: 16
        
        // Header with toggle
        RowLayout {
            Layout.fillWidth: true
            
            Label {
                text: "My Application"
                font.pixelSize: 24
                font.bold: true
                color: window.textColor
            }
            
            Item { Layout.fillWidth: true }
            
            // Toggle switch
            RowLayout {
                spacing: 8
                
                Label {
                    text: "☀️"
                    font.pixelSize: 16
                    opacity: darkMode ? 0.5 : 1
                }
                
                Switch {
                    id: themeSwitch
                    checked: darkMode
                    onCheckedChanged: darkMode = checked
                    
                    indicator: Rectangle {
                        width: 52; height: 28
                        radius: 14
                        color: themeSwitch.checked ? "#3498db" : "#bdc3c7"
                        
                        Rectangle {
                            x: themeSwitch.checked ? parent.width - 26 : 2
                            y: 2
                            width: 24; height: 24
                            radius: 12
                            color: "white"
                            
                            Behavior on x {
                                NumberAnimation { duration: 200; easing.type: Easing.OutCubic }
                            }
                        }
                        
                        Behavior on color {
                            ColorAnimation { duration: 200 }
                        }
                    }
                }
                
                Label {
                    text: "🌙"
                    font.pixelSize: 16
                    opacity: darkMode ? 1 : 0.5
                }
            }
        }
        
        // Stats cards
        RowLayout {
            Layout.fillWidth: true
            spacing: 16
            
            Repeater {
                model: [
                    {title: "Users", value: "1,234", icon: "👤", trend: "+5%"},
                    {title: "Revenue", value: "฿89,000", icon: "💰", trend: "+12%"},
                    {title: "Orders", value: "456", icon: "📦", trend: "-2%"}
                ]
                
                Rectangle {
                    Layout.fillWidth: true
                    height: 100
                    radius: 12
                    color: window.cardColor
                    
                    Behavior on color { ColorAnimation { duration: 300 } }
                    
                    ColumnLayout {
                        anchors.fill: parent
                        anchors.margins: 16
                        spacing: 4
                        
                        RowLayout {
                            Label { text: modelData.icon; font.pixelSize: 20 }
                            Item { Layout.fillWidth: true }
                            Label {
                                text: modelData.trend
                                color: modelData.trend.startsWith("+") ? "#27ae60" : "#e74c3c"
                                font.pixelSize: 12
                            }
                        }
                        
                        Label {
                            text: modelData.value
                            font.pixelSize: 24
                            font.bold: true
                            color: window.textColor
                        }
                        
                        Label {
                            text: modelData.title
                            color: window.mutedColor
                            font.pixelSize: 13
                        }
                    }
                }
            }
        }
        
        // Content area
        Rectangle {
            Layout.fillWidth: true
            Layout.fillHeight: true
            radius: 12
            color: window.cardColor
            
            Behavior on color { ColorAnimation { duration: 300 } }
            
            ListView {
                anchors.fill: parent
                anchors.margins: 16
                model: 8
                spacing: 8
                clip: true
                
                delegate: Rectangle {
                    width: ListView.view.width
                    height: 50
                    radius: 8
                    color: darkMode ? "#0f3460" : "#f8f9fa"
                    
                    Behavior on color { ColorAnimation { duration: 300 } }
                    
                    RowLayout {
                        anchors.fill: parent
                        anchors.margins: 12
                        spacing: 12
                        
                        Rectangle {
                            width: 36; height: 36
                            radius: 18
                            color: ["#3498db","#27ae60","#e74c3c","#f39c12"][index % 4]
                            
                            Label {
                                anchors.centerIn: parent
                                text: "U" + (index + 1)
                                color: "white"
                                font.bold: true
                            }
                        }
                        
                        Column {
                            Layout.fillWidth: true
                            spacing: 2
                            
                            Label {
                                text: "User " + (index + 1)
                                color: window.textColor
                                font.pixelSize: 14
                                font.bold: true
                            }
                            Label {
                                text: "user" + (index + 1) + "@example.com"
                                color: window.mutedColor
                                font.pixelSize: 12
                            }
                        }
                        
                        Label {
                            text: index % 3 === 0 ? "Admin" : index % 3 === 1 ? "Manager" : "User"
                            color: ["#e74c3c","#f39c12","#3498db"][index % 3]
                            font.pixelSize: 12
                        }
                    }
                }
            }
        }
    }
}
```

---

## สรุป Part 034

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Material Design Controls (ToolBar, TabBar, Dialog, FAB)
2. ✅ Custom Theme singleton
3. ✅ Responsive layout (breakpoints, sidebar, grid)
4. ✅ Dark mode toggle ด้วย animated color transitions
5. ✅ Stats cards และ list view

---

⬅️ [Part 033](part033.md) | ➡️ [Part 035: Qt 3D (Qt Quick 3D)](part035.md)
