# Part 032: Qt Network Advanced

## ขั้นตอนที่ 446-460

---

## ขั้นตอนที่ 446: REST API Client

```cpp
#include <QNetworkAccessManager>
#include <QNetworkRequest>
#include <QNetworkReply>
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <QUrlQuery>
#include <QTimer>

// === Modern REST Client with error handling ===
class RestClient : public QObject {
    Q_OBJECT
    
public:
    struct Response {
        int statusCode;
        QByteArray body;
        QNetworkReply::NetworkError error;
        QString errorString;
        
        bool ok() const { return error == QNetworkReply::NoError && statusCode < 400; }
        QJsonDocument json() const { return QJsonDocument::fromJson(body); }
    };
    
    using Callback = std::function<void(const Response&)>;
    
    explicit RestClient(const QString& baseUrl, QObject* parent = nullptr)
        : QObject(parent), m_baseUrl(baseUrl) {
        m_manager = new QNetworkAccessManager(this);
        m_manager->setTransferTimeout(30000); // 30s timeout
    }
    
    void setAuthHeader(const QString& token) {
        m_authHeader = "Bearer " + token;
    }
    
    void setDefaultHeader(const QString& key, const QString& value) {
        m_headers[key] = value;
    }
    
    // GET request
    void get(const QString& endpoint, const QUrlQuery& params, Callback callback) {
        QUrl url(m_baseUrl + endpoint);
        if (!params.isEmpty()) url.setQuery(params);
        
        QNetworkRequest request(url);
        applyHeaders(request);
        
        auto* reply = m_manager->get(request);
        connectReply(reply, callback);
    }
    
    void get(const QString& endpoint, Callback callback) {
        get(endpoint, {}, callback);
    }
    
    // POST request
    void post(const QString& endpoint, const QJsonObject& body, Callback callback) {
        QNetworkRequest request(QUrl(m_baseUrl + endpoint));
        applyHeaders(request);
        request.setHeader(QNetworkRequest::ContentTypeHeader, "application/json");
        
        QByteArray data = QJsonDocument(body).toJson(QJsonDocument::Compact);
        auto* reply = m_manager->post(request, data);
        connectReply(reply, callback);
    }
    
    // PUT request
    void put(const QString& endpoint, const QJsonObject& body, Callback callback) {
        QNetworkRequest request(QUrl(m_baseUrl + endpoint));
        applyHeaders(request);
        request.setHeader(QNetworkRequest::ContentTypeHeader, "application/json");
        
        QByteArray data = QJsonDocument(body).toJson(QJsonDocument::Compact);
        auto* reply = m_manager->put(request, data);
        connectReply(reply, callback);
    }
    
    // DELETE request
    void del(const QString& endpoint, Callback callback) {
        QNetworkRequest request(QUrl(m_baseUrl + endpoint));
        applyHeaders(request);
        
        auto* reply = m_manager->deleteResource(request);
        connectReply(reply, callback);
    }
    
    // File download
    void download(const QUrl& url, const QString& savePath, 
                  std::function<void(int)> progressCallback,
                  std::function<void(bool, const QString&)> doneCallback) {
        QNetworkRequest request(url);
        auto* reply = m_manager->get(request);
        
        auto* file = new QFile(savePath, reply);
        if (!file->open(QIODevice::WriteOnly)) {
            reply->deleteLater();
            doneCallback(false, "Cannot create file: " + savePath);
            return;
        }
        
        connect(reply, &QNetworkReply::readyRead, [reply, file]() {
            file->write(reply->readAll());
        });
        
        connect(reply, &QNetworkReply::downloadProgress, 
                [progressCallback](qint64 received, qint64 total) {
            if (total > 0) progressCallback(received * 100 / total);
        });
        
        connect(reply, &QNetworkReply::finished, [reply, file, doneCallback]() {
            file->close();
            if (reply->error() == QNetworkReply::NoError) {
                doneCallback(true, {});
            } else {
                QFile::remove(file->fileName());
                doneCallback(false, reply->errorString());
            }
            reply->deleteLater();
        });
    }
    
private:
    void applyHeaders(QNetworkRequest& request) {
        if (!m_authHeader.isEmpty()) {
            request.setRawHeader("Authorization", m_authHeader.toUtf8());
        }
        for (const auto& [k, v] : m_headers.asKeyValueRange()) {
            request.setRawHeader(k.toUtf8(), v.toUtf8());
        }
    }
    
    void connectReply(QNetworkReply* reply, Callback callback) {
        connect(reply, &QNetworkReply::finished, [reply, callback]() {
            Response r;
            r.error = reply->error();
            r.errorString = reply->errorString();
            r.statusCode = reply->attribute(QNetworkRequest::HttpStatusCodeAttribute).toInt();
            r.body = reply->readAll();
            
            callback(r);
            reply->deleteLater();
        });
    }
    
    QNetworkAccessManager* m_manager;
    QString m_baseUrl;
    QString m_authHeader;
    QMap<QString, QString> m_headers;
};

// === Usage ===
class GithubClient : public QObject {
    Q_OBJECT
    
public:
    GithubClient(QObject* parent = nullptr) : QObject(parent) {
        m_rest = new RestClient("https://api.github.com", this);
        m_rest->setDefaultHeader("Accept", "application/vnd.github+json");
        m_rest->setDefaultHeader("X-GitHub-Api-Version", "2022-11-28");
    }
    
    void setToken(const QString& token) {
        m_rest->setAuthHeader(token);
    }
    
    void getUser(const QString& username, std::function<void(const QJsonObject&)> cb) {
        m_rest->get("/users/" + username, [cb](const RestClient::Response& r) {
            if (r.ok()) {
                cb(r.json().object());
            } else {
                qWarning() << "GitHub API error:" << r.statusCode << r.errorString;
                cb({});
            }
        });
    }
    
    void listRepos(const QString& username, std::function<void(const QJsonArray&)> cb) {
        QUrlQuery params;
        params.addQueryItem("per_page", "10");
        params.addQueryItem("sort", "updated");
        
        m_rest->get("/users/" + username + "/repos", params,
                    [cb](const RestClient::Response& r) {
            if (r.ok()) {
                cb(r.json().array());
            } else {
                cb({});
            }
        });
    }
    
private:
    RestClient* m_rest;
};
```

