# Part 059: World-Class — REST API Client & WebSocket

## ขั้นตอนที่ 851-865

---

## ขั้นตอนที่ 851: HTTP Client Architecture

```
Qt Network Stack:
  QNetworkAccessManager (shared instance, thread-safe)
    ├── GET/POST/PUT/DELETE/PATCH
    ├── SSL/TLS support
    ├── Redirects, cookies, authentication
    └── QNetworkReply (async, signals)

Best Practices:
  - ใช้ QNetworkAccessManager ตัวเดียวต่อ application
  - เชื่อม signals จาก QNetworkReply ไม่ใช่ wait blocking
  - Handle SSL errors อย่างระมัดระวัง
  - Set timeout via QNetworkReply abort + QTimer
  - Cancel requests ด้วย reply->abort()
```

---

## ขั้นตอนที่ 852: Generic HTTP Client

```cpp
// network/httpclient.h
#pragma once
#include <QObject>
#include <QNetworkAccessManager>
#include <QNetworkRequest>
#include <QNetworkReply>
#include <QJsonDocument>
#include <QTimer>
#include <functional>

class HttpClient : public QObject {
    Q_OBJECT
    
public:
    struct Response {
        int statusCode = 0;
        QByteArray body;
        QMap<QString, QString> headers;
        QString errorString;
        
        bool isOk() const { return statusCode >= 200 && statusCode < 300; }
        
        QJsonDocument json() const {
            return QJsonDocument::fromJson(body);
        }
        
        QJsonObject jsonObject() const { return json().object(); }
        QJsonArray  jsonArray()  const { return json().array(); }
    };
    
    struct RequestOptions {
        int timeoutMs = 30000;
        bool followRedirects = true;
        QMap<QString, QString> headers;
        QByteArray body;
        QString contentType = "application/json";
    };
    
    using Callback = std::function<void(const Response&)>;
    
    explicit HttpClient(const QString& baseUrl, QObject* parent = nullptr)
        : QObject(parent)
        , m_baseUrl(baseUrl)
        , m_nam(new QNetworkAccessManager(this))
    {
        m_nam->setAutoDeleteReplies(true);
    }
    
    void setAuthToken(const QString& token) {
        m_authToken = token;
    }
    
    void setBaseHeader(const QString& key, const QString& value) {
        m_baseHeaders[key] = value;
    }
    
    void get(const QString& path, Callback callback,
             const RequestOptions& opts = {}) {
        send(QNetworkAccessManager::GetOperation,
             path, {}, callback, opts);
    }
    
    void post(const QString& path, const QJsonObject& body,
              Callback callback, const RequestOptions& opts = {}) {
        RequestOptions o = opts;
        o.body = QJsonDocument(body).toJson(QJsonDocument::Compact);
        send(QNetworkAccessManager::PostOperation, path, o.body, callback, o);
    }
    
    void put(const QString& path, const QJsonObject& body,
             Callback callback, const RequestOptions& opts = {}) {
        RequestOptions o = opts;
        o.body = QJsonDocument(body).toJson(QJsonDocument::Compact);
        send(QNetworkAccessManager::PutOperation, path, o.body, callback, o);
    }
    
    void patch(const QString& path, const QJsonObject& body,
               Callback callback, const RequestOptions& opts = {}) {
        QNetworkRequest req = buildRequest(path, opts);
        auto* reply = m_nam->sendCustomRequest(req, "PATCH",
            QJsonDocument(body).toJson(QJsonDocument::Compact));
        handleReply(reply, callback, opts.timeoutMs);
    }
    
    void del(const QString& path, Callback callback,
             const RequestOptions& opts = {}) {
        send(QNetworkAccessManager::DeleteOperation, path, {}, callback, opts);
    }
    
    void upload(const QString& path, const QString& filePath,
                std::function<void(qint64, qint64)> progressCallback,
                Callback callback)
    {
        auto* file = new QFile(filePath, this);
        if (!file->open(QIODevice::ReadOnly)) {
            callback({0, {}, {}, "Cannot open file: " + filePath});
            return;
        }
        
        QHttpMultiPart* multiPart = new QHttpMultiPart(
            QHttpMultiPart::FormDataType, this);
        
        QHttpPart filePart;
        filePart.setHeader(QNetworkRequest::ContentDispositionHeader,
            QString("form-data; name=\"file\"; filename=\"%1\"")
                .arg(QFileInfo(filePath).fileName()));
        filePart.setBodyDevice(file);
        file->setParent(multiPart);
        multiPart->append(filePart);
        
        QNetworkRequest req = buildRequest(path, {});
        req.removeRawHeader("Content-Type");
        
        auto* reply = m_nam->post(req, multiPart);
        multiPart->setParent(reply);
        
        connect(reply, &QNetworkReply::uploadProgress, this,
                [progressCallback](qint64 sent, qint64 total) {
                    progressCallback(sent, total);
                });
        
        handleReply(reply, callback, 120000);
    }
    
private:
    void send(QNetworkAccessManager::Operation op,
              const QString& path,
              const QByteArray& body,
              Callback callback,
              const RequestOptions& opts)
    {
        QNetworkRequest req = buildRequest(path, opts);
        QNetworkReply* reply = nullptr;
        
        switch (op) {
            case QNetworkAccessManager::GetOperation:
                reply = m_nam->get(req);
                break;
            case QNetworkAccessManager::PostOperation:
                reply = m_nam->post(req, body);
                break;
            case QNetworkAccessManager::PutOperation:
                reply = m_nam->put(req, body);
                break;
            case QNetworkAccessManager::DeleteOperation:
                reply = m_nam->deleteResource(req);
                break;
            default:
                break;
        }
        
        if (reply) {
            handleReply(reply, callback, opts.timeoutMs);
        }
    }
    
    QNetworkRequest buildRequest(const QString& path, const RequestOptions& opts) {
        QString url = m_baseUrl + path;
        QNetworkRequest req{QUrl(url)};
        
        req.setHeader(QNetworkRequest::ContentTypeHeader, opts.contentType);
        req.setAttribute(QNetworkRequest::RedirectPolicyAttribute,
                         opts.followRedirects
                             ? QNetworkRequest::NoLessSafeRedirectPolicy
                             : QNetworkRequest::ManualRedirectPolicy);
        
        // Base headers
        for (auto it = m_baseHeaders.begin(); it != m_baseHeaders.end(); ++it) {
            req.setRawHeader(it.key().toUtf8(), it.value().toUtf8());
        }
        
        // Auth
        if (!m_authToken.isEmpty()) {
            req.setRawHeader("Authorization", ("Bearer " + m_authToken).toUtf8());
        }
        
        // Custom headers
        for (auto it = opts.headers.begin(); it != opts.headers.end(); ++it) {
            req.setRawHeader(it.key().toUtf8(), it.value().toUtf8());
        }
        
        return req;
    }
    
    void handleReply(QNetworkReply* reply, Callback callback, int timeoutMs) {
        // Timeout
        auto* timer = new QTimer(reply);
        timer->setSingleShot(true);
        connect(timer, &QTimer::timeout, reply, &QNetworkReply::abort);
        timer->start(timeoutMs);
        
        connect(reply, &QNetworkReply::finished, this, [=] {
            timer->stop();
            
            Response resp;
            resp.statusCode = reply->attribute(
                QNetworkRequest::HttpStatusCodeAttribute).toInt();
            resp.body = reply->readAll();
            resp.errorString = reply->errorString();
            
            // Read response headers
            for (const auto& header : reply->rawHeaderList()) {
                resp.headers[header] = reply->rawHeader(header);
            }
            
            if (reply->error() != QNetworkReply::NoError
                && resp.statusCode == 0) {
                resp.errorString = reply->errorString();
            }
            
            callback(resp);
        });
    }
    
    QString m_baseUrl;
    QString m_authToken;
    QMap<QString, QString> m_baseHeaders;
    QNetworkAccessManager* m_nam;
};
```

