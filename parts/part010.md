# Part 010: Advanced Memory Management

## ขั้นตอนที่ 116-130

---

## ขั้นตอนที่ 116: Memory Model และ Stack/Heap

```
Memory Layout ของโปรแกรม C++:

┌─────────────────────┐  High Address
│       Stack         │  ← Local variables, function calls
│         ↓           │
│                     │
│         ↑           │
│        Heap         │  ← Dynamic allocation (new/delete)
│─────────────────────│
│    BSS Segment      │  ← Uninitialized global/static
│─────────────────────│
│    Data Segment     │  ← Initialized global/static
│─────────────────────│
│    Text Segment     │  ← Program code (read-only)
└─────────────────────┘  Low Address
```

```cpp
#include <iostream>
#include <memory>
#include <chrono>
using namespace std;

// Stack vs Heap Benchmark
template<typename Func>
long long benchmark(Func f, int iterations = 1000000) {
    auto start = chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; i++) f();
    auto end = chrono::high_resolution_clock::now();
    return chrono::duration_cast<chrono::microseconds>(end - start).count();
}

struct SmallObject {
    int x, y, z;
    SmallObject(int a, int b, int c) : x(a), y(b), z(c) {}
};

int main() {
    cout << "=== Memory Model ===" << endl;
    
    // Stack allocation (เร็วมาก)
    int stackInt = 42;
    double stackDouble = 3.14;
    int stackArray[1000];
    
    // Heap allocation (ช้ากว่า แต่ lifespan ยาวกว่า)
    int* heapInt = new int(42);
    double* heapDouble = new double(3.14);
    int* heapArray = new int[1000];
    
    // Address comparison
    cout << "Stack address: " << &stackInt << endl;
    cout << "Heap address:  " << heapInt << endl;
    
    // Stack addresses ใกล้กัน
    int a = 1, b = 2, c = 3;
    cout << "&a=" << &a << " &b=" << &b << " &c=" << &c << endl;
    
    delete heapInt;
    delete heapDouble;
    delete[] heapArray;
    
    // === Benchmark: Stack vs Heap ===
    cout << "\n=== Benchmark ===" << endl;
    
    auto stackBench = []() {
        SmallObject obj(1, 2, 3);
        return obj.x + obj.y + obj.z;
    };
    
    auto heapBench = []() {
        SmallObject* obj = new SmallObject(1, 2, 3);
        int result = obj->x + obj->y + obj->z;
        delete obj;
        return result;
    };
    
    auto heapUniqueBench = []() {
        auto obj = make_unique<SmallObject>(1, 2, 3);
        return obj->x + obj->y + obj->z;
    };
    
    long long stackTime = benchmark(stackBench);
    long long heapTime = benchmark(heapBench);
    long long uniqueTime = benchmark(heapUniqueBench);
    
    cout << "Stack: " << stackTime << " us" << endl;
    cout << "Heap (raw): " << heapTime << " us" << endl;
    cout << "Heap (unique_ptr): " << uniqueTime << " us" << endl;
    
    // === Memory Fragmentation ===
    cout << "\n=== Large Allocation ===" << endl;
    
    try {
        // ทดลอง allocate memory ขนาดใหญ่
        size_t size = 100 * 1024 * 1024;  // 100 MB
        auto* big = new char[size];
        cout << "Allocated " << size / (1024*1024) << " MB" << endl;
        delete[] big;
        cout << "Deallocated" << endl;
    } catch (const bad_alloc& e) {
        cout << "Allocation failed: " << e.what() << endl;
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 117: Custom Allocator

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <list>
#include <chrono>
using namespace std;

// === Pool Allocator ===
class PoolAllocator {
private:
    struct Block {
        Block* next;
    };
    
    char* pool;
    Block* freeList;
    size_t blockSize;
    size_t poolSize;
    int allocated;
    
public:
    PoolAllocator(size_t blockSz, size_t numBlocks)
        : blockSize(max(blockSz, sizeof(Block*))),
          poolSize(numBlocks),
          allocated(0) {
        
        pool = new char[blockSize * numBlocks];
        freeList = nullptr;
        
        // เชื่อมทุก block เข้า free list
        for (size_t i = 0; i < numBlocks; i++) {
            Block* b = reinterpret_cast<Block*>(pool + i * blockSize);
            b->next = freeList;
            freeList = b;
        }
    }
    
    ~PoolAllocator() {
        delete[] pool;
    }
    
    void* allocate() {
        if (!freeList) return nullptr;  // Pool เต็ม
        Block* b = freeList;
        freeList = b->next;
        allocated++;
        return b;
    }
    
    void deallocate(void* ptr) {
        Block* b = reinterpret_cast<Block*>(ptr);
        b->next = freeList;
        freeList = b;
        allocated--;
    }
    
    int getAllocated() const { return allocated; }
};

// === STL Allocator ===
template <typename T>
class TrackingAllocator {
public:
    using value_type = T;
    
    static int allocations;
    static int deallocations;
    static size_t totalBytes;
    
    TrackingAllocator() = default;
    
    template <typename U>
    TrackingAllocator(const TrackingAllocator<U>&) noexcept {}
    
    T* allocate(size_t n) {
        allocations++;
        totalBytes += n * sizeof(T);
        cout << "[Alloc] " << n << " x " << sizeof(T) << " bytes" << endl;
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }
    
    void deallocate(T* p, size_t n) noexcept {
        deallocations++;
        cout << "[Dealloc] " << n << " x " << sizeof(T) << " bytes" << endl;
        ::operator delete(p);
    }
    
    static void printStats() {
        cout << "Allocations: " << allocations << endl;
        cout << "Deallocations: " << deallocations << endl;
        cout << "Total bytes: " << totalBytes << endl;
    }
    
    static void reset() {
        allocations = deallocations = 0;
        totalBytes = 0;
    }
};

template <typename T>
int TrackingAllocator<T>::allocations = 0;
template <typename T>
int TrackingAllocator<T>::deallocations = 0;
template <typename T>
size_t TrackingAllocator<T>::totalBytes = 0;

template <typename T, typename U>
bool operator==(const TrackingAllocator<T>&, const TrackingAllocator<U>&) { return true; }
template <typename T, typename U>
bool operator!=(const TrackingAllocator<T>&, const TrackingAllocator<U>&) { return false; }

struct Particle {
    float x, y, z;
    float vx, vy, vz;
    float mass;
    
    Particle(float x=0, float y=0, float z=0)
        : x(x), y(y), z(z), vx(0), vy(0), vz(0), mass(1.0f) {}
};

int main() {
    // === Pool Allocator Benchmark ===
    cout << "=== Pool Allocator ===" << endl;
    
    const int N = 10000;
    PoolAllocator pool(sizeof(Particle), N);
    
    // Allocate กับ Pool
    auto start = chrono::high_resolution_clock::now();
    vector<Particle*> particles;
    particles.reserve(N);
    
    for (int i = 0; i < N; i++) {
        void* mem = pool.allocate();
        particles.push_back(new(mem) Particle(i, i*2, i*3));
    }
    
    auto mid = chrono::high_resolution_clock::now();
    
    for (Particle* p : particles) {
        p->~Particle();
        pool.deallocate(p);
    }
    
    auto end = chrono::high_resolution_clock::now();
    
    long long poolTime = chrono::duration_cast<chrono::microseconds>(end - start).count();
    cout << "Pool allocator: " << poolTime << " us for " << N << " particles" << endl;
    
    // เปรียบเทียบกับ new/delete
    start = chrono::high_resolution_clock::now();
    vector<Particle*> rawParticles;
    rawParticles.reserve(N);
    
    for (int i = 0; i < N; i++) {
        rawParticles.push_back(new Particle(i, i*2, i*3));
    }
    for (Particle* p : rawParticles) {
        delete p;
    }
    
    end = chrono::high_resolution_clock::now();
    long long rawTime = chrono::duration_cast<chrono::microseconds>(end - start).count();
    cout << "Raw new/delete: " << rawTime << " us for " << N << " particles" << endl;
    
    if (poolTime < rawTime)
        cout << "Pool is " << (rawTime / max(poolTime, 1LL)) << "x faster!" << endl;
    
    // === STL Custom Allocator ===
    cout << "\n=== STL Custom Allocator ===" << endl;
    
    using TrackedVector = vector<int, TrackingAllocator<int>>;
    
    TrackingAllocator<int>::reset();
    {
        TrackedVector v;
        cout << "--- push_back 10 items ---" << endl;
        for (int i = 0; i < 10; i++) v.push_back(i);
    }
    
    cout << "\nStats:" << endl;
    TrackingAllocator<int>::printStats();
    
    return 0;
}
```

