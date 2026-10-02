# Part 049: World-Class Level — High Performance Qt

## ขั้นตอนที่ 701-715

---

## ขั้นตอนที่ 701: Performance Principles

```
Rule of thumb for Qt Performance:
  - Measure first (profiler), optimize second
  - Model updates: batch with begin/end Reset/Insert/Remove
  - Painting: cache pixmaps, avoid translucent widgets
  - Signals: avoid chained signals (use direct connections)
  - Memory: pool allocators, avoid heap fragmentation
  - Network: HTTP/2, connection reuse, response caching
  - SQL: prepared statements, indexes, connection pool
  - Threads: use QtConcurrent for CPU-bound, not I/O-bound

Tools:
  - Valgrind/Callgrind: CPU profiling
  - Heaptrack: memory profiling
  - QtCreator Profiler: UI profiling
  - perf: Linux system profiler
```

---

## ขั้นตอนที่ 702: Fast Item View

```cpp
// === Virtualized list view for 1M+ items ===
class FastListModel : public QAbstractListModel {
    Q_OBJECT
    
public:
    explicit FastListModel(QObject* parent = nullptr) : QAbstractListModel(parent) {}
    
    void setItemCount(int count) {
        beginResetModel();
        m_count = count;
        endResetModel();
    }
    
    void setDataGenerator(std::function<QString(int)> gen) {
        m_generator = std::move(gen);
    }
    
    int rowCount(const QModelIndex&) const override { return m_count; }
    
    QVariant data(const QModelIndex& idx, int role) const override {
        if (!idx.isValid() || idx.row() >= m_count) return {};
        
        if (role == Qt::DisplayRole || role == Qt::UserRole) {
            // Check cache first
            if (m_cache.contains(idx.row())) {
                return m_cache[idx.row()];
            }
            
            // Generate and cache
            QString value = m_generator ? m_generator(idx.row())
                                        : QString("Item %1").arg(idx.row());
            
            // LRU cache management
            if (m_cache.size() >= m_cacheSize) {
                m_cache.remove(m_cache.begin().key());
            }
            m_cache[idx.row()] = value;
            return value;
        }
        return {};
    }
    
    // Batch update: notify range instead of individual items
    void updateRange(int from, int to) {
        // Clear affected cache entries
        for (int i = from; i <= to; ++i) m_cache.remove(i);
        
        emit dataChanged(index(from), index(to));
    }
    
private:
    int m_count = 0;
    std::function<QString(int)> m_generator;
    mutable QMap<int, QString> m_cache;
    int m_cacheSize = 10000;
};

// === High-performance table: flat vector storage ===
class FastTableModel : public QAbstractTableModel {
    Q_OBJECT
    
public:
    struct Column {
        QString header;
        Qt::Alignment align = Qt::AlignLeft;
    };
    
    FastTableModel(int rows, int cols, QObject* parent = nullptr)
        : QAbstractTableModel(parent)
        , m_rows(rows), m_cols(cols)
        , m_data(rows * cols)  // flat vector: row-major
    {}
    
    int rowCount(const QModelIndex&) const override { return m_rows; }
    int columnCount(const QModelIndex&) const override { return m_cols; }
    
    QVariant data(const QModelIndex& idx, int role) const override {
        if (!idx.isValid()) return {};
        
        if (role == Qt::DisplayRole || role == Qt::EditRole) {
            return m_data[idx.row() * m_cols + idx.column()];
        }
        
        if (role == Qt::TextAlignmentRole && idx.column() < m_columns.size()) {
            return static_cast<int>(m_columns[idx.column()].align);
        }
        
        return {};
    }
    
    bool setData(const QModelIndex& idx, const QVariant& value, int role) override {
        if (!idx.isValid() || role != Qt::EditRole) return false;
        m_data[idx.row() * m_cols + idx.column()] = value;
        emit dataChanged(idx, idx, {role});
        return true;
    }
    
    QVariant headerData(int section, Qt::Orientation orientation, int role) const override {
        if (role != Qt::DisplayRole) return {};
        
        if (orientation == Qt::Horizontal) {
            if (section < m_columns.size()) return m_columns[section].header;
            return QString("Col %1").arg(section);
        }
        return section + 1;
    }
    
    Qt::ItemFlags flags(const QModelIndex& idx) const override {
        if (!idx.isValid()) return Qt::NoItemFlags;
        return Qt::ItemIsEnabled | Qt::ItemIsSelectable | Qt::ItemIsEditable;
    }
    
    void setColumns(const QList<Column>& cols) { m_columns = cols; }
    
    // Batch fill
    void fillColumn(int col, std::function<QVariant(int)> gen) {
        for (int row = 0; row < m_rows; ++row) {
            m_data[row * m_cols + col] = gen(row);
        }
        emit dataChanged(index(0, col), index(m_rows - 1, col));
    }
    
    void fillAll(std::function<QVariant(int, int)> gen) {
        for (int r = 0; r < m_rows; ++r) {
            for (int c = 0; c < m_cols; ++c) {
                m_data[r * m_cols + c] = gen(r, c);
            }
        }
        emit dataChanged(index(0, 0), index(m_rows - 1, m_cols - 1));
    }
    
private:
    int m_rows, m_cols;
    std::vector<QVariant> m_data;
    QList<Column> m_columns;
};
```

