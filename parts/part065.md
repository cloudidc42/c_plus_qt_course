# Part 065: Qt for Embedded & IoT

## ขั้นตอนที่ 941-955

---

## ขั้นตอนที่ 941: Qt on Embedded Linux (Raspberry Pi)

```cmake
# Cross-compilation สำหรับ Raspberry Pi 4
# toolchain/rpi4-toolchain.cmake

set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(SYSROOT /opt/sysroot-rpi4)
set(CROSS_COMPILE /opt/aarch64-linux-gnu/bin/aarch64-linux-gnu-)

set(CMAKE_C_COMPILER   ${CROSS_COMPILE}gcc)
set(CMAKE_CXX_COMPILER ${CROSS_COMPILE}g++)

set(CMAKE_SYSROOT ${SYSROOT})
set(CMAKE_FIND_ROOT_PATH ${SYSROOT})
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

# Qt6 on RPi uses EGLFS (no X11)
set(QT_HOST_PATH /opt/Qt6-host)
set(Qt6_DIR ${SYSROOT}/usr/lib/aarch64-linux-gnu/cmake/Qt6)
```

```bash
# Build command
cmake -B build-rpi4 \
    -DCMAKE_TOOLCHAIN_FILE=toolchain/rpi4-toolchain.cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -DQT_HOST_PATH=/opt/Qt6-host

cmake --build build-rpi4 --parallel

# Deploy to RPi4
rsync -avz build-rpi4/ErpPro pi@192.168.1.100:/home/pi/erp/

# Run on RPi4 with EGLFS
ssh pi@192.168.1.100 \
    "QT_QPA_PLATFORM=eglfs ./erp/ErpPro"
```

---

## ขั้นตอนที่ 942: GPIO Control ผ่าน QProcess/pigpio

```cpp
// iot/gpiocontroller.h
#pragma once
#include <QObject>
#include <QSocketNotifier>
#include <fcntl.h>
#include <unistd.h>

class GpioController : public QObject {
    Q_OBJECT
    
public:
    enum class PinMode { Input, Output, InputPullUp, InputPullDown };
    enum class EdgeType { Rising, Falling, Both };
    
    explicit GpioController(int gpioPin, QObject* parent = nullptr)
        : QObject(parent), m_pin(gpioPin)
    {}
    
    bool setup(PinMode mode) {
        // Export GPIO via sysfs
        QFile exportFile("/sys/class/gpio/export");
        if (!exportFile.open(QFile::WriteOnly)) return false;
        exportFile.write(QByteArray::number(m_pin));
        exportFile.close();
        
        QThread::msleep(100);  // Wait for sysfs to create files
        
        // Set direction
        QString dirPath = QString("/sys/class/gpio/gpio%1/direction").arg(m_pin);
        QFile dirFile(dirPath);
        if (!dirFile.open(QFile::WriteOnly)) return false;
        
        switch (mode) {
        case PinMode::Output:
            dirFile.write("out");
            break;
        case PinMode::Input:
        case PinMode::InputPullUp:
        case PinMode::InputPullDown:
            dirFile.write("in");
            break;
        }
        
        m_mode = mode;
        return true;
    }
    
    void write(bool high) {
        Q_ASSERT(m_mode == PinMode::Output);
        
        QString valuePath = QString("/sys/class/gpio/gpio%1/value").arg(m_pin);
        QFile f(valuePath);
        if (f.open(QFile::WriteOnly)) {
            f.write(high ? "1" : "0");
        }
    }
    
    bool read() const {
        QString valuePath = QString("/sys/class/gpio/gpio%1/value").arg(m_pin);
        QFile f(valuePath);
        if (f.open(QFile::ReadOnly)) {
            return f.readAll().trimmed() == "1";
        }
        return false;
    }
    
    void enableInterrupt(EdgeType edge) {
        // Set edge trigger
        QString edgePath = QString("/sys/class/gpio/gpio%1/edge").arg(m_pin);
        QFile f(edgePath);
        if (f.open(QFile::WriteOnly)) {
            switch (edge) {
            case EdgeType::Rising:  f.write("rising");  break;
            case EdgeType::Falling: f.write("falling"); break;
            case EdgeType::Both:    f.write("both");    break;
            }
        }
        
        // Open value file for polling
        QString valuePath = QString("/sys/class/gpio/gpio%1/value").arg(m_pin);
        m_fd = ::open(valuePath.toLatin1(), O_RDONLY | O_NONBLOCK);
        if (m_fd < 0) return;
        
        // Use QSocketNotifier for async notification
        m_notifier = new QSocketNotifier(m_fd, QSocketNotifier::Exception, this);
        connect(m_notifier, &QSocketNotifier::activated, this, [this]() {
            char buf[2];
            ::lseek(m_fd, 0, SEEK_SET);
            ::read(m_fd, buf, sizeof(buf));
            bool high = (buf[0] == '1');
            emit stateChanged(high);
        });
    }
    
    ~GpioController() {
        if (m_fd >= 0) ::close(m_fd);
        
        // Unexport
        QFile unexportFile("/sys/class/gpio/unexport");
        if (unexportFile.open(QFile::WriteOnly)) {
            unexportFile.write(QByteArray::number(m_pin));
        }
    }
    
signals:
    void stateChanged(bool high);
    
private:
    int m_pin;
    int m_fd{-1};
    PinMode m_mode{PinMode::Input};
    QSocketNotifier* m_notifier{nullptr};
};
```

