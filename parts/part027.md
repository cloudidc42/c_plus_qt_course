# Part 027: Template Metaprogramming (TMP)

## ขั้นตอนที่ 371-385

---

## ขั้นตอนที่ 371: Templates พื้นฐาน

```cpp
#include <type_traits>
#include <tuple>
#include <functional>
#include <array>

// === Function Templates ===
template<typename T>
T max_val(T a, T b) { return a > b ? a : b; }

// Specialization
template<>
const char* max_val<const char*>(const char* a, const char* b) {
    return std::strcmp(a, b) > 0 ? a : b;
}

// Variadic template
template<typename T>
T sum(T t) { return t; }

template<typename T, typename... Args>
T sum(T first, Args... rest) {
    return first + sum(rest...);
}

// Fold expressions (C++17)
template<typename... Ts>
auto sum_fold(Ts... args) {
    return (... + args);  // unary left fold
}

template<typename... Ts>
void print_all(Ts... args) {
    ((std::cout << args << ' '), ...);  // fold over comma
    std::cout << '\n';
}

// === Class Templates ===
template<typename T, std::size_t N>
class FixedArray {
public:
    FixedArray() : m_size(0) {}
    
    void push(const T& val) {
        if (m_size < N) m_data[m_size++] = val;
    }
    
    T pop() {
        if (m_size == 0) throw std::underflow_error("empty");
        return m_data[--m_size];
    }
    
    T& operator[](std::size_t idx) { return m_data[idx]; }
    const T& operator[](std::size_t idx) const { return m_data[idx]; }
    
    std::size_t size() const { return m_size; }
    static constexpr std::size_t capacity() { return N; }
    
    T* begin() { return m_data.data(); }
    T* end() { return m_data.data() + m_size; }
    
private:
    std::array<T, N> m_data;
    std::size_t m_size;
};

// Usage
void demoTemplates() {
    std::cout << max_val(3, 7) << '\n';     // 7
    std::cout << max_val(3.14, 2.71) << '\n'; // 3.14
    
    std::cout << sum(1, 2, 3, 4, 5) << '\n';    // 15
    std::cout << sum_fold(1, 2, 3, 4, 5) << '\n'; // 15
    
    print_all("Hello", 42, 3.14, true);
    
    FixedArray<int, 10> arr;
    arr.push(1); arr.push(2); arr.push(3);
    for (int x : arr) std::cout << x << ' '; // 1 2 3
}
```

---

## ขั้นตอนที่ 372: Type Traits

```cpp
#include <type_traits>

// === Using type_traits ===
template<typename T>
void printTypeInfo() {
    std::cout << "Type: " << typeid(T).name() << '\n';
    std::cout << "  is_integral:    " << std::is_integral_v<T> << '\n';
    std::cout << "  is_floating:    " << std::is_floating_point_v<T> << '\n';
    std::cout << "  is_pointer:     " << std::is_pointer_v<T> << '\n';
    std::cout << "  is_const:       " << std::is_const_v<T> << '\n';
    std::cout << "  sizeof:         " << sizeof(T) << '\n';
    std::cout << "  is_trivially_copyable: " << std::is_trivially_copyable_v<T> << '\n';
}

// Enable/disable overloads with enable_if
template<typename T, std::enable_if_t<std::is_integral_v<T>, bool> = true>
T square(T x) {
    std::cout << "[integer] ";
    return x * x;
}

template<typename T, std::enable_if_t<std::is_floating_point_v<T>, bool> = true>
T square(T x) {
    std::cout << "[float] ";
    return x * x;
}

// Custom type trait
template<typename T>
struct is_string : std::false_type {};

template<>
struct is_string<std::string> : std::true_type {};

template<>
struct is_string<std::string_view> : std::true_type {};

template<>
struct is_string<QString> : std::true_type {};

template<typename T>
constexpr bool is_string_v = is_string<T>::value;

// Conditional type selection
template<typename T>
using AddPtr = std::add_pointer_t<T>;  // T -> T*

template<bool Cond, typename TrueT, typename FalseT>
using SelectType = std::conditional_t<Cond, TrueT, FalseT>;

// Type list operations
template<typename... Ts>
struct TypeList {
    static constexpr std::size_t size = sizeof...(Ts);
};

template<std::size_t N, typename... Ts>
using NthType = std::tuple_element_t<N, std::tuple<Ts...>>;

// Usage
void demoTypeTraits() {
    printTypeInfo<int>();
    printTypeInfo<double>();
    printTypeInfo<const char*>();
    
    std::cout << square(5) << '\n';   // [integer] 25
    std::cout << square(2.5) << '\n'; // [float] 6.25
    
    static_assert(is_string_v<std::string>);
    static_assert(!is_string_v<int>);
    
    using BigInt = SelectType<sizeof(int) >= 4, int64_t, int32_t>;
    std::cout << sizeof(BigInt) << '\n';
    
    using MyList = TypeList<int, double, std::string, bool>;
    static_assert(MyList::size == 4);
    using Third = NthType<2, int, double, std::string, bool>; // std::string
}
```