---

## ขั้นตอนที่ 447: WebSocket Client

```cpp
#include <QWebSocket>
#include <QJsonDocument>

class WebSocketClient : public QObject {
    Q_OBJECT
    
public:
    explicit WebSocketClient(const QUrl& url, QObject* parent = nullptr)
        : QObject(parent), m_url(url) {
        
        m_socket = new QWebSocket("MyApp", QWebSocketProtocol::VersionLatest, this);
        
        connect(m_socket, &QWebSocket::connected, this, &WebSocketClient::onConnected);
        connect(m_socket, &QWebSocket::disconnected, this, &WebSocketClient::onDisconnected);
        connect(m_socket, &QWebSocket::textMessageReceived, this, &WebSocketClient::onMessage);
        connect(m_socket, QOverload<QAbstractSocket::SocketError>::of(&QWebSocket::errorOccurred),
                [this](QAbstractSocket::SocketError error) {
            qWarning() << "WebSocket error:" << error << m_socket->errorString();
            scheduleReconnect();
        });
        
        // Heartbeat
        m_heartbeat = new QTimer(this);
        m_heartbeat->setInterval(30000); // 30s
        connect(m_heartbeat, &QTimer::timeout, [this]() {
            if (m_socket->state() == QAbstractSocket::ConnectedState) {
                sendPing();
            }
        });
    }
    
    void connectToServer() {
        if (m_socket->state() != QAbstractSocket::ConnectedState) {
            m_socket->open(m_url);
        }
    }
    
    void disconnect() {
        m_reconnectTimer.stop();
        m_socket->close();
    }
    
    bool isConnected() const {
        return m_socket->state() == QAbstractSocket::ConnectedState;
    }
    
    // Send JSON message
    void send(const QJsonObject& msg) {
        if (!isConnected()) {
            m_queue.enqueue(msg); // Buffer while disconnected
            return;
        }
        
        QString text = QJsonDocument(msg).toJson(QJsonDocument::Compact);
        m_socket->sendTextMessage(text);
    }
    
    void sendPing() {
        send({{"type", "ping"}, {"ts", QDateTime::currentMSecsSinceEpoch()}});
    }
    
signals:
    void connected();
    void disconnected();
    void messageReceived(const QJsonObject& msg);
    
private slots:
    void onConnected() {
        qDebug() << "WebSocket connected";
        m_heartbeat->start();
        m_reconnectCount = 0;
        
        // Flush queued messages
        while (!m_queue.isEmpty()) {
            send(m_queue.dequeue());
        }
        
        emit connected();
    }
    
    void onDisconnected() {
        qDebug() << "WebSocket disconnected";
        m_heartbeat->stop();
        emit disconnected();
        scheduleReconnect();
    }
    
    void onMessage(const QString& text) {
        QJsonDocument doc = QJsonDocument::fromJson(text.toUtf8());
        if (doc.isObject()) {
            QJsonObject msg = doc.object();
            
            // Handle pong
            if (msg["type"].toString() == "pong") return;
            
            emit messageReceived(msg);
        }
    }
    
private:
    void scheduleReconnect() {
        if (m_reconnectCount >= 5) {
            qWarning() << "Max reconnect attempts reached";
            return;
        }
        
        int delay = std::min(1000 * (1 << m_reconnectCount), 30000); // Exponential backoff
        m_reconnectCount++;
        
        QTimer::singleShot(delay, this, &WebSocketClient::connectToServer);
        qDebug() << "Reconnecting in" << delay << "ms";
    }
    
    QWebSocket* m_socket;
    QUrl m_url;
    QTimer* m_heartbeat;
    QTimer m_reconnectTimer;
    QQueue<QJsonObject> m_queue;
    int m_reconnectCount = 0;
};
```

