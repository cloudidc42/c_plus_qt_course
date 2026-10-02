# Part 057: World-Class — Advanced QML & Fluid Animations

## ขั้นตอนที่ 821-835

---

## ขั้นตอนที่ 821: QML Integration Architecture

```
Qt C++ + QML Integration:
  
  C++ Side:
    - QQmlApplicationEngine: โหลด .qml entry point
    - QQmlContext: expose C++ objects ไปยัง QML
    - Q_PROPERTY: reactive properties
    - Q_INVOKABLE: callable methods from QML
    - QAbstractListModel: data for QML ListView
  
  QML Side:
    - import QtQuick 2.15
    - import QtQuick.Controls 2.15
    - import QtQuick.Layouts 1.15
    - ApplicationWindow + StackView + Drawer
  
  Communication:
    C++ → QML: property notify signals, emit
    QML → C++: Q_INVOKABLE, Connections{}
```

---

## ขั้นตอนที่ 822: C++ Backend for QML

```cpp
// backend/appbackend.h
#pragma once
#include <QObject>
#include <QAbstractListModel>
#include <QColor>

// Contact model for QML
class ContactModel : public QAbstractListModel {
    Q_OBJECT
    Q_PROPERTY(int count READ rowCount NOTIFY countChanged)
    
public:
    enum Roles {
        IdRole = Qt::UserRole,
        NameRole,
        EmailRole,
        PhoneRole,
        AvatarColorRole,
        IsFavoriteRole
    };
    Q_ENUM(Roles)
    
    struct Contact {
        int id;
        QString name;
        QString email;
        QString phone;
        bool isFavorite = false;
        
        QColor avatarColor() const {
            // Deterministic color from name hash
            uint hash = qHash(name);
            static const QList<QColor> colors = {
                "#3498DB", "#2ECC71", "#9B59B6", "#E67E22",
                "#E74C3C", "#1ABC9C", "#34495E", "#F39C12"
            };
            return colors[hash % colors.size()];
        }
    };
    
    explicit ContactModel(QObject* parent = nullptr) : QAbstractListModel(parent) {
        // Seed data
        m_contacts = {
            {1, "สมชาย ใจดี",    "somchai@email.com",  "081-234-5678", true},
            {2, "สมหญิง รักเรียน", "somying@email.com",  "089-876-5432", false},
            {3, "วิทยา เก่งมาก",  "witaya@email.com",   "062-111-2222", true},
            {4, "พิมพ์ สวยงาม",  "pim@email.com",      "093-333-4444", false},
            {5, "นภา ฟ้าใส",     "napa@email.com",     "087-555-6666", false}
        };
    }
    
    int rowCount(const QModelIndex& = {}) const override {
        return m_contacts.size();
    }
    
    QVariant data(const QModelIndex& index, int role = Qt::DisplayRole) const override {
        if (!index.isValid() || index.row() >= m_contacts.size()) return {};
        
        const auto& c = m_contacts[index.row()];
        switch (role) {
            case IdRole:         return c.id;
            case NameRole:       return c.name;
            case EmailRole:      return c.email;
            case PhoneRole:      return c.phone;
            case AvatarColorRole:return c.avatarColor().name();
            case IsFavoriteRole: return c.isFavorite;
        }
        return {};
    }
    
    QHash<int, QByteArray> roleNames() const override {
        return {
            {IdRole,          "contactId"},
            {NameRole,        "name"},
            {EmailRole,       "email"},
            {PhoneRole,       "phone"},
            {AvatarColorRole, "avatarColor"},
            {IsFavoriteRole,  "isFavorite"}
        };
    }
    
    Q_INVOKABLE void toggleFavorite(int id) {
        for (int i = 0; i < m_contacts.size(); ++i) {
            if (m_contacts[i].id == id) {
                m_contacts[i].isFavorite = !m_contacts[i].isFavorite;
                auto idx = index(i);
                emit dataChanged(idx, idx, {IsFavoriteRole});
                break;
            }
        }
    }
    
    Q_INVOKABLE void addContact(const QString& name, const QString& email, const QString& phone) {
        int newId = m_contacts.isEmpty() ? 1 : m_contacts.last().id + 1;
        
        beginInsertRows({}, m_contacts.size(), m_contacts.size());
        m_contacts.append({newId, name, email, phone, false});
        endInsertRows();
        
        emit countChanged();
    }
    
    Q_INVOKABLE void removeContact(int id) {
        for (int i = 0; i < m_contacts.size(); ++i) {
            if (m_contacts[i].id == id) {
                beginRemoveRows({}, i, i);
                m_contacts.removeAt(i);
                endRemoveRows();
                emit countChanged();
                return;
            }
        }
    }
    
signals:
    void countChanged();
    
private:
    QList<Contact> m_contacts;
};

// App Backend
class AppBackend : public QObject {
    Q_OBJECT
    Q_PROPERTY(QString userName READ userName WRITE setUserName NOTIFY userNameChanged)
    Q_PROPERTY(bool darkMode READ darkMode WRITE setDarkMode NOTIFY darkModeChanged)
    Q_PROPERTY(int unreadCount READ unreadCount NOTIFY unreadCountChanged)
    Q_PROPERTY(ContactModel* contacts READ contacts CONSTANT)
    
public:
    explicit AppBackend(QObject* parent = nullptr) : QObject(parent) {
        m_contacts = new ContactModel(this);
    }
    
    QString userName() const { return m_userName; }
    bool darkMode() const { return m_darkMode; }
    int unreadCount() const { return m_unreadCount; }
    ContactModel* contacts() const { return m_contacts; }
    
    void setUserName(const QString& name) {
        if (m_userName != name) { m_userName = name; emit userNameChanged(); }
    }
    
    void setDarkMode(bool dark) {
        if (m_darkMode != dark) { m_darkMode = dark; emit darkModeChanged(); }
    }
    
    Q_INVOKABLE void showNotification(const QString& title, const QString& message) {
        // Use QSystemTrayIcon for desktop notification
        qDebug() << "Notification:" << title << message;
        emit notificationRequested(title, message);
    }
    
    Q_INVOKABLE QString formatDate(const QDate& date) {
        return date.toString("dd MMMM yyyy");
    }
    
    Q_INVOKABLE QVariantList recentActivity() {
        return {
            QVariantMap{{"text", "เพิ่มลูกค้าใหม่: บริษัท ABC"}, {"time", "10 นาทีที่แล้ว"}, {"icon", "👥"}},
            QVariantMap{{"text", "คำสั่งซื้อ SO202401001 ยืนยันแล้ว"}, {"time", "1 ชั่วโมงที่แล้ว"}, {"icon", "📋"}},
            QVariantMap{{"text", "รับสินค้าเข้าคลัง: 50 ชิ้น"}, {"time", "3 ชั่วโมงที่แล้ว"}, {"icon", "📦"}}
        };
    }
    
signals:
    void userNameChanged();
    void darkModeChanged();
    void unreadCountChanged();
    void notificationRequested(const QString& title, const QString& message);
    
private:
    QString m_userName = "ผู้ดูแลระบบ";
    bool m_darkMode = false;
    int m_unreadCount = 3;
    ContactModel* m_contacts;
};
```

