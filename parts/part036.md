# Part 036: Qt Bluetooth & Serial Port

## ขั้นตอนที่ 506-520

---

## ขั้นตอนที่ 506: Qt Serial Port

```
Qt Serial Port:
  - สื่อสารกับ Arduino, sensors, PLCs, embedded systems
  
CMakeLists.txt:
  find_package(Qt6 REQUIRED COMPONENTS SerialPort)
  target_link_libraries(app PRIVATE Qt6::SerialPort)

Linux permissions:
  sudo usermod -a -G dialout $USER
  # หรือ
  sudo chmod 666 /dev/ttyUSB0
```

---

## ขั้นตอนที่ 507: Serial Port Basic

```cpp
#include <QSerialPort>
#include <QSerialPortInfo>

// === List Available Ports ===
void listSerialPorts() {
    for (const QSerialPortInfo& info : QSerialPortInfo::availablePorts()) {
        qDebug() << "Port:" << info.portName();
        qDebug() << "  Description:" << info.description();
        qDebug() << "  Manufacturer:" << info.manufacturer();
        qDebug() << "  VID:PID:" << info.vendorIdentifier() << ":" << info.productIdentifier();
        qDebug() << "  System Location:" << info.systemLocation();
    }
}

// === SerialManager class ===
class SerialManager : public QObject {
    Q_OBJECT
    Q_PROPERTY(bool connected READ isConnected NOTIFY connectionChanged)
    
public:
    explicit SerialManager(QObject* parent = nullptr) : QObject(parent) {
        serial = new QSerialPort(this);
        
        connect(serial, &QSerialPort::readyRead, this, &SerialManager::onDataReceived);
        connect(serial, &QSerialPort::errorOccurred, this, &SerialManager::onError);
    }
    
    bool open(const QString& portName, int baudRate = QSerialPort::Baud9600) {
        if (serial->isOpen()) serial->close();
        
        serial->setPortName(portName);
        serial->setBaudRate(baudRate);
        serial->setDataBits(QSerialPort::Data8);
        serial->setParity(QSerialPort::NoParity);
        serial->setStopBits(QSerialPort::OneStop);
        serial->setFlowControl(QSerialPort::NoFlowControl);
        
        if (!serial->open(QIODevice::ReadWrite)) {
            qWarning() << "Cannot open port:" << serial->errorString();
            return false;
        }
        
        qDebug() << "Port opened:" << portName << "@" << baudRate;
        emit connectionChanged(true);
        return true;
    }
    
    void close() {
        if (serial->isOpen()) {
            serial->close();
            emit connectionChanged(false);
        }
    }
    
    bool isConnected() const { return serial->isOpen(); }
    
    bool writeData(const QByteArray& data) {
        if (!serial->isOpen()) return false;
        
        qint64 written = serial->write(data);
        serial->flush();
        return written == data.size();
    }
    
    bool writeText(const QString& text) {
        return writeData(text.toUtf8());
    }
    
    bool writeLine(const QString& text) {
        return writeData(text.toUtf8() + "\n");
    }
    
    QStringList availablePorts() const {
        QStringList ports;
        for (const QSerialPortInfo& info : QSerialPortInfo::availablePorts()) {
            ports << info.portName();
        }
        return ports;
    }
    
signals:
    void connectionChanged(bool connected);
    void dataReceived(const QByteArray& data);
    void lineReceived(const QString& line);
    void errorOccurred(const QString& msg);
    
private slots:
    void onDataReceived() {
        QByteArray data = serial->readAll();
        m_buffer.append(data);
        emit dataReceived(data);
        
        // Parse lines
        while (m_buffer.contains('\n')) {
            int idx = m_buffer.indexOf('\n');
            QString line = QString::fromUtf8(m_buffer.left(idx)).trimmed();
            m_buffer.remove(0, idx + 1);
            
            if (!line.isEmpty()) {
                emit lineReceived(line);
            }
        }
    }
    
    void onError(QSerialPort::SerialPortError error) {
        if (error != QSerialPort::NoError) {
            qWarning() << "Serial error:" << error << serial->errorString();
            emit errorOccurred(serial->errorString());
        }
    }
    
private:
    QSerialPort* serial;
    QByteArray m_buffer;
};
```

