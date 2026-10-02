# Part 009: Modern C++11/14/17/20

## ขั้นตอนที่ 101-115

---

## ขั้นตอนที่ 101: Move Semantics

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <utility>
using namespace std;

class BigData {
private:
    int* data;
    size_t size;
    string name;
    
public:
    // Constructor
    BigData(const string& n, size_t s) : name(n), size(s) {
        data = new int[size];
        for (size_t i = 0; i < size; i++) data[i] = i;
        cout << "[Construct] " << name << " (size=" << size << ")" << endl;
    }
    
    // Copy Constructor - คัดลอกทุกอย่าง (ช้า)
    BigData(const BigData& other) : name(other.name + "_copy"), size(other.size) {
        data = new int[size];
        copy(other.data, other.data + size, data);
        cout << "[Copy] " << name << " (copied from " << other.name << ")" << endl;
    }
    
    // Move Constructor (C++11) - โอน ownership (เร็ว!)
    BigData(BigData&& other) noexcept 
        : name(move(other.name)), size(other.size), data(other.data) {
        other.data = nullptr;
        other.size = 0;
        cout << "[Move] " << name << " (moved)" << endl;
    }
    
    // Copy Assignment
    BigData& operator=(const BigData& other) {
        if (this != &other) {
            delete[] data;
            size = other.size;
            data = new int[size];
            copy(other.data, other.data + size, data);
            name = other.name + "_assigned";
            cout << "[Copy Assign] " << name << endl;
        }
        return *this;
    }
    
    // Move Assignment (C++11)
    BigData& operator=(BigData&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            size = other.size;
            name = move(other.name);
            other.data = nullptr;
            other.size = 0;
            cout << "[Move Assign] " << name << endl;
        }
        return *this;
    }
    
    ~BigData() {
        delete[] data;
        cout << "[Destroy] " << (name.empty() ? "(moved-from)" : name) << endl;
    }
    
    string getName() const { return name; }
    size_t getSize() const { return size; }
};

// Return Value Optimization (RVO/NRVO)
BigData createBigData(const string& name) {
    return BigData(name, 1000);  // Compiler ทำ RVO อัตโนมัติ
}