---

## ขั้นตอนที่ 823: QML Application Entry

```qml
// qml/main.qml
import QtQuick 2.15
import QtQuick.Controls 2.15
import QtQuick.Controls.Material 2.15
import QtQuick.Layouts 1.15

ApplicationWindow {
    id: root
    width: 1280
    height: 720
    visible: true
    title: "ERP Pro QML"
    
    // Apply Material theme
    Material.theme: backend.darkMode ? Material.Dark : Material.Light
    Material.accent: Material.Blue
    Material.primary: "#2C3E50"
    
    // Drawer (side navigation)
    Drawer {
        id: drawer
        width: 280
        height: parent.height
        
        background: Rectangle {
            color: Material.theme === Material.Dark ? "#1E1E2E" : "#2C3E50"
        }
        
        ColumnLayout {
            anchors.fill: parent
            spacing: 0
            
            // Header
            Rectangle {
                Layout.fillWidth: true
                height: 120
                color: "#16202C"
                
                Column {
                    anchors.centerIn: parent
                    spacing: 8
                    
                    Rectangle {
                        width: 56; height: 56
                        radius: 28
                        color: Material.accentColor
                        anchors.horizontalCenter: parent.horizontalCenter
                        
                        Text {
                            anchors.centerIn: parent
                            text: backend.userName.substring(0, 1)
                            font.pixelSize: 24
                            font.bold: true
                            color: "white"
                        }
                    }
                    
                    Text {
                        text: backend.userName
                        color: "white"
                        font.pixelSize: 14
                        anchors.horizontalCenter: parent.horizontalCenter
                    }
                }
            }
            
            // Nav items
            ListView {
                Layout.fillWidth: true
                Layout.fillHeight: true
                model: navModel
                
                delegate: ItemDelegate {
                    width: ListView.view.width
                    height: 52
                    
                    contentItem: RowLayout {
                        spacing: 16
                        
                        Text {
                            text: model.icon
                            font.pixelSize: 20
                            Layout.leftMargin: 16
                        }
                        
                        Text {
                            text: model.label
                            color: stackView.currentItem?.pageName === model.id
                                   ? Material.accentColor : "#BDC3C7"
                            font.pixelSize: 14
                            font.bold: stackView.currentItem?.pageName === model.id
                            Layout.fillWidth: true
                        }
                        
                        // Badge
                        Rectangle {
                            visible: model.badge > 0
                            width: 22; height: 22
                            radius: 11
                            color: "#E74C3C"
                            Layout.rightMargin: 12
                            
                            Text {
                                anchors.centerIn: parent
                                text: model.badge
                                color: "white"
                                font.pixelSize: 11
                                font.bold: true
                            }
                        }
                    }
                    
                    highlighted: stackView.currentItem?.pageName === model.id
                    Material.foreground: highlighted ? Material.accentColor : "#BDC3C7"
                    
                    onClicked: {
                        drawer.close()
                        stackView.navigate(model.id)
                    }
                }
            }
            
            // Dark mode toggle
            ItemDelegate {
                Layout.fillWidth: true
                height: 52
                
                contentItem: RowLayout {
                    spacing: 16
                    
                    Text {
                        text: "🌙"
                        font.pixelSize: 20
                        Layout.leftMargin: 16
                    }
                    
                    Text {
                        text: "โหมดมืด"
                        color: "#BDC3C7"
                        font.pixelSize: 14
                        Layout.fillWidth: true
                    }
                    
                    Switch {
                        checked: backend.darkMode
                        onCheckedChanged: backend.darkMode = checked
                        Layout.rightMargin: 8
                    }
                }
            }
        }
    }
    
    // Main layout
    ColumnLayout {
        anchors.fill: parent
        spacing: 0
        
        // Top App Bar
        ToolBar {
            Layout.fillWidth: true
            Material.background: "#2C3E50"
            height: 56
            
            RowLayout {
                anchors.fill: parent
                anchors.leftMargin: 8
                anchors.rightMargin: 8
                
                ToolButton {
                    icon.name: "drawer"
                    text: "☰"
                    font.pixelSize: 20
                    onClicked: drawer.open()
                    Material.foreground: "white"
                }
                
                Text {
                    text: stackView.currentItem?.title ?? "ERP Pro"
                    color: "white"
                    font.pixelSize: 18
                    font.bold: true
                    Layout.fillWidth: true
                }
                
                // Notification badge
                ToolButton {
                    text: "🔔"
                    font.pixelSize: 20
                    Material.foreground: "white"
                    
                    Rectangle {
                        visible: backend.unreadCount > 0
                        width: 18; height: 18; radius: 9
                        color: "#E74C3C"
                        anchors { top: parent.top; right: parent.right; margins: 6 }
                        
                        Text {
                            anchors.centerIn: parent
                            text: backend.unreadCount
                            color: "white"
                            font.pixelSize: 10
                            font.bold: true
                        }
                    }
                }
            }
        }
        
        // Page Stack
        StackView {
            id: stackView
            Layout.fillWidth: true
            Layout.fillHeight: true
            
            initialItem: dashboardPage
            
            function navigate(pageId) {
                var component;
                switch(pageId) {
                    case "dashboard": component = dashboardPage; break;
                    case "contacts":  component = contactsPage; break;
                    default: return;
                }
                
                if (currentItem?.pageName !== pageId) {
                    replace(null, component, {},
                            StackView.Transition)
                }
            }
            
            // Slide transition
            replaceEnter: Transition {
                XAnimator { from: stackView.width; to: 0; duration: 250; easing.type: Easing.OutCubic }
            }
            replaceExit: Transition {
                XAnimator { from: 0; to: -stackView.width / 3; duration: 250 }
            }
        }
    }
    
    // Pages (components)
    Component {
        id: dashboardPage
        DashboardPage { pageName: "dashboard" }
    }
    
    Component {
        id: contactsPage
        ContactsPage { pageName: "contacts" }
    }
    
    // Nav model
    ListModel {
        id: navModel
        ListElement { id: "dashboard"; icon: "📊"; label: "แดชบอร์ด"; badge: 0 }
        ListElement { id: "contacts";  icon: "👥"; label: "ผู้ติดต่อ";   badge: 0 }
        ListElement { id: "orders";    icon: "📋"; label: "คำสั่งซื้อ"; badge: 2 }
        ListElement { id: "products";  icon: "📦"; label: "สินค้า";     badge: 0 }
        ListElement { id: "reports";   icon: "📈"; label: "รายงาน";    badge: 0 }
    }
}
```