---

## ขั้นตอนที่ 508: Arduino Monitor

```cpp
class ArduinoMonitor : public QMainWindow {
    Q_OBJECT
    
public:
    ArduinoMonitor(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Arduino Monitor");
        setMinimumSize(700, 500);
        
        serial = new SerialManager(this);
        
        setupUi();
        
        connect(serial, &SerialManager::lineReceived, this, &ArduinoMonitor::onLine);
        connect(serial, &SerialManager::connectionChanged, this, &ArduinoMonitor::onConnectionChanged);
        connect(serial, &SerialManager::errorOccurred, [this](const QString& msg) {
            logMessage("ERROR: " + msg);
        });
        
        // Refresh port list periodically
        auto* refreshTimer = new QTimer(this);
        connect(refreshTimer, &QTimer::timeout, this, &ArduinoMonitor::refreshPorts);
        refreshTimer->start(2000);
        refreshPorts();
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* layout = new QVBoxLayout(central);
        setCentralWidget(central);
        
        // Connection bar
        auto* connBar = new QHBoxLayout();
        
        portCombo = new QComboBox();
        portCombo->setMinimumWidth(150);
        
        baudCombo = new QComboBox();
        baudCombo->addItems({"9600", "19200", "38400", "57600", "115200"});
        baudCombo->setCurrentText("9600");
        
        connectBtn = new QPushButton("Connect");
        connectBtn->setStyleSheet("background: #27ae60; color: white; padding: 6px 16px;");
        
        connBar->addWidget(new QLabel("Port:"));
        connBar->addWidget(portCombo);
        connBar->addWidget(new QLabel("Baud:"));
        connBar->addWidget(baudCombo);
        connBar->addWidget(connectBtn);
        connBar->addStretch();
        
        statusLed = new QLabel("●");
        statusLed->setStyleSheet("color: #e74c3c; font-size: 18px;");
        connBar->addWidget(statusLed);
        
        // Output
        outputView = new QTextEdit();
        outputView->setReadOnly(true);
        outputView->setFont(QFont("Monospace", 10));
        outputView->setStyleSheet("background: #0d1117; color: #58a6ff;");
        
        // Input
        auto* inputBar = new QHBoxLayout();
        inputEdit = new QLineEdit();
        inputEdit->setPlaceholderText("Send command...");
        sendBtn = new QPushButton("Send");
        sendBtn->setEnabled(false);
        
        auto* clearBtn = new QPushButton("Clear");
        
        inputBar->addWidget(inputEdit, 1);
        inputBar->addWidget(sendBtn);
        inputBar->addWidget(clearBtn);
        
        // Chart area
        chartView = new QChartView();
        chartView->setMinimumHeight(150);
        chartView->setRenderHint(QPainter::Antialiasing);
        
        chart = new QChart();
        chart->legend()->hide();
        chart->setBackgroundBrush(QBrush(Qt::black));
        chart->setPlotAreaBackgroundBrush(QBrush(QColor("#0d1117")));
        
        lineSeries = new QLineSeries();
        lineSeries->setColor(QColor("#27ae60"));
        chart->addSeries(lineSeries);
        
        auto* axisX = new QValueAxis();
        axisX->setRange(0, 100);
        axisX->setLabelsBrush(QBrush(Qt::white));
        axisX->setGridLineColor(QColor("#333"));
        chart->addAxis(axisX, Qt::AlignBottom);
        lineSeries->attachAxis(axisX);
        
        auto* axisY = new QValueAxis();
        axisY->setRange(0, 1023);
        axisY->setLabelsBrush(QBrush(Qt::white));
        axisY->setGridLineColor(QColor("#333"));
        chart->addAxis(axisY, Qt::AlignLeft);
        lineSeries->attachAxis(axisY);
        
        chartView->setChart(chart);
        
        layout->addLayout(connBar);
        layout->addWidget(outputView, 2);
        layout->addWidget(chartView, 1);
        layout->addLayout(inputBar);
        
        connect(connectBtn, &QPushButton::clicked, this, &ArduinoMonitor::toggleConnection);
        connect(sendBtn, &QPushButton::clicked, this, &ArduinoMonitor::sendCommand);
        connect(inputEdit, &QLineEdit::returnPressed, this, &ArduinoMonitor::sendCommand);
        connect(clearBtn, &QPushButton::clicked, outputView, &QTextEdit::clear);
    }
    
    void refreshPorts() {
        QString current = portCombo->currentText();
        portCombo->clear();
        portCombo->addItems(serial->availablePorts());
        
        if (portCombo->findText(current) >= 0) {
            portCombo->setCurrentText(current);
        }
    }
    
    void onLine(const QString& line) {
        logMessage(line);
        
        // Parse sensor data: "SENSOR:512" or just a number
        bool ok;
        double val = line.toDouble(&ok);
        
        if (!ok && line.contains(':')) {
            val = line.split(':').last().trimmed().toDouble(&ok);
        }
        
        if (ok) {
            m_sampleCount++;
            lineSeries->append(m_sampleCount, val);
            
            // Keep only last 100 samples
            if (lineSeries->count() > 100) {
                lineSeries->remove(0);
                auto* axisX = qobject_cast<QValueAxis*>(chart->axes(Qt::Horizontal).first());
                axisX->setRange(m_sampleCount - 100, m_sampleCount);
            }
        }
    }
    
    void toggleConnection() {
        if (serial->isConnected()) {
            serial->close();
        } else {
            int baud = baudCombo->currentText().toInt();
            serial->open(portCombo->currentText(), baud);
        }
    }
    
    void onConnectionChanged(bool connected) {
        connectBtn->setText(connected ? "Disconnect" : "Connect");
        connectBtn->setStyleSheet(connected 
            ? "background: #e74c3c; color: white; padding: 6px 16px;"
            : "background: #27ae60; color: white; padding: 6px 16px;");
        sendBtn->setEnabled(connected);
        statusLed->setStyleSheet(connected ? "color: #27ae60;" : "color: #e74c3c;");
        statusLed->setText("●");
    }
    
    void sendCommand() {
        QString cmd = inputEdit->text().trimmed();
        if (cmd.isEmpty() || !serial->isConnected()) return;
        
        logMessage("> " + cmd);
        serial->writeLine(cmd);
        inputEdit->clear();
    }
    
    void logMessage(const QString& msg) {
        QString ts = QDateTime::currentDateTime().toString("hh:mm:ss.zzz");
        outputView->append(QString("[%1] %2").arg(ts, msg));
    }
    
    SerialManager* serial;
    QComboBox* portCombo;
    QComboBox* baudCombo;
    QPushButton* connectBtn;
    QPushButton* sendBtn;
    QLineEdit* inputEdit;
    QTextEdit* outputView;
    QLabel* statusLed;
    QChartView* chartView;
    QChart* chart;
    QLineSeries* lineSeries;
    int m_sampleCount = 0;
};
```

