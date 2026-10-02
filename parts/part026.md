# Part 026: Qt WebEngine

## ขั้นตอนที่ 356-370

---

## ขั้นตอนที่ 356: QtWebEngine Overview

```
QtWebEngine Components:
  ├── QWebEngineView    - widget แสดง web page
  ├── QWebEnginePage    - page instance (หลายได้ต่อ view)
  ├── QWebEngineProfile - settings, cookies, cache
  ├── QWebEngineSettings - ตั้งค่า JavaScript, plugins
  └── QWebChannel      - สื่อสาร C++ <-> JavaScript

CMakeLists.txt:
  find_package(Qt6 REQUIRED COMPONENTS WebEngineWidgets)
  target_link_libraries(myapp PRIVATE Qt6::WebEngineWidgets)
```

---

## ขั้นตอนที่ 357: Basic WebView

```cpp
#include <QWebEngineView>
#include <QWebEnginePage>
#include <QWebEngineSettings>

class SimpleBrowser : public QMainWindow {
    Q_OBJECT
    
public:
    SimpleBrowser(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Simple Browser");
        setMinimumSize(1024, 768);
        
        webView = new QWebEngineView(this);
        setCentralWidget(webView);
        
        // Configure settings
        auto* settings = webView->settings();
        settings->setAttribute(QWebEngineSettings::JavascriptEnabled, true);
        settings->setAttribute(QWebEngineSettings::LocalStorageEnabled, true);
        settings->setAttribute(QWebEngineSettings::ScrollAnimatorEnabled, true);
        settings->setDefaultTextEncoding("UTF-8");
        
        // Navigation toolbar
        auto* toolbar = addToolBar("Navigation");
        
        auto* backBtn    = toolbar->addAction("◀");
        auto* forwardBtn = toolbar->addAction("▶");
        auto* reloadBtn  = toolbar->addAction("↻");
        auto* homeBtn    = toolbar->addAction("⌂");
        
        urlBar = new QLineEdit();
        urlBar->setMinimumWidth(500);
        toolbar->addWidget(urlBar);
        
        auto* goBtn = toolbar->addAction("Go");
        
        // Progress bar
        progressBar = new QProgressBar(this);
        progressBar->setMaximumHeight(3);
        progressBar->setTextVisible(false);
        progressBar->setRange(0, 100);
        progressBar->setStyleSheet("QProgressBar::chunk { background: #3498db; }");
        statusBar()->addPermanentWidget(progressBar, 1);
        
        // Connections
        connect(backBtn,    &QAction::triggered, webView, &QWebEngineView::back);
        connect(forwardBtn, &QAction::triggered, webView, &QWebEngineView::forward);
        connect(reloadBtn,  &QAction::triggered, webView, &QWebEngineView::reload);
        connect(homeBtn,    &QAction::triggered, [this]() {
            webView->load(QUrl("https://www.google.com"));
        });
        
        connect(goBtn, &QAction::triggered, this, &SimpleBrowser::navigate);
        connect(urlBar, &QLineEdit::returnPressed, this, &SimpleBrowser::navigate);
        
        connect(webView, &QWebEngineView::urlChanged, [this](const QUrl& url) {
            urlBar->setText(url.toString());
        });
        
        connect(webView, &QWebEngineView::titleChanged, [this](const QString& title) {
            setWindowTitle(title + " - Browser");
        });
        
        connect(webView, &QWebEngineView::loadProgress, [this](int pct) {
            progressBar->setValue(pct);
            progressBar->setVisible(pct < 100);
        });
        
        // Load initial page
        webView->load(QUrl("https://www.example.com"));
    }
    
private slots:
    void navigate() {
        QString url = urlBar->text();
        if (!url.startsWith("http")) url = "https://" + url;
        webView->load(QUrl(url));
    }
    
private:
    QWebEngineView* webView;
    QLineEdit* urlBar;
    QProgressBar* progressBar;
};
```

---

## ขั้นตอนที่ 358: Load HTML & Local Content