---

## ขั้นตอนที่ 703: Custom Delegate สำหรับ Performance

```cpp
// === Fast custom delegate (minimal repaint) ===
class FastDelegate : public QStyledItemDelegate {
    Q_OBJECT
    
public:
    explicit FastDelegate(QObject* parent = nullptr) : QStyledItemDelegate(parent) {}
    
    void paint(QPainter* painter, const QStyleOptionViewItem& option,
               const QModelIndex& index) const override {
        // Save state once
        painter->save();
        
        bool selected = option.state & QStyle::State_Selected;
        bool hovered  = option.state & QStyle::State_MouseOver;
        
        QRect r = option.rect;
        
        // Background
        if (selected) {
            painter->fillRect(r, QColor("#3498db"));
        } else if (hovered) {
            painter->fillRect(r, QColor("#e8f4fd"));
        } else {
            painter->fillRect(r, (index.row() % 2 == 0) ? Qt::white : QColor("#f9f9f9"));
        }
        
        // Content without calling base class (faster)
        QString text = index.data(Qt::DisplayRole).toString();
        
        QColor textColor = selected ? Qt::white : Qt::black;
        painter->setPen(textColor);
        painter->setFont(option.font);
        
        QRect textRect = r.adjusted(8, 0, -8, 0);
        painter->drawText(textRect, Qt::AlignVCenter | Qt::AlignLeft, text);
        
        // Bottom separator line
        painter->setPen(QPen(QColor("#e8e8e8"), 1));
        painter->drawLine(r.bottomLeft(), r.bottomRight());
        
        painter->restore();
    }
    
    QSize sizeHint(const QStyleOptionViewItem&, const QModelIndex&) const override {
        return QSize(0, 36);  // Fixed height = faster layout
    }
};
```

---

## ขั้นตอนที่ 704: Object Pool

```cpp
// === Memory pool for frequent object creation ===
template<typename T, int PoolSize = 1024>
class ObjectPool {
public:
    ObjectPool() {
        m_pool.reserve(PoolSize);
        for (int i = 0; i < PoolSize; ++i) {
            m_pool.push_back(std::make_unique<T>());
            m_free.push_back(m_pool.back().get());
        }
    }
    
    T* acquire() {
        if (m_free.empty()) {
            // Pool exhausted: allocate new (slow path)
            m_pool.push_back(std::make_unique<T>());
            return m_pool.back().get();
        }
        
        T* obj = m_free.back();
        m_free.pop_back();
        return obj;
    }
    
    void release(T* obj) {
        obj->~T();
        new (obj) T();  // Placement new to reset state
        m_free.push_back(obj);
    }
    
    int available() const { return m_free.size(); }
    int total() const { return m_pool.size(); }
    
private:
    std::vector<std::unique_ptr<T>> m_pool;
    std::vector<T*> m_free;
};

// === RAII pool handle ===
template<typename T>
class PoolHandle {
public:
    PoolHandle(ObjectPool<T>& pool)
        : m_pool(&pool), m_obj(pool.acquire()) {}
    
    ~PoolHandle() { if (m_obj) m_pool->release(m_obj); }
    
    PoolHandle(const PoolHandle&) = delete;
    PoolHandle(PoolHandle&& other) noexcept
        : m_pool(other.m_pool), m_obj(other.m_obj) {
        other.m_obj = nullptr;
    }
    
    T* get() { return m_obj; }
    T& operator*() { return *m_obj; }
    T* operator->() { return m_obj; }
    
private:
    ObjectPool<T>* m_pool;
    T* m_obj;
};

// Usage:
// struct Particle { float x, y, vx, vy; };
// ObjectPool<Particle> particlePool(10000);
// auto handle = PoolHandle<Particle>(particlePool);
// handle->x = 100.0f;
```

