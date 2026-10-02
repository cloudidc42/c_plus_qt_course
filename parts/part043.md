# Part 043: Real-World Project — Chat Application

## ขั้นตอนที่ 611-625

---

## ขั้นตอนที่ 611: Chat App Architecture

```
ChatApp/
├── ChatServer/           # QTcpServer-based server
│   ├── ChatServer.h/.cpp
│   ├── ClientSession.h/.cpp
│   └── main.cpp
└── ChatClient/           # Qt GUI client
    ├── ChatClient.h/.cpp
    ├── ChatWindow.h/.cpp
    ├── MessageBubble.h/.cpp
    ├── ContactList.h/.cpp
    └── main.cpp

Protocol (JSON over TCP):
  {"type": "connect", "username": "สมชาย"}
  {"type": "message", "to": "all", "text": "Hello"}
  {"type": "private", "to": "user123", "text": "Hi"}
  {"type": "users_list", "users": ["user1", "user2"]}
```

---

## ขั้นตอนที่ 612: Chat Protocol

```cpp
namespace Protocol {
    enum class MessageType {
        Connect, Disconnect,
        Message, PrivateMessage,
        UsersList, UserJoined, UserLeft,
        Error, Ping, Pong
    };
    
    struct ChatMessage {
        MessageType type;
        QString from;
        QString to;        // "all" or username
        QString text;
        QDateTime timestamp;
        QString id;        // UUID
        
        QJsonObject toJson() const {
            return {
                {"type",      static_cast<int>(type)},
                {"from",      from},
                {"to",        to},
                {"text",      text},
                {"ts",        timestamp.toMSecsSinceEpoch()},
                {"id",        id},
            };
        }
        
        static ChatMessage fromJson(const QJsonObject& obj) {
            ChatMessage m;
            m.type      = static_cast<MessageType>(obj["type"].toInt());
            m.from      = obj["from"].toString();
            m.to        = obj["to"].toString();
            m.text      = obj["text"].toString();
            m.timestamp = QDateTime::fromMSecsSinceEpoch(obj["ts"].toVariant().toLongLong());
            m.id        = obj["id"].toString();
            return m;
        }
        
        static ChatMessage make(MessageType type, const QString& from,
                                const QString& to, const QString& text) {
            ChatMessage m;
            m.type      = type;
            m.from      = from;
            m.to        = to;
            m.text      = text;
            m.timestamp = QDateTime::currentDateTime();
            m.id        = QUuid::createUuid().toString(QUuid::WithoutBraces);
            return m;
        }
    };
    
    // Serialize message to frame (length-prefixed)
    QByteArray encode(const ChatMessage& msg) {
        QByteArray json = QJsonDocument(msg.toJson()).toJson(QJsonDocument::Compact);
        QByteArray frame;
        QDataStream ds(&frame, QIODevice::WriteOnly);
        ds.setByteOrder(QDataStream::BigEndian);
        ds << static_cast<quint32>(json.size());
        frame.append(json);
        return frame;
    }
}
```

---

## ขั้นตอนที่ 613: Chat Server