```cpp
class HtmlViewer : public QWidget {
    Q_OBJECT
    
public:
    HtmlViewer() {
        auto* layout = new QVBoxLayout(this);
        
        view = new QWebEngineView();
        layout->addWidget(view);
        
        auto* btnBar = new QHBoxLayout();
        
        auto* loadHtmlBtn = new QPushButton("Load HTML String");
        auto* loadFileBtn = new QPushButton("Load Local File");
        auto* loadUrlBtn  = new QPushButton("Load URL");
        
        btnBar->addWidget(loadHtmlBtn);
        btnBar->addWidget(loadFileBtn);
        btnBar->addWidget(loadUrlBtn);
        layout->addLayout(btnBar);
        
        connect(loadHtmlBtn, &QPushButton::clicked, this, &HtmlViewer::loadHtmlString);
        connect(loadFileBtn, &QPushButton::clicked, this, &HtmlViewer::loadLocalFile);
        connect(loadUrlBtn,  &QPushButton::clicked, this, &HtmlViewer::loadRemoteUrl);
    }
    
private slots:
    void loadHtmlString() {
        QString html = R"(
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<style>
  body { font-family: 'Sarabun', sans-serif; background: #f0f2f5; padding: 20px; }
  .card { background: white; border-radius: 12px; padding: 24px;
          box-shadow: 0 4px 20px rgba(0,0,0,0.1); max-width: 600px; margin: auto; }
  h1 { color: #2c3e50; }
  .btn { background: #3498db; color: white; border: none; padding: 10px 20px;
         border-radius: 6px; cursor: pointer; font-size: 16px; }
  .btn:hover { background: #2980b9; }
  #result { margin-top: 16px; color: #27ae60; }
</style>
</head>
<body>
<div class="card">
  <h1>สวัสดี Qt WebEngine!</h1>
  <p>นี่คือ HTML ที่โหลดจาก C++ string</p>
  <button class="btn" onclick="handleClick()">คลิกฉัน</button>
  <div id="result"></div>
</div>
<script>
function handleClick() {
  document.getElementById('result').textContent = 'สวัสดีจาก JavaScript! 🎉';
}
</script>
</body>
</html>
        )";
        
        // Load with base URL (needed for relative paths)
        view->setHtml(html, QUrl("qrc:/"));
    }
    
    void loadLocalFile() {
        QString path = QFileDialog::getOpenFileName(this, "Open HTML", "", "HTML (*.html *.htm)");
        if (!path.isEmpty()) {
            view->load(QUrl::fromLocalFile(path));
        }
    }
    
    void loadRemoteUrl() {
        bool ok;
        QString url = QInputDialog::getText(this, "URL", "Enter URL:", 
                                             QLineEdit::Normal, "https://", &ok);
        if (ok && !url.isEmpty()) {
            if (!url.startsWith("http")) url = "https://" + url;
            view->load(QUrl(url));
        }
    }
    
private:
    QWebEngineView* view;
};
```

---

## ขั้นตอนที่ 359: C++ ↔ JavaScript Communication

