# Part 061: Advanced C++ — SIMD Optimization & Profiling

## ขั้นตอนที่ 881-895

---

## ขั้นตอนที่ 881: SIMD Fundamentals

```cpp
// SIMD (Single Instruction, Multiple Data)
// ทำให้ CPU คำนวณข้อมูลหลายชุดพร้อมกันในคำสั่งเดียว

// Intel SSE/AVX:
//   SSE:  128-bit (4 floats หรือ 2 doubles ต่อครั้ง)
//   AVX:  256-bit (8 floats หรือ 4 doubles ต่อครั้ง)
//   AVX-512: 512-bit (16 floats ต่อครั้ง)

// ARM NEON (Raspberry Pi, Apple M1/M2):
//   128-bit (4 floats ต่อครั้ง)

// CMake: เปิด SIMD
// target_compile_options(app PRIVATE -mavx2 -mfma)

#include <immintrin.h>  // Intel AVX/SSE
#include <cstring>
#include <vector>
#include <chrono>

// Scalar (ทั่วไป) vs SIMD comparison
namespace Scalar {
    void addArrays(const float* a, const float* b, float* out, int n) {
        for (int i = 0; i < n; ++i) {
            out[i] = a[i] + b[i];
        }
    }
    
    float dotProduct(const float* a, const float* b, int n) {
        float sum = 0;
        for (int i = 0; i < n; ++i) sum += a[i] * b[i];
        return sum;
    }
}

namespace Simd {
    // AVX: ประมวลผล 8 float ต่อครั้ง
    void addArrays_AVX(const float* a, const float* b, float* out, int n) {
        int i = 0;
        
        // Process 8 floats at a time
        for (; i <= n - 8; i += 8) {
            __m256 va = _mm256_loadu_ps(a + i);
            __m256 vb = _mm256_loadu_ps(b + i);
            __m256 vr = _mm256_add_ps(va, vb);
            _mm256_storeu_ps(out + i, vr);
        }
        
        // Handle remaining elements
        for (; i < n; ++i) {
            out[i] = a[i] + b[i];
        }
    }
    
    float dotProduct_AVX(const float* a, const float* b, int n) {
        __m256 sum = _mm256_setzero_ps();
        int i = 0;
        
        for (; i <= n - 8; i += 8) {
            __m256 va = _mm256_loadu_ps(a + i);
            __m256 vb = _mm256_loadu_ps(b + i);
            sum = _mm256_fmadd_ps(va, vb, sum);  // Fused multiply-add
        }
        
        // Horizontal sum of 8 floats
        __m128 lo = _mm256_castps256_ps128(sum);
        __m128 hi = _mm256_extractf128_ps(sum, 1);
        __m128 tmp = _mm_add_ps(lo, hi);
        tmp = _mm_hadd_ps(tmp, tmp);
        tmp = _mm_hadd_ps(tmp, tmp);
        float result = _mm_cvtss_f32(tmp);
        
        // Handle remaining
        for (; i < n; ++i) result += a[i] * b[i];
        
        return result;
    }
    
    // SIMD-optimized RGB → Grayscale
    void rgbToGray_AVX(const uint8_t* rgb, uint8_t* gray, int pixelCount) {
        // Y = 0.299*R + 0.587*G + 0.114*B
        const __m256 wr = _mm256_set1_ps(0.299f);
        const __m256 wg = _mm256_set1_ps(0.587f);
        const __m256 wb = _mm256_set1_ps(0.114f);
        
        int i = 0;
        for (; i <= pixelCount - 8; i += 8) {
            // Load 8 pixels (each 3 bytes: R, G, B)
            float r[8], g[8], b[8];
            for (int j = 0; j < 8; ++j) {
                r[j] = rgb[(i + j) * 3];
                g[j] = rgb[(i + j) * 3 + 1];
                b[j] = rgb[(i + j) * 3 + 2];
            }
            
            __m256 vr = _mm256_loadu_ps(r);
            __m256 vg = _mm256_loadu_ps(g);
            __m256 vb = _mm256_loadu_ps(b);
            
            __m256 vy = _mm256_fmadd_ps(vr, wr,
                         _mm256_fmadd_ps(vg, wg,
                          _mm256_mul_ps(vb, wb)));
            
            float ybuf[8];
            _mm256_storeu_ps(ybuf, vy);
            
            for (int j = 0; j < 8; ++j) {
                gray[i + j] = static_cast<uint8_t>(ybuf[j]);
            }
        }
        
        for (; i < pixelCount; ++i) {
            gray[i] = static_cast<uint8_t>(
                0.299f * rgb[i*3] + 0.587f * rgb[i*3+1] + 0.114f * rgb[i*3+2]);
        }
    }
}
```