```cpp
#include <QTcpServer>
#include <QTcpSocket>

class ClientSession : public QObject {
    Q_OBJECT
    
public:
    explicit ClientSession(QTcpSocket* socket, QObject* parent = nullptr)
        : QObject(parent), m_socket(socket) {
        
        connect(socket, &QTcpSocket::readyRead, this, &ClientSession::onReadyRead);
        connect(socket, &QTcpSocket::disconnected, this, &ClientSession::disconnected);
    }
    
    void send(const Protocol::ChatMessage& msg) {
        if (m_socket->state() == QAbstractSocket::ConnectedState) {
            m_socket->write(Protocol::encode(msg));
        }
    }
    
    QString username() const { return m_username; }
    bool isAuthenticated() const { return !m_username.isEmpty(); }
    
signals:
    void messageReceived(ClientSession* session, const Protocol::ChatMessage& msg);
    void disconnected(ClientSession* session);
    
private slots:
    void onReadyRead() {
        m_buffer.append(m_socket->readAll());
        
        while (m_buffer.size() >= 4) {
            QDataStream ds(m_buffer);
            ds.setByteOrder(QDataStream::BigEndian);
            
            quint32 msgLen;
            ds >> msgLen;
            
            if (m_buffer.size() < 4 + static_cast<int>(msgLen)) break;
            
            QByteArray json = m_buffer.mid(4, msgLen);
            m_buffer.remove(0, 4 + msgLen);
            
            auto msg = Protocol::ChatMessage::fromJson(
                QJsonDocument::fromJson(json).object());
            
            // Handle connect
            if (msg.type == Protocol::MessageType::Connect) {
                m_username = msg.from;
            }
            
            emit messageReceived(this, msg);
        }
    }
    
    QTcpSocket* m_socket;
    QByteArray m_buffer;
    QString m_username;
};

// ===

class ChatServer : public QObject {
    Q_OBJECT
    
public:
    explicit ChatServer(QObject* parent = nullptr) : QObject(parent) {
        m_server = new QTcpServer(this);
        
        connect(m_server, &QTcpServer::newConnection,
                this, &ChatServer::onNewConnection);
    }
    
    bool listen(quint16 port = 12345) {
        if (!m_server->listen(QHostAddress::Any, port)) {
            qCritical() << "Server listen error:" << m_server->errorString();
            return false;
        }
        qInfo() << "Chat server listening on port" << port;
        return true;
    }
    
    void stop() { m_server->close(); }
    
    QStringList onlineUsers() const {
        QStringList users;
        for (const auto* session : m_sessions) {
            if (session->isAuthenticated()) users << session->username();
        }
        return users;
    }
    
signals:
    void userJoined(const QString& username);
    void userLeft(const QString& username);
    void messageReceived(const Protocol::ChatMessage& msg);
    
private slots:
    void onNewConnection() {
        while (m_server->hasPendingConnections()) {
            auto* socket = m_server->nextPendingConnection();
            auto* session = new ClientSession(socket, this);
            
            connect(session, &ClientSession::messageReceived,
                    this, &ChatServer::handleMessage);
            connect(session, &ClientSession::disconnected,
                    this, &ChatServer::onSessionDisconnected);
            
            m_sessions.append(session);
            qDebug() << "New connection from" << socket->peerAddress().toString();
        }
    }
    
    void handleMessage(ClientSession* from, const Protocol::ChatMessage& msg) {
        using MT = Protocol::MessageType;
        
        switch (msg.type) {
        case MT::Connect: {
            qInfo() << "User joined:" << msg.from;
            
            // Send current users list to newcomer
            auto usersMsg = Protocol::ChatMessage::make(
                MT::UsersList, "server", msg.from,
                onlineUsers().join(","));
            from->send(usersMsg);
            
            // Notify everyone
            auto joinMsg = Protocol::ChatMessage::make(
                MT::UserJoined, "server", "all", msg.from);
            broadcast(joinMsg, from);
            
            emit userJoined(msg.from);
            break;
        }
        
        case MT::Message: {
            // Broadcast to all
            emit messageReceived(msg);
            broadcast(msg, nullptr);
            break;
        }
        
        case MT::PrivateMessage: {
            // Send only to target
            auto* target = findSession(msg.to);
            if (target) {
                target->send(msg);
                from->send(msg);   // Echo back to sender
            } else {
                auto errMsg = Protocol::ChatMessage::make(
                    MT::Error, "server", msg.from,
                    QString("User '%1' not found").arg(msg.to));
                from->send(errMsg);
            }
            break;
        }
        
        case MT::Ping:
            from->send(Protocol::ChatMessage::make(MT::Pong, "server", msg.from, ""));
            break;
            
        default: break;
        }
    }
    
    void onSessionDisconnected(ClientSession* session) {
        QString username = session->username();
        
        if (!username.isEmpty()) {
            auto leaveMsg = Protocol::ChatMessage::make(
                Protocol::MessageType::UserLeft, "server", "all", username);
            broadcast(leaveMsg, nullptr);
            emit userLeft(username);
        }
        
        m_sessions.removeAll(session);
        session->deleteLater();
        qInfo() << "User disconnected:" << username;
    }
    
    void broadcast(const Protocol::ChatMessage& msg, ClientSession* exclude) {
        for (auto* session : m_sessions) {
            if (session != exclude && session->isAuthenticated()) {
                session->send(msg);
            }
        }
    }
    
    ClientSession* findSession(const QString& username) {
        for (auto* s : m_sessions) {
            if (s->username() == username) return s;
        }
        return nullptr;
    }
    
    QTcpServer* m_server;
    QList<ClientSession*> m_sessions;
};
```