---

## ขั้นตอนที่ 943: Serial Port Communication

```cpp
// iot/serialmonitor.h
#pragma once
#include <QObject>
#include <QSerialPort>
#include <QSerialPortInfo>
#include <QTimer>
#include <QQueue>

class SerialMonitor : public QObject {
    Q_OBJECT
    Q_PROPERTY(bool connected READ isConnected NOTIFY connectionChanged)
    
public:
    explicit SerialMonitor(QObject* parent = nullptr)
        : QObject(parent)
    {
        m_port = new QSerialPort(this);
        
        connect(m_port, &QSerialPort::readyRead,
                this, &SerialMonitor::onDataReceived);
        connect(m_port, &QSerialPort::errorOccurred,
                this, &SerialMonitor::onError);
        
        // Heartbeat timer
        m_heartbeatTimer = new QTimer(this);
        m_heartbeatTimer->setInterval(5000);
        connect(m_heartbeatTimer, &QTimer::timeout, this, [this]() {
            sendCommand("PING");
        });
    }
    
    static QStringList availablePorts() {
        QStringList ports;
        for (const auto& info : QSerialPortInfo::availablePorts()) {
            ports << info.portName();
        }
        return ports;
    }
    
    bool connect(const QString& portName,
                 qint32 baudRate = QSerialPort::Baud115200)
    {
        m_port->setPortName(portName);
        m_port->setBaudRate(baudRate);
        m_port->setDataBits(QSerialPort::Data8);
        m_port->setParity(QSerialPort::NoParity);
        m_port->setStopBits(QSerialPort::OneStop);
        m_port->setFlowControl(QSerialPort::NoFlowControl);
        
        if (!m_port->open(QIODevice::ReadWrite)) {
            emit error(tr("ไม่สามารถเปิด port %1: %2")
                      .arg(portName, m_port->errorString()));
            return false;
        }
        
        m_heartbeatTimer->start();
        emit connectionChanged(true);
        return true;
    }
    
    void disconnect() {
        m_heartbeatTimer->stop();
        if (m_port->isOpen()) m_port->close();
        emit connectionChanged(false);
    }
    
    bool isConnected() const { return m_port->isOpen(); }
    
    void sendCommand(const QString& cmd) {
        if (!m_port->isOpen()) return;
        QByteArray data = (cmd + "\r\n").toUtf8();
        m_port->write(data);
    }
    
    // Enqueue multiple commands
    void enqueueCommand(const QString& cmd) {
        m_commandQueue.enqueue(cmd);
        if (!m_busy) processQueue();
    }
    
signals:
    void connectionChanged(bool connected);
    void dataReceived(const QString& line);
    void sensorData(const QVariantMap& data);
    void error(const QString& message);
    
private slots:
    void onDataReceived() {
        m_buffer.append(m_port->readAll());
        
        // Process complete lines
        while (m_buffer.contains('\n')) {
            int idx = m_buffer.indexOf('\n');
            QByteArray line = m_buffer.left(idx).trimmed();
            m_buffer.remove(0, idx + 1);
            
            if (!line.isEmpty()) {
                processLine(QString::fromUtf8(line));
            }
        }
    }
    
    void processLine(const QString& line) {
        emit dataReceived(line);
        
        // Parse JSON sensor data: {"temp":25.5,"humidity":60.2,"pressure":1013.25}
        if (line.startsWith('{')) {
            QJsonDocument doc = QJsonDocument::fromJson(line.toUtf8());
            if (!doc.isNull()) {
                emit sensorData(doc.object().toVariantMap());
            }
        }
        
        // Handle ACK/NACK
        if (line == "ACK") {
            m_busy = false;
            processQueue();
        }
    }
    
    void processQueue() {
        if (m_commandQueue.isEmpty()) {
            m_busy = false;
            return;
        }
        m_busy = true;
        sendCommand(m_commandQueue.dequeue());
    }
    
    void onError(QSerialPort::SerialPortError err) {
        if (err != QSerialPort::NoError) {
            emit error(m_port->errorString());
            disconnect();
        }
    }
    
private:
    QSerialPort* m_port;
    QTimer* m_heartbeatTimer;
    QByteArray m_buffer;
    QQueue<QString> m_commandQueue;
    bool m_busy{false};
};
```

