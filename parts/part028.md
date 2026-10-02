# Part 028: Advanced STL Algorithms & Ranges

## ขั้นตอนที่ 386-400

---

## ขั้นตอนที่ 386: STL Algorithms Overview

```cpp
#include <algorithm>
#include <numeric>
#include <functional>
#include <iterator>
#include <ranges>
#include <vector>
#include <list>
#include <string>

// === Non-modifying algorithms ===
void demoNonModifying() {
    std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};
    
    // find
    auto it = std::find(v.begin(), v.end(), 5);
    std::cout << "First 5 at index: " << std::distance(v.begin(), it) << '\n'; // 4
    
    // find_if
    auto even = std::find_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });
    std::cout << "First even: " << *even << '\n'; // 4
    
    // count / count_if
    int fives = std::count(v.begin(), v.end(), 5);
    int evens = std::count_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });
    std::cout << "Fives: " << fives << ", Evens: " << evens << '\n'; // 2, 3
    
    // all_of, any_of, none_of
    bool allPos = std::all_of(v.begin(), v.end(), [](int x) { return x > 0; });
    bool anyGt8 = std::any_of(v.begin(), v.end(), [](int x) { return x > 8; });
    bool noneNeg = std::none_of(v.begin(), v.end(), [](int x) { return x < 0; });
    std::cout << "All pos: " << allPos << ", Any>8: " << anyGt8 << ", None neg: " << noneNeg << '\n';
    
    // min/max element
    auto [minIt, maxIt] = std::minmax_element(v.begin(), v.end());
    std::cout << "Min: " << *minIt << ", Max: " << *maxIt << '\n'; // 1, 9
    
    // accumulate / reduce
    int sum = std::accumulate(v.begin(), v.end(), 0);
    int product = std::accumulate(v.begin(), v.end(), 1, std::multiplies<int>());
    std::cout << "Sum: " << sum << ", Product: " << product << '\n';
    
    // inner_product
    std::vector<int> weights = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    int dotProduct = std::inner_product(v.begin(), v.end(), weights.begin(), 0);
    std::cout << "Dot product: " << dotProduct << '\n';
}

// === Modifying algorithms ===
void demoModifying() {
    std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};
    
    // sort
    std::sort(v.begin(), v.end());
    // {1, 1, 2, 3, 3, 4, 5, 5, 6, 9}
    
    // unique (หลัง sort)
    auto last = std::unique(v.begin(), v.end());
    v.erase(last, v.end());
    // {1, 2, 3, 4, 5, 6, 9}
    
    // binary_search / lower_bound / upper_bound
    bool found = std::binary_search(v.begin(), v.end(), 5);
    auto lo = std::lower_bound(v.begin(), v.end(), 4);
    auto hi = std::upper_bound(v.begin(), v.end(), 5);
    std::cout << "Range [4,5]: " << std::distance(lo, hi) << " elements\n";
    
    // reverse
    std::vector<int> r = {1, 2, 3, 4, 5};
    std::reverse(r.begin(), r.end()); // {5, 4, 3, 2, 1}
    
    // rotate
    std::rotate(r.begin(), r.begin() + 2, r.end()); // {3, 2, 1, 5, 4}
    
    // partition
    std::vector<int> p = {1, 2, 3, 4, 5, 6, 7, 8};
    auto mid = std::partition(p.begin(), p.end(), [](int x) { return x % 2 == 0; });
    // evens before mid, odds after
    
    // transform
    std::vector<int> src = {1, 2, 3, 4, 5};
    std::vector<int> dst(src.size());
    std::transform(src.begin(), src.end(), dst.begin(), [](int x) { return x * x; });
    // {1, 4, 9, 16, 25}
    
    // transform with two ranges
    std::vector<int> a = {1, 2, 3}, b = {4, 5, 6};
    std::vector<int> c(3);
    std::transform(a.begin(), a.end(), b.begin(), c.begin(), std::plus<int>());
    // {5, 7, 9}
    
    // copy_if
    std::vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8};
    std::vector<int> evens;
    std::copy_if(nums.begin(), nums.end(), std::back_inserter(evens),
                  [](int x) { return x % 2 == 0; });
    // evens = {2, 4, 6, 8}
    
    // fill, generate
    std::vector<int> filled(5);
    std::fill(filled.begin(), filled.end(), 42); // {42, 42, 42, 42, 42}
    
    int counter = 0;
    std::generate(filled.begin(), filled.end(), [&counter]() { return counter++ * counter; });
    // {0, 1, 4, 9, 16}
    
    // remove_if (Erase-Remove idiom)
    std::vector<int> toClean = {1, 2, 3, 4, 5, 6};
    toClean.erase(
        std::remove_if(toClean.begin(), toClean.end(), [](int x) { return x % 2 == 0; }),
        toClean.end()
    );
    // {1, 3, 5}
}
```