```cpp
#include <QWebChannel>
#include <QWebEngineView>
#include <QWebEnginePage>

// === C++ Backend exposed to JavaScript ===
class AppBackend : public QObject {
    Q_OBJECT
    Q_PROPERTY(QString userName READ userName WRITE setUserName NOTIFY userNameChanged)
    Q_PROPERTY(int    itemCount READ itemCount NOTIFY itemCountChanged)
    
public:
    explicit AppBackend(QObject* parent = nullptr) : QObject(parent) {}
    
    QString userName() const { return m_userName; }
    void setUserName(const QString& name) {
        if (m_userName != name) {
            m_userName = name;
            emit userNameChanged(m_userName);
        }
    }
    
    int itemCount() const { return m_items.size(); }
    
    // Q_INVOKABLE = callable from JavaScript
    Q_INVOKABLE QString greet(const QString& name) {
        return QString("สวัสดี, %1! ยินดีต้อนรับ").arg(name);
    }
    
    Q_INVOKABLE void addItem(const QString& text) {
        m_items.append(text);
        emit itemAdded(text);
        emit itemCountChanged(m_items.size());
    }
    
    Q_INVOKABLE QVariantList getItems() {
        QVariantList result;
        for (const QString& item : m_items)
            result << item;
        return result;
    }
    
    Q_INVOKABLE bool removeItem(int index) {
        if (index < 0 || index >= m_items.size()) return false;
        m_items.removeAt(index);
        emit itemCountChanged(m_items.size());
        return true;
    }
    
    Q_INVOKABLE void showNativeDialog(const QString& msg) {
        QMessageBox::information(nullptr, "From JavaScript", msg);
    }
    
signals:
    void userNameChanged(const QString& name);
    void itemCountChanged(int count);
    void itemAdded(const QString& text);
    void notificationFromCpp(const QString& msg); // Call from C++ to JS
    
private:
    QString m_userName = "Guest";
    QStringList m_items;
};

class ChannelDemo : public QMainWindow {
    Q_OBJECT
    
public:
    ChannelDemo() {
        setWindowTitle("WebChannel Demo");
        setMinimumSize(900, 700);
        
        backend = new AppBackend(this);
        
        auto* channel = new QWebChannel(this);
        channel->registerObject("backend", backend);
        
        view = new QWebEngineView(this);
        view->page()->setWebChannel(channel);
        
        setCentralWidget(view);
        
        // Button to send notification from C++ to JS
        auto* toolbar = addToolBar("Actions");
        auto* notifyBtn = toolbar->addAction("Send to JS");
        connect(notifyBtn, &QAction::triggered, [this]() {
            emit backend->notificationFromCpp("Hello from C++! " + 
                QDateTime::currentDateTime().toString("hh:mm:ss"));
        });
        
        loadPage();
    }
    
private:
    void loadPage() {
        QString html = R"(
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<script src="qrc:///qtwebchannel/qwebchannel.js"></script>
<style>
  body { font-family: sans-serif; padding: 20px; background: #f5f5f5; }
  .card { background: white; padding: 20px; border-radius: 10px; margin-bottom: 16px;
          box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
  button { background: #3498db; color: white; border: none; padding: 8px 16px;
           border-radius: 5px; cursor: pointer; margin: 4px; }
  button:hover { background: #2980b9; }
  input { border: 1px solid #ccc; padding: 8px; border-radius: 5px; width: 200px; }
  #log { background: #2c3e50; color: #2ecc71; padding: 16px; border-radius: 8px;
         font-family: monospace; min-height: 100px; max-height: 200px; overflow-y: auto; }
</style>
</head>
<body>
<div class="card">
  <h2>Qt WebChannel Demo</h2>
  
  <div>
    <input id="nameInput" placeholder="ชื่อของคุณ">
    <button onclick="greet()">Greet</button>
  </div>
  <p id="greetResult"></p>
</div>

<div class="card">
  <h3>Item List</h3>
  <input id="itemInput" placeholder="ชื่อ item">
  <button onclick="addItem()">เพิ่ม</button>
  <button onclick="loadItems()">โหลดทั้งหมด</button>
  <ul id="itemList"></ul>
  <p>จำนวน: <span id="count">0</span></p>
</div>

<div class="card">
  <h3>Log</h3>
  <div id="log"></div>
</div>

<script>
let backend = null;

new QWebChannel(qt.webChannelTransport, function(channel) {
    backend = channel.objects.backend;
    log('Connected to C++ backend');
    
    // Listen to C++ signals
    backend.itemAdded.connect(function(text) {
        log('Item added from C++: ' + text);
    });
    
    backend.notificationFromCpp.connect(function(msg) {
        log('Notification: ' + msg);
    });
    
    backend.itemCountChanged.connect(function(count) {
        document.getElementById('count').textContent = count;
    });
    
    loadItems();
});

function greet() {
    let name = document.getElementById('nameInput').value;
    if (!name) return;
    backend.greet(name, function(result) {
        document.getElementById('greetResult').textContent = result;
        log('greet() result: ' + result);
    });
}

function addItem() {
    let text = document.getElementById('itemInput').value;
    if (!text) return;
    backend.addItem(text);
    document.getElementById('itemInput').value = '';
    setTimeout(loadItems, 50);
}

function loadItems() {
    backend.getItems(function(items) {
        let ul = document.getElementById('itemList');
        ul.innerHTML = '';
        items.forEach(function(item, idx) {
            let li = document.createElement('li');
            li.textContent = item;
            let btn = document.createElement('button');
            btn.textContent = '✕';
            btn.onclick = function() {
                backend.removeItem(idx, function() { loadItems(); });
            };
            li.appendChild(btn);
            ul.appendChild(li);
        });
    });
}

function log(msg) {
    let el = document.getElementById('log');
    el.innerHTML += '<div>' + new Date().toLocaleTimeString() + ' - ' + msg + '</div>';
    el.scrollTop = el.scrollHeight;
}
</script>
</body>
</html>
        )";
        view->setHtml(html, QUrl("qrc:/"));
    }
    
    QWebEngineView* view;
    AppBackend* backend;
};
```