---

## ขั้นตอนที่ 944: IoT Dashboard (Sensor Monitoring)

```cpp
// iot/sensordashboard.h
#pragma once
#include <QWidget>
#include <QtCharts>
#include <QDateTime>

class SensorDashboard : public QWidget {
    Q_OBJECT
    
public:
    explicit SensorDashboard(QWidget* parent = nullptr) : QWidget(parent) {
        setupUi();
    }
    
    void updateSensorData(const QVariantMap& data) {
        QDateTime now = QDateTime::currentDateTime();
        
        if (data.contains("temp")) {
            double temp = data["temp"].toDouble();
            addPoint(m_tempSeries, now, temp);
            m_tempGauge->setValue(temp);
            m_tempLabel->setText(QString("%1°C").arg(temp, 0, 'f', 1));
        }
        
        if (data.contains("humidity")) {
            double hum = data["humidity"].toDouble();
            addPoint(m_humSeries, now, hum);
            m_humLabel->setText(QString("%1%").arg(hum, 0, 'f', 1));
        }
        
        if (data.contains("pressure")) {
            double pres = data["pressure"].toDouble();
            m_presLabel->setText(QString("%1 hPa").arg(pres, 0, 'f', 1));
        }
    }
    
private:
    void setupUi() {
        auto* mainLayout = new QVBoxLayout(this);
        
        // Current values row
        auto* valuesRow = new QHBoxLayout;
        
        m_tempLabel = makeValueCard("อุณหภูมิ", "°C", "#E74C3C");
        m_humLabel  = makeValueCard("ความชื้น", "%",  "#3498DB");
        m_presLabel = makeValueCard("ความดัน", "hPa", "#27AE60");
        
        valuesRow->addWidget(m_tempLabel->parentWidget());
        valuesRow->addWidget(m_humLabel->parentWidget());
        valuesRow->addWidget(m_presLabel->parentWidget());
        
        // Real-time chart
        m_chart = new QChart;
        m_chart->setTitle("ข้อมูลเซ็นเซอร์ (Real-time)");
        m_chart->setAnimationOptions(QChart::NoAnimation);
        m_chart->legend()->setVisible(true);
        
        m_tempSeries = new QLineSeries;
        m_tempSeries->setName("อุณหภูมิ (°C)");
        m_tempSeries->setColor(QColor("#E74C3C"));
        
        m_humSeries = new QLineSeries;
        m_humSeries->setName("ความชื้น (%)");
        m_humSeries->setColor(QColor("#3498DB"));
        
        m_chart->addSeries(m_tempSeries);
        m_chart->addSeries(m_humSeries);
        
        // X axis: DateTime
        auto* axisX = new QDateTimeAxis;
        axisX->setFormat("HH:mm:ss");
        axisX->setTitleText("เวลา");
        axisX->setTickCount(6);
        m_chart->addAxis(axisX, Qt::AlignBottom);
        m_tempSeries->attachAxis(axisX);
        m_humSeries->attachAxis(axisX);
        
        // Y axis
        auto* axisY = new QValueAxis;
        axisY->setRange(0, 100);
        axisY->setTitleText("ค่า");
        m_chart->addAxis(axisY, Qt::AlignLeft);
        m_tempSeries->attachAxis(axisY);
        m_humSeries->attachAxis(axisY);
        
        auto* chartView = new QChartView(m_chart);
        chartView->setRenderHint(QPainter::Antialiasing);
        chartView->setMinimumHeight(300);
        
        mainLayout->addLayout(valuesRow);
        mainLayout->addWidget(chartView);
        
        // Alarm threshold
        auto* alarmGroup = new QGroupBox(tr("การแจ้งเตือน"));
        auto* alarmLayout = new QFormLayout(alarmGroup);
        
        auto* tempAlarm = new QDoubleSpinBox;
        tempAlarm->setRange(0, 100);
        tempAlarm->setValue(35.0);
        tempAlarm->setSuffix(" °C");
        alarmLayout->addRow(tr("อุณหภูมิเกิน:"), tempAlarm);
        
        auto* humAlarm = new QDoubleSpinBox;
        humAlarm->setRange(0, 100);
        humAlarm->setValue(80.0);
        humAlarm->setSuffix(" %");
        alarmLayout->addRow(tr("ความชื้นเกิน:"), humAlarm);
        
        mainLayout->addWidget(alarmGroup);
        
        connect(tempAlarm, &QDoubleSpinBox::valueChanged, this, [this](double val) {
            m_tempAlarmThreshold = val;
        });
    }
    
    QLabel* makeValueCard(const QString& title, const QString& unit,
                           const QString& color)
    {
        auto* card = new QFrame(this);
        card->setFrameShape(QFrame::StyledPanel);
        card->setStyleSheet(QString("border-left: 4px solid %1;").arg(color));
        
        auto* vl = new QVBoxLayout(card);
        auto* titleLbl = new QLabel(title, card);
        titleLbl->setStyleSheet("color: #7F8C8D; font-size: 11px;");
        
        auto* valueLbl = new QLabel("--" + unit, card);
        valueLbl->setStyleSheet(
            QString("color: %1; font-size: 28px; font-weight: bold;").arg(color));
        
        vl->addWidget(titleLbl);
        vl->addWidget(valueLbl);
        
        return valueLbl;
    }
    
    void addPoint(QLineSeries* series, const QDateTime& dt, double value) {
        series->append(dt.toMSecsSinceEpoch(), value);
        
        // Keep last 5 minutes
        qint64 cutoff = QDateTime::currentMSecsSinceEpoch() - 300000;
        while (series->count() > 0 &&
               series->at(0).x() < cutoff) {
            series->remove(0);
        }
        
        // Update X axis range
        if (auto* axis = qobject_cast<QDateTimeAxis*>(
                m_chart->axes(Qt::Horizontal).first())) {
            axis->setRange(QDateTime::fromMSecsSinceEpoch(
                               series->at(0).x()),
                           QDateTime::currentDateTime());
        }
        
        // Check alarm
        if (series == m_tempSeries && value > m_tempAlarmThreshold) {
            emit temperatureAlarm(value);
        }
    }
    
signals:
    void temperatureAlarm(double temp);
    
private:
    QLabel* m_tempLabel{};
    QLabel* m_humLabel{};
    QLabel* m_presLabel{};
    QChart* m_chart{};
    QLineSeries* m_tempSeries{};
    QLineSeries* m_humSeries{};
    QProgressBar* m_tempGauge{};
    double m_tempAlarmThreshold{35.0};
};
```

