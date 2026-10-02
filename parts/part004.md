# Part 004: ฟังก์ชัน (Functions)

## ขั้นตอนที่ 31-40

---

## ขั้นตอนที่ 31: พื้นฐานฟังก์ชัน

ฟังก์ชันคือกลุ่มของคำสั่งที่รวมกันเป็นหน่วยหนึ่ง สามารถเรียกใช้ซ้ำได้และช่วยทำให้โค้ดอ่านง่ายขึ้น

```cpp
#include <iostream>
#include <string>
using namespace std;

// === การประกาศฟังก์ชัน (Function Declaration / Prototype) ===
void printGreeting();                    // ไม่รับ ไม่คืน
int addNumbers(int a, int b);           // รับ int 2 ตัว คืน int
double calculateArea(double r);          // รับ double คืน double
string getFullName(string first, string last);  // รับ string คืน string

int main() {
    // === เรียกใช้ฟังก์ชัน ===
    
    printGreeting();
    
    int sum = addNumbers(10, 20);
    cout << "ผลรวม: " << sum << endl;
    
    double area = calculateArea(5.0);
    cout << "พื้นที่: " << area << endl;
    
    string fullName = getFullName("สมชาย", "ใจดี");
    cout << "ชื่อเต็ม: " << fullName << endl;
    
    return 0;
}

// === นิยามฟังก์ชัน (Function Definition) ===

void printGreeting() {
    cout << "============================" << endl;
    cout << "  ยินดีต้อนรับสู่ C++ 🎉   " << endl;
    cout << "============================" << endl;
}

int addNumbers(int a, int b) {
    return a + b;  // คืนค่าผลรวม
}

double calculateArea(double r) {
    const double PI = 3.14159265358979;
    return PI * r * r;  // คืนค่าพื้นที่
}

string getFullName(string first, string last) {
    return first + " " + last;  // คืนค่าชื่อเต็ม
}
```

### โครงสร้างฟังก์ชัน

```
return_type function_name(parameter_list) {
    // function body
    return value;  // ถ้า return_type ไม่ใช่ void
}
```

**ประเภทฟังก์ชันตาม Parameter และ Return:**

```
1. void function()           - ไม่รับ ไม่คืน
2. void function(params)     - รับแต่ไม่คืน
3. type function()           - ไม่รับแต่คืน
4. type function(params)     - รับและคืน
```

---

## ขั้นตอนที่ 32: การส่งผ่านค่า (Pass by Value vs Reference)

```cpp
#include <iostream>
using namespace std;

// === Pass by Value ===
// Function รับสำเนาของค่า การเปลี่ยนแปลงไม่กระทบต้นฉบับ
void doubleValue(int x) {
    x = x * 2;  // เปลี่ยนแค่สำเนา
    cout << "ใน function: x = " << x << endl;
}

// === Pass by Reference ===
// Function รับ Reference ของตัวแปรจริง การเปลี่ยนแปลงกระทบต้นฉบับ
void doubleValueRef(int& x) {
    x = x * 2;  // เปลี่ยนตัวแปรจริง
    cout << "ใน function: x = " << x << endl;
}

// === Pass by Const Reference ===
// ส่งผ่าน Reference แต่ป้องกันการแก้ไข (efficient สำหรับ object ใหญ่)
void printInfo(const string& name, const int& age) {
    // name = "แก้ไข";  // Error! const ไม่สามารถแก้ไขได้
    cout << "ชื่อ: " << name << ", อายุ: " << age << endl;
}

// === Pass by Pointer ===
void doubleValuePtr(int* x) {
    *x = *x * 2;  // Dereference pointer เพื่อเปลี่ยนค่า
}

// === Swap Functions ===
void swapByValue(int a, int b) {
    int temp = a;
    a = b;
    b = temp;
    // ไม่มีผลภายนอก!
}

void swapByRef(int& a, int& b) {
    int temp = a;
    a = b;
    b = temp;
    // สลับค่าจริง!
}

int main() {
    // === Pass by Value ===
    cout << "=== Pass by Value ===" << endl;
    int num = 10;
    cout << "ก่อน: num = " << num << endl;
    doubleValue(num);
    cout << "หลัง: num = " << num << endl;  // ยังเป็น 10!
    
    // === Pass by Reference ===
    cout << "\n=== Pass by Reference ===" << endl;
    num = 10;
    cout << "ก่อน: num = " << num << endl;
    doubleValueRef(num);
    cout << "หลัง: num = " << num << endl;  // เป็น 20!
    
    // === Pass by Const Reference ===
    cout << "\n=== Pass by Const Reference ===" << endl;
    string name = "สมหญิง";
    int age = 25;
    printInfo(name, age);
    
    // === Pass by Pointer ===
    cout << "\n=== Pass by Pointer ===" << endl;
    num = 15;
    cout << "ก่อน: num = " << num << endl;
    doubleValuePtr(&num);  // ส่ง address
    cout << "หลัง: num = " << num << endl;  // เป็น 30!
    
    // === Swap ===
    cout << "\n=== Swap Functions ===" << endl;
    int x = 5, y = 10;
    cout << "ก่อน: x=" << x << ", y=" << y << endl;
    swapByValue(x, y);
    cout << "หลัง swapByValue: x=" << x << ", y=" << y << endl;  // ไม่เปลี่ยน
    
    swapByRef(x, y);
    cout << "หลัง swapByRef: x=" << x << ", y=" << y << endl;  // สลับแล้ว
    
    // === Multiple Return Values ด้วย Reference ===
    cout << "\n=== Multiple Return Values ===" << endl;
    
    auto divmod = [](int a, int b, int& quotient, int& remainder) {
        quotient = a / b;
        remainder = a % b;
    };
    
    int quot, rem;
    divmod(17, 5, quot, rem);
    cout << "17 / 5 = " << quot << " เหลือ " << rem << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 33: ค่าเริ่มต้น (Default Parameters)

```cpp
#include <iostream>
#include <string>
#include <cmath>
using namespace std;

