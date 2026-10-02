# Part 062: Advanced — Lock-Free Data Structures & Concurrency

## ขั้นตอนที่ 896-910

---

## ขั้นตอนที่ 896: Lock-Free Fundamentals

```cpp
// Lock-free programming ด้วย std::atomic
// หลักการ: แทนที่ mutex ด้วย atomic CAS (Compare-And-Swap)

#include <atomic>
#include <memory>
#include <optional>

// === Lock-free stack (Treiber Stack) ===
template<typename T>
class LockFreeStack {
public:
    void push(T value) {
        auto* newNode = new Node{std::move(value), nullptr};
        
        Node* oldHead;
        do {
            oldHead = m_head.load(std::memory_order_relaxed);
            newNode->next = oldHead;
        } while (!m_head.compare_exchange_weak(
            oldHead, newNode,
            std::memory_order_release,
            std::memory_order_relaxed));
        
        m_size.fetch_add(1, std::memory_order_relaxed);
    }
    
    std::optional<T> pop() {
        Node* oldHead;
        do {
            oldHead = m_head.load(std::memory_order_relaxed);
            if (!oldHead) return std::nullopt;
        } while (!m_head.compare_exchange_weak(
            oldHead, oldHead->next,
            std::memory_order_acquire,
            std::memory_order_relaxed));
        
        T value = std::move(oldHead->value);
        m_size.fetch_sub(1, std::memory_order_relaxed);
        
        // Note: ABA problem mitigation needed for production
        // Use hazard pointers or epoch-based reclamation
        delete oldHead;
        
        return value;
    }
    
    std::size_t size() const {
        return m_size.load(std::memory_order_relaxed);
    }
    
    bool empty() const { return size() == 0; }
    
    ~LockFreeStack() {
        while (pop()) {}
    }
    
private:
    struct Node {
        T value;
        Node* next;
    };
    
    std::atomic<Node*> m_head{nullptr};
    std::atomic<std::size_t> m_size{0};
};

// === Lock-free single-producer single-consumer ring buffer ===
template<typename T, std::size_t Capacity>
class SPSCQueue {
    static_assert((Capacity & (Capacity - 1)) == 0,
                  "Capacity must be power of 2");
    
public:
    bool push(T value) {
        std::size_t tail = m_tail.load(std::memory_order_relaxed);
        std::size_t nextTail = (tail + 1) & (Capacity - 1);
        
        if (nextTail == m_head.load(std::memory_order_acquire)) {
            return false;  // Queue is full
        }
        
        m_buffer[tail] = std::move(value);
        m_tail.store(nextTail, std::memory_order_release);
        return true;
    }
    
    std::optional<T> pop() {
        std::size_t head = m_head.load(std::memory_order_relaxed);
        
        if (head == m_tail.load(std::memory_order_acquire)) {
            return std::nullopt;  // Queue is empty
        }
        
        T value = std::move(m_buffer[head]);
        m_head.store((head + 1) & (Capacity - 1), std::memory_order_release);
        return value;
    }
    
    bool empty() const {
        return m_head.load(std::memory_order_acquire)
            == m_tail.load(std::memory_order_acquire);
    }
    
    std::size_t size() const {
        std::size_t h = m_head.load(std::memory_order_acquire);
        std::size_t t = m_tail.load(std::memory_order_acquire);
        return (t - h + Capacity) & (Capacity - 1);
    }
    
private:
    alignas(64) std::atomic<std::size_t> m_head{0};
    alignas(64) std::atomic<std::size_t> m_tail{0};
    T m_buffer[Capacity];
};
```

---

## ขั้นตอนที่ 897: Thread Pool Implementation

