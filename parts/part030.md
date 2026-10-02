# Part 030: Performance Optimization

## ขั้นตอนที่ 416-430

---

## ขั้นตอนที่ 416: Profiling & Benchmarking

```cpp
#include <chrono>
#include <iostream>
#include <functional>
#include <vector>

// === High-resolution timer ===
class Timer {
public:
    using Clock = std::chrono::high_resolution_clock;
    using TimePoint = Clock::time_point;
    using Duration = std::chrono::duration<double, std::milli>;
    
    Timer() { reset(); }
    
    void reset() { m_start = Clock::now(); }
    
    double elapsedMs() const {
        return Duration(Clock::now() - m_start).count();
    }
    
    double elapsedUs() const {
        using Micro = std::chrono::duration<double, std::micro>;
        return Micro(Clock::now() - m_start).count();
    }
    
    void report(const std::string& label) const {
        std::cout << label << ": " << elapsedMs() << " ms\n";
    }
    
private:
    TimePoint m_start;
};

// === Benchmark utility ===
struct BenchResult {
    std::string name;
    double meanMs;
    double minMs;
    double maxMs;
    int iterations;
};

BenchResult benchmark(const std::string& name, int iterations, std::function<void()> fn) {
    // Warmup
    for (int i = 0; i < 3; i++) fn();
    
    std::vector<double> times;
    times.reserve(iterations);
    
    for (int i = 0; i < iterations; i++) {
        Timer t;
        fn();
        times.push_back(t.elapsedMs());
    }
    
    double sum = 0, minVal = times[0], maxVal = times[0];
    for (double t : times) {
        sum += t;
        minVal = std::min(minVal, t);
        maxVal = std::max(maxVal, t);
    }
    
    return {name, sum / iterations, minVal, maxVal, iterations};
}

void printBenchResult(const BenchResult& r) {
    std::cout << r.name << ":\n";
    std::cout << "  mean=" << r.meanMs << "ms  min=" << r.minMs << "ms  max=" << r.maxMs << "ms\n";
}

// Benchmark examples
void runBenchmarks() {
    std::vector<int> data(10000);
    std::iota(data.begin(), data.end(), 0);
    
    // Comparison: linear search vs. binary search
    auto r1 = benchmark("linear_search", 1000, [&]() {
        auto it = std::find(data.begin(), data.end(), 9999);
        (void)it;
    });
    
    auto r2 = benchmark("binary_search", 1000, [&]() {
        bool found = std::binary_search(data.begin(), data.end(), 9999);
        (void)found;
    });
    
    printBenchResult(r1);
    printBenchResult(r2);
    std::cout << "Speedup: " << r1.meanMs / r2.meanMs << "x\n";
}
```

---

## ขั้นตอนที่ 417: Memory Layout Optimization

```cpp
// === Structure of Arrays (SoA) vs Array of Structures (AoS) ===

// AoS - poor cache performance for simulation
struct ParticleAoS {
    float x, y, z;       // position
    float vx, vy, vz;    // velocity
    float mass;
    int id;
    bool active;
    // Padding waste
};

// SoA - cache-friendly for SIMD/vectorization
struct ParticleSystemSoA {
    std::vector<float> x, y, z;
    std::vector<float> vx, vy, vz;
    std::vector<float> mass;
    std::vector<int>   id;
    std::vector<bool>  active;
    std::size_t count;
    
    ParticleSystemSoA(std::size_t n) : count(n) {
        x.resize(n);  y.resize(n);  z.resize(n);
        vx.resize(n); vy.resize(n); vz.resize(n);
        mass.resize(n, 1.0f);
        id.resize(n);
        active.resize(n, true);
    }
    
    // Update positions - vectorizable!
    void updatePositions(float dt) {
        for (std::size_t i = 0; i < count; i++) {
            x[i] += vx[i] * dt;
            y[i] += vy[i] * dt;
            z[i] += vz[i] * dt;
        }
    }
    
    // Kinetic energy - all mass[i] accessed sequentially
    double totalKineticEnergy() const {
        double total = 0;
        for (std::size_t i = 0; i < count; i++) {
            double v2 = vx[i]*vx[i] + vy[i]*vy[i] + vz[i]*vz[i];
            total += 0.5 * mass[i] * v2;
        }
        return total;
    }
};

// === Cache-aligned allocation ===
#include <memory>
#include <cstdlib>

template<typename T, std::size_t Alignment = 64>
struct AlignedAllocator {
    using value_type = T;
    
    T* allocate(std::size_t n) {
        void* ptr;
        if (posix_memalign(&ptr, Alignment, n * sizeof(T)) != 0)
            throw std::bad_alloc();
        return static_cast<T*>(ptr);
    }
    
    void deallocate(T* ptr, std::size_t) {
        free(ptr);
    }
};

using AlignedVecF = std::vector<float, AlignedAllocator<float>>;
```

---

## ขั้นตอนที่ 418: Move Semantics & RVO