---

## ขั้นตอนที่ 387: Numeric Algorithms

```cpp
#include <numeric>

void demoNumeric() {
    std::vector<int> v = {1, 2, 3, 4, 5};
    
    // iota: fill with increasing values
    std::iota(v.begin(), v.end(), 10); // {10, 11, 12, 13, 14}
    
    // partial_sum
    std::vector<int> src = {1, 2, 3, 4, 5};
    std::vector<int> cumulative(5);
    std::partial_sum(src.begin(), src.end(), cumulative.begin());
    // {1, 3, 6, 10, 15}
    
    // adjacent_difference
    std::vector<int> diffs(5);
    std::adjacent_difference(src.begin(), src.end(), diffs.begin());
    // {1, 1, 1, 1, 1} (differences between adjacent elements)
    
    // reduce (parallel-friendly, C++17)
    int total = std::reduce(src.begin(), src.end()); // 15
    int product = std::reduce(src.begin(), src.end(), 1, std::multiplies<int>()); // 120
    
    // transform_reduce
    std::vector<double> data = {1.0, 2.0, 3.0, 4.0, 5.0};
    double sumSquares = std::transform_reduce(
        data.begin(), data.end(),
        0.0,
        std::plus<double>(),
        [](double x) { return x * x; }
    ); // 1 + 4 + 9 + 16 + 25 = 55
    
    // exclusive_scan / inclusive_scan (C++17)
    std::vector<int> excl(5), incl(5);
    std::exclusive_scan(src.begin(), src.end(), excl.begin(), 0);
    // {0, 1, 3, 6, 10}
    std::inclusive_scan(src.begin(), src.end(), incl.begin());
    // {1, 3, 6, 10, 15}
    
    std::cout << "Sum: " << total << ", Product: " << product << '\n';
    std::cout << "Sum of squares: " << sumSquares << '\n';
}
```

---

## ขั้นตอนที่ 388: C++20 Ranges