```cpp
// concurrency/threadpool.h
#pragma once
#include <QObject>
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>
#include <functional>
#include <atomic>

class ThreadPool {
public:
    explicit ThreadPool(std::size_t numThreads = std::thread::hardware_concurrency())
        : m_stop(false)
    {
        for (std::size_t i = 0; i < numThreads; ++i) {
            m_workers.emplace_back([this] {
                for (;;) {
                    std::function<void()> task;
                    
                    {
                        std::unique_lock lock(m_mutex);
                        m_cv.wait(lock, [this] {
                            return m_stop || !m_tasks.empty();
                        });
                        
                        if (m_stop && m_tasks.empty()) return;
                        
                        task = std::move(m_tasks.front());
                        m_tasks.pop();
                    }
                    
                    m_activeTasks++;
                    task();
                    m_activeTasks--;
                    
                    if (m_activeTasks == 0 && m_tasks.empty()) {
                        m_allDone.notify_all();
                    }
                }
            });
        }
    }
    
    ~ThreadPool() {
        {
            std::lock_guard lock(m_mutex);
            m_stop = true;
        }
        m_cv.notify_all();
        for (auto& t : m_workers) t.join();
    }
    
    template<typename F, typename... Args>
    auto submit(F&& fn, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>
    {
        using ReturnType = std::invoke_result_t<F, Args...>;
        
        auto task = std::make_shared<std::packaged_task<ReturnType()>>(
            std::bind(std::forward<F>(fn), std::forward<Args>(args)...)
        );
        
        std::future<ReturnType> result = task->get_future();
        
        {
            std::lock_guard lock(m_mutex);
            if (m_stop) throw std::runtime_error("ThreadPool is stopped");
            m_tasks.emplace([task] { (*task)(); });
        }
        
        m_cv.notify_one();
        return result;
    }
    
    void waitAll() {
        std::unique_lock lock(m_mutex);
        m_allDone.wait(lock, [this] {
            return m_tasks.empty() && m_activeTasks == 0;
        });
    }
    
    std::size_t threadCount() const { return m_workers.size(); }
    std::size_t pendingTasks() const {
        std::lock_guard lock(m_mutex);
        return m_tasks.size();
    }
    
private:
    std::vector<std::thread> m_workers;
    std::queue<std::function<void()>> m_tasks;
    mutable std::mutex m_mutex;
    std::condition_variable m_cv;
    std::condition_variable m_allDone;
    std::atomic<bool> m_stop;
    std::atomic<int> m_activeTasks{0};
};

// Qt-friendly wrapper using futures → QFuture
class QtThreadPool : public QObject {
    Q_OBJECT
    
public:
    static QtThreadPool& instance() {
        static QtThreadPool inst;
        return inst;
    }
    
    template<typename F>
    QFuture<typename std::invoke_result_t<F>> run(F&& fn) {
        QPromise<typename std::invoke_result_t<F>> promise;
        QFuture<typename std::invoke_result_t<F>> future = promise.future();
        
        m_pool.submit([fn = std::forward<F>(fn), p = std::move(promise)]() mutable {
            p.start();
            try {
                if constexpr (std::is_void_v<typename std::invoke_result_t<F>>) {
                    fn();
                    p.finish();
                } else {
                    p.addResult(fn());
                    p.finish();
                }
            } catch (...) {
                p.setException(std::current_exception());
                p.finish();
            }
        });
        
        return future;
    }
    
private:
    QtThreadPool() = default;
    ThreadPool m_pool;
};
```

---

## ขั้นตอนที่ 898: Parallel Algorithms

