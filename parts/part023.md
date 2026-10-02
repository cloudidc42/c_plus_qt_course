# Part 023: Qt Multimedia & Graphics

## ขั้นตอนที่ 311-325

---

## ขั้นตอนที่ 311: Qt Multimedia

```qmake
QT += multimedia multimediawidgets
```

```cpp
#include <QMediaPlayer>
#include <QAudioOutput>
#include <QVideoWidget>
#include <QSlider>
#include <QLabel>
#include <QPushButton>
#include <QFileDialog>

class MediaPlayer : public QMainWindow {
    Q_OBJECT
    
public:
    MediaPlayer(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Qt Media Player");
        setMinimumSize(700, 500);
        
        // Media player
        player = new QMediaPlayer(this);
        audioOutput = new QAudioOutput(this);
        player->setAudioOutput(audioOutput);
        audioOutput->setVolume(0.7);
        
        // Video widget
        videoWidget = new QVideoWidget();
        videoWidget->setMinimumHeight(300);
        player->setVideoOutput(videoWidget);
        
        // Controls
        playBtn  = new QPushButton("▶");
        pauseBtn = new QPushButton("⏸");
        stopBtn  = new QPushButton("⏹");
        openBtn  = new QPushButton("📁 Open");
        
        progressSlider = new QSlider(Qt::Horizontal);
        volumeSlider = new QSlider(Qt::Horizontal);
        volumeSlider->setRange(0, 100);
        volumeSlider->setValue(70);
        volumeSlider->setMaximumWidth(100);
        
        timeLabel = new QLabel("0:00 / 0:00");
        titleLabel = new QLabel("No media");
        titleLabel->setAlignment(Qt::AlignCenter);
        
        // Layout
        auto* central = new QWidget(this);
        auto* layout = new QVBoxLayout(central);
        setCentralWidget(central);
        
        layout->addWidget(titleLabel);
        layout->addWidget(videoWidget, 1);
        
        layout->addWidget(progressSlider);
        layout->addWidget(timeLabel);
        
        auto* ctrlBar = new QHBoxLayout();
        ctrlBar->addWidget(openBtn);
        ctrlBar->addStretch();
        ctrlBar->addWidget(playBtn);
        ctrlBar->addWidget(pauseBtn);
        ctrlBar->addWidget(stopBtn);
        ctrlBar->addStretch();
        ctrlBar->addWidget(new QLabel("🔊"));
        ctrlBar->addWidget(volumeSlider);
        layout->addLayout(ctrlBar);
        
        // Connections
        connect(openBtn,  &QPushButton::clicked, this, &MediaPlayer::openFile);
        connect(playBtn,  &QPushButton::clicked, player, &QMediaPlayer::play);
        connect(pauseBtn, &QPushButton::clicked, player, &QMediaPlayer::pause);
        connect(stopBtn,  &QPushButton::clicked, player, &QMediaPlayer::stop);
        
        connect(player, &QMediaPlayer::positionChanged, this, &MediaPlayer::updateProgress);
        connect(player, &QMediaPlayer::durationChanged, this, &MediaPlayer::updateDuration);
        
        connect(progressSlider, &QSlider::sliderMoved, [this](int pos) {
            player->setPosition(pos * 1000LL);
        });
        
        connect(volumeSlider, &QSlider::valueChanged, [this](int vol) {
            audioOutput->setVolume(vol / 100.0);
        });
        
        connect(player, &QMediaPlayer::playbackStateChanged, this, &MediaPlayer::onStateChanged);
        connect(player, &QMediaPlayer::errorOccurred, [this](QMediaPlayer::Error, const QString& err) {
            QMessageBox::warning(this, "Error", "Playback error: " + err);
        });
    }
    
private:
    QMediaPlayer* player;
    QAudioOutput* audioOutput;
    QVideoWidget* videoWidget;
    QPushButton* playBtn;
    QPushButton* pauseBtn;
    QPushButton* stopBtn;
    QPushButton* openBtn;
    QSlider* progressSlider;
    QSlider* volumeSlider;
    QLabel* timeLabel;
    QLabel* titleLabel;
    
    QString formatTime(qint64 ms) {
        qint64 secs = ms / 1000;
        qint64 mins = secs / 60;
        secs %= 60;
        return QString("%1:%2").arg(mins).arg(secs, 2, 10, QChar('0'));
    }
    
    void openFile() {
        QString file = QFileDialog::getOpenFileName(this, "Open Media",
            QDir::homePath(),
            "Media Files (*.mp4 *.avi *.mkv *.mp3 *.wav *.flac);;All Files (*)");
        
        if (!file.isEmpty()) {
            player->setSource(QUrl::fromLocalFile(file));
            titleLabel->setText(QFileInfo(file).fileName());
            player->play();
        }
    }
    
    void updateProgress(qint64 position) {
        qint64 duration = player->duration();
        if (duration > 0) {
            progressSlider->setValue(position / 1000);
        }
        timeLabel->setText(formatTime(position) + " / " + formatTime(duration));
    }
    
    void updateDuration(qint64 duration) {
        progressSlider->setRange(0, duration / 1000);
    }
    
    void onStateChanged(QMediaPlayer::PlaybackState state) {
        playBtn->setEnabled(state != QMediaPlayer::PlayingState);
        pauseBtn->setEnabled(state == QMediaPlayer::PlayingState);
    }
};
```

