# Part 008: Templates และ STL (Standard Template Library)

## ขั้นตอนที่ 86-100

---

## ขั้นตอนที่ 86: Class Templates

```cpp
#include <iostream>
#include <stdexcept>
using namespace std;

// === Stack Template ===
template <typename T>
class Stack {
private:
    T* data;
    int capacity;
    int topIndex;
    
    void resize() {
        int newCapacity = capacity * 2;
        T* newData = new T[newCapacity];
        for (int i = 0; i <= topIndex; i++) {
            newData[i] = data[i];
        }
        delete[] data;
        data = newData;
        capacity = newCapacity;
    }
    
public:
    Stack(int cap = 10) : capacity(cap), topIndex(-1) {
        data = new T[capacity];
    }
    
    ~Stack() { delete[] data; }
    
    // Copy constructor
    Stack(const Stack& other) : capacity(other.capacity), topIndex(other.topIndex) {
        data = new T[capacity];
        for (int i = 0; i <= topIndex; i++) {
            data[i] = other.data[i];
        }
    }
    
    void push(const T& item) {
        if (topIndex + 1 >= capacity) resize();
        data[++topIndex] = item;
    }
    
    T pop() {
        if (isEmpty()) throw underflow_error("Stack is empty");
        return data[topIndex--];
    }
    
    T& top() {
        if (isEmpty()) throw underflow_error("Stack is empty");
        return data[topIndex];
    }
    
    bool isEmpty() const { return topIndex == -1; }
    int size() const { return topIndex + 1; }
    
    void print() const {
        cout << "Stack [bottom -> top]: ";
        for (int i = 0; i <= topIndex; i++) {
            cout << data[i];
            if (i < topIndex) cout << ", ";
        }
        cout << " (size=" << size() << ")" << endl;
    }
};

// === Queue Template ===
template <typename T>
class Queue {
private:
    T* data;
    int capacity;
    int frontIdx, backIdx, count;
    
public:
    Queue(int cap = 10) : capacity(cap), frontIdx(0), backIdx(-1), count(0) {
        data = new T[capacity];
    }
    
    ~Queue() { delete[] data; }
    
    void enqueue(const T& item) {
        if (count >= capacity) throw overflow_error("Queue is full");
        backIdx = (backIdx + 1) % capacity;
        data[backIdx] = item;
        count++;
    }
    
    T dequeue() {
        if (isEmpty()) throw underflow_error("Queue is empty");
        T item = data[frontIdx];
        frontIdx = (frontIdx + 1) % capacity;
        count--;
        return item;
    }
    
    T& front() {
        if (isEmpty()) throw underflow_error("Queue is empty");
        return data[frontIdx];
    }
    
    bool isEmpty() const { return count == 0; }
    int size() const { return count; }
};

// === Linked List Template ===
template <typename T>
class LinkedList {
private:
    struct Node {
        T data;
        Node* next;
        Node(const T& d) : data(d), next(nullptr) {}
    };
    
    Node* head;
    int size_;
    
public:
    LinkedList() : head(nullptr), size_(0) {}
    
    ~LinkedList() {
        Node* current = head;
        while (current) {
            Node* next = current->next;
            delete current;
            current = next;
        }
    }
    
    void pushFront(const T& data) {
        Node* node = new Node(data);
        node->next = head;
        head = node;
        size_++;
    }
    
    void pushBack(const T& data) {
        Node* node = new Node(data);
        if (!head) {
            head = node;
        } else {
            Node* current = head;
            while (current->next) current = current->next;
            current->next = node;
        }
        size_++;
    }
    
    void print() const {
        cout << "LinkedList: ";
        Node* current = head;
        while (current) {
            cout << current->data;
            if (current->next) cout << " -> ";
            current = current->next;
        }
        cout << " (size=" << size_ << ")" << endl;
    }
    
    bool contains(const T& data) const {
        Node* current = head;
        while (current) {
            if (current->data == data) return true;
            current = current->next;
        }
        return false;
    }
    
    void reverse() {
        Node* prev = nullptr;
        Node* current = head;
        while (current) {
            Node* next = current->next;
            current->next = prev;
            prev = current;
            current = next;
        }
        head = prev;
    }
    
    int size() const { return size_; }
};

int main() {
    // === Stack ===
    cout << "=== Stack<int> ===" << endl;
    
    Stack<int> intStack;
    for (int i = 1; i <= 5; i++) intStack.push(i * 10);
    intStack.print();
    
    cout << "Pop: " << intStack.pop() << endl;
    cout << "Top: " << intStack.top() << endl;
    intStack.print();
    
    cout << "\n=== Stack<string> ===" << endl;
    
    Stack<string> strStack;
    strStack.push("แรก");
    strStack.push("สอง");
    strStack.push("สาม");
    strStack.print();
    
    // ใช้สำหรับตรวจสอบ parentheses
    cout << "\n=== Balanced Parentheses ===" << endl;
    
    auto isBalanced = [](const string& s) {
        Stack<char> st;
        for (char c : s) {
            if (c == '(' || c == '[' || c == '{') {
                st.push(c);
            } else if (c == ')' || c == ']' || c == '}') {
                if (st.isEmpty()) return false;
                char top = st.pop();
                if ((c == ')' && top != '(') ||
                    (c == ']' && top != '[') ||
                    (c == '}' && top != '{')) return false;
            }
        }
        return st.isEmpty();
    };
    
    vector<string> tests = {"((()))", "([{}])", "(()", "([)]", "{[()]}"};
    for (const string& t : tests) {
        cout << "\"" << t << "\": " << (isBalanced(t) ? "balanced" : "not balanced") << endl;
    }
    
    // === Queue ===
    cout << "\n=== Queue<string> ===" << endl;
    
    Queue<string> q(5);
    q.enqueue("ลูกค้า A");
    q.enqueue("ลูกค้า B");
    q.enqueue("ลูกค้า C");
    
    cout << "ให้บริการ: " << q.dequeue() << endl;
    cout << "ให้บริการ: " << q.dequeue() << endl;
    q.enqueue("ลูกค้า D");
    cout << "ลูกค้าหน้าคิว: " << q.front() << endl;
    
    // === LinkedList ===
    cout << "\n=== LinkedList<int> ===" << endl;
    
    LinkedList<int> list;
    for (int i = 1; i <= 5; i++) list.pushBack(i);
    list.print();
    
    list.pushFront(0);
    list.print();
    
    list.reverse();
    cout << "Reversed: ";
    list.print();
    
    return 0;
}
```

