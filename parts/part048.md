# Part 048: World-Class Level — Advanced C++ Metaprogramming II

## ขั้นตอนที่ 686-700

---

## ขั้นตอนที่ 686: Compile-Time String Processing

```cpp
#include <array>
#include <string_view>

// === Compile-time string ===
template<std::size_t N>
struct FixedString {
    constexpr FixedString(const char (&arr)[N]) {
        for (std::size_t i = 0; i < N; ++i) data[i] = arr[i];
    }
    
    constexpr std::size_t size() const { return N - 1; }
    constexpr const char* c_str() const { return data.data(); }
    constexpr std::string_view view() const { return {data.data(), N - 1}; }
    
    constexpr char operator[](std::size_t i) const { return data[i]; }
    
    constexpr bool operator==(const FixedString& other) const {
        if (N != other.size() + 1) return false;
        for (std::size_t i = 0; i < N - 1; ++i)
            if (data[i] != other.data[i]) return false;
        return true;
    }
    
    std::array<char, N> data{};
};

// Deduction guide
template<std::size_t N>
FixedString(const char (&)[N]) -> FixedString<N>;

// === Compile-time hash ===
consteval std::uint32_t hash32(std::string_view s) {
    std::uint32_t h = 2166136261u;
    for (char c : s) {
        h ^= static_cast<unsigned char>(c);
        h *= 16777619u;
    }
    return h;
}

// === Usage in templates ===
template<FixedString Name, typename T>
struct TypedField {
    static constexpr std::string_view name = Name.view();
    T value{};
    
    TypedField() = default;
    explicit TypedField(T v) : value(std::move(v)) {}
};

// Define a typed record at compile time
struct UserRecord {
    TypedField<"id",    int>         id;
    TypedField<"name",  std::string> name;
    TypedField<"email", std::string> email;
    TypedField<"score", double>      score;
};

void compileTimeStringDemo() {
    constexpr auto name = FixedString("HelloWorld");
    static_assert(name.size() == 10);
    static_assert(name[0] == 'H');
    
    constexpr auto h = hash32("qt_app");
    static_assert(h != 0);
    
    UserRecord user;
    user.id.value = 42;
    user.name.value = "Alice";
    user.email.value = "alice@example.com";
    
    // Field names available at compile time
    qDebug() << "Field name:" << QString::fromStdString(std::string(UserRecord::id_type::name));
}
```

---

## ขั้นตอนที่ 687: Variadic Templates Advanced

```cpp
// === Type list ===
template<typename... Ts>
struct TypeList {
    static constexpr std::size_t size = sizeof...(Ts);
};

// Concatenate type lists
template<typename L1, typename L2>
struct Concat;

template<typename... T1, typename... T2>
struct Concat<TypeList<T1...>, TypeList<T2...>> {
    using type = TypeList<T1..., T2...>;
};

// Get Nth type
template<std::size_t N, typename... Ts>
struct NthType;

template<std::size_t N, typename T, typename... Rest>
struct NthType<N, T, Rest...> {
    using type = typename NthType<N - 1, Rest...>::type;
};

template<typename T, typename... Rest>
struct NthType<0, T, Rest...> {
    using type = T;
};

// Check if type is in list
template<typename T, typename... Ts>
struct Contains : std::disjunction<std::is_same<T, Ts>...> {};

// Filter type list
template<template<typename> class Pred, typename... Ts>
struct Filter {
    using type = TypeList<>;
};

template<template<typename> class Pred, typename T, typename... Rest>
struct Filter<Pred, T, Rest...> {
    using RestFiltered = typename Filter<Pred, Rest...>::type;
    using type = std::conditional_t<
        Pred<T>::value,
        typename Concat<TypeList<T>, RestFiltered>::type,
        RestFiltered
    >;
};

// === Tuple apply ===
template<typename Func, typename Tuple, std::size_t... Indices>
auto tupleApplyImpl(Func&& func, Tuple&& tup, std::index_sequence<Indices...>) {
    return func(std::get<Indices>(std::forward<Tuple>(tup))...);
}

template<typename Func, typename Tuple>
auto tupleApply(Func&& func, Tuple&& tup) {
    constexpr auto size = std::tuple_size_v<std::decay_t<Tuple>>;
    return tupleApplyImpl(
        std::forward<Func>(func),
        std::forward<Tuple>(tup),
        std::make_index_sequence<size>{}
    );
}

// === Fold expressions with complex operations ===
template<typename... Args>
void printAll(Args&&... args) {
    ((std::cout << std::forward<Args>(args) << " "), ...);
    std::cout << '\n';
}

template<typename... Containers>
auto mergeAll(Containers&&... containers) {
    using value_type = typename std::common_type_t<
        typename std::decay_t<Containers>::value_type...>;
    
    std::vector<value_type> result;
    result.reserve((containers.size() + ...));
    
    (result.insert(result.end(), containers.begin(), containers.end()), ...);
    return result;
}

// Usage:
// printAll(1, "hello", 3.14, 'x');
// auto merged = mergeAll(vec1, vec2, vec3);
```