---

## ขั้นตอนที่ 373: SFINAE & Concepts (C++20)

```cpp
// === SFINAE (Substitution Failure Is Not An Error) ===

// detect member function existence
template<typename T, typename = void>
struct has_size : std::false_type {};

template<typename T>
struct has_size<T, std::void_t<decltype(std::declval<T>().size())>> : std::true_type {};

template<typename T>
struct has_begin : std::false_type {};

template<typename T>
struct has_begin<T, std::void_t<decltype(std::declval<T>().begin())>> : std::true_type {};

template<typename T>
constexpr bool is_container_v = has_size<T>::value && has_begin<T>::value;

// Print container if it has begin/end
template<typename T>
std::enable_if_t<is_container_v<T>> printContainer(const T& c) {
    std::cout << "[";
    bool first = true;
    for (const auto& item : c) {
        if (!first) std::cout << ", ";
        std::cout << item;
        first = false;
    }
    std::cout << "]\n";
}

template<typename T>
std::enable_if_t<!is_container_v<T>> printContainer(const T& val) {
    std::cout << val << '\n';
}

// === C++20 Concepts ===
#include <concepts>

template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

template<typename T>
concept Container = requires(T t) {
    { t.begin() } -> std::input_iterator;
    { t.end() }   -> std::input_iterator;
    { t.size() }  -> std::convertible_to<std::size_t>;
    typename T::value_type;
};

template<typename T>
concept Printable = requires(T t) {
    { std::cout << t };
};

template<typename T>
concept Comparable = requires(T a, T b) {
    { a == b } -> std::convertible_to<bool>;
    { a < b  } -> std::convertible_to<bool>;
};

template<typename T>
concept Sortable = Container<T> && Comparable<typename T::value_type>;

// Constrained functions
template<Numeric T>
T absoluteValue(T x) { return x < 0 ? -x : x; }

template<Container C>
void printAll(const C& container) {
    for (const auto& item : container) {
        if constexpr (Printable<decltype(item)>) {
            std::cout << item << ' ';
        }
    }
    std::cout << '\n';
}

template<Sortable C>
void sortContainer(C& c) {
    std::sort(c.begin(), c.end());
}

// Custom concept with compound requirements
template<typename T>
concept Serializable = requires(T t, std::ostream& os, std::istream& is) {
    { t.serialize(os) }   -> std::same_as<void>;
    { T::deserialize(is) } -> std::same_as<T>;
};
```

---

## ขั้นตอนที่ 374: Policy-Based Design