---

## ขั้นตอนที่ 87: STL Containers

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <deque>
#include <set>
#include <unordered_set>
#include <map>
#include <unordered_map>
#include <queue>
#include <stack>
#include <string>
using namespace std;

int main() {
    // === vector ===
    cout << "=== vector ===" << endl;
    vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};
    v.push_back(5);
    v.insert(v.begin(), 0);
    cout << "size=" << v.size() << " capacity=" << v.capacity() << endl;
    
    // === list (Doubly Linked List) ===
    cout << "\n=== list ===" << endl;
    list<int> l = {1, 2, 3, 4, 5};
    l.push_front(0);
    l.push_back(6);
    l.remove(3);  // ลบค่า 3 ทั้งหมด
    
    cout << "list: ";
    for (int x : l) cout << x << " ";
    cout << endl;
    
    // === deque (Double-ended queue) ===
    cout << "\n=== deque ===" << endl;
    deque<int> d;
    d.push_back(3);
    d.push_back(4);
    d.push_front(2);
    d.push_front(1);
    
    cout << "deque: ";
    for (int x : d) cout << x << " ";
    cout << endl;
    d.pop_front();
    d.pop_back();
    cout << "after pop: ";
    for (int x : d) cout << x << " ";
    cout << endl;
    
    // === set (Sorted, unique) ===
    cout << "\n=== set ===" << endl;
    set<int> s = {5, 2, 8, 1, 9, 3, 7, 2, 5};  // duplicates ถูกลบ
    
    cout << "set: ";
    for (int x : s) cout << x << " ";  // sorted automatically
    cout << endl;
    
    s.insert(6);
    s.erase(3);
    cout << "contains 5: " << (s.count(5) ? "yes" : "no") << endl;
    cout << "contains 3: " << (s.count(3) ? "yes" : "no") << endl;
    
    // Find
    auto it = s.find(7);
    if (it != s.end()) cout << "found: " << *it << endl;
    
    // === multiset (Sorted, allows duplicates) ===
    cout << "\n=== multiset ===" << endl;
    multiset<int> ms = {3, 1, 4, 1, 5, 9, 2, 6, 5};
    cout << "multiset: ";
    for (int x : ms) cout << x << " ";
    cout << endl;
    cout << "count(5): " << ms.count(5) << endl;
    
    // === map (Key-Value, sorted by key) ===
    cout << "\n=== map ===" << endl;
    
    map<string, int> ages;
    ages["สมชาย"] = 25;
    ages["สมหญิง"] = 23;
    ages["สมศรี"] = 28;
    ages.insert({"สมพร", 30});
    ages.emplace("สมทรง", 35);
    
    for (const auto& [name, age] : ages) {
        cout << name << ": " << age << endl;
    }
    
    // Check key exists
    if (ages.count("สมชาย")) {
        cout << "สมชาย อายุ: " << ages["สมชาย"] << endl;
    }
    
    // Find
    auto mapIt = ages.find("สมหญิง");
    if (mapIt != ages.end()) {
        cout << "Found: " << mapIt->first << " = " << mapIt->second << endl;
    }
    
    ages.erase("สมศรี");
    cout << "Size after erase: " << ages.size() << endl;
    
    // === unordered_map (Hash Map - O(1) average) ===
    cout << "\n=== unordered_map ===" << endl;
    
    unordered_map<string, vector<string>> phoneBook;
    phoneBook["สมชาย"] = {"081-111-1111", "02-222-2222"};
    phoneBook["สมหญิง"] = {"082-333-3333"};
    
    for (const auto& [name, phones] : phoneBook) {
        cout << name << ": ";
        for (const string& phone : phones) cout << phone << " ";
        cout << endl;
    }
    
    // === priority_queue (Max Heap) ===
    cout << "\n=== priority_queue ===" << endl;
    
    priority_queue<int> pq;
    pq.push(3);
    pq.push(1);
    pq.push(7);
    pq.push(5);
    
    cout << "priority_queue (max first): ";
    while (!pq.empty()) {
        cout << pq.top() << " ";
        pq.pop();
    }
    cout << endl;
    
    // Min Heap
    priority_queue<int, vector<int>, greater<int>> minPq;
    minPq.push(3);
    minPq.push(1);
    minPq.push(7);
    minPq.push(5);
    
    cout << "min priority_queue: ";
    while (!minPq.empty()) {
        cout << minPq.top() << " ";
        minPq.pop();
    }
    cout << endl;
    
    // === stack ===
    cout << "\n=== stack ===" << endl;
    
    stack<string> stk;
    stk.push("ชั้น 1");
    stk.push("ชั้น 2");
    stk.push("ชั้น 3");
    
    cout << "Top: " << stk.top() << endl;
    while (!stk.empty()) {
        cout << stk.top() << " ";
        stk.pop();
    }
    cout << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 88: STL Algorithms

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <functional>
#include <iterator>
using namespace std;

