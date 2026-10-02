# Part 016: Qt Network & HTTP

## ขั้นตอนที่ 206-220

---

## ขั้นตอนที่ 206: QtNetwork Module

**เพิ่มใน .pro:**
```qmake
QT += network
```

```
Qt Network Classes:
  ├── QNetworkAccessManager  - HTTP Client
  ├── QNetworkRequest        - HTTP Request
  ├── QNetworkReply          - HTTP Response
  ├── QTcpServer             - TCP Server
  ├── QTcpSocket             - TCP Client
  ├── QUdpSocket             - UDP
  ├── QSslSocket             - SSL/TLS
  └── QWebSocket             - WebSocket
```

---

## ขั้นตอนที่ 207: HTTP GET Request

```cpp
#include <QApplication>
#include <QNetworkAccessManager>
#include <QNetworkRequest>
#include <QNetworkReply>
#include <QUrl>
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <QTimer>

class HttpClient : public QObject {
    Q_OBJECT
    
private:
    QNetworkAccessManager* manager;
    
public:
    explicit HttpClient(QObject* parent = nullptr) : QObject(parent) {
        manager = new QNetworkAccessManager(this);
        
        // SSL certificate error handling
        connect(manager, &QNetworkAccessManager::sslErrors,
                [](QNetworkReply* reply, const QList<QSslError>& errors) {
            qWarning() << "SSL errors:" << errors;
            // reply->ignoreSslErrors();  // เฉพาะ dev!
        });
    }
    
    void get(const QString& url, 
             std::function<void(const QByteArray&, int statusCode)> callback) {
        QNetworkRequest req(QUrl(url));
        req.setHeader(QNetworkRequest::UserAgentHeader, "QtApp/1.0");
        req.setRawHeader("Accept", "application/json");
        
        QNetworkReply* reply = manager->get(req);
        
        connect(reply, &QNetworkReply::finished, [=]() {
            int status = reply->attribute(QNetworkRequest::HttpStatusCodeAttribute).toInt();
            
            if (reply->error() == QNetworkReply::NoError) {
                callback(reply->readAll(), status);
            } else {
                qWarning() << "HTTP error:" << reply->errorString();
                callback({}, -1);
            }
            reply->deleteLater();
        });
        
        // Timeout
        QTimer* timer = new QTimer(this);
        timer->setSingleShot(true);
        timer->start(10000);  // 10 seconds
        connect(timer, &QTimer::timeout, [=]() {
            if (reply->isRunning()) {
                reply->abort();
                qWarning() << "Request timeout";
            }
            timer->deleteLater();
        });
    }
    
    void post(const QString& url, const QByteArray& data,
              const QString& contentType,
              std::function<void(const QByteArray&, int)> callback) {
        QNetworkRequest req(QUrl(url));
        req.setHeader(QNetworkRequest::ContentTypeHeader, contentType);
        req.setHeader(QNetworkRequest::UserAgentHeader, "QtApp/1.0");
        
        QNetworkReply* reply = manager->post(req, data);
        
        connect(reply, &QNetworkReply::finished, [=]() {
            int status = reply->attribute(QNetworkRequest::HttpStatusCodeAttribute).toInt();
            QByteArray body = reply->readAll();
            callback(body, status);
            reply->deleteLater();
        });
    }
    
    void postJson(const QString& url, const QJsonObject& json,
                  std::function<void(const QJsonDocument&, int)> callback) {
        QByteArray data = QJsonDocument(json).toJson(QJsonDocument::Compact);
        
        post(url, data, "application/json", [=](const QByteArray& body, int status) {
            QJsonDocument doc = QJsonDocument::fromJson(body);
            callback(doc, status);
        });
    }
    
    void download(const QString& url, const QString& savePath,
                  std::function<void(bool success, const QString& path)> callback,
                  std::function<void(qint64 received, qint64 total)> progressCallback = nullptr) {
        
        QNetworkRequest req(QUrl(url));
        QNetworkReply* reply = manager->get(req);
        
        if (progressCallback) {
            connect(reply, &QNetworkReply::downloadProgress, progressCallback);
        }
        
        connect(reply, &QNetworkReply::finished, [=]() {
            if (reply->error() != QNetworkReply::NoError) {
                callback(false, reply->errorString());
                reply->deleteLater();
                return;
            }
            
            QFile file(savePath);
            if (file.open(QIODevice::WriteOnly)) {
                file.write(reply->readAll());
                file.close();
                callback(true, savePath);
            } else {
                callback(false, "Cannot create file: " + savePath);
            }
            reply->deleteLater();
        });
    }
};
```