---

## ขั้นตอนที่ 312: QGraphicsScene & QGraphicsView

```cpp
#include <QGraphicsScene>
#include <QGraphicsView>
#include <QGraphicsItem>
#include <QGraphicsRectItem>
#include <QGraphicsEllipseItem>
#include <QGraphicsLineItem>
#include <QGraphicsTextItem>
#include <QGraphicsPixmapItem>

class GraphicsDemo : public QWidget {
    Q_OBJECT
    
public:
    GraphicsDemo(QWidget* parent = nullptr) : QWidget(parent) {
        setWindowTitle("QGraphicsScene Demo");
        setMinimumSize(800, 600);
        
        scene = new QGraphicsScene(0, 0, 780, 580, this);
        view = new QGraphicsView(scene, this);
        view->setRenderHint(QPainter::Antialiasing);
        view->setDragMode(QGraphicsView::ScrollHandDrag);
        
        auto* layout = new QVBoxLayout(this);
        
        // Tools
        auto* toolBar = new QHBoxLayout();
        auto* addRectBtn = new QPushButton("+ Rect");
        auto* addCircBtn = new QPushButton("+ Circle");
        auto* addTextBtn = new QPushButton("+ Text");
        auto* clearBtn = new QPushButton("Clear");
        auto* zoomInBtn = new QPushButton("Zoom +");
        auto* zoomOutBtn = new QPushButton("Zoom -");
        
        toolBar->addWidget(addRectBtn);
        toolBar->addWidget(addCircBtn);
        toolBar->addWidget(addTextBtn);
        toolBar->addWidget(clearBtn);
        toolBar->addStretch();
        toolBar->addWidget(zoomInBtn);
        toolBar->addWidget(zoomOutBtn);
        
        layout->addLayout(toolBar);
        layout->addWidget(view);
        
        // Background
        scene->setBackgroundBrush(QBrush(QColor(240, 240, 250)));
        
        // Add items
        addSampleItems();
        
        // Connections
        connect(addRectBtn, &QPushButton::clicked, this, &GraphicsDemo::addRect);
        connect(addCircBtn, &QPushButton::clicked, this, &GraphicsDemo::addCircle);
        connect(addTextBtn, &QPushButton::clicked, this, &GraphicsDemo::addText);
        connect(clearBtn, &QPushButton::clicked, scene, &QGraphicsScene::clear);
        connect(zoomInBtn, &QPushButton::clicked, [this]() { view->scale(1.2, 1.2); });
        connect(zoomOutBtn, &QPushButton::clicked, [this]() { view->scale(1/1.2, 1/1.2); });
    }
    
private:
    QGraphicsScene* scene;
    QGraphicsView* view;
    int itemCount = 0;
    
    void addSampleItems() {
        // Rectangles
        for (int i = 0; i < 5; i++) {
            auto* rect = scene->addRect(
                20 + i * 100, 20, 80, 60,
                QPen(QColor(52, 152, 219), 2),
                QBrush(QColor(52, 152, 219, 80))
            );
            rect->setFlag(QGraphicsItem::ItemIsMovable);
            rect->setFlag(QGraphicsItem::ItemIsSelectable);
            rect->setToolTip(QString("Rect %1").arg(i+1));
        }
        
        // Ellipses
        for (int i = 0; i < 5; i++) {
            auto* ell = scene->addEllipse(
                20 + i * 100, 120, 80, 60,
                QPen(QColor(231, 76, 60), 2),
                QBrush(QColor(231, 76, 60, 80))
            );
            ell->setFlag(QGraphicsItem::ItemIsMovable);
            ell->setFlag(QGraphicsItem::ItemIsSelectable);
        }
        
        // Text
        auto* text = scene->addText("สวัสดี QGraphicsScene!", 
                                     QFont("Arial", 16, QFont::Bold));
        text->setDefaultTextColor(QColor(44, 62, 80));
        text->setPos(50, 220);
        text->setFlag(QGraphicsItem::ItemIsMovable);
        
        // Line with arrow
        scene->addLine(50, 320, 300, 320, QPen(Qt::darkGray, 2));
        
        // Polygon (star)
        QPolygonF star;
        for (int i = 0; i < 10; i++) {
            double angle = i * M_PI / 5 - M_PI / 2;
            double r = (i % 2 == 0) ? 50 : 20;
            star << QPointF(400 + r * cos(angle), 350 + r * sin(angle));
        }
        auto* starItem = scene->addPolygon(star,
            QPen(QColor(241, 196, 15), 2),
            QBrush(QColor(241, 196, 15, 150)));
        starItem->setFlag(QGraphicsItem::ItemIsMovable);
    }
    
    void addRect() {
        QColor color(qrand() % 200 + 55, qrand() % 200 + 55, qrand() % 200 + 55);
        auto* rect = scene->addRect(
            qrand() % 600, qrand() % 400, 80, 60,
            QPen(color, 2),
            QBrush(QColor(color.red(), color.green(), color.blue(), 100))
        );
        rect->setFlag(QGraphicsItem::ItemIsMovable);
        rect->setFlag(QGraphicsItem::ItemIsSelectable);
        itemCount++;
    }
    
    void addCircle() {
        QColor color(qrand() % 200 + 55, qrand() % 200 + 55, qrand() % 200 + 55);
        auto* ell = scene->addEllipse(
            qrand() % 600, qrand() % 400, 70, 70,
            QPen(color, 2),
            QBrush(QColor(color.red(), color.green(), color.blue(), 100))
        );
        ell->setFlag(QGraphicsItem::ItemIsMovable);
        ell->setFlag(QGraphicsItem::ItemIsSelectable);
    }
    
    void addText() {
        QStringList words = {"Qt", "C++", "QML", "Programming", "Hello", "World"};
        auto* text = scene->addText(
            words[qrand() % words.size()],
            QFont("Arial", qrand() % 12 + 10)
        );
        text->setPos(qrand() % 600, qrand() % 400);
        text->setDefaultTextColor(QColor(qrand() % 256, qrand() % 256, qrand() % 256));
        text->setFlag(QGraphicsItem::ItemIsMovable);
    }
};
```