// === Default Parameters ===
// ต้องกำหนดจากขวาไปซ้าย!
void printBox(int width = 20, int height = 5, char fillChar = '*') {
    for (int i = 0; i < height; i++) {
        if (i == 0 || i == height - 1) {
            // แถวบนสุดและล่างสุด
            for (int j = 0; j < width; j++) cout << fillChar;
        } else {
            // แถวกลาง
            cout << fillChar;
            for (int j = 0; j < width - 2; j++) cout << " ";
            cout << fillChar;
        }
        cout << endl;
    }
}

// ฟังก์ชันคำนวณพื้นที่ รูปร่างต่างๆ
double calculateAreaShape(double value1, double value2 = -1, string shape = "rectangle") {
    const double PI = 3.14159265358979;
    
    if (shape == "rectangle") {
        return value1 * value2;
    } else if (shape == "circle") {
        return PI * value1 * value1;
    } else if (shape == "triangle") {
        return 0.5 * value1 * value2;  // base * height / 2
    } else {
        return -1;  // Unknown shape
    }
}

// ฟังก์ชันแสดงข้อมูลนักเรียน
void showStudentInfo(string name, int grade = 10, string school = "โรงเรียนตัวอย่าง") {
    cout << "ชื่อ: " << name << endl;
    cout << "ชั้น: ม." << grade << endl;
    cout << "โรงเรียน: " << school << endl;
}

int main() {
    // === เรียกฟังก์ชันที่มี Default Parameters ===
    
    cout << "=== printBox กับ Default Parameters ===" << endl;
    
    cout << "\nใช้ค่า Default ทั้งหมด:" << endl;
    printBox();  // width=20, height=5, char='*'
    
    cout << "\nกำหนดแค่ width=30:" << endl;
    printBox(30);  // width=30, height=5, char='*'
    
    cout << "\nกำหนด width=15, height=3:" << endl;
    printBox(15, 3);  // width=15, height=3, char='*'
    
    cout << "\nกำหนดทุกค่า:" << endl;
    printBox(25, 4, '#');  // width=25, height=4, char='#'
    
    // === Shape Area ===
    cout << "\n=== คำนวณพื้นที่ ===" << endl;
    
    cout << "พื้นที่สี่เหลี่ยม 5x3 = " 
         << calculateAreaShape(5, 3) << endl;
    
    cout << "พื้นที่วงกลม r=7 = " 
         << calculateAreaShape(7, -1, "circle") << endl;
    
    cout << "พื้นที่สามเหลี่ยม ฐาน=6 สูง=4 = " 
         << calculateAreaShape(6, 4, "triangle") << endl;
    
    // === Student Info ===
    cout << "\n=== ข้อมูลนักเรียน ===" << endl;
    
    showStudentInfo("สมชาย");  // ใช้ Default grade และ school
    cout << "---" << endl;
    showStudentInfo("สมหญิง", 12);  // กำหนด grade
    cout << "---" << endl;
    showStudentInfo("สมศรี", 11, "โรงเรียนมัธยมปลาย");  // กำหนดทั้งหมด
    
    return 0;
}
```

---

## ขั้นตอนที่ 34: Function Overloading

```cpp
#include <iostream>
#include <string>
#include <cmath>
using namespace std;

// === Function Overloading ===
// ฟังก์ชันชื่อเดียวกัน แต่ Parameter ต่างกัน

// area() แบบวงกลม
double area(double radius) {
    const double PI = 3.14159265358979;
    return PI * radius * radius;
}

// area() แบบสี่เหลี่ยมผืนผ้า
double area(double length, double width) {
    return length * width;
}

// area() แบบสามเหลี่ยม
double area(double base, double height, bool isTriangle) {
    if (isTriangle) return 0.5 * base * height;
    return base * height;  // สี่เหลี่ยม
}

// area() แบบ int
int area(int side) {
    return side * side;  // สี่เหลี่ยมจัตุรัส
}

// === print() Overloading ===
void print(int value) {
    cout << "int: " << value << endl;
}