---

## ขั้นตอนที่ 824: QML Contacts Page with Animations

```qml
// qml/ContactsPage.qml
import QtQuick 2.15
import QtQuick.Controls 2.15
import QtQuick.Controls.Material 2.15
import QtQuick.Layouts 1.15

Page {
    id: root
    property string pageName: "contacts"
    title: "ผู้ติดต่อ"
    
    header: ToolBar {
        RowLayout {
            anchors.fill: parent
            
            TextField {
                id: searchField
                Layout.fillWidth: true
                Layout.margins: 8
                placeholderText: "ค้นหาผู้ติดต่อ..."
                Material.accent: Material.Blue
                
                leftPadding: 40
                
                Image {
                    anchors { left: parent.left; verticalCenter: parent.verticalCenter; leftMargin: 12 }
                    source: "qrc:/icons/search.svg"
                    width: 18; height: 18
                    opacity: 0.5
                }
            }
            
            RoundButton {
                text: "+"
                font.pixelSize: 22
                Material.background: Material.accentColor
                Material.foreground: "white"
                Layout.rightMargin: 8
                onClicked: addDialog.open()
            }
        }
    }
    
    // Contacts list
    ListView {
        id: contactsList
        anchors.fill: parent
        model: backend.contacts
        clip: true
        spacing: 1
        
        ScrollBar.vertical: ScrollBar {}
        
        // Filter by search
        property string filter: searchField.text.toLowerCase()
        
        delegate: ContactDelegate {
            width: contactsList.width
            visible: name.toLowerCase().includes(contactsList.filter)
                     || email.toLowerCase().includes(contactsList.filter)
            height: visible ? 72 : 0
            
            Behavior on height {
                NumberAnimation { duration: 200; easing.type: Easing.OutCubic }
            }
        }
        
        // Empty state
        Text {
            anchors.centerIn: parent
            visible: contactsList.count === 0
            text: "ไม่มีผู้ติดต่อ\nกด + เพื่อเพิ่ม"
            horizontalAlignment: Text.AlignHCenter
            color: Material.hintTextColor
            font.pixelSize: 16
        }
        
        // Add animation when item added
        add: Transition {
            NumberAnimation { property: "opacity"; from: 0; to: 1; duration: 300 }
            NumberAnimation { property: "height"; from: 0; to: 72; duration: 300; easing.type: Easing.OutBack }
        }
        
        remove: Transition {
            NumberAnimation { property: "opacity"; from: 1; to: 0; duration: 200 }
            NumberAnimation { property: "height"; from: 72; to: 0; duration: 200 }
        }
        
        displaced: Transition {
            NumberAnimation { properties: "x,y"; duration: 200; easing.type: Easing.OutCubic }
        }
    }
    
    // Add contact dialog
    Dialog {
        id: addDialog
        parent: Overlay.overlay
        anchors.centerIn: parent
        title: "เพิ่มผู้ติดต่อ"
        standardButtons: Dialog.Ok | Dialog.Cancel
        modal: true
        width: 400
        
        enter: Transition {
            NumberAnimation { property: "opacity"; from: 0; to: 1; duration: 200 }
            NumberAnimation { property: "scale"; from: 0.8; to: 1; duration: 200; easing.type: Easing.OutBack }
        }
        
        ColumnLayout {
            anchors.fill: parent
            spacing: 12
            
            TextField { id: nameField;  Layout.fillWidth: true; placeholderText: "ชื่อ-นามสกุล *" }
            TextField { id: emailField; Layout.fillWidth: true; placeholderText: "อีเมล" }
            TextField { id: phoneField; Layout.fillWidth: true; placeholderText: "โทรศัพท์" }
        }
        
        onAccepted: {
            if (nameField.text.trim().length > 0) {
                backend.contacts.addContact(
                    nameField.text.trim(),
                    emailField.text.trim(),
                    phoneField.text.trim()
                )
                nameField.text = ""
                emailField.text = ""
                phoneField.text = ""
            }
        }
    }
}

// ContactDelegate.qml
Component {
    Rectangle {
        required property string name
        required property string email
        required property string phone
        required property string avatarColor
        required property bool isFavorite
        required property int contactId
        
        color: mouseArea.containsMouse ? Material.dividerColor : Material.backgroundColor
        
        RowLayout {
            anchors { fill: parent; margins: 12 }
            spacing: 16
            
            // Avatar
            Rectangle {
                width: 46; height: 46; radius: 23
                color: avatarColor
                
                Text {
                    anchors.centerIn: parent
                    text: name.substring(0, 1)
                    font.pixelSize: 20
                    font.bold: true
                    color: "white"
                }
            }
            
            // Info
            Column {
                Layout.fillWidth: true
                spacing: 4
                
                Text {
                    text: name
                    font.pixelSize: 15
                    font.bold: true
                    color: Material.primaryTextColor
                }
                
                Text {
                    text: email || phone || "ไม่มีข้อมูล"
                    font.pixelSize: 13
                    color: Material.hintTextColor
                    elide: Text.ElideRight
                    width: parent.width
                }
            }
            
            // Favorite button
            IconButton {
                text: isFavorite ? "⭐" : "☆"
                font.pixelSize: 20
                opacity: isFavorite ? 1.0 : 0.4
                
                Behavior on opacity {
                    NumberAnimation { duration: 150 }
                }
                
                onClicked: backend.contacts.toggleFavorite(contactId)
            }
        }
        
        MouseArea {
            id: mouseArea
            anchors.fill: parent
            hoverEnabled: true
            onClicked: contactDetail.show(contactId, name, email, phone)
        }
    }
}
```

---

## สรุป Part 057

ใน Part นี้คุณได้เรียนรู้:

1. ✅ ContactModel ด้วย QAbstractListModel + Q_INVOKABLE + roleNames
2. ✅ AppBackend ด้วย Q_PROPERTY + NOTIFY signals
3. ✅ QML ApplicationWindow + Drawer + Material theme
4. ✅ StackView navigation ด้วย slide transitions
5. ✅ ListView ด้วย add/remove/displaced animations + filter
6. ✅ Dialog ด้วย scale+opacity enter transition

---

⬅️ [Part 056](part056.md) | ➡️ [Part 058: World-Class — Security & Authentication](part058.md)