---

## ขั้นตอนที่ 313: Custom Graphics Item

```cpp
class NodeItem : public QGraphicsObject {
    Q_OBJECT
    
private:
    QString m_label;
    QColor m_color;
    QList<NodeItem*> m_connections;
    bool m_hovered = false;
    
public:
    NodeItem(const QString& label, const QColor& color, QGraphicsItem* parent = nullptr)
        : QGraphicsObject(parent), m_label(label), m_color(color) {
        setFlag(ItemIsMovable);
        setFlag(ItemIsSelectable);
        setFlag(ItemSendsGeometryChanges);
        setAcceptHoverEvents(true);
        setCacheMode(DeviceCoordinateCache);
        setZValue(-1);
    }
    
    QRectF boundingRect() const override {
        return QRectF(-50, -30, 100, 60);
    }
    
    void paint(QPainter* p, const QStyleOptionGraphicsItem* option, QWidget*) override {
        p->setRenderHint(QPainter::Antialiasing);
        
        QColor c = m_color;
        if (option->state & QStyle::State_Selected) c = c.darker(130);
        if (m_hovered) c = c.lighter(120);
        
        // Shadow
        if (!m_hovered) {
            p->setPen(Qt::NoPen);
            p->setBrush(QColor(0, 0, 0, 30));
            p->drawRoundedRect(-48, -28, 100, 60, 8, 8);
        }
        
        // Node
        p->setPen(QPen(c.darker(120), 2));
        p->setBrush(c);
        p->drawRoundedRect(-50, -30, 100, 60, 8, 8);
        
        // Label
        p->setPen(Qt::white);
        p->setFont(QFont("Arial", 10, QFont::Bold));
        p->drawText(QRectF(-50, -30, 100, 60), Qt::AlignCenter, m_label);
    }
    
    void connectTo(NodeItem* other) {
        if (!m_connections.contains(other)) {
            m_connections.append(other);
        }
    }
    
    const QList<NodeItem*>& connections() const { return m_connections; }
    
protected:
    void hoverEnterEvent(QGraphicsSceneHoverEvent*) override {
        m_hovered = true;
        update();
    }
    
    void hoverLeaveEvent(QGraphicsSceneHoverEvent*) override {
        m_hovered = false;
        update();
    }
    
    QVariant itemChange(GraphicsItemChange change, const QVariant& value) override {
        if (change == ItemPositionHasChanged) {
            emit positionChanged();
        }
        return QGraphicsItem::itemChange(change, value);
    }
    
signals:
    void positionChanged();
};

class EdgeItem : public QGraphicsItem {
private:
    NodeItem* src;
    NodeItem* dst;
    
public:
    EdgeItem(NodeItem* s, NodeItem* d) : src(s), dst(d) {
        setZValue(-1);
        
        auto update = [this]() { prepareGeometryChange(); this->update(); };
        connect(src, &NodeItem::positionChanged, update);
        connect(dst, &NodeItem::positionChanged, update);
    }
    
    QRectF boundingRect() const override {
        QPointF s = src->pos();
        QPointF d = dst->pos();
        return QRectF(qMin(s.x(), d.x()) - 10, qMin(s.y(), d.y()) - 10,
                      qAbs(s.x() - d.x()) + 20, qAbs(s.y() - d.y()) + 20);
    }
    
    void paint(QPainter* p, const QStyleOptionGraphicsItem*, QWidget*) override {
        QPointF s = src->pos();
        QPointF d = dst->pos();
        
        p->setPen(QPen(QColor(100, 100, 100, 180), 2, Qt::SolidLine, Qt::RoundCap));
        p->drawLine(s, d);
        
        // Arrow at destination
        QLineF line(s, d);
        double angle = line.angle();
        
        QPointF arrowP1 = d + QPointF(
            sin(qDegreesToRadians(angle + 150)) * 10,
            cos(qDegreesToRadians(angle + 150)) * 10);
        QPointF arrowP2 = d + QPointF(
            sin(qDegreesToRadians(angle - 150)) * 10,
            cos(qDegreesToRadians(angle - 150)) * 10);
        
        p->setBrush(QColor(100, 100, 100));
        p->drawPolygon(QPolygonF({d, arrowP1, arrowP2}));
    }
};
```