---

## ขั้นตอนที่ 448: Chat Application with WebSocket

```cpp
class ChatWindow : public QMainWindow {
    Q_OBJECT
    
public:
    ChatWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Qt Chat");
        setMinimumSize(600, 500);
        
        ws = new WebSocketClient(QUrl("wss://chat.example.com/ws"), this);
        
        setupUi();
        
        connect(ws, &WebSocketClient::connected, [this]() {
            statusLabel->setText("Connected");
            statusLabel->setStyleSheet("color: #27ae60;");
            appendSystemMessage("Connected to server");
        });
        
        connect(ws, &WebSocketClient::disconnected, [this]() {
            statusLabel->setText("Disconnected (reconnecting...)");
            statusLabel->setStyleSheet("color: #e74c3c;");
        });
        
        connect(ws, &WebSocketClient::messageReceived, this, &ChatWindow::onMessage);
        
        ws->connectToServer();
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* layout = new QVBoxLayout(central);
        setCentralWidget(central);
        
        // Header
        auto* header = new QHBoxLayout();
        auto* titleLabel = new QLabel("Qt Chat");
        titleLabel->setStyleSheet("font-size: 18px; font-weight: bold;");
        statusLabel = new QLabel("Connecting...");
        statusLabel->setStyleSheet("color: orange;");
        header->addWidget(titleLabel);
        header->addStretch();
        header->addWidget(statusLabel);
        
        // Chat display
        chatDisplay = new QTextBrowser();
        chatDisplay->setOpenExternalLinks(true);
        
        // Input area
        auto* inputArea = new QHBoxLayout();
        messageInput = new QLineEdit();
        messageInput->setPlaceholderText("พิมพ์ข้อความ...");
        messageInput->setEnabled(false);
        
        auto* sendBtn = new QPushButton("ส่ง");
        sendBtn->setStyleSheet("background: #3498db; color: white; padding: 8px 16px;");
        sendBtn->setEnabled(false);
        
        inputArea->addWidget(messageInput, 1);
        inputArea->addWidget(sendBtn);
        
        // Username
        auto* userArea = new QHBoxLayout();
        userArea->addWidget(new QLabel("ชื่อ:"));
        usernameEdit = new QLineEdit("User" + QString::number(std::rand() % 1000));
        userArea->addWidget(usernameEdit);
        userArea->addStretch();
        
        layout->addLayout(header);
        layout->addWidget(chatDisplay, 1);
        layout->addLayout(inputArea);
        layout->addLayout(userArea);
        
        connect(ws, &WebSocketClient::connected, [sendBtn, this]() {
            sendBtn->setEnabled(true);
            messageInput->setEnabled(true);
        });
        
        auto sendMessage = [this]() {
            QString text = messageInput->text().trimmed();
            if (text.isEmpty()) return;
            
            ws->send({
                {"type", "message"},
                {"from", usernameEdit->text()},
                {"text", text},
                {"ts", QDateTime::currentMSecsSinceEpoch()}
            });
            
            messageInput->clear();
        };
        
        connect(sendBtn, &QPushButton::clicked, sendMessage);
        connect(messageInput, &QLineEdit::returnPressed, sendMessage);
    }
    
    void onMessage(const QJsonObject& msg) {
        QString type = msg["type"].toString();
        
        if (type == "message") {
            QString from = msg["from"].toString();
            QString text = msg["text"].toString();
            bool isMe = from == usernameEdit->text();
            
            appendChatMessage(from, text, isMe);
        } else if (type == "system") {
            appendSystemMessage(msg["text"].toString());
        } else if (type == "users") {
            // User list update
        }
    }
    
    void appendChatMessage(const QString& from, const QString& text, bool isMe) {
        QString style = isMe 
            ? "color: #2980b9; font-weight: bold;"
            : "color: #27ae60; font-weight: bold;";
        
        QString html = QString(
            "<div style='margin:4px;'>"
            "<span style='%1'>%2</span>: "
            "<span>%3</span>"
            "</div>"
        ).arg(style).arg(from.toHtmlEscaped()).arg(text.toHtmlEscaped());
        
        chatDisplay->append(html);
    }
    
    void appendSystemMessage(const QString& text) {
        chatDisplay->append(QString(
            "<div style='color: gray; font-style: italic; text-align: center;'>%1</div>"
        ).arg(text));
    }
    
    WebSocketClient* ws;
    QTextBrowser* chatDisplay;
    QLineEdit* messageInput;
    QLineEdit* usernameEdit;
    QLabel* statusLabel;
};
```