int main() {
    cout << "=== Move Semantics ===" << endl;
    
    // Create
    BigData a("Alpha", 1000);
    
    // Copy (ช้า)
    cout << "\n--- Copy ---" << endl;
    BigData b = a;
    
    // Move (เร็ว!)
    cout << "\n--- Move ---" << endl;
    BigData c = move(a);  // a ว่างเปล่าหลังจากนี้
    cout << "a after move: name=" << a.getName() << " size=" << a.getSize() << endl;
    
    // RVO
    cout << "\n--- RVO ---" << endl;
    BigData d = createBigData("Delta");
    
    // swap กับ move
    cout << "\n--- swap ---" << endl;
    BigData e("Echo", 500);
    BigData f("Fox", 500);
    swap(e, f);  // ใช้ Move Semantics
    
    cout << "\n--- Cleanup ---" << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 102: Smart Pointers

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>
using namespace std;

class Resource {
private:
    string name;
    static int count;
    
public:
    Resource(const string& n) : name(n) {
        count++;
        cout << "[+] Resource: " << name << " (total=" << count << ")" << endl;
    }
    
    ~Resource() {
        count--;
        cout << "[-] Resource: " << name << " (total=" << count << ")" << endl;
    }
    
    void use() const { cout << "Using: " << name << endl; }
    string getName() const { return name; }
    static int getCount() { return count; }
};

int Resource::count = 0;

// === Node กับ shared_ptr ===
struct Node {
    int value;
    shared_ptr<Node> next;
    weak_ptr<Node> prev;  // ใช้ weak_ptr เพื่อหลีกเลี่ยง circular reference
    
    Node(int v) : value(v) {
        cout << "Node(" << v << ") created" << endl;
    }
    
    ~Node() {
        cout << "Node(" << value << ") destroyed" << endl;
    }
};

int main() {
    // === unique_ptr ===
    cout << "=== unique_ptr ===" << endl;
    {
        unique_ptr<Resource> res1 = make_unique<Resource>("A");
        res1->use();
        
        // unique_ptr ไม่สามารถ copy ได้
        // unique_ptr<Resource> res2 = res1;  // Error!
        
        // แต่ move ได้
        unique_ptr<Resource> res2 = move(res1);
        cout << "res1 is null: " << (res1 == nullptr) << endl;
        res2->use();
        
        // Array version
        unique_ptr<int[]> arr = make_unique<int[]>(5);
        for (int i = 0; i < 5; i++) arr[i] = i * 10;
        cout << "arr[2] = " << arr[2] << endl;
        
    }  // res2 ถูก delete อัตโนมัติ
    cout << "Resource count: " << Resource::getCount() << endl;
    
    // === shared_ptr ===
    cout << "\n=== shared_ptr ===" << endl;
    {
        shared_ptr<Resource> sp1 = make_shared<Resource>("B");
        cout << "use_count: " << sp1.use_count() << endl;  // 1
        
        {
            shared_ptr<Resource> sp2 = sp1;  // copy = share
            cout << "use_count: " << sp1.use_count() << endl;  // 2
            
            shared_ptr<Resource> sp3 = sp1;
            cout << "use_count: " << sp1.use_count() << endl;  // 3
            sp1->use();
        }
        // sp2, sp3 ออกจาก scope
        cout << "use_count after inner scope: " << sp1.use_count() << endl;  // 1
        
    }  // sp1 ออกจาก scope -> delete
    cout << "Resource count: " << Resource::getCount() << endl;
    
    // === weak_ptr ===
    cout << "\n=== weak_ptr ===" << endl;
    {
        shared_ptr<Resource> sp = make_shared<Resource>("C");
        weak_ptr<Resource> wp = sp;
        
        cout << "use_count: " << sp.use_count() << endl;  // 1 (weak_ptr ไม่นับ)
        
        // ใช้ weak_ptr ต้อง lock() ก่อน
        if (auto locked = wp.lock()) {
            locked->use();
            cout << "use_count while locked: " << sp.use_count() << endl;  // 2
        }
        
        cout << "use_count after unlock: " << sp.use_count() << endl;  // 1
        
        // Check if expired
        sp.reset();  // ลบ Resource
        cout << "wp expired: " << wp.expired() << endl;  // true
        
        if (wp.lock() == nullptr) {
            cout << "Resource no longer exists" << endl;
        }
    }
    
    // === Doubly Linked List กับ weak_ptr ===
    cout << "\n=== Doubly Linked List ===" << endl;
    {
        auto n1 = make_shared<Node>(1);
        auto n2 = make_shared<Node>(2);
        auto n3 = make_shared<Node>(3);
        
        n1->next = n2;
        n2->prev = n1;  // weak_ptr!
        n2->next = n3;
        n3->prev = n2;  // weak_ptr!
        
        // ไม่มี circular reference
        cout << "use_count n1: " << n1.use_count() << endl;  // 1
        cout << "use_count n2: " << n2.use_count() << endl;  // 2 (n1->next, n2)
        
    }  // ทั้งหมดถูก destroy อย่างถูกต้อง
    
    // === Custom Deleter ===
    cout << "\n=== Custom Deleter ===" << endl;
    
    auto fileDeleter = [](FILE* f) {
        if (f) {
            fclose(f);
            cout << "File closed" << endl;
        }
    };
    
    // unique_ptr กับ custom deleter (ตัวอย่าง)
    // unique_ptr<FILE, decltype(fileDeleter)> filePtr(fopen("test.txt", "w"), fileDeleter);
    
    // === Factory with smart pointers ===
    cout << "\n=== Factory Pattern ===" << endl;
    
    auto createResource = [](const string& type) -> unique_ptr<Resource> {
        return make_unique<Resource>(type);
    };
    
    vector<unique_ptr<Resource>> resources;
    resources.push_back(createResource("X"));
    resources.push_back(createResource("Y"));
    resources.push_back(createResource("Z"));
    
    for (const auto& r : resources) {
        r->use();
    }
    cout << "Resources in vector: " << resources.size() << endl;
    
    // clear = delete ทั้งหมด
    resources.clear();
    cout << "Resource count after clear: " << Resource::getCount() << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 103: Concurrency พื้นฐาน (C++11)

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <atomic>
#include <condition_variable>
#include <future>
#include <chrono>
#include <vector>
using namespace std;

// === Thread พื้นฐาน ===
void printNumbers(int start, int count, const string& name) {
    for (int i = start; i < start + count; i++) {
        cout << "[" << name << "] " << i << endl;
        this_thread::sleep_for(chrono::milliseconds(10));
    }
}

// === Mutex ===
mutex mtx;
int sharedCounter = 0;

void incrementWithMutex(int iterations) {
    for (int i = 0; i < iterations; i++) {
        lock_guard<mutex> lock(mtx);  // RAII lock
        sharedCounter++;
    }
}

// === Atomic ===
atomic<int> atomicCounter(0);

void incrementAtomic(int iterations) {
    for (int i = 0; i < iterations; i++) {
        atomicCounter++;  // Thread-safe!
    }
}

// === Producer-Consumer ===
queue<int> dataQueue;
mutex qMutex;
condition_variable cv;
bool producerDone = false;

void producer() {
    for (int i = 1; i <= 5; i++) {
        this_thread::sleep_for(chrono::milliseconds(100));
        
        unique_lock<mutex> lock(qMutex);
        dataQueue.push(i * 10);
        cout << "[Producer] pushed " << i * 10 << endl;
        cv.notify_one();
    }
    
    unique_lock<mutex> lock(qMutex);
    producerDone = true;
    cv.notify_all();
}

void consumer(int id) {
    while (true) {
        unique_lock<mutex> lock(qMutex);
        cv.wait(lock, []{ return !dataQueue.empty() || producerDone; });
        
        if (dataQueue.empty() && producerDone) break;
        
        if (!dataQueue.empty()) {
            int val = dataQueue.front();
            dataQueue.pop();
            lock.unlock();
            cout << "[Consumer " << id << "] got " << val << endl;
        }
    }
}

// === async/future ===
int computeHeavy(int n) {
    cout << "Computing " << n << "^2 in thread..." << endl;
    this_thread::sleep_for(chrono::milliseconds(500));
    return n * n;
}

int main() {
    // === Basic Threads ===
    cout << "=== Basic Threads ===" << endl;
    
    thread t1(printNumbers, 1, 3, "Thread1");
    thread t2(printNumbers, 10, 3, "Thread2");
    
    t1.join();
    t2.join();
    
    // === Mutex ===
    cout << "\n=== Mutex ===" << endl;
    
    sharedCounter = 0;
    vector<thread> threads;
    
    for (int i = 0; i < 5; i++) {
        threads.emplace_back(incrementWithMutex, 1000);
    }
    
    for (auto& t : threads) t.join();
    cout << "Counter (with mutex): " << sharedCounter << " (expected: 5000)" << endl;
    
    // === Atomic ===
    cout << "\n=== Atomic ===" << endl;
    
    atomicCounter = 0;
    threads.clear();
    
    for (int i = 0; i < 5; i++) {
        threads.emplace_back(incrementAtomic, 1000);
    }
    
    for (auto& t : threads) t.join();
    cout << "Counter (atomic): " << atomicCounter << " (expected: 5000)" << endl;
    
    // === async/future ===
    cout << "\n=== async/future ===" << endl;
    
    cout << "Starting async computation..." << endl;
    auto future1 = async(launch::async, computeHeavy, 10);
    auto future2 = async(launch::async, computeHeavy, 20);
    auto future3 = async(launch::async, computeHeavy, 30);
    
    cout << "Doing other work..." << endl;
    this_thread::sleep_for(chrono::milliseconds(100));
    
    // รอผลลัพธ์
    cout << "Results: " << future1.get() << ", " << future2.get() << ", " << future3.get() << endl;
    
    // === promise/future ===
    cout << "\n=== promise/future ===" << endl;
    
    promise<string> prom;
    future<string> fut = prom.get_future();
    
    thread promThread([&prom]() {
        this_thread::sleep_for(chrono::milliseconds(200));
        prom.set_value("ผลลัพธ์จาก promise!");
    });
    
    cout << "รอผลลัพธ์..." << endl;
    cout << "ได้รับ: " << fut.get() << endl;
    promThread.join();
    
    // === Producer-Consumer ===
    cout << "\n=== Producer-Consumer ===" << endl;
    
    producerDone = false;
    
    thread producerThread(producer);
    thread consumerThread1(consumer, 1);
    thread consumerThread2(consumer, 2);
    
    producerThread.join();
    consumerThread1.join();
    consumerThread2.join();
    
    return 0;
}
```

---

## ขั้นตอนที่ 104: C++17 Features

```cpp
#include <iostream>
#include <string>
#include <optional>
#include <variant>
#include <any>
#include <string_view>
#include <filesystem>
#include <charconv>
#include <algorithm>
#include <numeric>
#include <tuple>
using namespace std;
namespace fs = filesystem;

// === std::optional ===
optional<int> findElement(const vector<int>& v, int target) {
    auto it = find(v.begin(), v.end(), target);
    if (it != v.end()) return *it;
    return nullopt;  // ไม่พบ
}

optional<double> safeDivide(double a, double b) {
    if (b == 0) return nullopt;
    return a / b;
}

// === std::variant ===
using JsonValue = variant<nullptr_t, bool, int, double, string>;

string jsonToString(const JsonValue& v) {
    return visit([](auto&& val) -> string {
        using T = decay_t<decltype(val)>;
        if constexpr (is_same_v<T, nullptr_t>) return "null";
        else if constexpr (is_same_v<T, bool>) return val ? "true" : "false";
        else if constexpr (is_same_v<T, int>) return to_string(val);
        else if constexpr (is_same_v<T, double>) return to_string(val);
        else if constexpr (is_same_v<T, string>) return "\"" + val + "\"";
    }, v);
}

// === Structured Bindings ===
tuple<string, int, double> getPersonInfo() {
    return {"สมชาย", 25, 3.8};
}

map<string, int> getScores() {
    return {{"Math", 95}, {"Science", 88}, {"English", 92}};
}

int main() {
    // === std::optional ===
    cout << "=== std::optional ===" << endl;
    
    vector<int> v = {1, 2, 3, 4, 5};
    
    auto result = findElement(v, 3);
    if (result) {
        cout << "Found: " << *result << endl;
    }
    
    auto notFound = findElement(v, 10);
    cout << "notFound has value: " << boolalpha << notFound.has_value() << endl;
    
    // value_or
    cout << "result.value_or(-1): " << result.value_or(-1) << endl;
    cout << "notFound.value_or(-1): " << notFound.value_or(-1) << endl;
    
    // safe divide
    if (auto div = safeDivide(10, 3)) {
        cout << "10/3 = " << *div << endl;
    }
    
    if (!safeDivide(10, 0)) {
        cout << "Division by zero!" << endl;
    }
    
    // === std::variant ===
    cout << "\n=== std::variant ===" << endl;
    
    vector<JsonValue> jsonValues = {
        nullptr,
        true,
        42,
        3.14,
        string("Hello")
    };
    
    for (const auto& val : jsonValues) {
        cout << "type index: " << val.index() << " value: " << jsonToString(val) << endl;
    }
    
    // Check type
    JsonValue jv = string("World");
    if (holds_alternative<string>(jv)) {
        cout << "It's a string: " << get<string>(jv) << endl;
    }
    
    // === std::any ===
    cout << "\n=== std::any ===" << endl;
    
    any a = 42;
    cout << "int: " << any_cast<int>(a) << endl;
    
    a = string("Hello");
    cout << "string: " << any_cast<string>(a) << endl;
    
    a = 3.14;
    cout << "double: " << any_cast<double>(a) << endl;
    
    // Check type
    if (a.type() == typeid(double)) {
        cout << "a is double" << endl;
    }
    
    // Safe cast
    try {
        auto i = any_cast<int>(a);  // จะ throw
    } catch (const bad_any_cast& e) {
        cout << "Bad cast: " << e.what() << endl;
    }
    
    // === string_view ===
    cout << "\n=== string_view ===" << endl;
    
    // string_view ไม่สร้าง copy ของ string
    string_view sv = "Hello, World!";
    cout << sv.substr(0, 5) << endl;   // "Hello"
    cout << sv.length() << endl;        // 13
    cout << (sv.find("World") != string_view::npos) << endl;  // true
    
    // === Structured Bindings (C++17) ===
    cout << "\n=== Structured Bindings ===" << endl;
    
    // Tuple
    auto [name, age, gpa] = getPersonInfo();
    cout << name << " | อายุ: " << age << " | GPA: " << gpa << endl;
    
    // Map
    auto scores = getScores();
    for (const auto& [subject, score] : scores) {
        cout << subject << ": " << score << endl;
    }
    
    // Array
    int arr[] = {1, 2, 3};
    auto [x, y, z] = arr;
    cout << x << " " << y << " " << z << endl;
    
    // Pair
    pair<string, int> p = {"test", 100};
    auto [key, val] = p;
    cout << key << " = " << val << endl;
    
    // === if/switch with initializer (C++17) ===
    cout << "\n=== if with initializer ===" << endl;
    
    map<string, int> wordCount = {{"hello", 5}, {"world", 3}};
    
    if (auto it = wordCount.find("hello"); it != wordCount.end()) {
        cout << "hello count: " << it->second << endl;
    }
    
    if (auto it = wordCount.find("missing"); it != wordCount.end()) {
        cout << "found";
    } else {
        cout << "'missing' not found" << endl;
    }
    
    // === fold expressions (C++17) ===
    cout << "\n=== Fold Expressions ===" << endl;
    
    auto sum = [](auto... args) {
        return (args + ...);  // Fold expression
    };
    
    auto printAll = [](auto... args) {
        ((cout << args << " "), ...);  // Comma fold
        cout << endl;
    };
    
    cout << "sum(1,2,3,4,5) = " << sum(1, 2, 3, 4, 5) << endl;
    printAll(1, "hello", 3.14, true);
    
    // === std::filesystem ===
    cout << "\n=== Filesystem ===" << endl;
    
    fs::path current = fs::current_path();
    cout << "Current path: " << current << endl;
    
    // สร้าง path
    fs::path filepath = current / "test" / "example.txt";
    cout << "Parent: " << filepath.parent_path() << endl;
    cout << "Filename: " << filepath.filename() << endl;
    cout << "Extension: " << filepath.extension() << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 105: C++20 Features

```cpp
#include <iostream>
#include <concepts>
#include <ranges>
#include <coroutine>
#include <format>
#include <span>
#include <vector>
#include <string>
using namespace std;

// === Concepts ===
template <typename T>
concept Numeric = is_arithmetic_v<T>;

template <typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> same_as<T>;
};

template <typename T>
concept Printable = requires(T t) {
    { cout << t } -> same_as<ostream&>;
};

template <Numeric T>
T add(T a, T b) {
    return a + b;
}

template <Numeric T>
T multiply(T a, T b) {
    return a * b;
}

template <Printable T>
void printValue(const T& value) {
    cout << value << endl;
}

// Container concept
template <typename C>
concept Container = requires(C c) {
    { c.begin() };
    { c.end() };
    { c.size() } -> convertible_to<size_t>;
};

template <Container C>
void printContainer(const C& c) {
    for (const auto& item : c) {
        cout << item << " ";
    }
    cout << endl;
}

// === Ranges ===
int main() {
    // === Concepts ===
    cout << "=== Concepts ===" << endl;
    
    cout << add(10, 20) << endl;
    cout << add(3.14, 2.71) << endl;
    // add("hello", "world");  // Compile error - not numeric
    
    printValue(42);
    printValue(3.14);
    printValue(string("Hello"));
    
    vector<int> v = {1, 2, 3, 4, 5};
    printContainer(v);
    
    // === Ranges (C++20) ===
    cout << "\n=== Ranges ===" << endl;
    
    vector<int> nums = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    
    // Range-based views
    auto even = nums | views::filter([](int x) { return x % 2 == 0; });
    cout << "evens: ";
    for (int x : even) cout << x << " ";
    cout << endl;
    
    auto doubled = nums | views::transform([](int x) { return x * 2; });
    cout << "doubled: ";
    for (int x : doubled) cout << x << " ";
    cout << endl;
    
    // Pipeline
    auto result = nums 
        | views::filter([](int x) { return x % 2 != 0; })
        | views::transform([](int x) { return x * x; })
        | views::take(3);
    
    cout << "odd squares (first 3): ";
    for (int x : result) cout << x << " ";
    cout << endl;
    
    // ranges::sort
    ranges::sort(nums);
    cout << "sorted: ";
    for (int x : nums) cout << x << " ";
    cout << endl;
    
    // iota view
    cout << "iota 1..10: ";
    for (int x : views::iota(1, 11)) cout << x << " ";
    cout << endl;
    
    // zip view (C++23 actual, but show concept)
    // auto zipped = views::zip(v1, v2);
    
    // === std::format (C++20) ===
    cout << "\n=== std::format ===" << endl;
    
    string name = "สมชาย";
    int age = 25;
    double gpa = 3.85;
    
    // string msg = format("ชื่อ: {}, อายุ: {}, GPA: {:.2f}", name, age, gpa);
    // cout << msg << endl;
    
    // Fallback (ถ้า compiler ไม่รองรับ format)
    cout << "ชื่อ: " << name << ", อายุ: " << age << endl;
    
    // === span ===
    cout << "\n=== span ===" << endl;
    
    int arr[] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    span<int> s(arr, 10);
    cout << "span size: " << s.size() << endl;
    
    // Sub-span
    auto sub = s.subspan(2, 5);
    cout << "subspan(2, 5): ";
    for (int x : sub) cout << x << " ";
    cout << endl;
    
    // ฟังก์ชันที่รับ span
    auto sumSpan = [](span<const int> data) {
        return accumulate(data.begin(), data.end(), 0);
    };
    
    cout << "sum(s): " << sumSpan(s) << endl;
    cout << "sum(sub): " << sumSpan(sub) << endl;
    
    // span กับ vector
    vector<int> vec = {10, 20, 30, 40, 50};
    span<int> vecSpan(vec);
    cout << "vector span first 3: ";
    for (int x : vecSpan.first(3)) cout << x << " ";
    cout << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 106-115: โปรแกรมขั้นสูง

```cpp
// ============================================
// โปรแกรม: Task Manager ด้วย Modern C++
// ============================================
#include <iostream>
#include <string>
#include <vector>
#include <optional>
#include <memory>
#include <chrono>
#include <algorithm>
#include <functional>
#include <sstream>
#include <iomanip>
using namespace std;

// === Task System ===
enum class Priority { LOW = 1, MEDIUM = 2, HIGH = 3, CRITICAL = 4 };
enum class Status { TODO, IN_PROGRESS, DONE, CANCELLED };

string priorityToString(Priority p) {
    switch (p) {
        case Priority::LOW:      return "Low";
        case Priority::MEDIUM:   return "Medium";
        case Priority::HIGH:     return "High";
        case Priority::CRITICAL: return "Critical";
        default:                 return "Unknown";
    }
}

string statusToString(Status s) {
    switch (s) {
        case Status::TODO:        return "TODO";
        case Status::IN_PROGRESS: return "In Progress";
        case Status::DONE:        return "Done";
        case Status::CANCELLED:   return "Cancelled";
        default:                  return "Unknown";
    }
}

struct Task {
    int id;
    string title;
    string description;
    Priority priority;
    Status status;
    string assignee;
    vector<string> tags;
    
    // Factory method
    static Task create(int id, string title, string desc,
                       Priority p = Priority::MEDIUM,
                       string assignee = "Unassigned") {
        return {id, move(title), move(desc), p, Status::TODO, move(assignee), {}};
    }
    
    void addTag(const string& tag) {
        if (find(tags.begin(), tags.end(), tag) == tags.end()) {
            tags.push_back(tag);
        }
    }
    
    bool hasTag(const string& tag) const {
        return find(tags.begin(), tags.end(), tag) != tags.end();
    }
    
    string toString() const {
        ostringstream oss;
        oss << "[" << id << "] " << title
            << " | " << priorityToString(priority)
            << " | " << statusToString(status)
            << " | " << assignee;
        if (!tags.empty()) {
            oss << " | Tags: ";
            for (const string& t : tags) oss << "#" << t << " ";
        }
        return oss.str();
    }
};

class TaskManager {
private:
    vector<Task> tasks;
    int nextId = 1;
    
    using Predicate = function<bool(const Task&)>;
    
    vector<Task> filter(Predicate pred) const {
        vector<Task> result;
        copy_if(tasks.begin(), tasks.end(), back_inserter(result), pred);
        return result;
    }
    
public:
    int addTask(string title, string desc, Priority p = Priority::MEDIUM,
                string assignee = "Unassigned") {
        Task t = Task::create(nextId++, move(title), move(desc), p, move(assignee));
        int id = t.id;
        tasks.push_back(move(t));
        return id;
    }
    
    optional<Task*> findById(int id) {
        auto it = find_if(tasks.begin(), tasks.end(),
                          [id](const Task& t) { return t.id == id; });
        if (it != tasks.end()) return &(*it);
        return nullopt;
    }
    
    bool updateStatus(int id, Status status) {
        if (auto task = findById(id)) {
            (*task)->status = status;
            return true;
        }
        return false;
    }
    
    bool addTag(int id, const string& tag) {
        if (auto task = findById(id)) {
            (*task)->addTag(tag);
            return true;
        }
        return false;
    }
    
    // Query methods
    vector<Task> getByStatus(Status status) const {
        return filter([status](const Task& t) { return t.status == status; });
    }
    
    vector<Task> getByPriority(Priority priority) const {
        return filter([priority](const Task& t) { return t.priority == priority; });
    }
    
    vector<Task> getByAssignee(const string& assignee) const {
        return filter([&assignee](const Task& t) { return t.assignee == assignee; });
    }
    
    vector<Task> getByTag(const string& tag) const {
        return filter([&tag](const Task& t) { return t.hasTag(tag); });
    }
    
    vector<Task> getSorted(function<bool(const Task&, const Task&)> comp) const {
        vector<Task> sorted = tasks;
        sort(sorted.begin(), sorted.end(), comp);
        return sorted;
    }
    
    void display(const vector<Task>& taskList = {}) const {
        const vector<Task>& toDisplay = taskList.empty() ? tasks : taskList;
        
        cout << string(70, '-') << endl;
        for (const Task& t : toDisplay) {
            cout << t.toString() << endl;
        }
        cout << string(70, '-') << endl;
        cout << "Total: " << toDisplay.size() << " tasks" << endl;
    }
    
    void summary() const {
        cout << "\n=== Task Summary ===" << endl;
        cout << "Total: " << tasks.size() << endl;
        cout << "TODO: " << getByStatus(Status::TODO).size() << endl;
        cout << "In Progress: " << getByStatus(Status::IN_PROGRESS).size() << endl;
        cout << "Done: " << getByStatus(Status::DONE).size() << endl;
        cout << "Cancelled: " << getByStatus(Status::CANCELLED).size() << endl;
    }
};

int main() {
    TaskManager tm;
    
    // เพิ่ม tasks
    int t1 = tm.addTask("สร้าง Login Page", "ออกแบบและสร้างหน้า Login", 
                         Priority::HIGH, "สมชาย");
    int t2 = tm.addTask("เชื่อมต่อ Database", "เชื่อมต่อ PostgreSQL", 
                         Priority::CRITICAL, "สมหญิง");
    int t3 = tm.addTask("เขียน Unit Tests", "เขียนเทสสำหรับ API", 
                         Priority::MEDIUM, "สมชาย");
    int t4 = tm.addTask("อัปเดต Documentation", "อัปเดตเอกสาร API", 
                         Priority::LOW, "สมศรี");
    int t5 = tm.addTask("Fix Bug #42", "แก้ไข Memory Leak", 
                         Priority::CRITICAL, "สมหญิง");
    
    // เพิ่ม tags
    tm.addTag(t1, "frontend");
    tm.addTag(t1, "ui");
    tm.addTag(t2, "backend");
    tm.addTag(t2, "database");
    tm.addTag(t3, "testing");
    tm.addTag(t4, "docs");
    tm.addTag(t5, "bug");
    tm.addTag(t5, "backend");
    
    // อัปเดตสถานะ
    tm.updateStatus(t1, Status::IN_PROGRESS);
    tm.updateStatus(t2, Status::IN_PROGRESS);
    
    cout << "=== All Tasks ===" << endl;
    tm.display();
    
    cout << "\n=== Critical Tasks ===" << endl;
    tm.display(tm.getByPriority(Priority::CRITICAL));
    
    cout << "\n=== Tasks by สมชาย ===" << endl;
    tm.display(tm.getByAssignee("สมชาย"));
    
    cout << "\n=== Backend Tasks ===" << endl;
    tm.display(tm.getByTag("backend"));
    
    cout << "\n=== Sorted by Priority ===" << endl;
    tm.display(tm.getSorted([](const Task& a, const Task& b) {
        return static_cast<int>(a.priority) > static_cast<int>(b.priority);
    }));
    
    tm.updateStatus(t5, Status::DONE);
    tm.updateStatus(t1, Status::DONE);
    
    tm.summary();
    
    return 0;
}
```

---

## สรุป Part 009

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Move Semantics (Move Constructor, Move Assignment)
2. ✅ Smart Pointers (unique_ptr, shared_ptr, weak_ptr)
3. ✅ Threading และ Concurrency พื้นฐาน
4. ✅ C++17: optional, variant, any, string_view, structured bindings
5. ✅ C++17: Filesystem, fold expressions
6. ✅ C++20: Concepts, Ranges, span
7. ✅ Task Manager ด้วย Modern C++

---

## แบบฝึกหัด Part 009

**ฝึกหัดที่ 1:** Implement Thread-safe Queue ด้วย mutex และ condition_variable

**ฝึกหัดที่ 2:** สร้าง Pipeline ด้วย Ranges ที่ประมวลผลข้อมูลหลายขั้นตอน

**ฝึกหัดที่ 3:** สร้าง Type-safe Event System ด้วย variant

---

⬅️ **ก่อนหน้า:** [Part 008](part008.md) | ➡️ **ต่อไป:** [Part 010: Memory Management](part010.md)