---

## ขั้นตอนที่ 688: Compile-Time State Machine

```cpp
// === Compile-time FSM using templates ===
template<typename StateEnum, StateEnum From, typename Event, StateEnum To>
struct Transition {
    using state_type = StateEnum;
    using event_type = Event;
    static constexpr StateEnum from = From;
    static constexpr StateEnum to = To;
};

template<typename... Transitions>
struct TransitionTable {};

// State machine
template<typename StateEnum, StateEnum Initial, typename Table>
class StateMachine;

template<typename StateEnum, StateEnum Initial, typename... Ts>
class StateMachine<StateEnum, Initial, TransitionTable<Ts...>> {
public:
    StateEnum state() const { return m_state; }
    
    template<typename Event>
    bool process(const Event& event) {
        return processImpl<Event, Ts...>(event);
    }
    
    template<typename Event>
    bool can(const Event& = {}) const {
        return canImpl<Event, Ts...>();
    }
    
private:
    template<typename Event>
    bool processImpl(const Event&) { return false; }
    
    template<typename Event, typename T, typename... Rest>
    bool processImpl(const Event& event) {
        if constexpr (std::is_same_v<typename T::event_type, Event>) {
            if (m_state == T::from) {
                m_state = T::to;
                return true;
            }
        }
        return processImpl<Event, Rest...>(event);
    }
    
    template<typename Event>
    bool canImpl() const { return false; }
    
    template<typename Event, typename T, typename... Rest>
    bool canImpl() const {
        if constexpr (std::is_same_v<typename T::event_type, Event>) {
            if (m_state == T::from) return true;
        }
        return canImpl<Event, Rest...>();
    }
    
    StateEnum m_state = Initial;
};

// === Task FSM example ===
enum class TaskState { Todo, InProgress, Review, Done };
struct StartWork {};
struct SubmitReview {};
struct Approve {};
struct Reject {};
struct Reset {};

using TaskFSM = StateMachine<
    TaskState, TaskState::Todo,
    TransitionTable<
        Transition<TaskState, TaskState::Todo,       StartWork,    TaskState::InProgress>,
        Transition<TaskState, TaskState::InProgress,  SubmitReview, TaskState::Review>,
        Transition<TaskState, TaskState::Review,       Approve,      TaskState::Done>,
        Transition<TaskState, TaskState::Review,       Reject,       TaskState::InProgress>,
        Transition<TaskState, TaskState::Done,         Reset,        TaskState::Todo>
    >
>;

void fsmDemo() {
    TaskFSM fsm;
    
    qDebug() << "State: Todo, can start?" << fsm.can<StartWork>();  // true
    
    fsm.process(StartWork{});
    qDebug() << "State: InProgress";
    
    fsm.process(SubmitReview{});
    qDebug() << "State: Review";
    
    fsm.process(Approve{});
    qDebug() << "State: Done";
    
    fsm.process(StartWork{});  // false - invalid transition
    qDebug() << "Still Done (invalid transition ignored)";
}
```

---

## ขั้นตอนที่ 689: Metaclass Simulation with CRTP

```cpp
// === Serializable metaclass via CRTP ===
template<typename Derived>
class Serializable {
public:
    std::string serialize() const {
        std::ostringstream oss;
        oss << "{";
        bool first = true;
        
        derived().forEachField([&](auto name, const auto& value) {
            if (!first) oss << ",";
            first = false;
            oss << "\"" << name << "\":";
            serializeValue(oss, value);
        });
        
        oss << "}";
        return oss.str();
    }
    
    static Derived deserialize(const std::string& json) {
        Derived obj;
        // Simplified: parse and assign via reflection-like access
        return obj;
    }
    
private:
    const Derived& derived() const { return static_cast<const Derived&>(*this); }
    
    template<typename T>
    static void serializeValue(std::ostream& os, const T& v) {
        if constexpr (std::is_arithmetic_v<T>) {
            os << v;
        } else if constexpr (std::is_same_v<T, std::string>) {
            os << "\"" << v << "\"";
        } else if constexpr (std::is_same_v<T, QString>) {
            os << "\"" << v.toStdString() << "\"";
        } else {
            os << "\"?\"";
        }
    }
};

// === Comparable metaclass ===
template<typename Derived>
class Comparable {
public:
    bool operator==(const Derived& other) const {
        return derived().tie() == other.tie();
    }
    bool operator!=(const Derived& other) const { return !(*this == other); }
    bool operator< (const Derived& other) const { return derived().tie() < other.tie(); }
    bool operator<=(const Derived& other) const { return !(other < derived()); }
    bool operator> (const Derived& other) const { return other < derived(); }
    bool operator>=(const Derived& other) const { return !(*this < other); }
    
private:
    const Derived& derived() const { return static_cast<const Derived&>(*this); }
};

// === Concrete class using metaclasses ===
struct Product 
    : Serializable<Product>
    , Comparable<Product>
{
    std::string name;
    double price;
    int stock;
    
    Product(std::string n, double p, int s)
        : name(std::move(n)), price(p), stock(s) {}
    
    auto tie() const { return std::tie(price, name); }
    
    // Reflection-like field enumeration
    template<typename Func>
    void forEachField(Func func) const {
        func("name",  name);
        func("price", price);
        func("stock", stock);
    }
};

void metaclassDemo() {
    Product p1("Widget A", 29.99, 100);
    Product p2("Widget B", 49.99, 50);
    
    qDebug() << QString::fromStdString(p1.serialize());
    // Output: {"name":"Widget A","price":29.99,"stock":100}
    
    qDebug() << "p1 < p2?" << (p1 < p2);  // by price: 29.99 < 49.99 = true
}
```