---

## ขั้นตอนที่ 705: Cache-Friendly Data Processing

```cpp
// === Structure of Arrays vs Array of Structures ===
struct ParticleSystem_SoA {
    // SoA layout — cache friendly for individual property access
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
    std::vector<float> life;  // remaining lifetime
    
    void resize(int n) {
        x.resize(n); y.resize(n); z.resize(n);
        vx.resize(n); vy.resize(n); vz.resize(n);
        mass.resize(n, 1.0f);
        life.resize(n, 1.0f);
    }
    
    // Process all positions in tight loop (cache friendly)
    void updatePositions(float dt) {
        int n = x.size();
        for (int i = 0; i < n; ++i) {
            x[i] += vx[i] * dt;
            y[i] += vy[i] * dt;
            z[i] += vz[i] * dt;
        }
    }
    
    // Remove dead particles efficiently
    void removeDeadParticles() {
        int alive = 0;
        int n = x.size();
        
        for (int i = 0; i < n; ++i) {
            if (life[i] > 0) {
                if (i != alive) {
                    x[alive] = x[i]; y[alive] = y[i]; z[alive] = z[i];
                    vx[alive] = vx[i]; vy[alive] = vy[i]; vz[alive] = vz[i];
                    life[alive] = life[i];
                }
                ++alive;
            }
        }
        
        x.resize(alive); y.resize(alive); z.resize(alive);
        vx.resize(alive); vy.resize(alive); vz.resize(alive);
        life.resize(alive);
    }
    
    // SIMD-friendly update (compiler vectorizable)
    void decreaseLife(float dt) {
        for (float& l : life) l -= dt;
    }
    
    // Batch rendering
    void renderToScene(QGraphicsScene* scene) {
        // Create/update graphics items in batch
        // Real implementation would use custom painter or OpenGL
    }
    
    int count() const { return x.size(); }
};

// === Parallel particle update with QtConcurrent ===
void updateParticlesParallel(ParticleSystem_SoA& ps, float dt) {
    int n = ps.x.size();
    int numCores = QThread::idealThreadCount();
    int chunkSize = n / numCores;
    
    QtConcurrent::blockingMap(
        QList<int>(numCores),
        [&, chunkSize, n, dt](int core) {
            int start = core * chunkSize;
            int end = (core == numCores - 1) ? n : start + chunkSize;
            
            for (int i = start; i < end; ++i) {
                ps.x[i] += ps.vx[i] * dt;
                ps.y[i] += ps.vy[i] * dt;
                ps.life[i] -= dt;
            }
        }
    );
}
```

---

## ขั้นตอนที่ 706-715: Real-time Dashboard