void print(double value) {
    cout << "double: " << value << endl;
}

void print(string value) {
    cout << "string: " << value << endl;
}

void print(bool value) {
    cout << "bool: " << (value ? "true" : "false") << endl;
}

void print(int value, string label) {
    cout << label << ": " << value << endl;
}

// === max() Overloading ===
int maxVal(int a, int b) {
    return (a > b) ? a : b;
}

double maxVal(double a, double b) {
    return (a > b) ? a : b;
}

int maxVal(int a, int b, int c) {
    return maxVal(maxVal(a, b), c);
}

double maxVal(double a, double b, double c) {
    return maxVal(maxVal(a, b), c);
}

int main() {
    // === area() ===
    cout << "=== area() Overloading ===" << endl;
    
    cout << "วงกลม r=5: " << area(5.0) << endl;
    cout << "สี่เหลี่ยม 4x6: " << area(4.0, 6.0) << endl;
    cout << "สามเหลี่ยม ฐาน=3 สูง=4: " << area(3.0, 4.0, true) << endl;
    cout << "สี่เหลี่ยมจัตุรัส ด้าน=7: " << area(7) << endl;
    
    // === print() ===
    cout << "\n=== print() Overloading ===" << endl;
    
    print(42);
    print(3.14);
    print(string("Hello C++"));
    print(true);
    print(100, "คะแนน");
    
    // === maxVal() ===
    cout << "\n=== maxVal() Overloading ===" << endl;
    
    cout << "max(3, 7) = " << maxVal(3, 7) << endl;
    cout << "max(3.5, 2.8) = " << maxVal(3.5, 2.8) << endl;
    cout << "max(5, 2, 9) = " << maxVal(5, 2, 9) << endl;
    cout << "max(1.1, 5.5, 3.3) = " << maxVal(1.1, 5.5, 3.3) << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 35: ฟังก์ชัน Recursive

```cpp
#include <iostream>
#include <string>
using namespace std;

// === Factorial ===
long long factorial(int n) {
    if (n <= 1) return 1;  // Base case
    return n * factorial(n - 1);  // Recursive case
}

// === Fibonacci ===
long long fibonacci(int n) {
    if (n <= 0) return 0;  // Base case 1
    if (n == 1) return 1;  // Base case 2
    return fibonacci(n - 1) + fibonacci(n - 2);  // Recursive case
}

// === หาผลรวม 1 ถึง n ===
int sumToN(int n) {
    if (n <= 0) return 0;  // Base case
    return n + sumToN(n - 1);  // Recursive case
}

// === Binary Search Recursive ===
int binarySearch(int arr[], int left, int right, int target) {
    if (left > right) return -1;  // ไม่พบ
    
    int mid = left + (right - left) / 2;
    
    if (arr[mid] == target) return mid;  // พบ
    if (arr[mid] < target) return binarySearch(arr, mid + 1, right, target);
    return binarySearch(arr, left, mid - 1, target);
}

// === Power Function Recursive ===
double power(double base, int exp) {
    if (exp == 0) return 1;           // Base case
    if (exp < 0) return 1.0 / power(base, -exp);  // Negative exponent
    if (exp % 2 == 0) {
        double half = power(base, exp / 2);
        return half * half;  // Efficient: O(log n)
    }
    return base * power(base, exp - 1);
}

// === Hanoi Tower ===
void hanoi(int n, char from, char to, char aux) {
    if (n == 1) {
        cout << "ย้ายจาน " << n << " จาก " << from << " ไป " << to << endl;
        return;
    }
    hanoi(n - 1, from, aux, to);
    cout << "ย้ายจาน " << n << " จาก " << from << " ไป " << to << endl;
    hanoi(n - 1, aux, to, from);
}

// === Reverse String Recursive ===
string reverseString(string s) {
    if (s.length() <= 1) return s;
    return reverseString(s.substr(1)) + s[0];
}

// === GCD Recursive (Euclidean Algorithm) ===
int gcd(int a, int b) {
    if (b == 0) return a;
    return gcd(b, a % b);
}

// === Count Digits Recursive ===
int countDigits(int n) {
    if (n == 0) return 0;
    return 1 + countDigits(n / 10);
}

// === Flatten Nested ===
void flatten(int depth) {
    if (depth <= 0) {
        cout << "ถึงจุดสิ้นสุด" << endl;
        return;
    }
    cout << string(4 * (5 - depth), ' ') << "ระดับ " << depth << endl;
    flatten(depth - 1);
    cout << string(4 * (5 - depth), ' ') << "กลับสู่ระดับ " << depth << endl;
}

int main() {
    // === Factorial ===
    cout << "=== Factorial ===" << endl;
    for (int i = 0; i <= 10; i++) {
        cout << i << "! = " << factorial(i) << endl;
    }
    
    // === Fibonacci ===
    cout << "\n=== Fibonacci ===" << endl;
    for (int i = 0; i < 15; i++) {
        cout << fibonacci(i) << " ";
    }
    cout << endl;
    
    // === Sum ===
    cout << "\n=== ผลรวม 1-10 = " << sumToN(10) << endl;
    
    // === Binary Search ===
    cout << "\n=== Binary Search ===" << endl;
    int arr[] = {2, 5, 8, 12, 16, 23, 38, 56, 72, 91};
    int n = sizeof(arr) / sizeof(arr[0]);
    
    int target = 23;
    int idx = binarySearch(arr, 0, n - 1, target);
    
    if (idx != -1) {
        cout << "พบ " << target << " ที่ตำแหน่ง " << idx << endl;
    } else {
        cout << "ไม่พบ " << target << endl;
    }
    
    // === Power ===
    cout << "\n=== Power ===" << endl;
    cout << "2^10 = " << power(2, 10) << endl;
    cout << "3^5 = " << power(3, 5) << endl;
    cout << "2^-3 = " << power(2, -3) << endl;
    
    // === Hanoi ===
    cout << "\n=== Hanoi Tower (3 จาน) ===" << endl;
    hanoi(3, 'A', 'C', 'B');
    
    // === Reverse String ===
    cout << "\n=== Reverse String ===" << endl;
    cout << reverseString("Hello World") << endl;
    cout << reverseString("สวัสดี") << endl;
    
    // === GCD ===
    cout << "\n=== GCD ===" << endl;
    cout << "gcd(48, 36) = " << gcd(48, 36) << endl;
    cout << "gcd(100, 75) = " << gcd(100, 75) << endl;
    
    // === Count Digits ===
    cout << "\n=== Count Digits ===" << endl;
    cout << "จำนวนหลักของ 12345 = " << countDigits(12345) << endl;
    
    // === Call Stack Visualization ===
    cout << "\n=== Call Stack ===" << endl;
    flatten(5);
    
    return 0;
}
```

---

## ขั้นตอนที่ 36: Inline Functions และ Lambda Functions

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <algorithm>
using namespace std;

// === Inline Functions ===
// Compiler แทนที่ function call ด้วย code ของ function โดยตรง (เร็วขึ้น)
inline int square(int x) {
    return x * x;
}

inline double circleArea(double r) {
    return 3.14159 * r * r;
}

inline bool isEven(int n) {
    return n % 2 == 0;
}

int main() {
    // === Inline Functions ===
    cout << "=== Inline Functions ===" << endl;
    cout << "square(5) = " << square(5) << endl;
    cout << "circleArea(3) = " << circleArea(3) << endl;
    cout << "isEven(4) = " << boolalpha << isEven(4) << endl;
    
    // === Lambda Functions (C++11) ===
    cout << "\n=== Lambda Functions ===" << endl;
    
    // Lambda พื้นฐาน: [capture](params) -> return_type { body }
    
    // Lambda ไม่มี Parameter
    auto greet = []() {
        cout << "สวัสดี Lambda!" << endl;
    };
    greet();
    
    // Lambda มี Parameters
    auto add = [](int a, int b) -> int {
        return a + b;
    };
    cout << "add(3, 4) = " << add(3, 4) << endl;
    
    // Lambda กับ auto return type (C++14)
    auto multiply = [](auto a, auto b) {
        return a * b;
    };
    cout << "multiply(3, 4) = " << multiply(3, 4) << endl;
    cout << "multiply(2.5, 3.0) = " << multiply(2.5, 3.0) << endl;
    
    // === Lambda Captures ===
    cout << "\n=== Lambda Captures ===" << endl;
    
    int x = 10, y = 20;
    
    // Capture by value [=]
    auto captureByValue = [=]() {
        cout << "By value: x=" << x << ", y=" << y << endl;
        // x = 100;  // Error! value captures are const by default
    };
    captureByValue();
    
    // Capture by reference [&]
    auto captureByRef = [&]() {
        x += 5;  // แก้ไขได้!
        cout << "By ref: x=" << x << ", y=" << y << endl;
    };
    captureByRef();
    cout << "x หลัง capture by ref: " << x << endl;  // 15
    
    // Capture เฉพาะตัวแปร
    int count = 0;
    auto increment = [&count]() {
        count++;
    };
    increment();
    increment();
    increment();
    cout << "count = " << count << endl;  // 3
    
    // Capture by value กับ mutable
    int base = 100;
    auto mutableLambda = [base]() mutable {
        base += 10;  // แก้ได้แต่ไม่กระทบตัวแปรต้นฉบับ
        return base;
    };
    cout << "mutableLambda() = " << mutableLambda() << endl;  // 110
    cout << "base = " << base << endl;  // ยังเป็น 100
    
    // === Lambda กับ STL Algorithms ===
    cout << "\n=== Lambda กับ STL ===" << endl;
    
    vector<int> nums = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    
    // sort กับ lambda
    sort(nums.begin(), nums.end(), [](int a, int b) {
        return a < b;  // ascending
    });
    
    cout << "Sorted: ";
    for (int n : nums) cout << n << " ";
    cout << endl;
    
    // sort descending
    sort(nums.begin(), nums.end(), [](int a, int b) {
        return a > b;
    });
    
    cout << "Sorted desc: ";
    for (int n : nums) cout << n << " ";
    cout << endl;
    
    // find_if
    auto it = find_if(nums.begin(), nums.end(), [](int n) {
        return n % 2 == 0;  // หาเลขคู่
    });
    
    if (it != nums.end()) {
        cout << "เลขคู่แรก: " << *it << endl;
    }
    
    // count_if
    int evenCount = count_if(nums.begin(), nums.end(), [](int n) {
        return n % 2 == 0;
    });
    cout << "จำนวนเลขคู่: " << evenCount << endl;
    
    // for_each
    cout << "Squared: ";
    for_each(nums.begin(), nums.end(), [](int n) {
        cout << n * n << " ";
    });
    cout << endl;
    
    // transform
    vector<int> doubled(nums.size());
    transform(nums.begin(), nums.end(), doubled.begin(), [](int n) {
        return n * 2;
    });
    cout << "Doubled: ";
    for (int n : doubled) cout << n << " ";
    cout << endl;
    
    // === std::function ===
    cout << "\n=== std::function ===" << endl;
    
    // เก็บ lambda ใน function variable
    function<int(int, int)> operation;
    
    char op;
    cout << "เลือกตัวดำเนินการ (+, -, *, /): ";
    cin >> op;
    
    switch (op) {
        case '+': operation = [](int a, int b) { return a + b; }; break;
        case '-': operation = [](int a, int b) { return a - b; }; break;
        case '*': operation = [](int a, int b) { return a * b; }; break;
        case '/': operation = [](int a, int b) { return b != 0 ? a / b : 0; }; break;
        default:  operation = [](int a, int b) { return 0; };
    }
    
    cout << "10 " << op << " 3 = " << operation(10, 3) << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 37: ฟังก์ชัน Template

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
using namespace std;

// === Function Templates ===
// เขียนครั้งเดียว ใช้ได้กับหลาย type

template <typename T>
T getMax(T a, T b) {
    return (a > b) ? a : b;
}

template <typename T>
T getMin(T a, T b) {
    return (a < b) ? a : b;
}

template <typename T>
void swap(T& a, T& b) {
    T temp = a;
    a = b;
    b = temp;
}

// Template กับหลาย Type Parameters
template <typename T, typename U>
void printPair(T first, U second) {
    cout << "(" << first << ", " << second << ")" << endl;
}

// Template Function ที่คืน Type ต่างออกไป
template <typename T>
double average(T arr[], int size) {
    double sum = 0;
    for (int i = 0; i < size; i++) {
        sum += arr[i];
    }
    return sum / size;
}

// Template กับ Vector
template <typename T>
void printVector(const vector<T>& v, const string& label = "Vector") {
    cout << label << ": [";
    for (int i = 0; i < v.size(); i++) {
        cout << v[i];
        if (i < v.size() - 1) cout << ", ";
    }
    cout << "]" << endl;
}

// Template สำหรับ Sort (Bubble Sort)
template <typename T>
void bubbleSort(vector<T>& arr) {
    int n = arr.size();
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                swap(arr[j], arr[j + 1]);
            }
        }
    }
}