---

## ขั้นตอนที่ 449-460: HTTP Server with Qt

```cpp
#include <QTcpServer>
#include <QTcpSocket>
#include <QHttpServer>  // Qt 6.4+

// Simple HTTP Server
class SimpleHttpServer : public QObject {
    Q_OBJECT
    
public:
    explicit SimpleHttpServer(QObject* parent = nullptr) : QObject(parent) {
        server = new QTcpServer(this);
        
        connect(server, &QTcpServer::newConnection, this, &SimpleHttpServer::handleConnection);
    }
    
    bool listen(quint16 port = 8080) {
        if (!server->listen(QHostAddress::LocalHost, port)) {
            qWarning() << "Server failed to start:" << server->errorString();
            return false;
        }
        qDebug() << "Server listening on port" << port;
        return true;
    }
    
    // Register route handler
    using Handler = std::function<void(const QString& path, 
                                        const QMap<QString, QString>& params,
                                        QTcpSocket* socket)>;
    
    void addRoute(const QString& method, const QString& path, Handler handler) {
        m_routes[method + ":" + path] = handler;
    }
    
    // Helpers
    static void sendResponse(QTcpSocket* socket, int statusCode, 
                              const QString& contentType, const QByteArray& body) {
        QString statusText = statusText_(statusCode);
        
        QByteArray response;
        response += QString("HTTP/1.1 %1 %2\r\n").arg(statusCode).arg(statusText).toUtf8();
        response += "Content-Type: " + contentType.toUtf8() + "\r\n";
        response += "Content-Length: " + QByteArray::number(body.size()) + "\r\n";
        response += "Connection: close\r\n";
        response += "\r\n";
        response += body;
        
        socket->write(response);
        socket->flush();
        socket->disconnectFromHost();
    }
    
    static void sendJson(QTcpSocket* socket, int status, const QJsonValue& json) {
        sendResponse(socket, status, "application/json",
                     QJsonDocument(json.toObject()).toJson(QJsonDocument::Compact));
    }
    
    static void sendHtml(QTcpSocket* socket, const QString& html) {
        sendResponse(socket, 200, "text/html; charset=utf-8", html.toUtf8());
    }
    
private slots:
    void handleConnection() {
        auto* socket = server->nextPendingConnection();
        
        connect(socket, &QTcpSocket::readyRead, [this, socket]() {
            QByteArray data = socket->readAll();
            parseRequest(data, socket);
        });
        
        connect(socket, &QTcpSocket::disconnected, socket, &QObject::deleteLater);
    }
    
private:
    void parseRequest(const QByteArray& data, QTcpSocket* socket) {
        QString request = QString::fromUtf8(data);
        QStringList lines = request.split("\r\n");
        if (lines.isEmpty()) return;
        
        // Parse request line: METHOD /path HTTP/1.1
        QStringList requestLine = lines[0].split(' ');
        if (requestLine.size() < 2) return;
        
        QString method = requestLine[0];
        QUrl url(requestLine[1]);
        QString path = url.path();
        
        // Parse query params
        QMap<QString, QString> params;
        QUrlQuery query(url);
        for (const auto& [k, v] : query.queryItems()) {
            params[k] = v;
        }
        
        // Find handler
        QString key = method + ":" + path;
        if (m_routes.contains(key)) {
            m_routes[key](path, params, socket);
        } else {
            sendJson(socket, 404, QJsonObject{{"error", "Not Found"}, {"path", path}});
        }
    }
    
    static QString statusText_(int code) {
        static QMap<int, QString> texts = {
            {200, "OK"}, {201, "Created"}, {204, "No Content"},
            {400, "Bad Request"}, {401, "Unauthorized"}, {403, "Forbidden"},
            {404, "Not Found"}, {500, "Internal Server Error"}
        };
        return texts.value(code, "Unknown");
    }
    
    QTcpServer* server;
    QMap<QString, Handler> m_routes;
};

// === Demo API server ===
class ApiServer : public QObject {
    Q_OBJECT
    
public:
    ApiServer() {
        server = new SimpleHttpServer(this);
        
        // Routes
        server->addRoute("GET", "/", [](const QString&, const QMap<QString,QString>&, QTcpSocket* s) {
            SimpleHttpServer::sendHtml(s, "<h1>Qt HTTP Server</h1><p>Hello World!</p>");
        });
        
        server->addRoute("GET", "/api/status", [](const QString&, const QMap<QString,QString>&, QTcpSocket* s) {
            SimpleHttpServer::sendJson(s, 200, QJsonObject{
                {"status", "ok"},
                {"time", QDateTime::currentDateTime().toString(Qt::ISODate)},
                {"version", "1.0"}
            });
        });
        
        server->addRoute("GET", "/api/users", [this](const QString&, const QMap<QString,QString>&, QTcpSocket* s) {
            QJsonArray arr;
            for (const auto& u : m_users) arr.append(u);
            SimpleHttpServer::sendJson(s, 200, arr);
        });
        
        server->addRoute("GET", "/api/greet", [](const QString&, const QMap<QString,QString>& p, QTcpSocket* s) {
            QString name = p.value("name", "World");
            SimpleHttpServer::sendJson(s, 200, QJsonObject{
                {"message", "สวัสดี, " + name + "!"}
            });
        });
        
        // Seed data
        m_users = {
            QJsonObject{{"id", 1}, {"name", "Alice"}, {"email", "alice@example.com"}},
            QJsonObject{{"id", 2}, {"name", "Bob"},   {"email", "bob@example.com"}}
        };
    }
    
    bool start(quint16 port = 8080) {
        return server->listen(port);
    }
    
private:
    SimpleHttpServer* server;
    QList<QJsonObject> m_users;
};

int main(int argc, char* argv[]) {
    QCoreApplication app(argc, argv);
    
    ApiServer apiServer;
    if (!apiServer.start(8080)) return 1;
    
    qDebug() << "API server running at http://localhost:8080";
    qDebug() << "  GET http://localhost:8080/api/status";
    qDebug() << "  GET http://localhost:8080/api/users";
    qDebug() << "  GET http://localhost:8080/api/greet?name=สมชาย";
    
    return app.exec();
}
```

---

## สรุป Part 032

ใน Part นี้คุณได้เรียนรู้:

1. ✅ REST Client (GET, POST, PUT, DELETE, download)
2. ✅ GitHub API Client example
3. ✅ WebSocket Client with reconnect + heartbeat
4. ✅ Chat application using WebSocket
5. ✅ Simple HTTP Server with routing
6. ✅ API Server with JSON responses

---

⬅️ [Part 031](part031.md) | ➡️ [Part 033: QML Advanced](part033.md)