---

## ขั้นตอนที่ 208: JSON Parsing

```cpp
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <QJsonValue>

// สมมติว่าได้รับ JSON จาก API
QString jsonString = R"({
    "status": "success",
    "count": 3,
    "users": [
        {"id": 1, "name": "Alice", "email": "alice@example.com", "active": true},
        {"id": 2, "name": "Bob",   "email": "bob@example.com",   "active": false},
        {"id": 3, "name": "Carol", "email": "carol@example.com", "active": true}
    ],
    "metadata": {
        "page": 1,
        "per_page": 10,
        "total": 3
    }
})";

struct User {
    int id;
    QString name;
    QString email;
    bool active;
    
    static User fromJson(const QJsonObject& obj) {
        User u;
        u.id     = obj["id"].toInt();
        u.name   = obj["name"].toString();
        u.email  = obj["email"].toString();
        u.active = obj["active"].toBool();
        return u;
    }
    
    QJsonObject toJson() const {
        QJsonObject obj;
        obj["id"]     = id;
        obj["name"]   = name;
        obj["email"]  = email;
        obj["active"] = active;
        return obj;
    }
    
    QString toString() const {
        return QString("[%1] %2 <%3> %4")
            .arg(id)
            .arg(name)
            .arg(email)
            .arg(active ? "active" : "inactive");
    }
};

void parseJsonExample() {
    // Parse JSON
    QJsonDocument doc = QJsonDocument::fromJson(jsonString.toUtf8());
    
    if (doc.isNull() || !doc.isObject()) {
        qWarning() << "Invalid JSON";
        return;
    }
    
    QJsonObject root = doc.object();
    
    // Read values
    QString status = root["status"].toString();
    int count = root["count"].toInt();
    
    qDebug() << "Status:" << status;
    qDebug() << "Count:" << count;
    
    // Parse array
    QJsonArray usersArr = root["users"].toArray();
    QList<User> users;
    
    for (const QJsonValue& val : usersArr) {
        if (val.isObject()) {
            users << User::fromJson(val.toObject());
        }
    }
    
    qDebug() << "\nUsers:";
    for (const User& u : users) {
        qDebug() << " -" << u.toString();
    }
    
    // Parse nested object
    QJsonObject meta = root["metadata"].toObject();
    qDebug() << "\nMetadata:";
    qDebug() << "Page:" << meta["page"].toInt();
    qDebug() << "Total:" << meta["total"].toInt();
    
    // Create JSON
    QJsonObject newUser;
    newUser["name"]  = "Dave";
    newUser["email"] = "dave@example.com";
    newUser["active"] = true;
    
    QJsonArray arr;
    for (const User& u : users) {
        arr.append(u.toJson());
    }
    arr.append(newUser);
    
    QJsonObject response;
    response["status"] = "success";
    response["users"]  = arr;
    
    QJsonDocument outDoc(response);
    QString jsonStr = outDoc.toJson(QJsonDocument::Indented);
    qDebug() << "\nGenerated JSON:" << jsonStr;
}
```

---

## ขั้นตอนที่ 209: REST API Client