---

## ขั้นตอนที่ 853: API Service Layer

```cpp
// api/erpapiclient.h
#pragma once
#include "httpclient.h"

class ErpApiClient : public QObject {
    Q_OBJECT
    
public:
    explicit ErpApiClient(const QString& apiBase, QObject* parent = nullptr)
        : QObject(parent)
        , m_http(new HttpClient(apiBase, this))
    {
        m_http->setBaseHeader("Accept", "application/json");
        m_http->setBaseHeader("X-Client-Version", "1.0.0");
    }
    
    void setToken(const QString& token) {
        m_http->setAuthToken(token);
    }
    
    // --- Auth ---
    void login(const QString& username, const QString& password,
               std::function<void(bool, const QString& token, const QString& error)> callback)
    {
        m_http->post("/auth/login",
            {{"username", username}, {"password", password}},
            [callback](const HttpClient::Response& r) {
                if (r.isOk()) {
                    QString token = r.jsonObject()["access_token"].toString();
                    callback(true, token, {});
                } else {
                    QString err = r.jsonObject()["detail"].toString("Invalid credentials");
                    callback(false, {}, err);
                }
            }
        );
    }
    
    // --- Customers ---
    void getCustomers(int page, int pageSize,
                      std::function<void(const QJsonArray&, int total)> callback)
    {
        m_http->get(
            QString("/api/customers?page=%1&page_size=%2").arg(page).arg(pageSize),
            [callback](const HttpClient::Response& r) {
                if (r.isOk()) {
                    auto obj = r.jsonObject();
                    callback(obj["items"].toArray(), obj["total"].toInt());
                } else {
                    callback({}, 0);
                }
            }
        );
    }
    
    void createCustomer(const QJsonObject& data,
                        std::function<void(bool, const QJsonObject&)> callback)
    {
        m_http->post("/api/customers", data, [callback](const HttpClient::Response& r) {
            callback(r.isOk(), r.jsonObject());
        });
    }
    
    void updateCustomer(int id, const QJsonObject& data,
                        std::function<void(bool)> callback)
    {
        m_http->put(QString("/api/customers/%1").arg(id), data,
                    [callback](const HttpClient::Response& r) {
                        callback(r.isOk());
                    });
    }
    
    void deleteCustomer(int id, std::function<void(bool)> callback) {
        m_http->del(QString("/api/customers/%1").arg(id),
                    [callback](const HttpClient::Response& r) {
                        callback(r.isOk());
                    });
    }
    
    // --- Products ---
    void searchProducts(const QString& query,
                        std::function<void(const QJsonArray&)> callback)
    {
        QString path = "/api/products/search?q=" +
                       QUrl::toPercentEncoding(query);
        m_http->get(path, [callback](const HttpClient::Response& r) {
            callback(r.isOk() ? r.jsonArray() : QJsonArray{});
        });
    }
    
    // --- Sales Orders ---
    void createOrder(const QJsonObject& order,
                     std::function<void(bool, int orderId)> callback)
    {
        m_http->post("/api/orders", order, [callback](const HttpClient::Response& r) {
            if (r.isOk()) {
                callback(true, r.jsonObject()["id"].toInt());
            } else {
                callback(false, 0);
            }
        });
    }
    
    // --- Reports ---
    void downloadReport(const QString& reportType,
                        const QVariantMap& filters,
                        const QString& savePath,
                        std::function<void(bool)> callback)
    {
        QJsonObject params;
        for (auto it = filters.begin(); it != filters.end(); ++it) {
            params[it.key()] = QJsonValue::fromVariant(it.value());
        }
        
        m_http->post(
            QString("/api/reports/%1/pdf").arg(reportType), params,
            [savePath, callback](const HttpClient::Response& r) {
                if (r.isOk()) {
                    QFile f(savePath);
                    if (f.open(QIODevice::WriteOnly)) {
                        f.write(r.body);
                        callback(true);
                    } else {
                        callback(false);
                    }
                } else {
                    callback(false);
                }
            },
            {.contentType = "application/json"}
        );
    }
    
private:
    HttpClient* m_http;
};
```