---

## ขั้นตอนที่ 614: Chat Client

```cpp
class ChatClient : public QObject {
    Q_OBJECT
    Q_PROPERTY(bool connected READ isConnected NOTIFY connectionChanged)
    Q_PROPERTY(QString username READ username)
    
public:
    explicit ChatClient(QObject* parent = nullptr) : QObject(parent) {
        socket = new QTcpSocket(this);
        
        connect(socket, &QTcpSocket::connected, this, &ChatClient::onConnected);
        connect(socket, &QTcpSocket::disconnected, this, &ChatClient::onDisconnected);
        connect(socket, &QTcpSocket::readyRead, this, &ChatClient::onReadyRead);
        connect(socket, &QTcpSocket::errorOccurred, [this](QAbstractSocket::SocketError) {
            emit errorOccurred(socket->errorString());
        });
        
        // Heartbeat
        pingTimer = new QTimer(this);
        pingTimer->setInterval(30000);
        connect(pingTimer, &QTimer::timeout, this, &ChatClient::sendPing);
    }
    
    bool isConnected() const { return socket->state() == QAbstractSocket::ConnectedState; }
    QString username() const { return m_username; }
    
public slots:
    void connectToServer(const QString& host, quint16 port, const QString& username) {
        m_username = username;
        socket->connectToHost(host, port);
    }
    
    void disconnect() {
        pingTimer->stop();
        socket->disconnectFromHost();
    }
    
    void sendMessage(const QString& text, const QString& to = "all") {
        if (!isConnected()) return;
        
        auto type = (to == "all")
            ? Protocol::MessageType::Message
            : Protocol::MessageType::PrivateMessage;
        
        auto msg = Protocol::ChatMessage::make(type, m_username, to, text);
        socket->write(Protocol::encode(msg));
        
        emit messageSent(msg);
    }
    
signals:
    void connectionChanged(bool connected);
    void messageReceived(const Protocol::ChatMessage& msg);
    void messageSent(const Protocol::ChatMessage& msg);
    void userJoined(const QString& name);
    void userLeft(const QString& name);
    void onlineUsersUpdated(const QStringList& users);
    void errorOccurred(const QString& msg);
    
private slots:
    void onConnected() {
        // Send auth
        auto msg = Protocol::ChatMessage::make(
            Protocol::MessageType::Connect, m_username, "server", "");
        socket->write(Protocol::encode(msg));
        
        pingTimer->start();
        emit connectionChanged(true);
    }
    
    void onDisconnected() {
        pingTimer->stop();
        emit connectionChanged(false);
    }
    
    void onReadyRead() {
        m_buffer.append(socket->readAll());
        
        while (m_buffer.size() >= 4) {
            QDataStream ds(m_buffer);
            ds.setByteOrder(QDataStream::BigEndian);
            quint32 len;
            ds >> len;
            
            if (m_buffer.size() < 4 + static_cast<int>(len)) break;
            
            QByteArray json = m_buffer.mid(4, len);
            m_buffer.remove(0, 4 + len);
            
            auto msg = Protocol::ChatMessage::fromJson(
                QJsonDocument::fromJson(json).object());
            
            using MT = Protocol::MessageType;
            
            switch (msg.type) {
            case MT::Message:
            case MT::PrivateMessage:
                emit messageReceived(msg);
                break;
            case MT::UserJoined:
                emit userJoined(msg.text);
                break;
            case MT::UserLeft:
                emit userLeft(msg.text);
                break;
            case MT::UsersList:
                emit onlineUsersUpdated(msg.text.split(',', Qt::SkipEmptyParts));
                break;
            default: break;
            }
        }
    }
    
    void sendPing() {
        auto msg = Protocol::ChatMessage::make(
            Protocol::MessageType::Ping, m_username, "server", "");
        socket->write(Protocol::encode(msg));
    }
    
    QTcpSocket* socket;
    QTimer* pingTimer;
    QByteArray m_buffer;
    QString m_username;
};
```