```cpp
#include <ranges>
#include <string>
#include <vector>

void demoRanges() {
    namespace rv = std::ranges::views;
    namespace r = std::ranges;
    
    std::vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // filter + transform (lazy)
    auto result = nums 
        | rv::filter([](int x) { return x % 2 == 0; })  // even numbers
        | rv::transform([](int x) { return x * x; });     // square them
    
    for (int x : result) std::cout << x << ' '; // 4 16 36 64 100
    std::cout << '\n';
    
    // take / drop
    auto first5 = nums | rv::take(5);
    auto after5 = nums | rv::drop(5);
    
    // reverse
    auto rev = nums | rv::reverse;
    for (int x : rev | rv::take(3)) std::cout << x << ' '; // 10 9 8
    std::cout << '\n';
    
    // iota view
    auto squares = rv::iota(1, 11) 
        | rv::transform([](int x) { return x * x; });
    
    // zip (ใน C++23, แต่ใน C++20 ใช้ views::zip ไม่ได้ตรงๆ)
    
    // chunk / stride (C++23)
    // auto chunks = nums | rv::chunk(3);
    
    // join_with (flatten nested)
    std::vector<std::vector<int>> nested = {{1,2,3}, {4,5}, {6,7,8,9}};
    auto flat = nested | rv::join;
    for (int x : flat) std::cout << x << ' '; // 1 2 3 4 5 6 7 8 9
    std::cout << '\n';
    
    // keys / values (for maps)
    std::map<std::string, int> scores = {{"Alice", 90}, {"Bob", 85}, {"Carol", 92}};
    for (auto& k : scores | rv::keys)   std::cout << k << ' ';  // Alice Bob Carol
    for (auto  v : scores | rv::values) std::cout << v << ' ';  // 90 85 92
    
    // Sorting ranges
    std::vector<int> toSort = {5, 3, 1, 4, 2};
    r::sort(toSort);
    r::sort(toSort, std::greater<int>{}); // descending
    
    // Finding
    auto it = r::find(nums, 7);
    auto pos = r::find_if(nums, [](int x) { return x > 8; });
    
    // Counting
    int count = r::count_if(nums, [](int x) { return x % 3 == 0; });
    
    // Check
    bool allPos = r::all_of(nums, [](int x) { return x > 0; });
    bool anyGt9 = r::any_of(nums, [](int x) { return x > 9; });
    
    std::cout << "Count div3: " << count << '\n';
    std::cout << "All positive: " << allPos << '\n';
    std::cout << "Any > 9: " << anyGt9 << '\n';
}

// Custom range view
template<typename R>
auto everyNth(R&& range, int n) {
    namespace rv = std::ranges::views;
    
    int i = 0;
    return std::forward<R>(range) | rv::filter([n, i](const auto&) mutable {
        return i++ % n == 0;
    });
}
```

---

## ขั้นตอนที่ 389: Custom Iterator

```cpp
// === Custom Iterator ===
class IntRange {
public:
    struct Iterator {
        using iterator_category = std::forward_iterator_tag;
        using value_type = int;
        using difference_type = std::ptrdiff_t;
        using pointer = const int*;
        using reference = const int&;
        
        int current;
        int step;
        
        Iterator(int val, int step = 1) : current(val), step(step) {}
        
        int operator*() const { return current; }
        
        Iterator& operator++() { current += step; return *this; }
        Iterator operator++(int) { Iterator tmp = *this; ++*this; return tmp; }
        
        bool operator==(const Iterator& o) const { return current == o.current; }
        bool operator!=(const Iterator& o) const { return current != o.current; }
    };
    
    IntRange(int start, int end, int step = 1)
        : m_start(start), m_end(end), m_step(step) {}
    
    Iterator begin() const { return {m_start, m_step}; }
    Iterator end() const {
        // Round up to nearest step multiple
        int steps = (m_end - m_start + m_step - 1) / m_step;
        return {m_start + steps * m_step, m_step};
    }
    
private:
    int m_start, m_end, m_step;
};

// Usage
void demoCustomIterator() {
    // Range-based for loop
    for (int i : IntRange(0, 10)) std::cout << i << ' '; // 0 1 2 3 4 5 6 7 8 9
    std::cout << '\n';
    
    for (int i : IntRange(0, 20, 3)) std::cout << i << ' '; // 0 3 6 9 12 15 18
    std::cout << '\n';
    
    // Works with STL algorithms
    IntRange r(1, 11);
    int sum = std::accumulate(r.begin(), r.end(), 0); // 55
    std::cout << "Sum: " << sum << '\n';
    
    // Works with ranges
    auto evens = IntRange(0, 20, 2);
    int total = std::ranges::fold_left(evens, 0, std::plus<int>{}); // 90
    std::cout << "Total: " << total << '\n';
}
```

---

## ขั้นตอนที่ 390-400: Practical Pipeline with Ranges