---

## ขั้นตอนที่ 854: WebSocket Client

```cpp
// network/websocketclient.h
#pragma once
#include <QObject>
#include <QWebSocket>
#include <QTimer>
#include <QJsonDocument>
#include <QJsonObject>

class WebSocketClient : public QObject {
    Q_OBJECT
    
    Q_PROPERTY(bool connected READ isConnected NOTIFY connectionChanged)
    
public:
    enum class State { Disconnected, Connecting, Connected, Reconnecting };
    Q_ENUM(State)
    
    struct Message {
        QString type;
        QJsonObject data;
        QDateTime timestamp;
    };
    
    explicit WebSocketClient(QObject* parent = nullptr)
        : QObject(parent)
        , m_socket(new QWebSocket(QString(), QWebSocketProtocol::VersionLatest, this))
        , m_pingTimer(new QTimer(this))
        , m_reconnectTimer(new QTimer(this))
    {
        connect(m_socket, &QWebSocket::connected, this, &WebSocketClient::onConnected);
        connect(m_socket, &QWebSocket::disconnected, this, &WebSocketClient::onDisconnected);
        connect(m_socket, &QWebSocket::textMessageReceived, this, &WebSocketClient::onTextMessage);
        connect(m_socket, &QWebSocket::errorOccurred, this, [=](QAbstractSocket::SocketError err) {
            emit error(m_socket->errorString());
        });
        
        m_pingTimer->setInterval(30000);
        connect(m_pingTimer, &QTimer::timeout, this, &WebSocketClient::sendPing);
        
        m_reconnectTimer->setSingleShot(true);
        connect(m_reconnectTimer, &QTimer::timeout, this, [=] {
            if (!isConnected()) {
                qDebug() << "WebSocket: attempting reconnect...";
                m_socket->open(m_url);
            }
        });
    }
    
    bool isConnected() const {
        return m_socket->state() == QAbstractSocket::ConnectedState;
    }
    
    void connectToServer(const QUrl& url, const QString& authToken = {}) {
        m_url = url;
        m_authToken = authToken;
        m_reconnectDelay = 1000;
        m_shouldReconnect = true;
        
        QNetworkRequest req(url);
        if (!authToken.isEmpty()) {
            req.setRawHeader("Authorization", ("Bearer " + authToken).toUtf8());
        }
        
        m_socket->open(req);
        setState(State::Connecting);
    }
    
    void disconnect() {
        m_shouldReconnect = false;
        m_reconnectTimer->stop();
        m_pingTimer->stop();
        m_socket->close();
    }
    
    void send(const QString& type, const QJsonObject& data = {}) {
        if (!isConnected()) {
            qWarning() << "WebSocket: not connected, cannot send" << type;
            return;
        }
        
        QJsonObject msg;
        msg["type"] = type;
        msg["data"] = data;
        msg["timestamp"] = QDateTime::currentMSecsSinceEpoch();
        
        m_socket->sendTextMessage(
            QJsonDocument(msg).toJson(QJsonDocument::Compact));
    }
    
    void subscribe(const QString& channel) {
        send("subscribe", {{"channel", channel}});
        m_subscriptions.insert(channel);
    }
    
    void unsubscribe(const QString& channel) {
        send("unsubscribe", {{"channel", channel}});
        m_subscriptions.remove(channel);
    }
    
signals:
    void connectionChanged();
    void messageReceived(const Message& msg);
    void error(const QString& error);
    void stateChanged(State state);
    
private slots:
    void onConnected() {
        qDebug() << "WebSocket: connected to" << m_url.toString();
        m_reconnectDelay = 1000;
        setState(State::Connected);
        m_pingTimer->start();
        
        // Re-subscribe after reconnect
        for (const auto& ch : m_subscriptions) {
            send("subscribe", {{"channel", ch}});
        }
        
        emit connectionChanged();
    }
    
    void onDisconnected() {
        qDebug() << "WebSocket: disconnected";
        m_pingTimer->stop();
        setState(State::Disconnected);
        emit connectionChanged();
        
        if (m_shouldReconnect) {
            scheduleReconnect();
        }
    }
    
    void onTextMessage(const QString& text) {
        QJsonObject obj = QJsonDocument::fromJson(text.toUtf8()).object();
        
        if (obj["type"].toString() == "pong") return;
        
        Message msg;
        msg.type = obj["type"].toString();
        msg.data = obj["data"].toObject();
        msg.timestamp = QDateTime::fromMSecsSinceEpoch(obj["timestamp"].toInteger());
        
        emit messageReceived(msg);
    }
    
    void sendPing() {
        if (isConnected()) {
            send("ping");
        }
    }
    
private:
    void setState(State s) {
        m_state = s;
        emit stateChanged(s);
    }
    
    void scheduleReconnect() {
        setState(State::Reconnecting);
        qDebug() << "WebSocket: reconnect in" << m_reconnectDelay << "ms";
        m_reconnectTimer->start(m_reconnectDelay);
        m_reconnectDelay = qMin(m_reconnectDelay * 2, 30000);
    }
    
    QWebSocket* m_socket;
    QTimer* m_pingTimer;
    QTimer* m_reconnectTimer;
    QUrl m_url;
    QString m_authToken;
    State m_state = State::Disconnected;
    bool m_shouldReconnect = false;
    int m_reconnectDelay = 1000;
    QSet<QString> m_subscriptions;
};

// Real-time notification handler
class RealtimeNotifications : public QObject {
    Q_OBJECT
    
public:
    explicit RealtimeNotifications(WebSocketClient* ws, QObject* parent = nullptr)
        : QObject(parent), m_ws(ws)
    {
        connect(ws, &WebSocketClient::messageReceived,
                this, &RealtimeNotifications::handleMessage);
        
        ws->subscribe("notifications");
        ws->subscribe("orders");
        ws->subscribe("inventory");
    }
    
signals:
    void newOrderReceived(int orderId, const QString& customerName);
    void lowStockAlert(const QString& productName, double qty);
    void paymentReceived(int orderId, double amount);
    void userLoggedIn(const QString& username);
    
private slots:
    void handleMessage(const WebSocketClient::Message& msg) {
        if (msg.type == "new_order") {
            emit newOrderReceived(
                msg.data["order_id"].toInt(),
                msg.data["customer_name"].toString()
            );
        }
        else if (msg.type == "low_stock") {
            emit lowStockAlert(
                msg.data["product_name"].toString(),
                msg.data["qty"].toDouble()
            );
        }
        else if (msg.type == "payment") {
            emit paymentReceived(
                msg.data["order_id"].toInt(),
                msg.data["amount"].toDouble()
            );
        }
        else if (msg.type == "user_login") {
            emit userLoggedIn(msg.data["username"].toString());
        }
    }
    
    WebSocketClient* m_ws;
};
```

---

## สรุป Part 059

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Generic HttpClient ด้วย GET/POST/PUT/PATCH/DELETE + timeout + multipart upload
2. ✅ ErpApiClient service layer ที่ครอบ HTTP client
3. ✅ WebSocketClient ด้วย auto-reconnect (exponential backoff) + ping/pong + subscriptions
4. ✅ RealtimeNotifications handler สำหรับ live updates
5. ✅ Bearer token authentication + custom headers

---

⬅️ [Part 058](part058.md) | ➡️ [Part 060: Capstone Final — Complete ERP Integration](part060.md)