---

## ขั้นตอนที่ 882: Memory-Aligned Allocations

```cpp
// Aligned memory for SIMD efficiency
#include <cstdlib>
#include <new>

template<typename T, std::size_t Alignment = 32>
class AlignedVector {
public:
    AlignedVector() = default;
    
    explicit AlignedVector(std::size_t n) : m_size(n) {
        m_data = static_cast<T*>(allocate(n));
        for (std::size_t i = 0; i < n; ++i) new(m_data + i) T{};
    }
    
    AlignedVector(std::size_t n, T fill) : m_size(n) {
        m_data = static_cast<T*>(allocate(n));
        for (std::size_t i = 0; i < n; ++i) new(m_data + i) T(fill);
    }
    
    ~AlignedVector() {
        for (std::size_t i = 0; i < m_size; ++i) m_data[i].~T();
        std::free(m_data);
    }
    
    AlignedVector(const AlignedVector&) = delete;
    AlignedVector& operator=(const AlignedVector&) = delete;
    
    T* data() { return m_data; }
    const T* data() const { return m_data; }
    std::size_t size() const { return m_size; }
    
    T& operator[](std::size_t i) { return m_data[i]; }
    const T& operator[](std::size_t i) const { return m_data[i]; }
    
    T* begin() { return m_data; }
    T* end() { return m_data + m_size; }
    
private:
    void* allocate(std::size_t n) {
        void* ptr;
#ifdef _WIN32
        ptr = _aligned_malloc(n * sizeof(T), Alignment);
        if (!ptr) throw std::bad_alloc{};
#else
        if (posix_memalign(&ptr, Alignment, n * sizeof(T)) != 0)
            throw std::bad_alloc{};
#endif
        return ptr;
    }
    
    T* m_data = nullptr;
    std::size_t m_size = 0;
};

// Usage: 32-byte aligned float array for AVX
// AlignedVector<float, 32> data(1024, 0.0f);
```

---

## ขั้นตอนที่ 883: Profiling Tools Integration