```cpp
#include <utility>
#include <vector>
#include <string>

// Measure move vs. copy
class HeavyObject {
public:
    std::vector<double> data;
    std::string name;
    
    HeavyObject(std::size_t size, std::string n) 
        : data(size, 3.14), name(std::move(n)) {}
    
    // Copy constructor
    HeavyObject(const HeavyObject& o) : data(o.data), name(o.name) {
        // std::cout << "COPY\n"; // Expensive!
    }
    
    // Move constructor
    HeavyObject(HeavyObject&& o) noexcept 
        : data(std::move(o.data)), name(std::move(o.name)) {
        // std::cout << "MOVE\n"; // Cheap - just pointer swap
    }
    
    HeavyObject& operator=(const HeavyObject& o) {
        if (this != &o) { data = o.data; name = o.name; }
        return *this;
    }
    
    HeavyObject& operator=(HeavyObject&& o) noexcept {
        if (this != &o) { data = std::move(o.data); name = std::move(o.name); }
        return *this;
    }
    
    ~HeavyObject() = default;
};

// RVO (Return Value Optimization) - compiler elides copy
HeavyObject createObject(std::size_t size) {
    return HeavyObject(size, "created"); // No copy - direct construction
}

// NRVO (Named RVO)
HeavyObject createNamedObject(std::size_t size) {
    HeavyObject obj(size, "named"); // Named variable
    // Compiler still optimizes
    return obj;
}

void demoMoveSemantics() {
    Timer t;
    
    // Option 1: Copy (slow)
    t.reset();
    HeavyObject src(100000, "source");
    HeavyObject copy1 = src; // copies 100k doubles
    t.report("Copy");
    
    // Option 2: Move (fast)
    t.reset();
    HeavyObject moved = std::move(src); // Just moves pointers
    t.report("Move");
    
    // Option 3: RVO (even faster - no move/copy at all)
    t.reset();
    HeavyObject rvo = createObject(100000);
    t.report("RVO");
    
    // emplace_back vs push_back
    std::vector<HeavyObject> vec;
    vec.reserve(3);
    
    t.reset();
    vec.push_back(HeavyObject(10000, "push")); // Creates temp, then moves
    t.report("push_back");
    
    t.reset();
    vec.emplace_back(10000, "emplace"); // Constructs in-place
    t.report("emplace_back");
}
```

---

## ขั้นตอนที่ 419: String Optimization

```cpp
#include <string_view>
#include <charconv>

// === Avoid unnecessary string copies ===
class StringProcessor {
public:
    // Bad: copies string
    static bool startsWithBad(const std::string& str, const std::string& prefix) {
        return str.substr(0, prefix.size()) == prefix;
    }
    
    // Good: no copy, no allocation
    static bool startsWith(std::string_view str, std::string_view prefix) {
        return str.size() >= prefix.size() && 
               str.substr(0, prefix.size()) == prefix;
    }
    
    static bool endsWith(std::string_view str, std::string_view suffix) {
        return str.size() >= suffix.size() &&
               str.substr(str.size() - suffix.size()) == suffix;
    }
    
    // Parse number without string allocation
    static std::optional<int> parseInt(std::string_view sv) {
        int result;
        auto [ptr, ec] = std::from_chars(sv.data(), sv.data() + sv.size(), result);
        if (ec != std::errc{}) return std::nullopt;
        return result;
    }
    
    static std::optional<double> parseDouble(std::string_view sv) {
        double result;
        auto [ptr, ec] = std::from_chars(sv.data(), sv.data() + sv.size(), result);
        if (ec != std::errc{}) return std::nullopt;
        return result;
    }
    
    // Fast split
    static std::vector<std::string_view> split(std::string_view str, char delim) {
        std::vector<std::string_view> parts;
        std::size_t start = 0;
        
        while (start <= str.size()) {
            auto end = str.find(delim, start);
            if (end == std::string_view::npos) end = str.size();
            parts.push_back(str.substr(start, end - start));
            start = end + 1;
        }
        
        return parts;
    }
};

// Small String Optimization (SSO) awareness
void demoStringOpt() {
    // Short strings don't allocate on heap (SSO)
    std::string short1 = "Hello";   // likely on stack
    std::string short2 = "World!";  // likely on stack
    
    // Long string allocates on heap
    std::string long1 = "This is a longer string that exceeds SSO buffer";
    
    // string_view over literals - zero cost
    std::string_view sv1 = "Hello World"; // no copy
    
    // Using string_view in function avoids copy
    std::cout << StringProcessor::startsWith("Hello World", "Hello") << '\n'; // true
    
    // Fast number parsing
    auto num = StringProcessor::parseInt("12345");
    std::cout << "Parsed: " << *num << '\n';
    
    // Efficient split
    auto parts = StringProcessor::split("a,b,c,d,e", ',');
    for (auto p : parts) std::cout << p << ' '; // a b c d e
    std::cout << '\n';
}
```