```cpp
// === Real-time chart with fast updates ===
class RealTimeDashboard : public QMainWindow {
    Q_OBJECT
    
public:
    RealTimeDashboard(QWidget* parent = nullptr) : QMainWindow(parent) {
        setWindowTitle("Real-time Dashboard");
        setMinimumSize(1200, 700);
        
        setupUi();
        setupCharts();
        
        // Simulate real-time data
        auto* dataTimer = new QTimer(this);
        connect(dataTimer, &QTimer::timeout, this, &RealTimeDashboard::updateData);
        dataTimer->start(50);  // 20 FPS
        
        // Stats update
        auto* statsTimer = new QTimer(this);
        connect(statsTimer, &QTimer::timeout, this, &RealTimeDashboard::updateStats);
        statsTimer->start(1000);  // 1 Hz
    }
    
private:
    void setupUi() {
        auto* central = new QWidget();
        auto* layout = new QGridLayout(central);
        setCentralWidget(central);
        
        // 4 chart panels
        for (int i = 0; i < 4; ++i) {
            auto* view = new QChartView();
            view->setRenderHint(QPainter::Antialiasing);
            view->setMinimumSize(400, 250);
            
            chartViews[i] = view;
            layout->addWidget(view, i / 2, i % 2);
        }
        
        // Status bar
        fpsLabel = new QLabel("FPS: --");
        pointsLabel = new QLabel("Points: --");
        memLabel = new QLabel("Memory: --");
        statusBar()->addWidget(fpsLabel);
        statusBar()->addWidget(pointsLabel);
        statusBar()->addWidget(memLabel);
    }
    
    void setupCharts() {
        // CPU chart
        cpuChart = new QChart();
        cpuChart->setTitle("CPU Usage (%)");
        cpuChart->legend()->hide();
        cpuChart->setBackgroundBrush(Qt::white);
        
        cpuSeries = new QLineSeries();
        cpuSeries->setColor(QColor("#e74c3c"));
        cpuSeries->setPen(QPen(QColor("#e74c3c"), 2));
        cpuChart->addSeries(cpuSeries);
        
        auto* cpuAxisX = new QValueAxis();
        cpuAxisX->setRange(0, 100);
        cpuAxisX->setLabelsVisible(false);
        
        auto* cpuAxisY = new QValueAxis();
        cpuAxisY->setRange(0, 100);
        cpuAxisY->setTitleText("%");
        
        cpuChart->addAxis(cpuAxisX, Qt::AlignBottom);
        cpuChart->addAxis(cpuAxisY, Qt::AlignLeft);
        cpuSeries->attachAxis(cpuAxisX);
        cpuSeries->attachAxis(cpuAxisY);
        
        chartViews[0]->setChart(cpuChart);
        
        // Memory chart
        memChart = new QChart();
        memChart->setTitle("Memory (MB)");
        memChart->legend()->hide();
        
        memSeries = new QAreaSeries();
        auto* memLine = new QLineSeries();
        memSeries->setUpperSeries(memLine);
        memSeries->setColor(QColor(52, 152, 219, 100));
        memSeries->setBorderColor(QColor("#3498db"));
        
        memChart->addSeries(memSeries);
        
        auto* memAxisX = new QValueAxis();
        memAxisX->setRange(0, 100);
        memAxisX->setLabelsVisible(false);
        
        auto* memAxisY = new QValueAxis();
        memAxisY->setRange(0, 2048);
        memAxisY->setTitleText("MB");
        
        memChart->addAxis(memAxisX, Qt::AlignBottom);
        memChart->addAxis(memAxisY, Qt::AlignLeft);
        memSeries->attachAxis(memAxisX);
        memSeries->attachAxis(memAxisY);
        
        chartViews[1]->setChart(memChart);
        
        // Network chart (area)
        networkChart = new QChart();
        networkChart->setTitle("Network (KB/s)");
        networkChart->legend()->hide();
        
        networkSeries = new QLineSeries();
        networkSeries->setColor(QColor("#27ae60"));
        networkChart->addSeries(networkSeries);
        
        auto* netAxisX = new QValueAxis();
        netAxisX->setRange(0, 100);
        netAxisX->setLabelsVisible(false);
        
        auto* netAxisY = new QValueAxis();
        netAxisY->setRange(0, 10000);
        netAxisY->setTitleText("KB/s");
        
        networkChart->addAxis(netAxisX, Qt::AlignBottom);
        networkChart->addAxis(netAxisY, Qt::AlignLeft);
        networkSeries->attachAxis(netAxisX);
        networkSeries->attachAxis(netAxisY);
        
        chartViews[2]->setChart(networkChart);
        
        // FPS chart
        fpsChart = new QChart();
        fpsChart->setTitle("Frame Rate (FPS)");
        fpsChart->legend()->hide();
        
        fpsSeries = new QLineSeries();
        fpsSeries->setColor(QColor("#9b59b6"));
        fpsChart->addSeries(fpsSeries);
        
        auto* fpsAxisX = new QValueAxis();
        fpsAxisX->setRange(0, 100);
        fpsAxisX->setLabelsVisible(false);
        
        auto* fpsAxisY = new QValueAxis();
        fpsAxisY->setRange(0, 120);
        fpsAxisY->setTitleText("FPS");
        
        fpsChart->addAxis(fpsAxisX, Qt::AlignBottom);
        fpsChart->addAxis(fpsAxisY, Qt::AlignLeft);
        fpsSeries->attachAxis(fpsAxisX);
        fpsSeries->attachAxis(fpsAxisY);
        
        chartViews[3]->setChart(fpsChart);
    }
    
    void updateData() {
        ++m_tick;
        
        // Simulate metrics
        double cpu = 30 + 20 * std::sin(m_tick * 0.1) + (rand() % 10 - 5);
        double mem = 512 + 200 * std::sin(m_tick * 0.05) + (rand() % 50);
        double net = 1000 + 3000 * std::abs(std::sin(m_tick * 0.08)) + (rand() % 500);
        
        // Calculate actual FPS
        static QElapsedTimer fpsTimer;
        static int frameCount = 0;
        ++frameCount;
        
        if (!fpsTimer.isValid()) fpsTimer.start();
        
        double actualFps = frameCount / (fpsTimer.elapsed() / 1000.0);
        
        // Append and scroll
        appendAndScroll(cpuSeries, m_tick, cpu);
        appendAndScroll(networkSeries, m_tick, net);
        appendAndScroll(fpsSeries, m_tick, actualFps);
        
        // Memory series (upper series of AreaSeries)
        if (auto* upper = memSeries->upperSeries()) {
            auto* lineSeries = static_cast<QLineSeries*>(upper);
            appendAndScroll(lineSeries, m_tick, mem);
        }
        
        // Update axes
        if (m_tick > 100) {
            static_cast<QValueAxis*>(cpuChart->axes(Qt::Horizontal).first())
                ->setRange(m_tick - 100, m_tick);
            static_cast<QValueAxis*>(networkChart->axes(Qt::Horizontal).first())
                ->setRange(m_tick - 100, m_tick);
            static_cast<QValueAxis*>(memChart->axes(Qt::Horizontal).first())
                ->setRange(m_tick - 100, m_tick);
            static_cast<QValueAxis*>(fpsChart->axes(Qt::Horizontal).first())
                ->setRange(m_tick - 100, m_tick);
        }
    }
    
    void appendAndScroll(QLineSeries* series, double x, double y) {
        series->append(x, y);
        if (series->count() > 150) {
            series->remove(0);
        }
    }
    
    void updateStats() {
        static QElapsedTimer t;
        static int lastTick = 0;
        
        if (!t.isValid()) { t.start(); lastTick = m_tick; return; }
        
        double fps = (m_tick - lastTick) / (t.elapsed() / 1000.0);
        t.restart();
        lastTick = m_tick;
        
        fpsLabel->setText(QString("FPS: %1").arg(fps, 0, 'f', 1));
        pointsLabel->setText(QString("Points: %1").arg(
            cpuSeries->count() + memSeries->upperSeries()->count() +
            networkSeries->count() + fpsSeries->count()));
    }
    
    QChartView* chartViews[4];
    QChart* cpuChart;
    QChart* memChart;
    QChart* networkChart;
    QChart* fpsChart;
    QLineSeries* cpuSeries;
    QAreaSeries* memSeries;
    QLineSeries* networkSeries;
    QLineSeries* fpsSeries;
    QLabel* fpsLabel;
    QLabel* pointsLabel;
    QLabel* memLabel;
    int m_tick = 0;
};
```

---

## สรุป Part 049

ใน Part นี้คุณได้เรียนรู้:

1. ✅ FastListModel ด้วย LRU cache สำหรับ 1M+ items
2. ✅ FastTableModel ด้วย flat vector storage (cache-friendly)
3. ✅ Fast custom delegate ที่ไม่ใช้ QStyleOptionViewItem
4. ✅ Object Pool pattern สำหรับ frequent allocations
5. ✅ SoA (Structure of Arrays) ด้วย parallel processing
6. ✅ Real-time dashboard ด้วย QChart + 20FPS update

---

⬅️ [Part 048](part048.md) | ➡️ [Part 050: World-Class Level — Qt OpenGL & Shader](part050.md)