---

## ขั้นตอนที่ 615: Message Bubble

```cpp
class MessageBubble : public QWidget {
public:
    enum Side { Left, Right };
    
    MessageBubble(const Protocol::ChatMessage& msg, Side side, QWidget* parent = nullptr)
        : QWidget(parent), m_side(side) {
        
        auto* layout = new QHBoxLayout(this);
        layout->setContentsMargins(8, 4, 8, 4);
        
        auto* bubble = new QWidget();
        auto* bLayout = new QVBoxLayout(bubble);
        bLayout->setContentsMargins(12, 8, 12, 8);
        bLayout->setSpacing(2);
        
        if (side == Left) {
            auto* sender = new QLabel(msg.from);
            sender->setStyleSheet("color: #3498db; font-weight: bold; font-size: 11px;");
            bLayout->addWidget(sender);
        }
        
        auto* text = new QLabel(msg.text);
        text->setWordWrap(true);
        text->setTextFormat(Qt::PlainText);
        text->setMaximumWidth(400);
        
        auto* ts = new QLabel(msg.timestamp.toString("hh:mm"));
        ts->setStyleSheet("color: rgba(0,0,0,0.4); font-size: 10px;");
        ts->setAlignment(side == Right ? Qt::AlignRight : Qt::AlignLeft);
        
        bLayout->addWidget(text);
        bLayout->addWidget(ts);
        
        bool isPrivate = (msg.type == Protocol::MessageType::PrivateMessage);
        
        QString bgColor = (side == Right) ? "#3498db" 
                          : isPrivate ? "#f0e6ff" : "#f0f0f0";
        QString textColor = (side == Right) ? "white" : "black";
        
        bubble->setStyleSheet(QString(
            "background: %1; color: %2; border-radius: 12px; "
            "border-top-%3-radius: 2px;")
            .arg(bgColor, textColor, side == Right ? "right" : "left"));
        
        if (side == Right) {
            layout->addStretch();
            layout->addWidget(bubble);
        } else {
            layout->addWidget(bubble);
            layout->addStretch();
        }
    }
    
private:
    Side m_side;
};
```

---

## ขั้นตอนที่ 616-625: Chat Window