```cpp
class ApiClient : public QObject {
    Q_OBJECT
    
private:
    HttpClient* http;
    QString baseUrl;
    QString authToken;
    
public:
    explicit ApiClient(const QString& url, QObject* parent = nullptr)
        : QObject(parent), baseUrl(url) {
        http = new HttpClient(this);
    }
    
    void setAuthToken(const QString& token) { authToken = token; }
    
    // User endpoints
    void getUsers(std::function<void(QList<User>)> callback) {
        http->get(baseUrl + "/api/users", [=](const QByteArray& body, int status) {
            if (status != 200) { callback({}); return; }
            
            QJsonDocument doc = QJsonDocument::fromJson(body);
            QJsonArray arr = doc.array();
            
            QList<User> users;
            for (const QJsonValue& v : arr) {
                users << User::fromJson(v.toObject());
            }
            callback(users);
        });
    }
    
    void createUser(const User& user,
                    std::function<void(User, bool)> callback) {
        http->postJson(baseUrl + "/api/users", user.toJson(),
                       [=](const QJsonDocument& doc, int status) {
            if (status == 201) {
                callback(User::fromJson(doc.object()), true);
            } else {
                callback({}, false);
            }
        });
    }
    
    void updateUser(int id, const User& user,
                    std::function<void(bool)> callback) {
        QString url = QString("%1/api/users/%2").arg(baseUrl).arg(id);
        QByteArray data = QJsonDocument(user.toJson()).toJson();
        
        http->post(url, data, "application/json",
                   [=](const QByteArray&, int status) {
            callback(status == 200);
        });
    }
};
```

---

## ขั้นตอนที่ 210-220: TCP Server/Client