---

## ขั้นตอนที่ 360-370: JavaScript Execution from C++

```cpp
class JsController : public QObject {
    Q_OBJECT
    
public:
    JsController(QWebEngineView* view) : m_view(view) {}
    
    // Run JS and get result
    void runScript(const QString& js, std::function<void(const QVariant&)> callback = {}) {
        m_view->page()->runJavaScript(js, [callback](const QVariant& result) {
            if (callback) callback(result);
        });
    }
    
    // Get page title
    void getTitle(std::function<void(const QString&)> cb) {
        runScript("document.title", [cb](const QVariant& v) {
            cb(v.toString());
        });
    }
    
    // Scroll page
    void scrollTo(int x, int y) {
        runScript(QString("window.scrollTo(%1, %2)").arg(x).arg(y));
    }
    
    // Get all links
    void getAllLinks(std::function<void(const QStringList&)> cb) {
        QString js = R"(
            Array.from(document.querySelectorAll('a[href]'))
                .map(a => a.href)
                .filter(h => h.startsWith('http'))
        )";
        runScript(js, [cb](const QVariant& v) {
            QStringList links;
            for (const QVariant& item : v.toList()) links << item.toString();
            cb(links);
        });
    }
    
    // Inject CSS
    void injectCss(const QString& css) {
        QString js = QString(R"(
            var style = document.createElement('style');
            style.textContent = '%1';
            document.head.appendChild(style);
        )").arg(css.simplified().replace("'", "\\'"));
        runScript(js);
    }
    
    // Screenshot via JS
    void takeScreenshot(const QString& filePath) {
        m_view->grab().save(filePath);
        qDebug() << "Screenshot saved to:" << filePath;
    }
    
    // Fill form fields
    void fillInput(const QString& selector, const QString& value) {
        QString js = QString(
            "var el = document.querySelector('%1');"
            "if(el) { el.value='%2'; el.dispatchEvent(new Event('input')); }"
        ).arg(selector).arg(value);
        runScript(js);
    }
    
    // Click element
    void clickElement(const QString& selector) {
        runScript(QString(
            "var el = document.querySelector('%1');"
            "if(el) el.click();"
        ).arg(selector));
    }
    
private:
    QWebEngineView* m_view;
};

// === Demo usage ===
class WebAutomation : public QMainWindow {
    Q_OBJECT
    
public:
    WebAutomation() {
        view = new QWebEngineView(this);
        setCentralWidget(view);
        
        js = new JsController(view, this);
        
        auto* toolbar = addToolBar("JS");
        
        auto* titleBtn = toolbar->addAction("Get Title");
        connect(titleBtn, &QAction::triggered, [this]() {
            js->getTitle([](const QString& t) {
                QMessageBox::information(nullptr, "Title", t);
            });
        });
        
        auto* linksBtn = toolbar->addAction("Get Links");
        connect(linksBtn, &QAction::triggered, [this]() {
            js->getAllLinks([](const QStringList& links) {
                qDebug() << "Links found:" << links.size();
                for (const QString& l : links.mid(0, 5))
                    qDebug() << " -" << l;
            });
        });
        
        auto* darkBtn = toolbar->addAction("Dark Mode");
        connect(darkBtn, &QAction::triggered, [this]() {
            js->injectCss(R"(
                body { background: #1a1a2e !important; color: #e0e0e0 !important; }
                a { color: #4fc3f7 !important; }
            )");
        });
        
        view->load(QUrl("https://www.example.com"));
    }
    
private:
    QWebEngineView* view;
    JsController* js;
};
```

---

## สรุป Part 026

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QWebEngineView พื้นฐาน
2. ✅ โหลด HTML string, local file, remote URL
3. ✅ QWebChannel: สื่อสาร C++ ↔ JavaScript
4. ✅ Q_INVOKABLE functions จาก JavaScript
5. ✅ Qt signals → JavaScript callbacks
6. ✅ รัน JavaScript จาก C++

---

⬅️ [Part 025](part025.md) | ➡️ [Part 027: Template Metaprogramming](part027.md)