```cpp
// profiler/profiler.h
#pragma once
#include <QElapsedTimer>
#include <QMap>
#include <QMutex>
#include <QString>
#include <QDebug>
#include <functional>

class Profiler {
public:
    struct Record {
        qint64 totalNs = 0;
        qint64 minNs = std::numeric_limits<qint64>::max();
        qint64 maxNs = 0;
        int count = 0;
        
        double avgMs() const { return count ? (totalNs / 1e6) / count : 0; }
        double totalMs() const { return totalNs / 1e6; }
        double minMs() const { return minNs / 1e6; }
        double maxMs() const { return maxNs / 1e6; }
    };
    
    static Profiler& instance() {
        static Profiler p;
        return p;
    }
    
    class Scope {
    public:
        explicit Scope(const QString& name) : m_name(name) {
            m_timer.start();
        }
        
        ~Scope() {
            qint64 ns = m_timer.nsecsElapsed();
            Profiler::instance().record(m_name, ns);
        }
        
    private:
        QString m_name;
        QElapsedTimer m_timer;
    };
    
    void record(const QString& name, qint64 ns) {
        QMutexLocker lock(&m_mutex);
        auto& r = m_records[name];
        r.totalNs += ns;
        r.count++;
        r.minNs = qMin(r.minNs, ns);
        r.maxNs = qMax(r.maxNs, ns);
    }
    
    template<typename F>
    auto measure(const QString& name, F&& fn) -> decltype(fn()) {
        Scope scope(name);
        return fn();
    }
    
    void reset() {
        QMutexLocker lock(&m_mutex);
        m_records.clear();
    }
    
    void printReport() const {
        QMutexLocker lock(&m_mutex);
        qDebug() << "\n=== Performance Report ===";
        
        // Sort by total time descending
        auto sorted = m_records.keys();
        std::sort(sorted.begin(), sorted.end(), [&](const QString& a, const QString& b) {
            return m_records[a].totalNs > m_records[b].totalNs;
        });
        
        for (const auto& name : sorted) {
            const auto& r = m_records[name];
            qDebug().noquote()
                << QString("[%1] calls=%2 total=%3ms avg=%4ms min=%5ms max=%6ms")
                   .arg(name, -40)
                   .arg(r.count, 6)
                   .arg(r.totalMs(), 8, 'f', 3)
                   .arg(r.avgMs(), 8, 'f', 3)
                   .arg(r.minMs(), 8, 'f', 3)
                   .arg(r.maxMs(), 8, 'f', 3);
        }
    }
    
    QMap<QString, Record> records() const {
        QMutexLocker lock(&m_mutex);
        return m_records;
    }
    
private:
    mutable QMutex m_mutex;
    QMap<QString, Record> m_records;
};

// Convenience macro
#define PROFILE(name) Profiler::Scope _prof_scope(name)
#define PROFILE_FN()  Profiler::Scope _prof_scope(__func__)

// ProfilerWidget for in-app display
class ProfilerWidget : public QWidget {
    Q_OBJECT
    
public:
    explicit ProfilerWidget(QWidget* parent = nullptr) : QWidget(parent) {
        auto* layout = new QVBoxLayout(this);
        
        auto* toolbar = new QHBoxLayout;
        auto* refreshBtn = new QPushButton("รีเฟรช", this);
        auto* resetBtn = new QPushButton("รีเซ็ต", this);
        auto* exportBtn = new QPushButton("ส่งออก", this);
        toolbar->addWidget(refreshBtn);
        toolbar->addWidget(resetBtn);
        toolbar->addStretch();
        toolbar->addWidget(exportBtn);
        
        m_table = new QTableWidget(0, 6, this);
        m_table->setHorizontalHeaderLabels({
            "ชื่อ", "เรียก (ครั้ง)", "รวม (ms)", "เฉลี่ย (ms)", "น้อยสุด", "มากสุด"
        });
        m_table->horizontalHeader()->setSectionResizeMode(0, QHeaderView::Stretch);
        m_table->setSortingEnabled(true);
        m_table->setEditTriggers(QAbstractItemView::NoEditTriggers);
        
        layout->addLayout(toolbar);
        layout->addWidget(m_table);
        
        // Auto-refresh
        auto* timer = new QTimer(this);
        timer->setInterval(1000);
        connect(timer, &QTimer::timeout, this, &ProfilerWidget::refresh);
        timer->start();
        
        connect(refreshBtn, &QPushButton::clicked, this, &ProfilerWidget::refresh);
        connect(resetBtn, &QPushButton::clicked, this, [=] {
            Profiler::instance().reset();
            refresh();
        });
        
        refresh();
    }
    
public slots:
    void refresh() {
        auto records = Profiler::instance().records();
        
        m_table->setRowCount(0);
        
        for (auto it = records.begin(); it != records.end(); ++it) {
            int row = m_table->rowCount();
            m_table->insertRow(row);
            
            const auto& r = it.value();
            
            m_table->setItem(row, 0, new QTableWidgetItem(it.key()));
            m_table->setItem(row, 1, new QTableWidgetItem(QString::number(r.count)));
            m_table->setItem(row, 2, new QTableWidgetItem(
                QString::number(r.totalMs(), 'f', 3)));
            m_table->setItem(row, 3, new QTableWidgetItem(
                QString::number(r.avgMs(), 'f', 3)));
            m_table->setItem(row, 4, new QTableWidgetItem(
                QString::number(r.minMs(), 'f', 3)));
            m_table->setItem(row, 5, new QTableWidgetItem(
                QString::number(r.maxMs(), 'f', 3)));
            
            // Color-code hot paths
            double avg = r.avgMs();
            QColor bgColor = avg > 16 ? QColor("#FDEDEC")
                           : avg > 8  ? QColor("#FEF9E7")
                                      : Qt::white;
            
            for (int col = 0; col < 6; ++col) {
                if (auto* item = m_table->item(row, col)) {
                    item->setBackground(bgColor);
                    item->setTextAlignment(col == 0 ? Qt::AlignLeft | Qt::AlignVCenter
                                                    : Qt::AlignRight | Qt::AlignVCenter);
                }
            }
        }
        
        m_table->sortByColumn(2, Qt::DescendingOrder);
    }
    
private:
    QTableWidget* m_table;
};
```

---

## ขั้นตอนที่ 884: Memory Leak Detection