```cpp
// === Policy Classes ===

// Storage policy
template<typename T>
class VectorStorage {
protected:
    std::vector<T> m_data;
    
    void store(const T& val) { m_data.push_back(val); }
    T retrieve(std::size_t idx) { return m_data[idx]; }
    std::size_t dataSize() const { return m_data.size(); }
};

template<typename T>
class ListStorage {
protected:
    std::list<T> m_data;
    
    void store(const T& val) { m_data.push_back(val); }
    T retrieve(std::size_t idx) {
        auto it = m_data.begin();
        std::advance(it, idx);
        return *it;
    }
    std::size_t dataSize() const { return m_data.size(); }
};

// Locking policy
class MutexLock {
protected:
    std::mutex m_mutex;
    
    void lock()   { m_mutex.lock(); }
    void unlock() { m_mutex.unlock(); }
};

class NoLock {
protected:
    void lock()   {}
    void unlock() {}
};

// Thread-safe collection using policy-based design
template<
    typename T,
    template<typename> class StoragePolicy = VectorStorage,
    class LockPolicy = NoLock
>
class PolicyCollection 
    : private StoragePolicy<T>
    , private LockPolicy 
{
public:
    void add(const T& val) {
        this->lock();
        this->store(val);
        this->unlock();
    }
    
    T get(std::size_t idx) {
        this->lock();
        T result = this->retrieve(idx);
        this->unlock();
        return result;
    }
    
    std::size_t size() const { return this->dataSize(); }
};

// Different configurations
using SimpleIntList = PolicyCollection<int>;
using ThreadSafeStringList = PolicyCollection<std::string, VectorStorage, MutexLock>;
using LinkedIntList = PolicyCollection<int, ListStorage>;

void demoPolicyDesign() {
    SimpleIntList list;
    list.add(1); list.add(2); list.add(3);
    
    ThreadSafeStringList safe;
    // Can be used from multiple threads
    safe.add("Hello"); safe.add("World");
    
    std::cout << "Simple size: " << list.size() << '\n';
    std::cout << "Safe: " << safe.get(0) << '\n';
}
```

---

## ขั้นตอนที่ 375: CRTP (Curiously Recurring Template Pattern)

```cpp
// === CRTP Base ===
template<typename Derived>
class Cloneable {
public:
    std::unique_ptr<Derived> clone() const {
        return std::make_unique<Derived>(static_cast<const Derived&>(*this));
    }
};

template<typename Derived>
class Comparable {
public:
    bool operator==(const Derived& other) const {
        return static_cast<const Derived*>(this)->equals(other);
    }
    bool operator!=(const Derived& other) const { return !(*this == other); }
    bool operator<=(const Derived& other) const {
        const Derived* d = static_cast<const Derived*>(this);
        return d->equals(other) || d->lessThan(other);
    }
    bool operator>=(const Derived& other) const {
        const Derived* d = static_cast<const Derived*>(this);
        return d->equals(other) || !d->lessThan(other);
    }
    bool operator<(const Derived& other) const {
        return static_cast<const Derived*>(this)->lessThan(other);
    }
    bool operator>(const Derived& other) const {
        const Derived* d = static_cast<const Derived*>(this);
        return !d->equals(other) && !d->lessThan(other);
    }
};

// CRTP for static polymorphism
template<typename Derived>
class Shape {
public:
    double area() const {
        return static_cast<const Derived*>(this)->areaImpl();
    }
    double perimeter() const {
        return static_cast<const Derived*>(this)->perimeterImpl();
    }
    void print() const {
        std::cout << "Area: " << area() << ", Perimeter: " << perimeter() << '\n';
    }
};

class Circle : public Shape<Circle>, public Comparable<Circle>, public Cloneable<Circle> {
public:
    explicit Circle(double r) : m_r(r) {}
    
    double areaImpl() const { return M_PI * m_r * m_r; }
    double perimeterImpl() const { return 2 * M_PI * m_r; }
    
    bool equals(const Circle& o) const { return m_r == o.m_r; }
    bool lessThan(const Circle& o) const { return m_r < o.m_r; }
    
    double radius() const { return m_r; }
    
private:
    double m_r;
};

class Rectangle : public Shape<Rectangle> {
public:
    Rectangle(double w, double h) : m_w(w), m_h(h) {}
    
    double areaImpl() const { return m_w * m_h; }
    double perimeterImpl() const { return 2 * (m_w + m_h); }
    
private:
    double m_w, m_h;
};

void demoCRTP() {
    Circle c1(5.0), c2(3.0);
    c1.print();
    c2.print();
    
    std::cout << "c1 > c2: " << (c1 > c2) << '\n'; // true
    std::cout << "c1 == c1: " << (c1 == c1) << '\n'; // true
    
    auto copy = c1.clone();
    std::cout << "Clone radius: " << copy->radius() << '\n'; // 5.0
    
    Rectangle rect(4, 6);
    rect.print();
}
```