// Template Specialization
template <typename T>
string toString(T value) {
    return to_string(value);
}

// Specialization สำหรับ bool
template <>
string toString<bool>(bool value) {
    return value ? "true" : "false";
}

// Specialization สำหรับ string
template <>
string toString<string>(string value) {
    return "\"" + value + "\"";
}

int main() {
    // === getMax / getMin ===
    cout << "=== Template getMax/getMin ===" << endl;
    
    cout << "max(3, 7) = " << getMax(3, 7) << endl;
    cout << "max(3.14, 2.71) = " << getMax(3.14, 2.71) << endl;
    cout << "max('a', 'z') = " << getMax('a', 'z') << endl;
    cout << "max(\"apple\", \"banana\") = " << getMax(string("apple"), string("banana")) << endl;
    
    // === swap ===
    cout << "\n=== Template swap ===" << endl;
    
    int a = 10, b = 20;
    cout << "ก่อน: a=" << a << ", b=" << b << endl;
    swap(a, b);
    cout << "หลัง: a=" << a << ", b=" << b << endl;
    
    string s1 = "Hello", s2 = "World";
    cout << "ก่อน: s1=" << s1 << ", s2=" << s2 << endl;
    swap(s1, s2);
    cout << "หลัง: s1=" << s1 << ", s2=" << s2 << endl;
    
    // === printPair ===
    cout << "\n=== Template printPair ===" << endl;
    
    printPair(1, "one");
    printPair(3.14, true);
    printPair('A', 65);
    
    // === average ===
    cout << "\n=== Template average ===" << endl;
    
    int intArr[] = {1, 2, 3, 4, 5};
    double dblArr[] = {1.5, 2.5, 3.5, 4.5, 5.5};
    
    cout << "Average int: " << average(intArr, 5) << endl;
    cout << "Average double: " << average(dblArr, 5) << endl;
    
    // === vector ===
    cout << "\n=== Template Vector ===" << endl;
    
    vector<int> intVec = {3, 1, 4, 1, 5, 9, 2, 6};
    vector<string> strVec = {"banana", "apple", "cherry", "date"};
    vector<double> dblVec = {3.14, 2.71, 1.41, 1.73};
    
    printVector(intVec, "Integers");
    printVector(strVec, "Strings");
    printVector(dblVec, "Doubles");
    
    bubbleSort(intVec);
    bubbleSort(strVec);
    
    printVector(intVec, "Sorted Integers");
    printVector(strVec, "Sorted Strings");
    
    // === toString Specialization ===
    cout << "\n=== Template Specialization ===" << endl;
    
    cout << toString(42) << endl;
    cout << toString(3.14) << endl;
    cout << toString(true) << endl;
    cout << toString(string("Hello")) << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 38: ตัวแปร Static และ Global

```cpp
#include <iostream>
using namespace std;

// === Global Variables ===
int globalCounter = 0;  // ทุก function เข้าถึงได้

void incrementGlobal() {
    globalCounter++;
}

// === Static Local Variables ===
void countCalls() {
    static int callCount = 0;  // ค่าคงอยู่ระหว่าง calls
    callCount++;
    cout << "เรียกฟังก์ชันครั้งที่: " << callCount << endl;
}

// === Static Function ===
static void privateFunction() {
    cout << "ฟังก์ชัน Private (ใช้ได้ใน file นี้เท่านั้น)" << endl;
}

// === Static Counter Generator ===
int generateID() {
    static int nextID = 1000;
    return nextID++;
}

// === Singleton Pattern ด้วย Static ===
class Config {
private:
    static Config* instance;
    string serverIP;
    int port;
    
    Config() : serverIP("127.0.0.1"), port(8080) {}
    
public:
    static Config* getInstance() {
        if (!instance) {
            instance = new Config();
        }
        return instance;
    }
    
    void setServer(string ip, int p) {
        serverIP = ip;
        port = p;
    }
    
    void showConfig() {
        cout << "Server: " << serverIP << ":" << port << endl;
    }
};

Config* Config::instance = nullptr;  // Initialize static member

int main() {
    // === Global Variable ===
    cout << "=== Global Variable ===" << endl;
    cout << "globalCounter = " << globalCounter << endl;
    incrementGlobal();
    incrementGlobal();
    incrementGlobal();
    cout << "globalCounter หลัง increment 3 ครั้ง = " << globalCounter << endl;
    
    // === Static Local Variable ===
    cout << "\n=== Static Local Variable ===" << endl;
    countCalls();  // ครั้งที่ 1
    countCalls();  // ครั้งที่ 2
    countCalls();  // ครั้งที่ 3
    
    // === Static Function ===
    cout << "\n=== Static Function ===" << endl;
    privateFunction();
    
    // === ID Generator ===
    cout << "\n=== ID Generator ===" << endl;
    for (int i = 0; i < 5; i++) {
        cout << "ID: " << generateID() << endl;
    }
    
    // === Singleton ===
    cout << "\n=== Singleton Config ===" << endl;
    Config* config1 = Config::getInstance();
    Config* config2 = Config::getInstance();
    
    cout << "config1 == config2: " << (config1 == config2 ? "true" : "false") << endl;
    
    config1->setServer("192.168.1.1", 3000);
    config2->showConfig();  // แสดงค่าที่ config1 set
    
    return 0;
}
```

---

## ขั้นตอนที่ 39: ฟังก์ชันขั้นสูง - Higher Order Functions

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <algorithm>
#include <numeric>
using namespace std;

// === Higher Order Functions ===

// รับ function เป็น parameter
void applyTwice(function<int(int)> f, int x) {
    cout << "apply once: " << f(x) << endl;
    cout << "apply twice: " << f(f(x)) << endl;
}

// คืน function
function<int(int)> multiplier(int factor) {
    return [factor](int x) { return x * factor; };
}

function<int(int)> adder(int addend) {
    return [addend](int x) { return x + addend; };
}

// Function Composition
function<int(int)> compose(function<int(int)> f, function<int(int)> g) {
    return [f, g](int x) { return f(g(x)); };
}

// Map function
template <typename T, typename U>
vector<U> map(const vector<T>& v, function<U(T)> f) {
    vector<U> result;
    for (const T& item : v) {
        result.push_back(f(item));
    }
    return result;
}

// Filter function
template <typename T>
vector<T> filter(const vector<T>& v, function<bool(T)> pred) {
    vector<T> result;
    for (const T& item : v) {
        if (pred(item)) {
            result.push_back(item);
        }
    }
    return result;
}

// Reduce function
template <typename T, typename U>
U reduce(const vector<T>& v, U init, function<U(U, T)> f) {
    U result = init;
    for (const T& item : v) {
        result = f(result, item);
    }
    return result;
}

int main() {
    // === applyTwice ===
    cout << "=== Higher Order Functions ===" << endl;
    
    auto double_ = [](int x) { return x * 2; };
    auto addTen = [](int x) { return x + 10; };
    
    cout << "applyTwice(double, 3):" << endl;
    applyTwice(double_, 3);  // 6, 12
    
    cout << "applyTwice(addTen, 5):" << endl;
    applyTwice(addTen, 5);  // 15, 25
    
    // === Closure / Currying ===
    cout << "\n=== Closure / Currying ===" << endl;
    
    auto triple = multiplier(3);
    auto quadruple = multiplier(4);
    auto add5 = adder(5);
    
    cout << "triple(7) = " << triple(7) << endl;       // 21
    cout << "quadruple(7) = " << quadruple(7) << endl;  // 28
    cout << "add5(10) = " << add5(10) << endl;           // 15
    
    // === Function Composition ===
    cout << "\n=== Function Composition ===" << endl;
    
    auto tripleAndAdd5 = compose(add5, triple);  // add5(triple(x))
    cout << "tripleAndAdd5(4) = " << tripleAndAdd5(4) << endl;  // add5(12) = 17
    
    // === Functional Pipeline ===
    cout << "\n=== Functional Pipeline ===" << endl;
    
    vector<int> numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // map: คูณด้วย 2
    auto doubled = map<int, int>(numbers, [](int x) { return x * 2; });
    cout << "map(*2): ";
    for (int n : doubled) cout << n << " ";
    cout << endl;
    
    // filter: เลือกเลขคู่
    auto evens = filter<int>(numbers, [](int x) { return x % 2 == 0; });
    cout << "filter(even): ";
    for (int n : evens) cout << n << " ";
    cout << endl;
    
    // reduce: หาผลรวม
    auto sum = reduce<int, int>(numbers, 0, [](int acc, int x) { return acc + x; });
    cout << "reduce(sum): " << sum << endl;  // 55
    
    // reduce: หาผลคูณ
    auto product = reduce<int, long long>(vector<int>{1,2,3,4,5}, 1LL, 
        [](long long acc, int x) { return acc * x; });
    cout << "reduce(product 1-5): " << product << endl;  // 120
    
    // Pipeline: filter even numbers then double then sum
    auto evenNums = filter<int>(numbers, [](int x) { return x % 2 == 0; });
    auto doubledEvens = map<int, int>(evenNums, [](int x) { return x * 2; });
    auto totalSum = reduce<int, int>(doubledEvens, 0, [](int acc, int x) { return acc + x; });
    
    cout << "sum of doubled evens: " << totalSum << endl;  // (2+4+6+8+10)*2 = 60
    
    // === สร้าง Callback System ===
    cout << "\n=== Callback System ===" << endl;
    
    using Callback = function<void(int)>;
    
    vector<Callback> callbacks;
    
    callbacks.push_back([](int n) {
        cout << "Callback 1: n = " << n << endl;
    });
    
    callbacks.push_back([](int n) {
        cout << "Callback 2: n^2 = " << n * n << endl;
    });
    
    callbacks.push_back([](int n) {
        cout << "Callback 3: n is " << (n % 2 == 0 ? "even" : "odd") << endl;
    });
    
    cout << "เรียก callbacks กับ n=7:" << endl;
    for (auto& cb : callbacks) {
        cb(7);
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 40: โปรแกรมสรุป - Student Grade System

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <iomanip>
#include <numeric>
#include <functional>
using namespace std;

// === Data Structures ===
struct Student {
    string name;
    int id;
    vector<double> scores;
};

// === ฟังก์ชัน utility ===
double calculateAverage(const vector<double>& scores) {
    if (scores.empty()) return 0.0;
    double sum = accumulate(scores.begin(), scores.end(), 0.0);
    return sum / scores.size();
}

char getGrade(double avg) {
    if (avg >= 90) return 'A';
    if (avg >= 80) return 'B';
    if (avg >= 70) return 'C';
    if (avg >= 60) return 'D';
    return 'F';
}

string getGradeDescription(char grade) {
    switch (grade) {
        case 'A': return "ดีเยี่ยม";
        case 'B': return "ดีมาก";
        case 'C': return "ดี";
        case 'D': return "พอใช้";
        case 'F': return "ตก";
        default:  return "ไม่ทราบ";
    }
}

void printStudentReport(const Student& s) {
    double avg = calculateAverage(s.scores);
    char grade = getGrade(avg);
    
    cout << fixed << setprecision(1);
    cout << setw(6) << s.id << "  ";
    cout << setw(15) << left << s.name << right << "  ";
    
    for (double score : s.scores) {
        cout << setw(6) << score;
    }
    
    cout << "  " << setw(7) << avg;
    cout << "  " << grade;
    cout << "  " << getGradeDescription(grade) << endl;
}

void printClassSummary(const vector<Student>& students) {
    cout << "\n=== สรุปผลการเรียน ===" << endl;
    
    vector<double> averages;
    for (const auto& s : students) {
        averages.push_back(calculateAverage(s.scores));
    }
    
    double classAvg = calculateAverage(averages);
    double maxAvg = *max_element(averages.begin(), averages.end());
    double minAvg = *min_element(averages.begin(), averages.end());
    
    // หานักเรียนที่คะแนนสูงสุด
    int maxIdx = max_element(averages.begin(), averages.end()) - averages.begin();
    int minIdx = min_element(averages.begin(), averages.end()) - averages.begin();
    
    cout << "จำนวนนักเรียน: " << students.size() << " คน" << endl;
    cout << "คะแนนเฉลี่ยห้อง: " << fixed << setprecision(1) << classAvg << endl;
    cout << "คะแนนสูงสุด: " << maxAvg << " (" << students[maxIdx].name << ")" << endl;
    cout << "คะแนนต่ำสุด: " << minAvg << " (" << students[minIdx].name << ")" << endl;
    
    // นับเกรด
    map<char, int> gradeCount;
    for (double avg : averages) {
        gradeCount[getGrade(avg)]++;
    }
    
    cout << "\nการกระจายเกรด:" << endl;
    for (auto& [grade, count] : gradeCount) {
        cout << "เกรด " << grade << ": " << count << " คน";
        // Bar chart
        cout << " |";
        for (int i = 0; i < count; i++) cout << "█";
        cout << endl;
    }
}

int main() {
    cout << "=====================================" << endl;
    cout << "       ระบบจัดการผลการเรียน          " << endl;
    cout << "=====================================" << endl;
    
    // สร้างข้อมูลนักเรียน
    vector<Student> students = {
        {"สมชาย ใจดี",    1001, {85, 90, 78, 92, 88}},
        {"สมหญิง งาม",   1002, {72, 68, 75, 80, 71}},
        {"สมศรี สุขใจ",  1003, {95, 98, 92, 96, 94}},
        {"สมพร รักดี",   1004, {55, 60, 58, 65, 62}},
        {"สมทรง ดีงาม",  1005, {88, 85, 90, 87, 91}},
        {"สมโชค มีสุข",  1006, {45, 50, 48, 52, 55}},
        {"สมบัติ ใหม่",  1007, {78, 82, 79, 85, 81}},
    };
    
    // แสดงหัวตาราง
    cout << "\n" << setw(6) << "รหัส" << "  ";
    cout << setw(15) << left << "ชื่อ" << right << "  ";
    cout << setw(6) << "วิชา1" << setw(6) << "วิชา2" 
         << setw(6) << "วิชา3" << setw(6) << "วิชา4" << setw(6) << "วิชา5";
    cout << "  " << setw(7) << "เฉลี่ย" << "  เกรด" << endl;
    cout << string(80, '-') << endl;
    
    // แสดงผลแต่ละนักเรียน
    for (const auto& student : students) {
        printStudentReport(student);
    }
    
    // สรุปผล
    printClassSummary(students);
    
    // เรียงลำดับตามคะแนน
    cout << "\n=== เรียงลำดับตามคะแนน (สูงสุด-ต่ำสุด) ===" << endl;
    
    vector<pair<double, string>> ranking;
    for (const auto& s : students) {
        ranking.push_back({calculateAverage(s.scores), s.name});
    }
    
    sort(ranking.begin(), ranking.end(), [](const auto& a, const auto& b) {
        return a.first > b.first;
    });
    
    cout << fixed << setprecision(1);
    for (int i = 0; i < ranking.size(); i++) {
        cout << setw(3) << (i + 1) << ". ";
        cout << setw(20) << left << ranking[i].second << right;
        cout << " = " << setw(5) << ranking[i].first;
        cout << " (เกรด " << getGrade(ranking[i].first) << ")" << endl;
    }
    
    return 0;
}
```

---

## สรุป Part 004

ใน Part นี้คุณได้เรียนรู้:

1. ✅ พื้นฐานฟังก์ชัน: Declaration, Definition, Calling
2. ✅ Pass by Value vs Reference vs Pointer
3. ✅ Default Parameters
4. ✅ Function Overloading
5. ✅ Recursive Functions (Factorial, Fibonacci, Binary Search, Hanoi)
6. ✅ Inline Functions
7. ✅ Lambda Functions และ Captures
8. ✅ Function Templates
9. ✅ Static Variables และ Functions
10. ✅ Higher Order Functions (Map, Filter, Reduce)

---

## แบบฝึกหัด Part 004

**ฝึกหัดที่ 1:** เขียน Template Function สำหรับ Linear Search ที่ใช้ได้กับทุก type

**ฝึกหัดที่ 2:** เขียน Recursive Function สำหรับหา Sum ของตัวเลขในอาร์เรย์

**ฝึกหัดที่ 3:** เขียนโปรแกรมคำนวณเลขฐาน n โดยใช้ Recursion

**ฝึกหัดที่ 4:** เขียน Higher Order Function ที่รับ vector และ function แล้วส่งคืน vector ที่ถูก transform

---

⬅️ **ก่อนหน้า:** [Part 003](part003.md) | ➡️ **ต่อไป:** [Part 005: Arrays, Strings, Pointers](part005.md)