```cpp
// debug/memorytracker.h
#pragma once
#include <QObject>
#include <QMap>
#include <QMutex>
#include <QDebug>

#ifdef QT_DEBUG

class MemoryTracker {
public:
    struct Allocation {
        std::size_t size;
        QString file;
        int line;
        QString function;
        QDateTime time;
    };
    
    static MemoryTracker& instance() {
        static MemoryTracker t;
        return t;
    }
    
    void track(void* ptr, std::size_t size,
               const char* file, int line, const char* func)
    {
        QMutexLocker lock(&m_mutex);
        m_allocations[ptr] = {size, file, line, func,
                               QDateTime::currentDateTime()};
        m_totalAllocated += size;
        m_allocationCount++;
    }
    
    void untrack(void* ptr) {
        QMutexLocker lock(&m_mutex);
        if (m_allocations.contains(ptr)) {
            m_totalAllocated -= m_allocations[ptr].size;
            m_allocations.remove(ptr);
        }
    }
    
    void printLeaks() const {
        QMutexLocker lock(&m_mutex);
        
        if (m_allocations.isEmpty()) {
            qDebug() << "No memory leaks detected!";
            return;
        }
        
        qDebug() << "\n=== MEMORY LEAKS DETECTED ===";
        qDebug() << "Leaked allocations:" << m_allocations.size();
        
        std::size_t total = 0;
        for (auto it = m_allocations.begin(); it != m_allocations.end(); ++it) {
            const auto& a = it.value();
            total += a.size;
            qDebug() << QString("  %1 bytes at %2:%3 [%4]")
                .arg(a.size)
                .arg(a.file)
                .arg(a.line)
                .arg(a.function);
        }
        
        qDebug() << "Total leaked:" << total << "bytes";
    }
    
    std::size_t totalAllocated() const { return m_totalAllocated; }
    int allocationCount() const { return m_allocationCount; }
    
private:
    mutable QMutex m_mutex;
    QMap<void*, Allocation> m_allocations;
    std::size_t m_totalAllocated = 0;
    int m_allocationCount = 0;
};

// Override global new/delete (use with care)
#define TRACK_ALLOC(size)        MemoryTracker::instance().track
#define TRACK_FREE(ptr)          MemoryTracker::instance().untrack

#endif // QT_DEBUG

// Resource usage monitor
class ResourceMonitor : public QObject {
    Q_OBJECT
    
public:
    struct Stats {
        qint64 memoryUsageMb = 0;
        double cpuPercent = 0;
        int threadCount = 0;
        int openFiles = 0;
    };
    
    explicit ResourceMonitor(QObject* parent = nullptr) : QObject(parent) {
        m_timer = new QTimer(this);
        m_timer->setInterval(2000);
        connect(m_timer, &QTimer::timeout, this, &ResourceMonitor::sample);
        m_timer->start();
    }
    
    Stats currentStats() const { return m_stats; }
    
signals:
    void statsUpdated(const Stats& stats);
    void highMemoryWarning(qint64 mb);
    
private slots:
    void sample() {
        Stats stats;
        
#ifdef Q_OS_LINUX
        // Parse /proc/self/status
        QFile status("/proc/self/status");
        if (status.open(QIODevice::ReadOnly)) {
            for (const auto& line : QString(status.readAll()).split('\n')) {
                if (line.startsWith("VmRSS:")) {
                    stats.memoryUsageMb = line.split(':').last()
                        .trimmed().split(' ').first().toLongLong() / 1024;
                }
                if (line.startsWith("Threads:")) {
                    stats.threadCount = line.split(':').last().trimmed().toInt();
                }
            }
        }
#endif
        
        m_stats = stats;
        emit statsUpdated(stats);
        
        if (stats.memoryUsageMb > 512) {
            emit highMemoryWarning(stats.memoryUsageMb);
        }
    }
    
    Stats m_stats;
    QTimer* m_timer;
};
```

---

## สรุป Part 061

ใน Part นี้คุณได้เรียนรู้:

1. ✅ SIMD ด้วย Intel AVX intrinsics (addArrays, dotProduct, RGB→Grayscale)
2. ✅ Aligned memory allocation สำหรับ SIMD efficiency
3. ✅ Profiler ด้วย RAII Scope, macro `PROFILE()`, ProfilerWidget
4. ✅ Memory leak detection ด้วย MemoryTracker
5. ✅ ResourceMonitor สำหรับ RAM + CPU + thread count

---

⬅️ [Part 060](part060.md) | ➡️ [Part 062: Advanced — Lock-Free Data Structures](part062.md)