```cpp
// concurrency/parallel.h
#pragma once
#include <vector>
#include <algorithm>
#include <future>
#include <numeric>
#include "threadpool.h"

// Parallel map
template<typename In, typename Out, typename Func>
std::vector<Out> parallelMap(const std::vector<In>& input, Func fn,
                              std::size_t chunkSize = 0)
{
    std::size_t n = input.size();
    if (n == 0) return {};
    
    std::size_t threads = std::thread::hardware_concurrency();
    if (chunkSize == 0) chunkSize = (n + threads - 1) / threads;
    
    std::vector<Out> output(n);
    std::vector<std::future<void>> futures;
    
    for (std::size_t start = 0; start < n; start += chunkSize) {
        std::size_t end = std::min(start + chunkSize, n);
        
        futures.push_back(
            std::async(std::launch::async, [&, start, end] {
                for (std::size_t i = start; i < end; ++i) {
                    output[i] = fn(input[i]);
                }
            })
        );
    }
    
    for (auto& f : futures) f.get();
    return output;
}

// Parallel reduce
template<typename T, typename Func>
T parallelReduce(const std::vector<T>& input, T identity, Func fn) {
    std::size_t n = input.size();
    if (n == 0) return identity;
    
    std::size_t threads = std::min(std::thread::hardware_concurrency(),
                                    (unsigned)n);
    std::size_t chunkSize = (n + threads - 1) / threads;
    
    std::vector<std::future<T>> futures;
    
    for (std::size_t start = 0; start < n; start += chunkSize) {
        std::size_t end = std::min(start + chunkSize, n);
        
        futures.push_back(
            std::async(std::launch::async, [&, start, end] {
                T acc = identity;
                for (std::size_t i = start; i < end; ++i) {
                    acc = fn(acc, input[i]);
                }
                return acc;
            })
        );
    }
    
    T result = identity;
    for (auto& f : futures) result = fn(result, f.get());
    return result;
}

// Parallel sort (merge sort based)
template<typename T, typename Compare = std::less<T>>
void parallelSort(std::vector<T>& data, int depth = 0, Compare cmp = {}) {
    std::size_t n = data.size();
    
    if (n < 10000 || depth >= 3) {
        std::sort(data.begin(), data.end(), cmp);
        return;
    }
    
    std::size_t mid = n / 2;
    std::vector<T> left(data.begin(), data.begin() + mid);
    std::vector<T> right(data.begin() + mid, data.end());
    
    auto future = std::async(std::launch::async, [&] {
        parallelSort(left, depth + 1, cmp);
    });
    
    parallelSort(right, depth + 1, cmp);
    future.get();
    
    std::merge(left.begin(), left.end(),
               right.begin(), right.end(),
               data.begin(), cmp);
}

// Demo: parallel image processing
void parallelImageProcess(QImage& image) {
    int height = image.height();
    int width = image.width();
    
    // Split image into horizontal strips
    int threads = QThreadPool::globalInstance()->maxThreadCount();
    int stripHeight = height / threads;
    
    QFutureSynchronizer<void> sync;
    
    for (int t = 0; t < threads; ++t) {
        int yStart = t * stripHeight;
        int yEnd = (t == threads - 1) ? height : yStart + stripHeight;
        
        sync.addFuture(QtConcurrent::run([=, &image]() {
            for (int y = yStart; y < yEnd; ++y) {
                QRgb* row = reinterpret_cast<QRgb*>(image.scanLine(y));
                for (int x = 0; x < width; ++x) {
                    // Sepia tone effect
                    int r = qRed(row[x]);
                    int g = qGreen(row[x]);
                    int b = qBlue(row[x]);
                    
                    int nr = qMin(255, (int)(r * 0.393 + g * 0.769 + b * 0.189));
                    int ng = qMin(255, (int)(r * 0.349 + g * 0.686 + b * 0.168));
                    int nb = qMin(255, (int)(r * 0.272 + g * 0.534 + b * 0.131));
                    
                    row[x] = qRgb(nr, ng, nb);
                }
            }
        }));
    }
    
    sync.waitForFinished();
}
```

---

## สรุป Part 062

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Lock-free Stack (Treiber Stack) ด้วย std::atomic CAS
2. ✅ SPSC Ring Buffer ด้วย atomic + cache-line alignment
3. ✅ Thread Pool ด้วย std::thread + packaged_task + condition_variable
4. ✅ QtThreadPool wrapper ด้วย QPromise + QFuture
5. ✅ Parallel Map/Reduce/Sort algorithms
6. ✅ Parallel image processing ด้วย QtConcurrent

---

⬅️ [Part 061](part061.md) | ➡️ [Part 063: Capstone — Real-time Trading Dashboard](part063.md)