---

## ขั้นตอนที่ 945: MQTT Integration

```cpp
// iot/mqttclient.h — Qt MQTT
#pragma once
#include <QObject>
#include <QtMqtt/QtMqtt>

class MqttClient : public QObject {
    Q_OBJECT
    Q_PROPERTY(bool connected READ isConnected NOTIFY connectionStateChanged)
    
public:
    explicit MqttClient(QObject* parent = nullptr) : QObject(parent) {
        m_client = new QMqttClient(this);
        
        connect(m_client, &QMqttClient::connected, this, [this]() {
            qDebug() << "MQTT Connected";
            emit connectionStateChanged(true);
            
            // Re-subscribe after reconnect
            for (const auto& topic : m_subscriptions.keys()) {
                m_client->subscribe(QMqttTopicFilter(topic),
                                   m_subscriptions[topic]);
            }
        });
        
        connect(m_client, &QMqttClient::disconnected, this, [this]() {
            emit connectionStateChanged(false);
            scheduleReconnect();
        });
        
        connect(m_client, &QMqttClient::messageReceived,
                this, [this](const QByteArray& message,
                             const QMqttTopicName& topic)
        {
            emit messageReceived(topic.name(), message);
            
            // Parse JSON
            QJsonDocument doc = QJsonDocument::fromJson(message);
            if (!doc.isNull()) {
                emit jsonReceived(topic.name(), doc.object());
            }
        });
        
        m_reconnectTimer = new QTimer(this);
        m_reconnectTimer->setSingleShot(true);
        connect(m_reconnectTimer, &QTimer::timeout,
                this, &MqttClient::connectToBroker);
    }
    
    void setup(const QString& host, quint16 port = 1883,
               const QString& clientId = QString())
    {
        m_client->setHostname(host);
        m_client->setPort(port);
        m_client->setClientId(clientId.isEmpty()
            ? "qt-client-" + QUuid::createUuid().toString(QUuid::WithoutBraces).left(8)
            : clientId);
    }
    
    void setCredentials(const QString& user, const QString& password) {
        m_client->setUsername(user);
        m_client->setPassword(password.toUtf8());
    }
    
    void connectToBroker() {
        m_client->connectToHost();
    }
    
    bool isConnected() const {
        return m_client->state() == QMqttClient::Connected;
    }
    
    void subscribe(const QString& topic, quint8 qos = 0) {
        m_subscriptions[topic] = qos;
        if (isConnected()) {
            m_client->subscribe(QMqttTopicFilter(topic), qos);
        }
    }
    
    void publish(const QString& topic, const QByteArray& payload,
                 quint8 qos = 0, bool retain = false)
    {
        if (!isConnected()) return;
        m_client->publish(QMqttTopicName(topic), payload, qos, retain);
    }
    
    void publishJson(const QString& topic, const QJsonObject& obj,
                     quint8 qos = 0)
    {
        publish(topic, QJsonDocument(obj).toJson(QJsonDocument::Compact), qos);
    }
    
signals:
    void connectionStateChanged(bool connected);
    void messageReceived(const QString& topic, const QByteArray& payload);
    void jsonReceived(const QString& topic, const QJsonObject& data);
    
private:
    void scheduleReconnect() {
        m_reconnectDelay = qMin(m_reconnectDelay * 2, 60000);
        m_reconnectTimer->start(m_reconnectDelay);
    }
    
    QMqttClient* m_client;
    QTimer* m_reconnectTimer;
    QMap<QString, quint8> m_subscriptions;
    int m_reconnectDelay{1000};
};

// Demo: IoT Hub ที่รับข้อมูลจาก MQTT และแสดงใน dashboard
class IoTHub : public QObject {
    Q_OBJECT
    
public:
    IoTHub(QObject* parent = nullptr) : QObject(parent) {
        m_mqtt = new MqttClient(this);
        m_mqtt->setup("mqtt.example.com");
        m_mqtt->subscribe("sensors/+/temperature");
        m_mqtt->subscribe("sensors/+/humidity");
        m_mqtt->subscribe("alerts/#");
        
        connect(m_mqtt, &MqttClient::jsonReceived,
                this, &IoTHub::onMqttMessage);
        
        m_mqtt->connectToBroker();
    }
    
private slots:
    void onMqttMessage(const QString& topic, const QJsonObject& data) {
        // Parse topic: sensors/{deviceId}/{metric}
        QStringList parts = topic.split('/');
        if (parts.size() >= 3 && parts[0] == "sensors") {
            QString deviceId = parts[1];
            QString metric   = parts[2];
            double value = data["value"].toDouble();
            
            emit sensorUpdate(deviceId, metric, value);
        }
    }
    
signals:
    void sensorUpdate(const QString& deviceId,
                      const QString& metric, double value);
    
private:
    MqttClient* m_mqtt;
};
```