---

## ขั้นตอนที่ 690-700: Type Erasure

```cpp
// === Type erasure: any callable ===
class Function {
    struct ICallable {
        virtual ~ICallable() = default;
        virtual void call(int) = 0;
        virtual std::unique_ptr<ICallable> clone() const = 0;
    };
    
    template<typename F>
    struct Callable : ICallable {
        F func;
        explicit Callable(F f) : func(std::move(f)) {}
        void call(int x) override { func(x); }
        std::unique_ptr<ICallable> clone() const override {
            return std::make_unique<Callable<F>>(func);
        }
    };
    
public:
    Function() = default;
    
    template<typename F>
    Function(F func)
        : m_callable(std::make_unique<Callable<std::decay_t<F>>>(std::forward<F>(func))) {}
    
    Function(const Function& other)
        : m_callable(other.m_callable ? other.m_callable->clone() : nullptr) {}
    
    Function(Function&&) = default;
    Function& operator=(Function&&) = default;
    
    void operator()(int x) const {
        if (m_callable) m_callable->call(x);
    }
    
    explicit operator bool() const { return m_callable != nullptr; }
    
private:
    std::unique_ptr<ICallable> m_callable;
};

// === Any type container ===
class Any {
    struct IHolder {
        virtual ~IHolder() = default;
        virtual std::unique_ptr<IHolder> clone() const = 0;
        virtual const std::type_info& type() const = 0;
    };
    
    template<typename T>
    struct Holder : IHolder {
        T value;
        explicit Holder(T v) : value(std::move(v)) {}
        
        std::unique_ptr<IHolder> clone() const override {
            return std::make_unique<Holder<T>>(value);
        }
        
        const std::type_info& type() const override { return typeid(T); }
    };
    
public:
    Any() = default;
    
    template<typename T>
    Any(T value)
        : m_holder(std::make_unique<Holder<std::decay_t<T>>>(std::forward<T>(value))) {}
    
    Any(const Any& other)
        : m_holder(other.m_holder ? other.m_holder->clone() : nullptr) {}
    
    Any(Any&&) = default;
    Any& operator=(Any&&) = default;
    
    template<typename T>
    T& get() {
        if (!m_holder || m_holder->type() != typeid(T)) {
            throw std::bad_cast();
        }
        return static_cast<Holder<T>*>(m_holder.get())->value;
    }
    
    template<typename T>
    bool holds() const {
        return m_holder && m_holder->type() == typeid(T);
    }
    
    bool empty() const { return !m_holder; }
    
private:
    std::unique_ptr<IHolder> m_holder;
};

// === Variant visitor pattern ===
template<typename... Visitors>
struct Overloaded : Visitors... {
    using Visitors::operator()...;
};

template<typename... Visitors>
Overloaded(Visitors...) -> Overloaded<Visitors...>;

void typeErasureDemo() {
    // Type-erased callable
    Function fn = [](int x) { qDebug() << "Value:" << x; };
    fn(42);
    
    // Any type
    Any a = 42;
    qDebug() << a.holds<int>();     // true
    qDebug() << a.get<int>();       // 42
    
    a = std::string("hello");
    qDebug() << a.holds<std::string>(); // true
    
    // std::variant with Overloaded visitor
    using Shape = std::variant<int, double, QString>;
    
    Shape s = QString("hello");
    
    auto desc = std::visit(Overloaded{
        [](int v)           { return QString("int: %1").arg(v); },
        [](double v)        { return QString("double: %1").arg(v); },
        [](const QString& v){ return "string: " + v; }
    }, s);
    
    qDebug() << desc;  // "string: hello"
}
```

---

## สรุป Part 048

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Compile-time string processing ด้วย FixedString + consteval hash
2. ✅ Type lists + variadic template metaprogramming
3. ✅ Compile-time State Machine ด้วย template transitions
4. ✅ Metaclass simulation ด้วย CRTP (Serializable + Comparable)
5. ✅ Type Erasure pattern (Function, Any, Overloaded visitor)

---

⬅️ [Part 047](part047.md) | ➡️ [Part 049: World-Class Level — High Performance Qt](part049.md)