---

## ขั้นตอนที่ 509: Bluetooth LE Scanner

```cpp
#include <QBluetoothDeviceDiscoveryAgent>
#include <QBluetoothDeviceInfo>
#include <QLowEnergyController>
#include <QLowEnergyService>
#include <QLowEnergyCharacteristic>

// CMakeLists.txt:
// find_package(Qt6 REQUIRED COMPONENTS Bluetooth)
// target_link_libraries(app PRIVATE Qt6::Bluetooth)

class BluetoothScanner : public QObject {
    Q_OBJECT
    Q_PROPERTY(bool scanning READ isScanning NOTIFY scanningChanged)
    Q_PROPERTY(QStringList devices READ deviceNames NOTIFY devicesChanged)
    
public:
    explicit BluetoothScanner(QObject* parent = nullptr) : QObject(parent) {
        agent = new QBluetoothDeviceDiscoveryAgent(this);
        agent->setLowEnergyDiscoveryTimeout(5000);
        
        connect(agent, &QBluetoothDeviceDiscoveryAgent::deviceDiscovered,
                this, &BluetoothScanner::onDeviceFound);
        
        connect(agent, &QBluetoothDeviceDiscoveryAgent::finished,
                [this]() {
            m_scanning = false;
            emit scanningChanged(false);
            qDebug() << "Scan finished, found" << m_devices.size() << "devices";
        });
        
        connect(agent, &QBluetoothDeviceDiscoveryAgent::errorOccurred,
                [this](QBluetoothDeviceDiscoveryAgent::Error err) {
            qWarning() << "BLE scan error:" << err;
            m_scanning = false;
            emit scanningChanged(false);
        });
    }
    
    bool isScanning() const { return m_scanning; }
    
    QStringList deviceNames() const {
        QStringList names;
        for (const auto& d : m_devices) {
            names << (d.name().isEmpty() ? d.address().toString() : d.name());
        }
        return names;
    }
    
public slots:
    void startScan() {
        m_devices.clear();
        emit devicesChanged();
        
        agent->start(QBluetoothDeviceDiscoveryAgent::LowEnergyMethod);
        m_scanning = true;
        emit scanningChanged(true);
        qDebug() << "BLE scan started";
    }
    
    void stopScan() {
        agent->stop();
    }
    
    void connectToDevice(int index) {
        if (index < 0 || index >= m_devices.size()) return;
        
        const QBluetoothDeviceInfo& info = m_devices[index];
        
        if (leController) {
            leController->disconnectFromDevice();
            delete leController;
        }
        
        leController = QLowEnergyController::createCentral(info, this);
        
        connect(leController, &QLowEnergyController::connected, [this]() {
            qDebug() << "BLE connected";
            emit connected();
            leController->discoverServices();
        });
        
        connect(leController, &QLowEnergyController::serviceDiscovered,
                this, &BluetoothScanner::onServiceFound);
        
        connect(leController, &QLowEnergyController::disconnected, [this]() {
            qDebug() << "BLE disconnected";
            emit disconnected();
        });
        
        leController->connectToDevice();
    }
    
signals:
    void scanningChanged(bool scanning);
    void devicesChanged();
    void deviceFound(const QString& name, const QString& address, int rssi);
    void connected();
    void disconnected();
    void serviceFound(const QString& uuid);
    void dataReceived(const QByteArray& data);
    
private slots:
    void onDeviceFound(const QBluetoothDeviceInfo& device) {
        if (device.coreConfigurations() & QBluetoothDeviceInfo::LowEnergyCoreConfiguration) {
            m_devices.append(device);
            emit devicesChanged();
            emit deviceFound(device.name(), device.address().toString(), device.rssi());
            qDebug() << "Found BLE device:" << device.name() << device.address().toString();
        }
    }
    
    void onServiceFound(const QBluetoothUuid& serviceUuid) {
        qDebug() << "Service found:" << serviceUuid.toString();
        emit serviceFound(serviceUuid.toString());
        
        auto* service = leController->createServiceObject(serviceUuid, this);
        if (!service) return;
        
        connect(service, &QLowEnergyService::stateChanged,
                [this, service](QLowEnergyService::ServiceState state) {
            if (state == QLowEnergyService::RemoteServiceDiscovered) {
                subscribeToCharacteristics(service);
            }
        });
        
        service->discoverDetails();
    }
    
    void subscribeToCharacteristics(QLowEnergyService* service) {
        for (const QLowEnergyCharacteristic& ch : service->characteristics()) {
            if (ch.properties() & QLowEnergyCharacteristic::Notify) {
                QLowEnergyDescriptor desc = ch.descriptor(
                    QBluetoothUuid::DescriptorType::ClientCharacteristicConfiguration);
                
                if (desc.isValid()) {
                    service->writeDescriptor(desc, QByteArray::fromHex("0100"));
                }
                
                connect(service, &QLowEnergyService::characteristicChanged,
                        [this](const QLowEnergyCharacteristic& c, const QByteArray& value) {
                    Q_UNUSED(c)
                    emit dataReceived(value);
                });
            }
        }
    }
    
    QBluetoothDeviceDiscoveryAgent* agent;
    QLowEnergyController* leController = nullptr;
    QList<QBluetoothDeviceInfo> m_devices;
    bool m_scanning = false;
};
```