---

## ขั้นตอนที่ 376-385: Expression Templates

```cpp
// === Expression Templates ===
// เพื่อหลีกเลี่ยงการสร้าง temporary objects ระหว่าง vector operations

template<typename E>
class VecExpr {
public:
    double operator[](std::size_t i) const {
        return static_cast<const E&>(*this)[i];
    }
    std::size_t size() const {
        return static_cast<const E&>(*this).size();
    }
};

class Vec : public VecExpr<Vec> {
public:
    Vec(std::size_t n, double val = 0) : m_data(n, val) {}
    
    // Construct from expression
    template<typename E>
    Vec(const VecExpr<E>& expr) : m_data(expr.size()) {
        for (std::size_t i = 0; i < m_data.size(); i++) {
            m_data[i] = expr[i];
        }
    }
    
    double operator[](std::size_t i) const { return m_data[i]; }
    double& operator[](std::size_t i) { return m_data[i]; }
    std::size_t size() const { return m_data.size(); }
    
    void print() const {
        std::cout << "[";
        for (std::size_t i = 0; i < m_data.size(); i++) {
            if (i > 0) std::cout << ", ";
            std::cout << m_data[i];
        }
        std::cout << "]\n";
    }
    
private:
    std::vector<double> m_data;
};

template<typename L, typename R>
class VecAdd : public VecExpr<VecAdd<L, R>> {
public:
    VecAdd(const L& l, const R& r) : m_l(l), m_r(r) {}
    
    double operator[](std::size_t i) const { return m_l[i] + m_r[i]; }
    std::size_t size() const { return m_l.size(); }
    
private:
    const L& m_l;
    const R& m_r;
};

template<typename L, typename R>
VecAdd<L, R> operator+(const VecExpr<L>& l, const VecExpr<R>& r) {
    return VecAdd<L, R>(static_cast<const L&>(l), static_cast<const R&>(r));
}

template<typename L, typename R>
class VecMul : public VecExpr<VecMul<L, R>> {
public:
    VecMul(const L& l, const R& r) : m_l(l), m_r(r) {}
    
    double operator[](std::size_t i) const { return m_l[i] * m_r[i]; }
    std::size_t size() const { return m_l.size(); }
    
private:
    const L& m_l;
    const R& m_r;
};

template<typename L, typename R>
VecMul<L, R> operator*(const VecExpr<L>& l, const VecExpr<R>& r) {
    return VecMul<L, R>(static_cast<const L&>(l), static_cast<const R&>(r));
}

void demoExpressionTemplates() {
    Vec a(5, 1.0), b(5, 2.0), c(5, 3.0);
    
    // a + b + c => ไม่มี temporary vector
    // คือ VecAdd<VecAdd<Vec,Vec>, Vec>
    Vec result = a + b + c;
    result.print(); // [6, 6, 6, 6, 6]
    
    Vec result2 = (a + b) * c;
    result2.print(); // [9, 9, 9, 9, 9]
}
```

---

## สรุป Part 027

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Function & Class templates
2. ✅ Type traits และ enable_if
3. ✅ SFINAE และ C++20 Concepts
4. ✅ Policy-Based Design
5. ✅ CRTP (static polymorphism)
6. ✅ Expression Templates

---

⬅️ [Part 026](part026.md) | ➡️ [Part 028: Advanced STL Algorithms](part028.md)