```cpp
// === TCP Chat Server ===
#include <QTcpServer>
#include <QTcpSocket>

class ChatServer : public QTcpServer {
    Q_OBJECT
    
private:
    QList<QTcpSocket*> clients;
    
public:
    ChatServer(QObject* parent = nullptr) : QTcpServer(parent) {}
    
    bool start(quint16 port) {
        if (!listen(QHostAddress::Any, port)) {
            qCritical() << "Cannot listen on port" << port;
            return false;
        }
        qInfo() << "Server started on port" << port;
        return true;
    }
    
    void broadcast(const QString& message, QTcpSocket* exclude = nullptr) {
        QByteArray data = (message + "\n").toUtf8();
        for (QTcpSocket* client : clients) {
            if (client != exclude) {
                client->write(data);
            }
        }
    }
    
protected:
    void incomingConnection(qintptr socketDescriptor) override {
        QTcpSocket* client = new QTcpSocket(this);
        client->setSocketDescriptor(socketDescriptor);
        clients.append(client);
        
        QString clientAddr = client->peerAddress().toString();
        qInfo() << "Client connected:" << clientAddr;
        broadcast("[Server] " + clientAddr + " joined");
        
        connect(client, &QTcpSocket::readyRead, [=]() {
            QByteArray data = client->readAll();
            QString message = QString::fromUtf8(data).trimmed();
            qInfo() << "[" << clientAddr << "]:" << message;
            broadcast("[" + clientAddr + "]: " + message, client);
        });
        
        connect(client, &QTcpSocket::disconnected, [=]() {
            clients.removeOne(client);
            qInfo() << "Client disconnected:" << clientAddr;
            broadcast("[Server] " + clientAddr + " left");
            client->deleteLater();
        });
    }
};

// === TCP Chat Client ===
class ChatClient : public QObject {
    Q_OBJECT
    
private:
    QTcpSocket* socket;
    
public:
    ChatClient(QObject* parent = nullptr) : QObject(parent) {
        socket = new QTcpSocket(this);
        
        connect(socket, &QTcpSocket::connected, [this]() {
            qInfo() << "Connected to server";
            emit connected();
        });
        
        connect(socket, &QTcpSocket::disconnected, [this]() {
            qInfo() << "Disconnected";
            emit disconnected();
        });
        
        connect(socket, &QTcpSocket::readyRead, [this]() {
            QByteArray data = socket->readAll();
            QString msg = QString::fromUtf8(data).trimmed();
            for (const QString& line : msg.split('\n')) {
                if (!line.isEmpty()) emit messageReceived(line);
            }
        });
        
        connect(socket, &QAbstractSocket::errorOccurred,
                [this](QAbstractSocket::SocketError) {
            emit error(socket->errorString());
        });
    }
    
    void connectToServer(const QString& host, quint16 port) {
        socket->connectToHost(host, port);
    }
    
    void sendMessage(const QString& msg) {
        if (socket->state() == QAbstractSocket::ConnectedState) {
            socket->write((msg + "\n").toUtf8());
        }
    }
    
    void disconnect() {
        socket->disconnectFromHost();
    }
    
signals:
    void connected();
    void disconnected();
    void messageReceived(const QString& msg);
    void error(const QString& err);
};

// === Chat Window ===
class ChatWindow : public QMainWindow {
    Q_OBJECT
    
public:
    ChatWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Qt Chat");
        setMinimumSize(500, 400);
        
        auto* central = new QWidget(this);
        setCentralWidget(central);
        auto* layout = new QVBoxLayout(central);
        
        // Chat display
        chatDisplay = new QTextEdit();
        chatDisplay->setReadOnly(true);
        
        // Input
        auto* inputBar = new QHBoxLayout();
        msgInput = new QLineEdit();
        msgInput->setPlaceholderText("พิมพ์ข้อความ...");
        auto* sendBtn = new QPushButton("ส่ง");
        inputBar->addWidget(msgInput);
        inputBar->addWidget(sendBtn);
        
        // Connect bar
        auto* connectBar = new QHBoxLayout();
        hostEdit = new QLineEdit("localhost");
        portSpin = new QSpinBox();
        portSpin->setRange(1, 65535);
        portSpin->setValue(12345);
        connectBtn = new QPushButton("เชื่อมต่อ");
        connectBar->addWidget(new QLabel("Host:"));
        connectBar->addWidget(hostEdit);
        connectBar->addWidget(new QLabel("Port:"));
        connectBar->addWidget(portSpin);
        connectBar->addWidget(connectBtn);
        
        layout->addLayout(connectBar);
        layout->addWidget(chatDisplay);
        layout->addLayout(inputBar);
        
        // Client
        client = new ChatClient(this);
        
        connect(connectBtn, &QPushButton::clicked, [this]() {
            if (connectBtn->text() == "เชื่อมต่อ") {
                client->connectToServer(hostEdit->text(), portSpin->value());
            } else {
                client->disconnect();
            }
        });
        
        connect(sendBtn, &QPushButton::clicked, this, &ChatWindow::sendMessage);
        connect(msgInput, &QLineEdit::returnPressed, this, &ChatWindow::sendMessage);
        
        connect(client, &ChatClient::connected, [this]() {
            connectBtn->setText("ตัดการเชื่อมต่อ");
            appendMessage("[System] เชื่อมต่อแล้ว");
        });
        
        connect(client, &ChatClient::disconnected, [this]() {
            connectBtn->setText("เชื่อมต่อ");
            appendMessage("[System] ตัดการเชื่อมต่อ");
        });
        
        connect(client, &ChatClient::messageReceived, this, &ChatWindow::appendMessage);
        connect(client, &ChatClient::error, [this](const QString& err) {
            appendMessage("[Error] " + err);
        });
    }
    
private:
    ChatClient* client;
    QTextEdit* chatDisplay;
    QLineEdit* msgInput;
    QLineEdit* hostEdit;
    QSpinBox* portSpin;
    QPushButton* connectBtn;
    
    void sendMessage() {
        QString msg = msgInput->text().trimmed();
        if (msg.isEmpty()) return;
        client->sendMessage(msg);
        appendMessage("ฉัน: " + msg);
        msgInput->clear();
    }
    
    void appendMessage(const QString& msg) {
        QString timestamp = QTime::currentTime().toString("HH:mm:ss");
        chatDisplay->append(QString("[%1] %2").arg(timestamp).arg(msg));
        chatDisplay->ensureCursorVisible();
    }
};
```

---

## สรุป Part 016

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QNetworkAccessManager - HTTP Client
2. ✅ JSON Parsing ด้วย QJsonDocument
3. ✅ REST API Client
4. ✅ TCP Server/Client
5. ✅ Chat Application

---

⬅️ [Part 015](part015.md) | ➡️ [Part 017: Qt File System & I/O](part017.md)