```cpp
class ChatWindow : public QMainWindow {
    Q_OBJECT
    
public:
    ChatWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Chat");
        setMinimumSize(900, 600);
        
        client = new ChatClient(this);
        
        setupUi();
        setupConnections();
        showLoginDialog();
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* mainLayout = new QHBoxLayout(central);
        mainLayout->setSpacing(0);
        mainLayout->setContentsMargins(0, 0, 0, 0);
        setCentralWidget(central);
        
        // Contacts sidebar
        auto* sidebar = new QWidget();
        sidebar->setFixedWidth(220);
        sidebar->setStyleSheet("background: #2c3e50; color: white;");
        auto* sLayout = new QVBoxLayout(sidebar);
        
        auto* statusWidget = new QWidget();
        statusWidget->setStyleSheet("background: #34495e; padding: 8px;");
        auto* sWLayout = new QHBoxLayout(statusWidget);
        
        statusIndicator = new QLabel("●");
        statusIndicator->setStyleSheet("color: #e74c3c;");
        usernameLabel = new QLabel("Not connected");
        usernameLabel->setStyleSheet("color: white;");
        
        sWLayout->addWidget(statusIndicator);
        sWLayout->addWidget(usernameLabel, 1);
        sLayout->addWidget(statusWidget);
        
        sLayout->addWidget(new QLabel("  Online Users:"));
        
        contactList = new QListWidget();
        contactList->setStyleSheet(
            "QListWidget { background: transparent; border: none; color: white; }"
            "QListWidget::item { padding: 8px 12px; }"
            "QListWidget::item:selected { background: rgba(255,255,255,0.15); }");
        contactList->addItem("All (Public)");
        sLayout->addWidget(contactList, 1);
        
        // Chat area
        auto* chatArea = new QWidget();
        auto* cLayout = new QVBoxLayout(chatArea);
        cLayout->setSpacing(0);
        cLayout->setContentsMargins(0, 0, 0, 0);
        
        // Messages
        scrollArea = new QScrollArea();
        scrollArea->setWidgetResizable(true);
        scrollArea->setHorizontalScrollBarPolicy(Qt::ScrollBarAlwaysOff);
        scrollArea->setFrameShape(QFrame::NoFrame);
        scrollArea->setStyleSheet("background: #f8f9fa;");
        
        messagesWidget = new QWidget();
        messagesLayout = new QVBoxLayout(messagesWidget);
        messagesLayout->setAlignment(Qt::AlignTop);
        messagesLayout->setSpacing(4);
        
        scrollArea->setWidget(messagesWidget);
        
        // Input
        auto* inputArea = new QWidget();
        inputArea->setStyleSheet("background: white; border-top: 1px solid #e0e0e0;");
        auto* iLayout = new QHBoxLayout(inputArea);
        
        messageEdit = new QLineEdit();
        messageEdit->setPlaceholderText("Type a message...");
        messageEdit->setStyleSheet("border: none; padding: 12px; font-size: 14px;");
        
        auto* sendBtn = new QPushButton("Send");
        sendBtn->setStyleSheet(
            "background: #3498db; color: white; padding: 12px 24px; border: none; font-weight: bold;");
        
        iLayout->addWidget(messageEdit, 1);
        iLayout->addWidget(sendBtn);
        
        cLayout->addWidget(scrollArea, 1);
        cLayout->addWidget(inputArea);
        
        mainLayout->addWidget(sidebar);
        mainLayout->addWidget(chatArea, 1);
        
        connect(sendBtn, &QPushButton::clicked, this, &ChatWindow::sendMessage);
        connect(messageEdit, &QLineEdit::returnPressed, this, &ChatWindow::sendMessage);
        
        connect(contactList, &QListWidget::itemClicked, [this](QListWidgetItem* item) {
            m_currentChat = (item->text() == "All (Public)") ? "all" : item->text();
        });
    }
    
    void setupConnections() {
        connect(client, &ChatClient::connectionChanged, [this](bool connected) {
            statusIndicator->setStyleSheet(
                connected ? "color: #27ae60;" : "color: #e74c3c;");
            statusIndicator->setText("●");
            
            if (!connected) {
                addSystemMessage("Disconnected from server");
            }
        });
        
        connect(client, &ChatClient::messageReceived, this, &ChatWindow::onMessage);
        connect(client, &ChatClient::messageSent, this, &ChatWindow::onMessageSent);
        
        connect(client, &ChatClient::userJoined, [this](const QString& name) {
            updateContacts(name, true);
            addSystemMessage(name + " joined the chat");
        });
        
        connect(client, &ChatClient::userLeft, [this](const QString& name) {
            updateContacts(name, false);
            addSystemMessage(name + " left the chat");
        });
        
        connect(client, &ChatClient::onlineUsersUpdated, [this](const QStringList& users) {
            contactList->clear();
            contactList->addItem("All (Public)");
            for (const QString& u : users) {
                if (u != client->username()) contactList->addItem(u);
            }
        });
        
        connect(client, &ChatClient::errorOccurred, [this](const QString& err) {
            QMessageBox::warning(this, "Connection Error", err);
        });
    }
    
    void showLoginDialog() {
        QDialog dlg(this);
        dlg.setWindowTitle("Connect to Chat Server");
        
        auto* layout = new QFormLayout(&dlg);
        auto* hostEdit = new QLineEdit("localhost");
        auto* portEdit = new QSpinBox();
        portEdit->setRange(1, 65535);
        portEdit->setValue(12345);
        auto* nameEdit = new QLineEdit();
        nameEdit->setPlaceholderText("Enter your username...");
        
        layout->addRow("Server:", hostEdit);
        layout->addRow("Port:", portEdit);
        layout->addRow("Username:", nameEdit);
        
        auto* btnBox = new QDialogButtonBox(QDialogButtonBox::Ok | QDialogButtonBox::Cancel);
        layout->addRow(btnBox);
        
        connect(btnBox, &QDialogButtonBox::accepted, &dlg, &QDialog::accept);
        connect(btnBox, &QDialogButtonBox::rejected, this, &QWidget::close);
        
        if (dlg.exec() != QDialog::Accepted) return;
        
        QString username = nameEdit->text().trimmed();
        if (username.isEmpty()) { showLoginDialog(); return; }
        
        usernameLabel->setText(username);
        client->connectToServer(hostEdit->text(), portEdit->value(), username);
    }
    
    void sendMessage() {
        QString text = messageEdit->text().trimmed();
        if (text.isEmpty() || !client->isConnected()) return;
        
        client->sendMessage(text, m_currentChat);
        messageEdit->clear();
    }
    
    void onMessage(const Protocol::ChatMessage& msg) {
        auto* bubble = new MessageBubble(msg, MessageBubble::Left, messagesWidget);
        messagesLayout->addWidget(bubble);
        scrollToBottom();
    }
    
    void onMessageSent(const Protocol::ChatMessage& msg) {
        auto* bubble = new MessageBubble(msg, MessageBubble::Right, messagesWidget);
        messagesLayout->addWidget(bubble);
        scrollToBottom();
    }
    
    void addSystemMessage(const QString& text) {
        auto* label = new QLabel(text);
        label->setAlignment(Qt::AlignCenter);
        label->setStyleSheet("color: #999; font-size: 12px; padding: 4px;");
        messagesLayout->addWidget(label);
        scrollToBottom();
    }
    
    void scrollToBottom() {
        QTimer::singleShot(50, [this]() {
            scrollArea->verticalScrollBar()->setValue(
                scrollArea->verticalScrollBar()->maximum());
        });
    }
    
    void updateContacts(const QString& name, bool online) {
        // Remove existing
        for (int i = 0; i < contactList->count(); ++i) {
            if (contactList->item(i)->text() == name) {
                delete contactList->takeItem(i);
                break;
            }
        }
        if (online) contactList->addItem(name);
    }
    
    ChatClient* client;
    QScrollArea* scrollArea;
    QWidget* messagesWidget;
    QVBoxLayout* messagesLayout;
    QListWidget* contactList;
    QLineEdit* messageEdit;
    QLabel* statusIndicator;
    QLabel* usernameLabel;
    QString m_currentChat = "all";
};
```

---

## สรุป Part 043

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Protocol design ด้วย length-prefixed JSON framing
2. ✅ Chat Server ด้วย QTcpServer + ClientSession routing
3. ✅ Chat Client ด้วย automatic reconnect heartbeat
4. ✅ MessageBubble widget แบบ iMessage-style
5. ✅ Chat Window ครบ: login, contacts sidebar, message history

---

⬅️ [Part 042](part042.md) | ➡️ [Part 044: Real-World Project — File Manager](part044.md)