---

## ขั้นตอนที่ 118: RAII Pattern

```cpp
#include <iostream>
#include <fstream>
#include <stdexcept>
#include <mutex>
#include <memory>
using namespace std;

// === RAII Wrapper สำหรับ File ===
class FileHandler {
private:
    FILE* file;
    string filename;
    
public:
    FileHandler(const string& name, const string& mode) 
        : filename(name), file(nullptr) {
        file = fopen(name.c_str(), mode.c_str());
        if (!file) throw runtime_error("Cannot open file: " + name);
        cout << "[FileHandler] Opened: " << name << endl;
    }
    
    ~FileHandler() {
        if (file) {
            fclose(file);
            cout << "[FileHandler] Closed: " << filename << endl;
        }
    }
    
    // ห้าม copy
    FileHandler(const FileHandler&) = delete;
    FileHandler& operator=(const FileHandler&) = delete;
    
    // อนุญาต move
    FileHandler(FileHandler&& other) noexcept
        : file(other.file), filename(move(other.filename)) {
        other.file = nullptr;
    }
    
    void write(const string& data) {
        fputs(data.c_str(), file);
    }
    
    string read() {
        fseek(file, 0, SEEK_END);
        long size = ftell(file);
        fseek(file, 0, SEEK_SET);
        string content(size, '\0');
        fread(&content[0], 1, size, file);
        return content;
    }
};

// === RAII สำหรับ Database Connection (สมมติ) ===
class DbConnection {
private:
    string connectionString;
    bool connected;
    
public:
    DbConnection(const string& connStr) : connectionString(connStr), connected(false) {
        // Simulate connection
        connected = true;
        cout << "[DB] Connected to: " << connectionString << endl;
    }
    
    ~DbConnection() {
        if (connected) {
            connected = false;
            cout << "[DB] Disconnected from: " << connectionString << endl;
        }
    }
    
    void query(const string& sql) {
        if (!connected) throw runtime_error("Not connected");
        cout << "[DB] Execute: " << sql << endl;
    }
};

// === ScopeGuard (Generic RAII) ===
template <typename Cleanup>
class ScopeGuard {
private:
    Cleanup cleanup;
    bool active;
    
public:
    explicit ScopeGuard(Cleanup c) : cleanup(c), active(true) {}
    
    ~ScopeGuard() {
        if (active) cleanup();
    }
    
    void dismiss() { active = false; }  // ยกเลิก cleanup
    
    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
};

template <typename Cleanup>
ScopeGuard<Cleanup> makeScopeGuard(Cleanup c) {
    return ScopeGuard<Cleanup>(c);
}

// === Transaction RAII ===
class Transaction {
private:
    DbConnection& db;
    bool committed;
    
public:
    Transaction(DbConnection& db) : db(db), committed(false) {
        db.query("BEGIN TRANSACTION");
    }
    
    ~Transaction() {
        if (!committed) {
            db.query("ROLLBACK");
            cout << "[Transaction] Rolled back" << endl;
        }
    }
    
    void commit() {
        db.query("COMMIT");
        committed = true;
        cout << "[Transaction] Committed" << endl;
    }
};

int main() {
    cout << "=== RAII Pattern ===" << endl;
    
    // === File RAII ===
    cout << "\n--- File RAII ---" << endl;
    try {
        FileHandler wf("test_raii.txt", "w");
        wf.write("Hello RAII World!\n");
        wf.write("Line 2\n");
    }  // file ถูก close อัตโนมัติ
    
    // ไฟล์ปิดแล้ว แต่ยังอ่านได้
    {
        FileHandler rf("test_raii.txt", "r");
        cout << "Content: " << rf.read();
    }
    
    // === Exception Safety ===
    cout << "\n--- Exception Safety ---" << endl;
    try {
        FileHandler f("test_raii.txt", "r");
        cout << "Before exception" << endl;
        throw runtime_error("Simulated error");
        // f.read();  // ไม่ถูก execute
    } catch (const runtime_error& e) {
        cout << "Caught: " << e.what() << endl;
        cout << "File closed safely before this line" << endl;
    }
    
    // === ScopeGuard ===
    cout << "\n--- ScopeGuard ---" << endl;
    
    cout << "Creating resource..." << endl;
    auto guard = makeScopeGuard([]() {
        cout << "ScopeGuard: Cleaning up resource!" << endl;
    });
    cout << "Using resource..." << endl;
    
    // guard ถูก dismiss ถ้าไม่ต้องการ cleanup
    // guard.dismiss();
    
    // === Database Transaction ===
    cout << "\n--- Transaction RAII ---" << endl;
    
    DbConnection db("postgresql://localhost/mydb");
    
    // Transaction สำเร็จ
    {
        Transaction tx(db);
        db.query("INSERT INTO users VALUES (1, 'Alice')");
        db.query("UPDATE accounts SET balance = 1000 WHERE id = 1");
        tx.commit();
    }
    
    // Transaction fail (rollback อัตโนมัติ)
    {
        Transaction tx(db);
        db.query("INSERT INTO users VALUES (2, 'Bob')");
        // exception เกิดขึ้น ไม่ได้ commit
        // tx จะ rollback ใน destructor
    }
    
    // Cleanup
    remove("test_raii.txt");
    
    return 0;
}
```