```cpp
#include <ranges>
#include <vector>
#include <string>
#include <sstream>

struct Employee {
    std::string name;
    std::string dept;
    double salary;
    int years;
    bool active;
};

// Functional pipeline using ranges
class EmployeeAnalytics {
public:
    std::vector<Employee> employees;
    
    // Get active employees
    auto getActive() const {
        namespace rv = std::ranges::views;
        return employees | rv::filter([](const Employee& e) { return e.active; });
    }
    
    // Average salary by department
    double avgSalary(const std::string& dept) const {
        namespace rv = std::ranges::views;
        namespace r = std::ranges;
        
        auto deptEmployees = employees 
            | rv::filter([&dept](const Employee& e) { 
                return e.dept == dept && e.active; 
              });
        
        double total = 0;
        int count = 0;
        for (const auto& e : deptEmployees) {
            total += e.salary;
            count++;
        }
        return count > 0 ? total / count : 0;
    }
    
    // Top N earners
    std::vector<Employee> topEarners(int n) const {
        std::vector<Employee> sorted = employees;
        std::ranges::sort(sorted, [](const Employee& a, const Employee& b) {
            return a.salary > b.salary;
        });
        
        int take = std::min(n, static_cast<int>(sorted.size()));
        return {sorted.begin(), sorted.begin() + take};
    }
    
    // Employees eligible for bonus (>= 5 years, active)
    std::vector<std::string> bonusEligible() const {
        namespace rv = std::ranges::views;
        
        auto names = employees
            | rv::filter([](const Employee& e) { return e.active && e.years >= 5; })
            | rv::transform([](const Employee& e) { return e.name; });
        
        return {names.begin(), names.end()};
    }
    
    // Department stats
    std::map<std::string, int> headcountByDept() const {
        std::map<std::string, int> result;
        for (const auto& e : employees) {
            if (e.active) result[e.dept]++;
        }
        return result;
    }
    
    // Total payroll
    double totalPayroll() const {
        namespace rv = std::ranges::views;
        
        auto activeSalaries = employees
            | rv::filter([](const Employee& e) { return e.active; })
            | rv::transform([](const Employee& e) { return e.salary; });
        
        return std::ranges::fold_left(activeSalaries, 0.0, std::plus<double>{});
    }
    
    // Print report
    void printReport() const {
        auto active = getActive();
        
        std::cout << "=== Employee Report ===\n";
        std::cout << "Active employees:\n";
        for (const auto& e : active) {
            std::cout << "  " << e.name << " | " << e.dept 
                      << " | $" << e.salary << " | " << e.years << " years\n";
        }
        
        auto bonus = bonusEligible();
        std::cout << "\nBonus eligible: ";
        for (const auto& name : bonus) std::cout << name << ", ";
        std::cout << '\n';
        
        std::cout << "\nTop 3 earners:\n";
        for (const auto& e : topEarners(3)) {
            std::cout << "  " << e.name << ": $" << e.salary << '\n';
        }
        
        std::cout << "\nPayroll: $" << totalPayroll() << '\n';
    }
};

int main() {
    EmployeeAnalytics analytics;
    analytics.employees = {
        {"Alice",   "Engineering", 95000, 8,  true},
        {"Bob",     "Marketing",   72000, 3,  true},
        {"Carol",   "Engineering", 88000, 6,  true},
        {"David",   "HR",          65000, 2,  true},
        {"Eve",     "Engineering", 105000, 12, true},
        {"Frank",   "Marketing",   68000, 7,  false},
        {"Grace",   "HR",          70000, 5,  true},
        {"Henry",   "Engineering", 91000, 9,  true}
    };
    
    analytics.printReport();
    
    std::cout << "\nAvg Engineering salary: $" 
              << analytics.avgSalary("Engineering") << '\n';
    
    auto headcount = analytics.headcountByDept();
    std::cout << "\nHeadcount by dept:\n";
    for (const auto& [dept, count] : headcount) {
        std::cout << "  " << dept << ": " << count << '\n';
    }
    
    return 0;
}
```

---

## สรุป Part 028

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Non-modifying algorithms (find, count, all_of, etc.)
2. ✅ Modifying algorithms (sort, transform, partition, etc.)
3. ✅ Numeric algorithms (accumulate, reduce, partial_sum)
4. ✅ C++20 Ranges และ views pipeline
5. ✅ Custom Iterator
6. ✅ Real-world pipeline สำหรับ data analysis

---

⬅️ [Part 027](part027.md) | ➡️ [Part 029: Advanced Error Handling](part029.md)