---

## ขั้นตอนที่ 420-430: Qt Performance

```cpp
// === Qt-specific Performance Tips ===

// 1. Reserve containers
void qtContainerPerf() {
    // Bad: multiple reallocations
    QList<int> bad;
    for (int i = 0; i < 10000; i++) bad.append(i);
    
    // Good: single allocation
    QList<int> good;
    good.reserve(10000);
    for (int i = 0; i < 10000; i++) good.append(i);
    
    // For QVector (QList in Qt6)
    QVector<double> vec;
    vec.reserve(1000);
}

// 2. Avoid implicitly shared copies
void qtCopyOnWrite() {
    QList<int> original;
    for (int i = 0; i < 100000; i++) original.append(i);
    
    // Detaches (copies) when modified - expensive!
    QList<int> copy = original;
    copy.append(100001); // causes copy of entire list
    
    // If you need to modify, use move or explicit copy upfront
    QList<int> moved = std::move(original); // original now empty
}

// 3. QString optimization
void qtStringPerf() {
    // Use QStringLiteral for compile-time string construction
    QString s1 = QStringLiteral("Hello World"); // No alloc at runtime
    
    // Use arg() chain instead of repeated +
    // Bad:
    QString bad = "Name: " + QString("Alice") + ", Age: " + QString::number(25);
    
    // Good:
    QString good = QString("Name: %1, Age: %2").arg("Alice").arg(25);
    
    // QByteArray for binary data
    QByteArray data;
    data.reserve(1024);
    data.append("Hello");
    data.append('\0');
    
    // Latin1 is faster than fromUtf8 when you know the encoding
    QString latin = QString::fromLatin1("ASCII text");
}

// 4. Widget painting optimization
class OptimizedWidget : public QWidget {
    Q_OBJECT
    
public:
    OptimizedWidget(QWidget* parent = nullptr) : QWidget(parent) {
        // Disable automatic background erase (paint everything ourselves)
        setAttribute(Qt::WA_OpaquePaintEvent);
        
        // Pre-allocate pixmap cache
        m_cache = QPixmap(size());
        m_dirty = true;
    }
    
protected:
    void paintEvent(QPaintEvent* event) override {
        if (m_dirty) {
            // Redraw cache
            m_cache = QPixmap(size());
            m_cache.fill(Qt::white);
            
            QPainter cachePainter(&m_cache);
            drawContent(cachePainter);
            m_dirty = false;
        }
        
        // Fast blit from cache
        QPainter painter(this);
        painter.drawPixmap(event->rect(), m_cache, event->rect());
    }
    
    void resizeEvent(QResizeEvent* event) override {
        QWidget::resizeEvent(event);
        m_cache = QPixmap(size());
        m_dirty = true;
    }
    
    void invalidate() {
        m_dirty = true;
        update();
    }
    
private:
    void drawContent(QPainter& p) {
        // Expensive drawing here (only when dirty)
        p.setPen(QPen(Qt::blue, 2));
        for (int i = 0; i < 1000; i++) {
            p.drawLine(i % width(), 0, (i * 37) % width(), height());
        }
    }
    
    QPixmap m_cache;
    bool m_dirty;
};

// 5. Model/View performance
class FastTableModel : public QAbstractTableModel {
    Q_OBJECT
    
    static const int ROWS = 100000;
    static const int COLS = 10;
    
    // Use flat array for cache performance
    std::vector<double> m_data;
    
public:
    FastTableModel(QObject* parent = nullptr) 
        : QAbstractTableModel(parent), m_data(ROWS * COLS) {
        // Initialize with random data
        for (auto& v : m_data) v = std::rand() / static_cast<double>(RAND_MAX);
    }
    
    int rowCount(const QModelIndex&) const override { return ROWS; }
    int columnCount(const QModelIndex&) const override { return COLS; }
    
    QVariant data(const QModelIndex& idx, int role) const override {
        if (role != Qt::DisplayRole) return {};
        return m_data[idx.row() * COLS + idx.column()];
    }
    
    // Batch update - much faster than one-at-a-time
    void updateRange(int startRow, int endRow) {
        for (int r = startRow; r <= endRow; r++)
            for (int c = 0; c < COLS; c++)
                m_data[r * COLS + c] = std::rand() / static_cast<double>(RAND_MAX);
        
        // Single emit covers all changed cells
        emit dataChanged(index(startRow, 0), index(endRow, COLS - 1));
    }
};
```

---

## สรุป Part 030

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Profiling ด้วย Timer + Benchmark utility
2. ✅ Memory layout: AoS vs. SoA
3. ✅ Move semantics & RVO/NRVO
4. ✅ String optimization (string_view, charconv, SSO)
5. ✅ Qt-specific optimization (reserve, QStringLiteral, pixmap cache)
6. ✅ FastTableModel with flat array

---

⬅️ [Part 029](part029.md) | ➡️ [Part 031: Qt Plugin System](part031.md)