---

## ขั้นตอนที่ 119-130: Memory Debugging

```cpp
#include <iostream>
#include <memory>
#include <unordered_map>
#include <string>
#include <cstring>
using namespace std;

// === Memory Leak Detector ===
class MemoryTracker {
private:
    struct AllocInfo {
        size_t size;
        string file;
        int line;
    };
    
    static unordered_map<void*, AllocInfo> allocations;
    static size_t totalAllocated;
    static int allocationCount;
    
public:
    static void* track(void* ptr, size_t size, const char* file, int line) {
        if (ptr) {
            allocations[ptr] = {size, file, line};
            totalAllocated += size;
            allocationCount++;
        }
        return ptr;
    }
    
    static void untrack(void* ptr) {
        auto it = allocations.find(ptr);
        if (it != allocations.end()) {
            totalAllocated -= it->second.size;
            allocations.erase(it);
        }
    }
    
    static void report() {
        if (allocations.empty()) {
            cout << "No memory leaks!" << endl;
        } else {
            cout << "Memory Leaks Detected!" << endl;
            for (const auto& [ptr, info] : allocations) {
                cout << "  Leak: " << info.size << " bytes at " 
                     << info.file << ":" << info.line << endl;
            }
        }
        cout << "Total allocated: " << totalAllocated << " bytes" << endl;
        cout << "Total allocations: " << allocationCount << endl;
    }
};

unordered_map<void*, MemoryTracker::AllocInfo> MemoryTracker::allocations;
size_t MemoryTracker::totalAllocated = 0;
int MemoryTracker::allocationCount = 0;

// Macros for tracking
#define TRACKED_NEW(type) \
    (type*)MemoryTracker::track(new type, sizeof(type), __FILE__, __LINE__)

#define TRACKED_DELETE(ptr) do { \
    MemoryTracker::untrack(ptr); \
    delete ptr; \
    ptr = nullptr; \
} while(0)

// === Buffer with Bounds Checking ===
template <typename T, size_t N>
class SafeArray {
private:
    T data[N];
    
public:
    T& operator[](size_t idx) {
        if (idx >= N) {
            throw out_of_range("Index " + to_string(idx) + 
                             " out of range [0," + to_string(N-1) + "]");
        }
        return data[idx];
    }
    
    const T& operator[](size_t idx) const {
        if (idx >= N) {
            throw out_of_range("Index " + to_string(idx) + 
                             " out of range [0," + to_string(N-1) + "]");
        }
        return data[idx];
    }
    
    size_t size() const { return N; }
    T* begin() { return data; }
    T* end() { return data + N; }
};

int main() {
    cout << "=== Memory Safety ===" << endl;
    
    // === Use Smart Pointers ===
    cout << "\n--- Smart Pointers ---" << endl;
    
    {
        auto p1 = make_unique<int>(42);
        auto p2 = make_shared<string>("Hello");
        
        cout << "*p1 = " << *p1 << endl;
        cout << "*p2 = " << *p2 << endl;
    }  // ทั้งคู่ถูก free อัตโนมัติ
    
    // === Safe Array ===
    cout << "\n--- Safe Array ---" << endl;
    
    SafeArray<int, 5> arr;
    for (size_t i = 0; i < arr.size(); i++) arr[i] = i * 10;
    
    cout << "arr[2] = " << arr[2] << endl;
    
    try {
        cout << arr[10] << endl;  // Out of bounds!
    } catch (const out_of_range& e) {
        cout << "Caught: " << e.what() << endl;
    }
    
    // === Dangling Pointer Demonstration ===
    cout << "\n--- Dangling Pointer (ห้ามทำ!) ---" << endl;
    
    // WRONG - ห้ามทำ:
    // int* p = new int(42);
    // delete p;
    // cout << *p;  // Undefined behavior!
    
    // CORRECT:
    int* p = new int(42);
    cout << "Before delete: " << *p << endl;
    delete p;
    p = nullptr;  // Set to null after delete
    
    if (p == nullptr) {
        cout << "Pointer is safely null" << endl;
    }
    
    // === Double Free (ห้ามทำ!) ===
    cout << "\n--- Double Free Prevention ---" << endl;
    
    // WRONG:
    // int* q = new int(10);
    // delete q;
    // delete q;  // Undefined behavior!
    
    // CORRECT ด้วย unique_ptr:
    {
        auto q = make_unique<int>(10);
        // q จะถูก delete แค่ครั้งเดียว
    }
    
    // === Buffer Overflow (ห้ามทำ!) ===
    cout << "\n--- Buffer Overflow Prevention ---" << endl;
    
    // WRONG:
    // char buf[10];
    // strcpy(buf, "This string is longer than 10 characters!");  // Buffer overflow!
    
    // CORRECT ด้วย string:
    string str = "This string is safe!";
    cout << str << endl;
    
    // CORRECT ด้วย strncpy:
    char buf[10];
    strncpy(buf, "Hello World", sizeof(buf) - 1);
    buf[sizeof(buf) - 1] = '\0';
    cout << "Safe copy: " << buf << endl;
    
    // === Memory Tracker ===
    cout << "\n--- Memory Tracking ---" << endl;
    
    int* tracked1 = TRACKED_NEW(int);
    *tracked1 = 100;
    
    int* tracked2 = TRACKED_NEW(int);
    *tracked2 = 200;
    
    // Free tracked1 แต่ไม่ free tracked2 (simulate leak)
    TRACKED_DELETE(tracked1);
    
    cout << "\nMemory Report:" << endl;
    MemoryTracker::report();
    
    // Clean up leak
    TRACKED_DELETE(tracked2);
    
    cout << "\nAfter cleanup:" << endl;
    MemoryTracker::report();
    
    return 0;
}
```

---

## สรุป Part 010

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Memory Model (Stack vs Heap)
2. ✅ Custom Pool Allocator 
3. ✅ STL Custom Allocator
4. ✅ RAII Pattern
5. ✅ Memory Safety Practices
6. ✅ Memory Leak Detection

---

⬅️ [Part 009](part009.md) | ➡️ [Part 011: Qt Framework Introduction](part011.md)