int main() {
    vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5};
    
    // === Non-modifying algorithms ===
    cout << "=== Non-modifying ===" << endl;
    
    // find
    auto it = find(v.begin(), v.end(), 9);
    cout << "find(9): " << (it != v.end() ? "found at " + to_string(it - v.begin()) : "not found") << endl;
    
    // count
    cout << "count(5): " << count(v.begin(), v.end(), 5) << endl;
    
    // count_if
    int evens = count_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });
    cout << "count_if(even): " << evens << endl;
    
    // all_of, any_of, none_of
    cout << boolalpha;
    cout << "all_of > 0: " << all_of(v.begin(), v.end(), [](int x) { return x > 0; }) << endl;
    cout << "any_of > 8: " << any_of(v.begin(), v.end(), [](int x) { return x > 8; }) << endl;
    cout << "none_of < 0: " << none_of(v.begin(), v.end(), [](int x) { return x < 0; }) << endl;
    
    // min/max element
    cout << "min: " << *min_element(v.begin(), v.end()) << endl;
    cout << "max: " << *max_element(v.begin(), v.end()) << endl;
    
    // accumulate
    cout << "sum: " << accumulate(v.begin(), v.end(), 0) << endl;
    cout << "product: " << accumulate(v.begin(), v.end(), 1, multiplies<int>()) << endl;
    
    // === Modifying algorithms ===
    cout << "\n=== Modifying ===" << endl;
    
    vector<int> v2 = v;
    
    // sort
    sort(v2.begin(), v2.end());
    cout << "sorted: ";
    for (int x : v2) cout << x << " ";
    cout << endl;
    
    // unique (ต้อง sort ก่อน)
    auto newEnd = unique(v2.begin(), v2.end());
    v2.erase(newEnd, v2.end());
    cout << "unique: ";
    for (int x : v2) cout << x << " ";
    cout << endl;
    
    // reverse
    vector<int> v3 = {1, 2, 3, 4, 5};
    reverse(v3.begin(), v3.end());
    cout << "reversed: ";
    for (int x : v3) cout << x << " ";
    cout << endl;
    
    // rotate
    vector<int> v4 = {1, 2, 3, 4, 5};
    rotate(v4.begin(), v4.begin() + 2, v4.end());
    cout << "rotated by 2: ";
    for (int x : v4) cout << x << " ";
    cout << endl;
    
    // transform
    vector<int> v5 = {1, 2, 3, 4, 5};
    vector<int> v6(v5.size());
    transform(v5.begin(), v5.end(), v6.begin(), [](int x) { return x * x; });
    cout << "squared: ";
    for (int x : v6) cout << x << " ";
    cout << endl;
    
    // transform กับ 2 ranges
    vector<int> a = {1, 2, 3, 4, 5};
    vector<int> b = {10, 20, 30, 40, 50};
    vector<int> c(5);
    transform(a.begin(), a.end(), b.begin(), c.begin(), plus<int>());
    cout << "a+b: ";
    for (int x : c) cout << x << " ";
    cout << endl;
    
    // replace
    vector<int> v7 = {1, 2, 3, 2, 4, 2, 5};
    replace(v7.begin(), v7.end(), 2, 99);
    cout << "replace 2->99: ";
    for (int x : v7) cout << x << " ";
    cout << endl;
    
    // fill
    vector<int> v8(5);
    fill(v8.begin(), v8.end(), 42);
    cout << "fill 42: ";
    for (int x : v8) cout << x << " ";
    cout << endl;
    
    // generate
    vector<int> v9(5);
    int gen_val = 0;
    generate(v9.begin(), v9.end(), [&gen_val]() { return gen_val += 5; });
    cout << "generate (multiples of 5): ";
    for (int x : v9) cout << x << " ";
    cout << endl;
    
    // remove_if + erase
    vector<int> v10 = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    v10.erase(remove_if(v10.begin(), v10.end(), [](int x) { return x % 2 == 0; }), v10.end());
    cout << "remove evens: ";
    for (int x : v10) cout << x << " ";
    cout << endl;
    
    // === Set operations ===
    cout << "\n=== Set Operations ===" << endl;
    
    vector<int> s1 = {1, 2, 3, 4, 5};
    vector<int> s2 = {3, 4, 5, 6, 7};
    vector<int> result;
    
    // Union
    set_union(s1.begin(), s1.end(), s2.begin(), s2.end(), back_inserter(result));
    cout << "union: ";
    for (int x : result) cout << x << " ";
    cout << endl;
    
    result.clear();
    set_intersection(s1.begin(), s1.end(), s2.begin(), s2.end(), back_inserter(result));
    cout << "intersection: ";
    for (int x : result) cout << x << " ";
    cout << endl;
    
    result.clear();
    set_difference(s1.begin(), s1.end(), s2.begin(), s2.end(), back_inserter(result));
    cout << "difference (s1-s2): ";
    for (int x : result) cout << x << " ";
    cout << endl;
    
    // === Partitioning ===
    cout << "\n=== Partitioning ===" << endl;
    
    vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    auto partIt = partition(nums.begin(), nums.end(), [](int x) { return x % 2 == 0; });
    
    cout << "evens first: ";
    for (int x : nums) cout << x << " ";
    cout << endl;
    cout << "partition point: " << (partIt - nums.begin()) << endl;
    
    // nth_element
    vector<int> data = {9, 1, 8, 2, 7, 3, 6, 4, 5};
    nth_element(data.begin(), data.begin() + 4, data.end());
    cout << "5th element: " << data[4] << endl;  // Median
    
    return 0;
}
```

---

## ขั้นตอนที่ 89-90: Iterators

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <map>
#include <set>
#include <iterator>
#include <algorithm>
using namespace std;

// Custom Iterator
template <typename T>
class Range {
private:
    T begin_, end_, step_;
    
public:
    Range(T begin, T end, T step = 1)
        : begin_(begin), end_(end), step_(step) {}
    
    class Iterator {
    private:
        T current, step;
    public:
        Iterator(T val, T s) : current(val), step(s) {}
        
        T operator*() const { return current; }
        
        Iterator& operator++() {
            current += step;
            return *this;
        }
        
        bool operator!=(const Iterator& other) const {
            return current < other.current;
        }
    };
    
    Iterator begin() const { return Iterator(begin_, step_); }
    Iterator end() const { return Iterator(end_, step_); }
};

int main() {
    // === Iterator Types ===
    cout << "=== Iterator Types ===" << endl;
    
    vector<int> v = {1, 2, 3, 4, 5};
    
    // Forward iterator
    cout << "forward: ";
    for (auto it = v.begin(); it != v.end(); ++it) {
        cout << *it << " ";
    }
    cout << endl;
    
    // Reverse iterator
    cout << "reverse: ";
    for (auto it = v.rbegin(); it != v.rend(); ++it) {
        cout << *it << " ";
    }
    cout << endl;
    
    // Const iterator
    cout << "const: ";
    for (auto it = v.cbegin(); it != v.cend(); ++it) {
        cout << *it << " ";
        // *it = 10;  // Error! const iterator
    }
    cout << endl;
    
    // === Iterator Operations ===
    cout << "\n=== Iterator Ops ===" << endl;
    
    auto it = v.begin();
    advance(it, 2);  // เลื่อนไป 2 ตำแหน่ง
    cout << "after advance(2): " << *it << endl;  // 3
    
    auto it2 = v.end();
    cout << "distance(begin, end): " << distance(v.begin(), v.end()) << endl;  // 5
    
    cout << "next(begin, 3): " << *next(v.begin(), 3) << endl;  // 4
    cout << "prev(end, 2): " << *prev(v.end(), 2) << endl;      // 4
    
    // === Insert Iterators ===
    cout << "\n=== Insert Iterators ===" << endl;
    
    vector<int> dest;
    vector<int> src = {10, 20, 30};
    
    // back_inserter
    copy(src.begin(), src.end(), back_inserter(dest));
    cout << "back_inserter: ";
    for (int x : dest) cout << x << " ";
    cout << endl;
    
    // front_inserter (ใช้กับ list)
    list<int> lst = {4, 5, 6};
    list<int> src2 = {1, 2, 3};
    copy(src2.begin(), src2.end(), front_inserter(lst));
    cout << "front_inserter: ";
    for (int x : lst) cout << x << " ";
    cout << endl;
    
    // inserter
    vector<int> target = {1, 5, 10};
    vector<int> toInsert = {2, 3, 4};
    auto insertPos = target.begin() + 1;
    copy(toInsert.begin(), toInsert.end(), inserter(target, insertPos));
    cout << "inserter: ";
    for (int x : target) cout << x << " ";
    cout << endl;
    
    // === Stream Iterators ===
    cout << "\n=== Stream Iterators ===" << endl;
    
    vector<int> nums = {1, 2, 3, 4, 5};
    
    // ostream_iterator
    cout << "ostream_iterator: ";
    copy(nums.begin(), nums.end(), ostream_iterator<int>(cout, " "));
    cout << endl;
    
    // === Custom Range Iterator ===
    cout << "\n=== Custom Range ===" << endl;
    
    for (int i : Range(0, 10)) {
        cout << i << " ";
    }
    cout << endl;
    
    for (int i : Range(0, 20, 3)) {
        cout << i << " ";
    }
    cout << endl;
    
    for (double d : Range(0.0, 1.0, 0.1)) {
        cout << d << " ";
    }
    cout << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 91-100: STL ในชีวิตจริง

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <unordered_map>
#include <set>
#include <algorithm>
#include <numeric>
#include <string>
#include <sstream>
#include <iomanip>
using namespace std;

// === Word Frequency Counter ===
map<string, int> countWords(const string& text) {
    map<string, int> freq;
    istringstream iss(text);
    string word;
    
    while (iss >> word) {
        // Remove punctuation
        word.erase(remove_if(word.begin(), word.end(), ::ispunct), word.end());
        // Lowercase
        transform(word.begin(), word.end(), word.begin(), ::tolower);
        if (!word.empty()) freq[word]++;
    }
    
    return freq;
}

// === Graph with Adjacency List ===
class Graph {
private:
    int vertices;
    unordered_map<int, vector<int>> adjList;
    
public:
    Graph(int v) : vertices(v) {}
    
    void addEdge(int u, int v, bool directed = false) {
        adjList[u].push_back(v);
        if (!directed) adjList[v].push_back(u);
    }
    
    void BFS(int start) {
        vector<bool> visited(vertices, false);
        queue<int> q;
        
        visited[start] = true;
        q.push(start);
        
        cout << "BFS from " << start << ": ";
        while (!q.empty()) {
            int v = q.front();
            q.pop();
            cout << v << " ";
            
            for (int neighbor : adjList[v]) {
                if (!visited[neighbor]) {
                    visited[neighbor] = true;
                    q.push(neighbor);
                }
            }
        }
        cout << endl;
    }
    
    void DFS(int start) {
        vector<bool> visited(vertices, false);
        stack<int> stk;
        
        stk.push(start);
        cout << "DFS from " << start << ": ";
        
        while (!stk.empty()) {
            int v = stk.top();
            stk.pop();
            
            if (!visited[v]) {
                visited[v] = true;
                cout << v << " ";
                
                for (int neighbor : adjList[v]) {
                    if (!visited[neighbor]) {
                        stk.push(neighbor);
                    }
                }
            }
        }
        cout << endl;
    }
};

// === Cache (LRU) ===
class LRUCache {
private:
    int capacity;
    list<pair<int, int>> cache;  // {key, value}
    unordered_map<int, list<pair<int, int>>::iterator> map;
    
public:
    LRUCache(int cap) : capacity(cap) {}
    
    int get(int key) {
        if (map.find(key) == map.end()) return -1;
        
        // Move to front (most recently used)
        cache.splice(cache.begin(), cache, map[key]);
        return map[key]->second;
    }
    
    void put(int key, int value) {
        if (map.find(key) != map.end()) {
            map[key]->second = value;
            cache.splice(cache.begin(), cache, map[key]);
        } else {
            if (cache.size() >= capacity) {
                // Remove LRU (back)
                auto last = cache.back();
                map.erase(last.first);
                cache.pop_back();
            }
            cache.push_front({key, value});
            map[key] = cache.begin();
        }
    }
    
    void display() const {
        cout << "LRU Cache [front=recent]: ";
        for (const auto& [k, v] : cache) {
            cout << k << ":" << v << " ";
        }
        cout << endl;
    }
};

// === Student Grade Analyzer ===
struct StudentRecord {
    string name;
    vector<double> scores;
    
    double getAverage() const {
        if (scores.empty()) return 0;
        return accumulate(scores.begin(), scores.end(), 0.0) / scores.size();
    }
    
    double getMax() const {
        return *max_element(scores.begin(), scores.end());
    }
    
    double getMin() const {
        return *min_element(scores.begin(), scores.end());
    }
    
    char getGrade() const {
        double avg = getAverage();
        if (avg >= 90) return 'A';
        if (avg >= 80) return 'B';
        if (avg >= 70) return 'C';
        if (avg >= 60) return 'D';
        return 'F';
    }
};

int main() {
    // === Word Frequency ===
    cout << "=== Word Frequency ===" << endl;
    
    string text = "the quick brown fox jumps over the lazy dog the fox";
    auto freq = countWords(text);
    
    // Sort by frequency (descending)
    vector<pair<string, int>> sortedFreq(freq.begin(), freq.end());
    sort(sortedFreq.begin(), sortedFreq.end(),
         [](const auto& a, const auto& b) { return a.second > b.second; });
    
    for (const auto& [word, count] : sortedFreq) {
        cout << setw(10) << word << ": " << count;
        for (int i = 0; i < count; i++) cout << "█";
        cout << endl;
    }
    
    // === Graph ===
    cout << "\n=== Graph Traversal ===" << endl;
    
    Graph g(6);
    g.addEdge(0, 1);
    g.addEdge(0, 2);
    g.addEdge(1, 3);
    g.addEdge(2, 4);
    g.addEdge(3, 5);
    g.addEdge(4, 5);
    
    g.BFS(0);
    g.DFS(0);
    
    // === LRU Cache ===
    cout << "\n=== LRU Cache ===" << endl;
    
    LRUCache cache(3);
    cache.put(1, 100);
    cache.put(2, 200);
    cache.put(3, 300);
    cache.display();
    
    cout << "get(1) = " << cache.get(1) << endl;
    cache.display();
    
    cache.put(4, 400);  // เอา LRU ออก (key 2)
    cache.display();
    
    cout << "get(2) = " << cache.get(2) << " (evicted)" << endl;
    
    // === Student Analysis ===
    cout << "\n=== Student Analysis ===" << endl;
    
    vector<StudentRecord> students = {
        {"สมชาย", {85, 90, 78, 92, 88}},
        {"สมหญิง", {72, 68, 75, 80, 71}},
        {"สมศรี", {95, 98, 92, 96, 94}},
        {"สมพร", {55, 60, 58, 65, 62}},
    };
    
    // Sort by average descending
    sort(students.begin(), students.end(), [](const StudentRecord& a, const StudentRecord& b) {
        return a.getAverage() > b.getAverage();
    });
    
    cout << fixed << setprecision(1);
    cout << setw(15) << "ชื่อ" << setw(8) << "เฉลี่ย" 
         << setw(6) << "สูงสุด" << setw(6) << "ต่ำสุด" << setw(6) << "เกรด" << endl;
    cout << string(41, '-') << endl;
    
    for (const auto& s : students) {
        cout << setw(15) << s.name 
             << setw(8) << s.getAverage()
             << setw(6) << s.getMax()
             << setw(6) << s.getMin()
             << setw(6) << s.getGrade() << endl;
    }
    
    // Class stats
    double classAvg = accumulate(students.begin(), students.end(), 0.0,
        [](double sum, const StudentRecord& s) { return sum + s.getAverage(); }) / students.size();
    
    auto best = max_element(students.begin(), students.end(),
        [](const StudentRecord& a, const StudentRecord& b) {
            return a.getAverage() < b.getAverage();
        });
    
    cout << "\nค่าเฉลี่ยห้อง: " << classAvg << endl;
    cout << "นักเรียนดีเด่น: " << best->name << " (" << best->getAverage() << ")" << endl;
    
    // Grade distribution
    map<char, int> gradeDist;
    for (const auto& s : students) gradeDist[s.getGrade()]++;
    
    cout << "\nการกระจายเกรด:" << endl;
    for (const auto& [grade, count] : gradeDist) {
        cout << "เกรด " << grade << ": " << count << " คน" << endl;
    }
    
    return 0;
}
```

---

## สรุป Part 008

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Class Templates (Stack, Queue, LinkedList)
2. ✅ STL Containers (vector, list, deque, set, map, unordered_map)
3. ✅ STL Algorithms (sort, find, transform, etc.)
4. ✅ Iterators ประเภทต่างๆ
5. ✅ Insert Iterators
6. ✅ Custom Iterators
7. ✅ Word Frequency Counter
8. ✅ Graph Traversal (BFS, DFS)
9. ✅ LRU Cache
10. ✅ Student Grade Analysis

---

## แบบฝึกหัด Part 008

**ฝึกหัดที่ 1:** สร้าง Generic `BinarySearchTree<T>` ที่รองรับ insert, search, inorder traversal

**ฝึกหัดที่ 2:** สร้าง `PriorityQueue<T>` ที่ใช้ Heap

**ฝึกหัดที่ 3:** สร้างโปรแกรม Anagram Checker ด้วย `map<char, int>`

---

⬅️ **ก่อนหน้า:** [Part 007](part007.md) | ➡️ **ต่อไป:** [Part 009: Modern C++](part009.md)