---

## ขั้นตอนที่ 314-325: Diagram Editor

```cpp
class DiagramEditor : public QMainWindow {
    Q_OBJECT
    
public:
    DiagramEditor(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Diagram Editor");
        setMinimumSize(900, 650);
        
        scene = new QGraphicsScene(-400, -300, 800, 600, this);
        view = new QGraphicsView(scene);
        view->setRenderHint(QPainter::Antialiasing);
        view->setDragMode(QGraphicsView::RubberBandDrag);
        
        setCentralWidget(view);
        setupToolbar();
        setupSampleDiagram();
    }
    
private:
    QGraphicsScene* scene;
    QGraphicsView* view;
    
    void setupToolbar() {
        auto* tb = addToolBar("Tools");
        
        tb->addAction("Add Node", [this]() {
            QString text = QInputDialog::getText(this, "Add Node", "Label:");
            if (!text.isEmpty()) {
                QColor colors[] = {
                    QColor("#3498db"), QColor("#e74c3c"), QColor("#2ecc71"),
                    QColor("#f39c12"), QColor("#9b59b6")
                };
                auto* node = new NodeItem(text, colors[qrand() % 5]);
                node->setPos(scene->width() * (qrand() % 100) / 100.0 - 400,
                             scene->height() * (qrand() % 100) / 100.0 - 300);
                scene->addItem(node);
            }
        });
        
        tb->addAction("Connect", [this]() {
            statusBar()->showMessage("Select source node...");
            // Connection mode implementation
        });
        
        tb->addAction("Delete", [this]() {
            for (auto* item : scene->selectedItems()) {
                scene->removeItem(item);
                delete item;
            }
        });
        
        tb->addSeparator();
        
        tb->addAction("Zoom In",  [this]() { view->scale(1.25, 1.25); });
        tb->addAction("Zoom Out", [this]() { view->scale(0.8, 0.8); });
        tb->addAction("Fit",      [this]() { view->fitInView(scene->itemsBoundingRect(), Qt::KeepAspectRatio); });
    }
    
    void setupSampleDiagram() {
        // สร้าง Flowchart
        auto* start  = new NodeItem("Start", QColor("#27ae60"));
        auto* input  = new NodeItem("Input Data", QColor("#3498db"));
        auto* process = new NodeItem("Process", QColor("#9b59b6"));
        auto* check  = new NodeItem("Valid?", QColor("#f39c12"));
        auto* output = new NodeItem("Output", QColor("#3498db"));
        auto* end    = new NodeItem("End", QColor("#e74c3c"));
        
        start->setPos(0, -200);
        input->setPos(0, -100);
        process->setPos(0, 0);
        check->setPos(0, 100);
        output->setPos(0, 200);
        end->setPos(0, 300);
        
        scene->addItem(start);
        scene->addItem(input);
        scene->addItem(process);
        scene->addItem(check);
        scene->addItem(output);
        scene->addItem(end);
        
        // Edges
        auto addEdge = [&](NodeItem* from, NodeItem* to) {
            scene->addItem(new EdgeItem(from, to));
        };
        
        addEdge(start, input);
        addEdge(input, process);
        addEdge(process, check);
        addEdge(check, output);
        addEdge(output, end);
    }
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    DiagramEditor editor;
    editor.show();
    return app.exec();
}
```

---

## สรุป Part 023

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt Multimedia - Media Player
2. ✅ QGraphicsScene & QGraphicsView
3. ✅ Custom QGraphicsItem
4. ✅ Diagram Editor

---

⬅️ [Part 022](part022.md) | ➡️ [Part 024: Advanced Qt Patterns](part024.md)