---

## ขั้นตอนที่ 510-520: BLE Monitor UI

```cpp
class BleMonitorWindow : public QMainWindow {
    Q_OBJECT
    
public:
    BleMonitorWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("BLE Monitor");
        setMinimumSize(800, 600);
        
        scanner = new BluetoothScanner(this);
        
        setupUi();
        
        connect(scanner, &BluetoothScanner::deviceFound, this, &BleMonitorWindow::onDeviceFound);
        connect(scanner, &BluetoothScanner::connected, [this]() {
            log("Connected to device");
            statusBar()->showMessage("Connected");
        });
        connect(scanner, &BluetoothScanner::disconnected, [this]() {
            log("Device disconnected");
            statusBar()->showMessage("Disconnected");
        });
        connect(scanner, &BluetoothScanner::dataReceived, this, &BleMonitorWindow::onDataReceived);
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* layout = new QHBoxLayout(central);
        setCentralWidget(central);
        
        // Left panel: devices
        auto* leftPanel = new QWidget();
        leftPanel->setMaximumWidth(300);
        auto* leftLayout = new QVBoxLayout(leftPanel);
        
        leftLayout->addWidget(new QLabel("BLE Devices:"));
        deviceList = new QListWidget();
        leftLayout->addWidget(deviceList, 1);
        
        auto* scanBtn = new QPushButton("Scan");
        scanBtn->setStyleSheet("background: #3498db; color: white; padding: 8px;");
        
        leftLayout->addWidget(scanBtn);
        
        // Right panel: data
        auto* rightPanel = new QWidget();
        auto* rightLayout = new QVBoxLayout(rightPanel);
        
        log_view = new QTextEdit();
        log_view->setReadOnly(true);
        log_view->setFont(QFont("Monospace", 10));
        
        rightLayout->addWidget(log_view);
        
        layout->addWidget(leftPanel);
        layout->addWidget(rightPanel, 1);
        
        connect(scanBtn, &QPushButton::clicked, [this, scanBtn]() {
            if (scanner->isScanning()) {
                scanner->stopScan();
                scanBtn->setText("Scan");
            } else {
                deviceList->clear();
                scanner->startScan();
                scanBtn->setText("Stop");
                log("Scanning for BLE devices...");
            }
        });
        
        connect(deviceList, &QListWidget::itemDoubleClicked, [this](QListWidgetItem* item) {
            int row = deviceList->row(item);
            scanner->connectToDevice(row);
            log("Connecting to: " + item->text());
        });
    }
    
    void onDeviceFound(const QString& name, const QString& address, int rssi) {
        QString display = QString("%1 (%2) RSSI:%3dBm")
            .arg(name.isEmpty() ? "Unknown" : name)
            .arg(address)
            .arg(rssi);
        
        deviceList->addItem(display);
        log("Found: " + display);
    }
    
    void onDataReceived(const QByteArray& data) {
        QString hex = data.toHex(' ');
        QString text = QString::fromUtf8(data).trimmed();
        log(QString("Data [hex]: %1 | [text]: %2").arg(hex, text));
    }
    
    void log(const QString& msg) {
        QString ts = QTime::currentTime().toString("hh:mm:ss.zzz");
        log_view->append(QString("[%1] %2").arg(ts, msg));
    }
    
    BluetoothScanner* scanner;
    QListWidget* deviceList;
    QTextEdit* log_view;
};
```

---

## สรุป Part 036

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QSerialPort พื้นฐาน (open, read, write, line parsing)
2. ✅ SerialManager class ที่ใช้งานได้จริง
3. ✅ Arduino Monitor GUI + real-time chart
4. ✅ Bluetooth LE scanning ด้วย QBluetoothDeviceDiscoveryAgent
5. ✅ BLE GATT connection + characteristic notifications
6. ✅ BLE Monitor UI

---

⬅️ [Part 035](part035.md) | ➡️ [Part 037: Qt PDF & Printing](part037.md)