---

## ขั้นตอนที่ 946: Touch-Optimized UI for Embedded

```cpp
// ui/touchui.h — UI สำหรับ touch screen
#pragma once
#include <QWidget>
#include <QGestureEvent>

// Touch-friendly button ขนาดใหญ่
class TouchButton : public QAbstractButton {
    Q_OBJECT
    Q_PROPERTY(QColor color READ color WRITE setColor NOTIFY colorChanged)
    
public:
    explicit TouchButton(const QString& text, QWidget* parent = nullptr)
        : QAbstractButton(parent)
        , m_color("#3498DB")
    {
        setText(text);
        setMinimumSize(100, 60);  // Minimum touch target: 44pt iOS / 48dp Android
        
        setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Preferred);
    }
    
    QColor color() const { return m_color; }
    void setColor(const QColor& c) {
        if (m_color != c) {
            m_color = c;
            update();
            emit colorChanged(c);
        }
    }
    
signals:
    void colorChanged(const QColor&);
    
protected:
    void paintEvent(QPaintEvent*) override {
        QPainter p(this);
        p.setRenderHint(QPainter::Antialiasing);
        
        QColor bg = isDown() ? m_color.darker(130) :
                    underMouse() ? m_color.lighter(110) : m_color;
        
        p.setBrush(bg);
        p.setPen(Qt::NoPen);
        p.drawRoundedRect(rect(), 8, 8);
        
        p.setPen(Qt::white);
        p.setFont(QFont(font().family(), 14, QFont::Medium));
        p.drawText(rect(), Qt::AlignCenter, text());
    }
    
    QSize sizeHint() const override {
        return QSize(120, 60);
    }
};

// NumPad สำหรับ touch input
class TouchNumPad : public QWidget {
    Q_OBJECT
    
public:
    explicit TouchNumPad(QWidget* parent = nullptr) : QWidget(parent) {
        auto* grid = new QGridLayout(this);
        grid->setSpacing(8);
        
        const QStringList keys = {
            "7", "8", "9",
            "4", "5", "6",
            "1", "2", "3",
            ".", "0", "⌫"
        };
        
        int row = 0, col = 0;
        for (const auto& key : keys) {
            auto* btn = new TouchButton(key, this);
            if (key == "⌫") btn->setColor("#E74C3C");
            else if (key == ".") btn->setColor("#95A5A6");
            
            connect(btn, &QAbstractButton::clicked, this, [this, key]() {
                if (key == "⌫") {
                    if (!m_value.isEmpty()) m_value.chop(1);
                } else if (key == "." && m_value.contains('.')) {
                    return;
                } else {
                    m_value += key;
                }
                emit valueChanged(m_value.toDouble());
                emit inputChanged(m_value);
            });
            
            grid->addWidget(btn, row, col);
            col++;
            if (col == 3) { col = 0; row++; }
        }
    }
    
    void clear() {
        m_value.clear();
        emit inputChanged(m_value);
    }
    
    double value() const { return m_value.toDouble(); }
    
signals:
    void valueChanged(double value);
    void inputChanged(const QString& text);
    
private:
    QString m_value;
};
```

---

## สรุป Part 065

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Cross-compilation สำหรับ Raspberry Pi 4 ด้วย CMake toolchain
2. ✅ GPIO control ผ่าน sysfs + QSocketNotifier สำหรับ interrupt
3. ✅ QSerialPort: full-duplex serial communication + command queue
4. ✅ IoT Sensor Dashboard ด้วย QChart (real-time) + alarm thresholds
5. ✅ MQTT integration ด้วย QMqttClient + auto-reconnect + JSON parsing
6. ✅ Touch-optimized UI: TouchButton, TouchNumPad สำหรับ embedded touchscreen

---

⬅️ [Part 064](part064.md) | ➡️ [Part 066: World-Class Final Review](part066.md)
