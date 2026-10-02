# Part 002: ตัวแปร ชนิดข้อมูล และตัวดำเนินการ

## ขั้นตอนที่ 11-20

---

## ขั้นตอนที่ 11: ชนิดข้อมูลพื้นฐาน (Fundamental Data Types)

C++ มีชนิดข้อมูลพื้นฐานหลายประเภท แต่ละประเภทมีขนาดและช่วงค่าที่แตกต่างกัน

### ตารางชนิดข้อมูลพื้นฐาน

```
| ชนิดข้อมูล      | ขนาด (bytes) | ช่วงค่า                                    |
|----------------|-------------|-------------------------------------------|
| bool           | 1           | true หรือ false                            |
| char           | 1           | -128 ถึง 127 (หรือ 0 ถึง 255)             |
| unsigned char  | 1           | 0 ถึง 255                                  |
| short          | 2           | -32,768 ถึง 32,767                         |
| unsigned short | 2           | 0 ถึง 65,535                               |
| int            | 4           | -2,147,483,648 ถึง 2,147,483,647          |
| unsigned int   | 4           | 0 ถึง 4,294,967,295                        |
| long           | 4 หรือ 8    | ขึ้นกับ Platform                           |
| long long      | 8           | -9,223,372,036,854,775,808 ถึง ...        |
| float          | 4           | ~3.4e-38 ถึง ~3.4e+38 (7 decimal digits)  |
| double         | 8           | ~1.7e-308 ถึง ~1.7e+308 (15 digits)       |
| long double    | 8-16        | ขึ้นกับ Platform                           |
| wchar_t        | 2 หรือ 4    | Wide Character                             |
```

### โค้ดตัวอย่าง: ชนิดข้อมูลพื้นฐาน

```cpp
#include <iostream>
#include <climits>    // สำหรับค่า INT_MAX, INT_MIN ฯลฯ
#include <cfloat>     // สำหรับ FLT_MAX, DBL_MAX ฯลฯ

using namespace std;

int main() {
    // === Boolean ===
    bool isTrue = true;
    bool isFalse = false;
    cout << "bool size: " << sizeof(bool) << " bytes" << endl;
    cout << "isTrue = " << isTrue << endl;   // แสดง 1
    cout << "isFalse = " << isFalse << endl; // แสดง 0
    
    // === Integer Types ===
    char ch = 'A';
    short s = 32767;
    int i = 2147483647;
    long l = 2147483647L;
    long long ll = 9223372036854775807LL;
    
    cout << "\n=== Integer Types ===" << endl;
    cout << "char: " << ch << " (size: " << sizeof(char) << ")" << endl;
    cout << "short max: " << SHRT_MAX << " (size: " << sizeof(short) << ")" << endl;
    cout << "int max: " << INT_MAX << " (size: " << sizeof(int) << ")" << endl;
    cout << "long max: " << LONG_MAX << " (size: " << sizeof(long) << ")" << endl;
    cout << "long long max: " << LLONG_MAX << " (size: " << sizeof(long long) << ")" << endl;
    
    // === Floating Point Types ===
    float f = 3.14f;
    double d = 3.14159265358979;
    long double ld = 3.14159265358979323846L;
    
    cout << "\n=== Floating Point Types ===" << endl;
    cout << "float: " << f << " (size: " << sizeof(float) << ")" << endl;
    cout << "double: " << d << " (size: " << sizeof(double) << ")" << endl;
    cout << "long double size: " << sizeof(long double) << endl;
    
    // === Unsigned Types ===
    unsigned int ui = 4294967295U;
    unsigned long long ull = 18446744073709551615ULL;
    
    cout << "\n=== Unsigned Types ===" << endl;
    cout << "unsigned int max: " << UINT_MAX << endl;
    cout << "unsigned long long max: " << ULLONG_MAX << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 12: การประกาศและกำหนดค่าตัวแปร

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    // === การประกาศตัวแปร ===
    
    // 1. ประกาศและกำหนดค่าพร้อมกัน
    int age = 25;
    double salary = 45000.50;
    string name = "สมชาย";
    bool isMarried = false;
    
    // 2. ประกาศก่อน กำหนดค่าทีหลัง
    int score;
    score = 95;
    
    // 3. ประกาศหลายตัวพร้อมกัน
    int x = 10, y = 20, z = 30;
    
    // 4. Uniform Initialization (C++11) - แนะนำ
    int num1{42};
    double pi{3.14159};
    string city{"กรุงเทพ"};
    
    // 5. Auto Type Deduction (C++11)
    auto weight = 70.5;        // double
    auto count = 100;          // int
    auto firstName = "John"s;  // string (ใช้ ""s suffix)
    
    cout << "ชื่อ: " << name << endl;
    cout << "อายุ: " << age << endl;
    cout << "เงินเดือน: " << salary << endl;
    cout << "คะแนน: " << score << endl;
    cout << "เมือง: " << city << endl;
    
    // === การตั้งชื่อตัวแปร ===
    // ✅ ถูกต้อง:
    int studentAge = 20;
    int student_age = 20;
    int _privateVar = 10;
    int myVariable2024 = 0;
    
    // ❌ ผิด:
    // int 2variable = 0;     // ขึ้นต้นด้วยตัวเลข
    // int my-var = 0;        // มีขีดกลาง
    // int int = 0;           // ใช้ Reserved Word
    // int my var = 0;        // มีช่องว่าง
    
    return 0;
}
```

---

## ขั้นตอนที่ 13: ค่าคงที่ (Constants)

```cpp
#include <iostream>
#include <cmath>
using namespace std;

int main() {
    // === 1. const Keyword ===
    const int MAX_SIZE = 100;
    const double PI = 3.14159265358979;
    const string GREETING = "สวัสดี";
    
    // MAX_SIZE = 200;  // Error! ไม่สามารถเปลี่ยนค่าได้
    
    cout << "PI = " << PI << endl;
    cout << "MAX_SIZE = " << MAX_SIZE << endl;
    
    // === 2. constexpr (C++11) - Compile-time Constant ===
    constexpr int ARRAY_SIZE = 50;
    constexpr double GRAVITY = 9.81;
    constexpr double EARTH_RADIUS_KM = 6371.0;
    
    // ใช้ใน Array (ต้องเป็น Compile-time constant)
    int arr[ARRAY_SIZE];
    
    // === 3. #define (Preprocessor Macro) - เก่า ไม่แนะนำ ===
    #define OLD_MAX 255
    #define SQUARE(x) ((x) * (x))
    
    cout << "OLD_MAX = " << OLD_MAX << endl;
    cout << "SQUARE(5) = " << SQUARE(5) << endl;
    
    // === 4. enum (Enumeration) ===
    enum Color { RED, GREEN, BLUE };
    enum Day { MON = 1, TUE, WED, THU, FRI, SAT, SUN };
    
    Color myColor = GREEN;
    Day today = WED;
    
    cout << "Color: " << myColor << endl;  // แสดง 1
    cout << "Day: " << today << endl;       // แสดง 3
    
    // === 5. enum class (Scoped Enum, C++11) - แนะนำ ===
    enum class Season { SPRING, SUMMER, AUTUMN, WINTER };
    enum class Direction { NORTH, SOUTH, EAST, WEST };
    
    Season s = Season::SUMMER;
    Direction d = Direction::NORTH;
    
    if (s == Season::SUMMER) {
        cout << "ฤดูร้อน!" << endl;
    }
    
    // Numeric values
    int seasonNum = static_cast<int>(Season::AUTUMN);
    cout << "AUTUMN = " << seasonNum << endl;  // แสดง 2
    
    return 0;
}
```

---

## ขั้นตอนที่ 14: ตัวดำเนินการทางคณิตศาสตร์ (Arithmetic Operators)

```cpp
#include <iostream>
#include <cmath>
using namespace std;

int main() {
    int a = 10, b = 3;
    
    // === ตัวดำเนินการพื้นฐาน ===
    cout << "=== Arithmetic Operators ===" << endl;
    cout << "a = " << a << ", b = " << b << endl;
    cout << "a + b = " << (a + b) << endl;   // 13
    cout << "a - b = " << (a - b) << endl;   // 7
    cout << "a * b = " << (a * b) << endl;   // 30
    cout << "a / b = " << (a / b) << endl;   // 3 (Integer Division!)
    cout << "a % b = " << (a % b) << endl;   // 1 (Modulo/Remainder)
    
    // Integer Division vs Float Division
    double da = 10.0, db = 3.0;
    cout << "\n=== Float Division ===" << endl;
    cout << "10.0 / 3.0 = " << (da / db) << endl;  // 3.33333
    cout << "10 / 3 = " << (a / b) << endl;          // 3 (Integer!)
    cout << "(double)10 / 3 = " << ((double)a / b) << endl;  // 3.33333
    
    // === ตัวดำเนินการเพิ่ม/ลด ===
    cout << "\n=== Increment/Decrement ===" << endl;
    int x = 5;
    cout << "x = " << x << endl;      // 5
    cout << "x++ = " << x++ << endl;  // 5 (Post-increment: ใช้ค่าก่อน เพิ่มทีหลัง)
    cout << "x = " << x << endl;      // 6
    cout << "++x = " << ++x << endl;  // 7 (Pre-increment: เพิ่มก่อน ใช้ค่าทีหลัง)
    cout << "x-- = " << x-- << endl;  // 7 (Post-decrement)
    cout << "x = " << x << endl;      // 6
    cout << "--x = " << --x << endl;  // 5 (Pre-decrement)
    
    // === ตัวดำเนินการมอบหมายประกอบ (Compound Assignment) ===
    cout << "\n=== Compound Assignment ===" << endl;
    int n = 10;
    cout << "n = " << n << endl;
    n += 5;  cout << "n += 5 -> " << n << endl;   // 15
    n -= 3;  cout << "n -= 3 -> " << n << endl;   // 12
    n *= 2;  cout << "n *= 2 -> " << n << endl;   // 24
    n /= 4;  cout << "n /= 4 -> " << n << endl;   // 6
    n %= 4;  cout << "n %= 4 -> " << n << endl;   // 2
    
    // === ฟังก์ชันคณิตศาสตร์จาก <cmath> ===
    cout << "\n=== Math Functions ===" << endl;
    cout << "sqrt(16) = " << sqrt(16) << endl;      // 4
    cout << "pow(2, 10) = " << pow(2, 10) << endl;  // 1024
    cout << "abs(-5) = " << abs(-5) << endl;         // 5
    cout << "ceil(3.2) = " << ceil(3.2) << endl;     // 4
    cout << "floor(3.8) = " << floor(3.8) << endl;   // 3
    cout << "round(3.5) = " << round(3.5) << endl;   // 4
    cout << "log(M_E) = " << log(M_E) << endl;       // 1 (Natural log)
    cout << "log10(100) = " << log10(100) << endl;   // 2
    cout << "sin(M_PI/2) = " << sin(M_PI/2) << endl; // 1
    cout << "cos(0) = " << cos(0) << endl;            // 1
    
    return 0;
}
```

---

## ขั้นตอนที่ 15: ตัวดำเนินการเปรียบเทียบ (Comparison Operators)

```cpp
#include <iostream>
using namespace std;

int main() {
    int a = 10, b = 20, c = 10;
    
    cout << "=== Comparison Operators ===" << endl;
    cout << "a=" << a << ", b=" << b << ", c=" << c << endl;
    
    // == Equal to
    cout << "\n(a == c) = " << (a == c) << endl;   // 1 (true)
    cout << "(a == b) = " << (a == b) << endl;   // 0 (false)
    
    // != Not equal to
    cout << "(a != b) = " << (a != b) << endl;   // 1 (true)
    cout << "(a != c) = " << (a != c) << endl;   // 0 (false)
    
    // > Greater than
    cout << "(b > a) = " << (b > a) << endl;     // 1 (true)
    cout << "(a > b) = " << (a > b) << endl;     // 0 (false)
    
    // < Less than
    cout << "(a < b) = " << (a < b) << endl;     // 1 (true)
    cout << "(b < a) = " << (b < a) << endl;     // 0 (false)
    
    // >= Greater than or equal
    cout << "(a >= c) = " << (a >= c) << endl;   // 1 (true, equal)
    cout << "(b >= a) = " << (b >= a) << endl;   // 1 (true, greater)
    cout << "(a >= b) = " << (a >= b) << endl;   // 0 (false)
    
    // <= Less than or equal
    cout << "(a <= c) = " << (a <= c) << endl;   // 1 (true, equal)
    cout << "(a <= b) = " << (a <= b) << endl;   // 1 (true, less)
    cout << "(b <= a) = " << (b <= a) << endl;   // 0 (false)
    
    // === Comparison กับ boolalpha ===
    cout << "\n=== ด้วย boolalpha ===" << endl;
    cout << boolalpha;
    cout << "(a == c) = " << (a == c) << endl;   // true
    cout << "(a == b) = " << (a == b) << endl;   // false
    cout << noboolalpha;  // Reset
    
    // === ระวัง: Float Comparison ===
    cout << "\n=== Float Comparison (ระวัง!) ===" << endl;
    double x = 0.1 + 0.2;
    double y = 0.3;
    cout << "0.1 + 0.2 = " << x << endl;
    cout << "0.3 = " << y << endl;
    cout << "(0.1 + 0.2 == 0.3) = " << (x == y) << endl;  // false!
    
    // วิธีที่ถูกต้องในการเปรียบเทียบ float
    double epsilon = 1e-9;
    cout << "(|x - y| < epsilon) = " << (abs(x - y) < epsilon) << endl;  // true
    
    return 0;
}
```

---

## ขั้นตอนที่ 16: ตัวดำเนินการตรรกศาสตร์ (Logical Operators)

```cpp
#include <iostream>
using namespace std;

int main() {
    bool p = true, q = false;
    
    cout << "=== Logical Operators ===" << endl;
    cout << boolalpha;
    cout << "p = " << p << ", q = " << q << endl;
    
    // && (AND) - true ถ้าทั้งสองเป็น true
    cout << "\n=== AND (&&) ===" << endl;
    cout << "(p && q) = " << (p && q) << endl;  // false
    cout << "(p && p) = " << (p && p) << endl;  // true
    cout << "(q && q) = " << (q && q) << endl;  // false
    
    // || (OR) - true ถ้าอย่างน้อยหนึ่งเป็น true
    cout << "\n=== OR (||) ===" << endl;
    cout << "(p || q) = " << (p || q) << endl;  // true
    cout << "(q || q) = " << (q || q) << endl;  // false
    cout << "(p || p) = " << (p || p) << endl;  // true
    
    // ! (NOT) - สลับค่า
    cout << "\n=== NOT (!) ===" << endl;
    cout << "(!p) = " << (!p) << endl;  // false
    cout << "(!q) = " << (!q) << endl;  // true
    
    // === Truth Table ===
    cout << "\n=== Truth Table ===" << endl;
    cout << "A\tB\tA&&B\tA||B\t!A" << endl;
    cout << "------------------------------------" << endl;
    
    bool vals[] = {true, false};
    for (bool a : vals) {
        for (bool b : vals) {
            cout << a << "\t" << b << "\t" 
                 << (a && b) << "\t" 
                 << (a || b) << "\t"
                 << (!a) << endl;
        }
    }
    
    // === Short-Circuit Evaluation ===
    cout << "\n=== Short-Circuit Evaluation ===" << endl;
    int x = 5;
    
    // && หยุดถ้า operand แรก false
    if (false && (++x > 0)) {
        // ไม่เข้ามาที่นี่
    }
    cout << "x หลัง false && (++x > 0): " << x << endl;  // 5 (ไม่เพิ่ม!)
    
    // || หยุดถ้า operand แรก true
    if (true || (++x > 0)) {
        // ++x ไม่ถูกประมวลผล
    }
    cout << "x หลัง true || (++x > 0): " << x << endl;  // 5 (ไม่เพิ่ม!)
    
    // === ตัวอย่างการใช้งานจริง ===
    cout << "\n=== ตัวอย่างจริง ===" << endl;
    int age = 25;
    double income = 50000;
    bool hasJob = true;
    
    bool canApplyLoan = (age >= 20 && age <= 65) && (income >= 30000) && hasJob;
    cout << "สมัครสินเชื่อได้: " << canApplyLoan << endl;  // true
    
    bool needsHelp = (age < 18) || (age > 65) || (income < 10000);
    cout << "ต้องการความช่วยเหลือ: " << needsHelp << endl;  // false
    
    return 0;
}
```

---

## ขั้นตอนที่ 17: ตัวดำเนินการ Bitwise

```cpp
#include <iostream>
#include <bitset>
using namespace std;

int main() {
    unsigned int a = 12;  // 1100 ในเลขฐานสอง
    unsigned int b = 10;  // 1010 ในเลขฐานสอง
    
    cout << "=== Bitwise Operators ===" << endl;
    cout << "a = " << a << " (" << bitset<4>(a) << ")" << endl;
    cout << "b = " << b << " (" << bitset<4>(b) << ")" << endl;
    
    // & (Bitwise AND)
    cout << "\na & b = " << (a & b) << " (" << bitset<4>(a & b) << ")" << endl;
    // 1100 & 1010 = 1000 = 8
    
    // | (Bitwise OR)
    cout << "a | b = " << (a | b) << " (" << bitset<4>(a | b) << ")" << endl;
    // 1100 | 1010 = 1110 = 14
    
    // ^ (Bitwise XOR)
    cout << "a ^ b = " << (a ^ b) << " (" << bitset<4>(a ^ b) << ")" << endl;
    // 1100 ^ 1010 = 0110 = 6
    
    // ~ (Bitwise NOT)
    cout << "~a = " << (~a) << " (เลขลบ)" << endl;
    
    // << (Left Shift) - คูณด้วย 2^n
    cout << "\na << 1 = " << (a << 1) << " (" << bitset<8>(a << 1) << ")" << endl;  // 24
    cout << "a << 2 = " << (a << 2) << " (" << bitset<8>(a << 2) << ")" << endl;  // 48
    
    // >> (Right Shift) - หารด้วย 2^n
    cout << "a >> 1 = " << (a >> 1) << " (" << bitset<8>(a >> 1) << ")" << endl;  // 6
    cout << "a >> 2 = " << (a >> 2) << " (" << bitset<8>(a >> 2) << ")" << endl;  // 3
    
    // === การใช้งาน Bitwise ในชีวิตจริง ===
    cout << "\n=== Flag Operations ===" << endl;
    
    // Flags สำหรับสิทธิ์ไฟล์ (เหมือน Linux permissions)
    const unsigned int READ    = 1 << 0;  // 0001
    const unsigned int WRITE   = 1 << 1;  // 0010
    const unsigned int EXECUTE = 1 << 2;  // 0100
    
    unsigned int permissions = READ | WRITE;  // 0011 = 3
    
    // ตรวจสอบ Flag
    if (permissions & READ)    cout << "มีสิทธิ์อ่าน" << endl;
    if (permissions & WRITE)   cout << "มีสิทธิ์เขียน" << endl;
    if (!(permissions & EXECUTE)) cout << "ไม่มีสิทธิ์รัน" << endl;
    
    // เพิ่ม Flag
    permissions |= EXECUTE;  // 0111 = 7
    if (permissions & EXECUTE) cout << "เพิ่มสิทธิ์รันแล้ว" << endl;
    
    // ลบ Flag
    permissions &= ~WRITE;  // ลบ WRITE flag
    if (!(permissions & WRITE)) cout << "ลบสิทธิ์เขียนแล้ว" << endl;
    
    // Toggle Flag
    permissions ^= READ;   // สลับ READ flag
    cout << "หลัง Toggle READ: " << (permissions & READ ? "มีสิทธิ์อ่าน" : "ไม่มีสิทธิ์อ่าน") << endl;
    
    // === ตรวจสอบเลขคู่/คี่ ===
    cout << "\n=== ตรวจสอบเลขคู่/คี่ ===" << endl;
    for (int i = 1; i <= 10; i++) {
        if (i & 1) {
            cout << i << " เป็นเลขคี่" << endl;
        } else {
            cout << i << " เป็นเลขคู่" << endl;
        }
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 18: ตัวดำเนินการ Ternary และอื่นๆ

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    // === Ternary Operator (? :) ===
    cout << "=== Ternary Operator ===" << endl;
    
    int x = 10;
    
    // รูปแบบ: condition ? value_if_true : value_if_false
    string result = (x > 5) ? "มากกว่า 5" : "น้อยกว่าหรือเท่ากับ 5";
    cout << "x = " << x << " -> " << result << endl;
    
    // ซ้อน Ternary (ระวัง อ่านยาก)
    int score = 85;
    string grade = (score >= 90) ? "A" :
                   (score >= 80) ? "B" :
                   (score >= 70) ? "C" :
                   (score >= 60) ? "D" : "F";
    cout << "คะแนน " << score << " -> เกรด " << grade << endl;
    
    // === sizeof Operator ===
    cout << "\n=== sizeof Operator ===" << endl;
    cout << "sizeof(bool) = " << sizeof(bool) << endl;
    cout << "sizeof(char) = " << sizeof(char) << endl;
    cout << "sizeof(int) = " << sizeof(int) << endl;
    cout << "sizeof(long) = " << sizeof(long) << endl;
    cout << "sizeof(double) = " << sizeof(double) << endl;
    cout << "sizeof(string) = " << sizeof(string) << endl;
    
    int arr[] = {1, 2, 3, 4, 5};
    cout << "sizeof(arr) = " << sizeof(arr) << endl;          // 20 (5*4)
    cout << "Array length = " << sizeof(arr)/sizeof(arr[0]) << endl;  // 5
    
    // === Comma Operator ===
    cout << "\n=== Comma Operator ===" << endl;
    int a, b;
    a = (b = 5, b + 3);  // b=5, แล้ว a=8
    cout << "a = " << a << ", b = " << b << endl;
    
    // === Type Casting ===
    cout << "\n=== Type Casting ===" << endl;
    
    // C-style cast (เก่า)
    int num = 7;
    double result2 = (double)num / 2;
    cout << "(double)7 / 2 = " << result2 << endl;  // 3.5
    
    // C++ style casts (แนะนำ)
    
    // static_cast: สำหรับ Numeric conversions
    double d = 3.99;
    int i = static_cast<int>(d);  // ตัดทศนิยมทิ้ง
    cout << "static_cast<int>(3.99) = " << i << endl;  // 3
    
    // ตัวอย่างเพิ่มเติม
    char c = 'A';
    int ascii = static_cast<int>(c);
    cout << "'A' -> ASCII: " << ascii << endl;  // 65
    
    int code = 97;
    char letter = static_cast<char>(code);
    cout << "ASCII 97 -> char: " << letter << endl;  // a
    
    // === Precedence ===
    cout << "\n=== Operator Precedence ===" << endl;
    // ลำดับความสำคัญ (สูงไปต่ำ):
    // 1. () [] -> .
    // 2. ! ~ ++ -- (unary)
    // 3. * / %
    // 4. + -
    // 5. << >>
    // 6. < <= > >=
    // 7. == !=
    // 8. &
    // 9. ^
    // 10. |
    // 11. &&
    // 12. ||
    // 13. ?:
    // 14. = += -= *= /=
    // 15. ,
    
    int val = 2 + 3 * 4;       // 2 + 12 = 14
    int val2 = (2 + 3) * 4;    // 5 * 4 = 20
    cout << "2 + 3 * 4 = " << val << endl;     // 14
    cout << "(2 + 3) * 4 = " << val2 << endl;  // 20
    
    bool check = 5 > 3 && 10 < 20 || false;
    // = (5>3) && (10<20) || false
    // = true && true || false
    // = true || false
    // = true
    cout << "5 > 3 && 10 < 20 || false = " << boolalpha << check << endl;  // true
    
    return 0;
}
```

---

## ขั้นตอนที่ 19: การรับข้อมูลจากผู้ใช้ (Input)

```cpp
#include <iostream>
#include <string>
#include <limits>
using namespace std;

int main() {
    // === cin: รับ Integer ===
    cout << "=== การรับข้อมูล ===" << endl;
    
    int age;
    cout << "กรอกอายุของคุณ: ";
    cin >> age;
    cout << "อายุของคุณคือ: " << age << " ปี" << endl;
    
    // === cin: รับ Double ===
    double height;
    cout << "\nกรอกส่วนสูงของคุณ (เมตร): ";
    cin >> height;
    cout << "ส่วนสูง: " << height << " เมตร" << endl;
    
    // === cin: รับ String (คำเดียว) ===
    string firstName;
    cout << "\nกรอกชื่อของคุณ: ";
    cin >> firstName;  // รับแค่คำแรก ถ้ามีช่องว่างจะหยุด
    cout << "ชื่อ: " << firstName << endl;
    
    // === getline: รับ String ทั้งบรรทัด ===
    cin.ignore(numeric_limits<streamsize>::max(), '\n');  // Clear buffer
    
    string fullName;
    cout << "\nกรอกชื่อเต็มของคุณ: ";
    getline(cin, fullName);  // รับทั้งบรรทัดรวมช่องว่าง
    cout << "ชื่อเต็ม: " << fullName << endl;
    
    // === รับข้อมูลหลายค่าพร้อมกัน ===
    int x, y;
    cout << "\nกรอกพิกัด x y (คั่นด้วยช่องว่าง): ";
    cin >> x >> y;
    cout << "พิกัด: (" << x << ", " << y << ")" << endl;
    
    // === ตรวจสอบ Input ===
    cout << "\n=== การตรวจสอบ Input ===" << endl;
    int number;
    cout << "กรอกตัวเลข: ";
    
    while (!(cin >> number)) {
        cin.clear();  // Clear error flag
        cin.ignore(numeric_limits<streamsize>::max(), '\n');  // Clear buffer
        cout << "กรุณากรอกตัวเลขเท่านั้น: ";
    }
    cout << "ตัวเลขที่กรอก: " << number << endl;
    
    // === โปรแกรมตัวอย่าง: Calculator ===
    cout << "\n=== Calculator ===" << endl;
    double num1, num2;
    char op;
    
    cout << "กรอก: num1 operator num2 (เช่น 10 + 5): ";
    cin >> num1 >> op >> num2;
    
    double calcResult;
    switch (op) {
        case '+': calcResult = num1 + num2; break;
        case '-': calcResult = num1 - num2; break;
        case '*': calcResult = num1 * num2; break;
        case '/': 
            if (num2 != 0) calcResult = num1 / num2;
            else { cout << "Error: หารด้วยศูนย์ไม่ได้!"; return 1; }
            break;
        default: cout << "ไม่รู้จักตัวดำเนินการ"; return 1;
    }
    
    cout << num1 << " " << op << " " << num2 << " = " << calcResult << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 20: ชนิดข้อมูลขั้นสูง - String และ Auto

```cpp
#include <iostream>
#include <string>
#include <typeinfo>
using namespace std;

int main() {
    // === std::string ===
    cout << "=== std::string ===" << endl;
    
    string s1 = "Hello";
    string s2 = "World";
    
    // การต่อ String
    string s3 = s1 + ", " + s2 + "!";
    cout << s3 << endl;  // Hello, World!
    
    // ความยาว
    cout << "ความยาว: " << s3.length() << endl;   // 13
    cout << "ขนาด: " << s3.size() << endl;         // 13 (เหมือนกัน)
    
    // เข้าถึงตัวอักษร
    cout << "ตัวอักษรแรก: " << s3[0] << endl;           // H
    cout << "ตัวอักษรสุดท้าย: " << s3[s3.length()-1] << endl;  // !
    cout << "ตัวอักษรที่ 1: " << s3.at(1) << endl;       // e (at() มีการตรวจสอบ bounds)
    
    // Substring
    cout << "Substring(7, 5): " << s3.substr(7, 5) << endl;  // World
    
    // หา String
    size_t pos = s3.find("World");
    if (pos != string::npos) {
        cout << "พบ 'World' ที่ตำแหน่ง: " << pos << endl;  // 7
    }
    
    // แทนที่
    string s4 = s3;
    s4.replace(7, 5, "C++");
    cout << "หลัง replace: " << s4 << endl;  // Hello, C++!
    
    // แปลงเป็น Upper/Lower case
    string upper = s1;
    for (char& c : upper) c = toupper(c);
    cout << "Uppercase: " << upper << endl;  // HELLO
    
    // เปรียบเทียบ
    cout << boolalpha;
    cout << "(s1 == \"Hello\"): " << (s1 == "Hello") << endl;  // true
    cout << "(s1 < s2): " << (s1 < s2) << endl;               // true (H < W)
    
    // === auto Type Deduction ===
    cout << "\n=== auto ===" << endl;
    
    auto i = 42;           // int
    auto d = 3.14;         // double
    auto f = 3.14f;        // float
    auto c = 'A';          // char
    auto b = true;         // bool
    auto s = string("Hi"); // string
    auto ptr = &i;         // int*
    
    cout << "i type: " << typeid(i).name() << " = " << i << endl;
    cout << "d type: " << typeid(d).name() << " = " << d << endl;
    cout << "f type: " << typeid(f).name() << " = " << f << endl;
    
    // auto กับ Range-based for loop
    int arr[] = {1, 2, 3, 4, 5};
    for (auto& val : arr) {
        val *= 2;  // แก้ไขค่าได้เพราะใช้ reference
    }
    for (auto val : arr) {
        cout << val << " ";  // 2 4 6 8 10
    }
    cout << endl;
    
    // === Fixed-width Integer Types (C++11) ===
    cout << "\n=== Fixed-width Types ===" << endl;
    #include <cstdint>  // จริงๆ ต้องอยู่บนสุด แต่แสดงให้เห็น
    
    // ขนาดแน่นอน ไม่ขึ้นกับ Platform
    int8_t   i8  = 127;
    int16_t  i16 = 32767;
    int32_t  i32 = 2147483647;
    int64_t  i64 = 9223372036854775807LL;
    
    uint8_t  u8  = 255;
    uint16_t u16 = 65535;
    uint32_t u32 = 4294967295U;
    
    cout << "int8_t max: " << (int)i8 << endl;
    cout << "int32_t max: " << i32 << endl;
    cout << "uint8_t max: " << (int)u8 << endl;
    
    // === String Conversion ===
    cout << "\n=== String Conversion ===" << endl;
    
    // Number to String
    int num = 42;
    string numStr = to_string(num);
    cout << "to_string(42) = \"" << numStr << "\"" << endl;
    
    // String to Number
    string str = "123";
    int converted = stoi(str);       // string to int
    double dConverted = stod("3.14"); // string to double
    cout << "stoi(\"123\") = " << converted << endl;
    cout << "stod(\"3.14\") = " << dConverted << endl;
    
    return 0;
}
```

---

## โปรแกรมตัวอย่างสรุป Part 002: BMI Calculator

```cpp
// ============================================
// โปรแกรม BMI Calculator
// =============================================
#include <iostream>
#include <string>
#include <iomanip>  // สำหรับ setprecision
using namespace std;

int main() {
    cout << "=====================================" << endl;
    cout << "       โปรแกรมคำนวณค่า BMI          " << endl;
    cout << "=====================================" << endl;
    
    // รับข้อมูล
    string name;
    double weight, height;
    
    cout << "\nกรอกชื่อของคุณ: ";
    cin >> name;
    
    cout << "กรอกน้ำหนัก (กิโลกรัม): ";
    cin >> weight;
    
    cout << "กรอกส่วนสูง (เซนติเมตร): ";
    cin >> height;
    
    // แปลงหน่วย
    double heightM = height / 100.0;  // เซนติเมตร -> เมตร
    
    // คำนวณ BMI
    double bmi = weight / (heightM * heightM);
    
    // วิเคราะห์ผล
    string category;
    string advice;
    
    if (bmi < 18.5) {
        category = "น้ำหนักน้อยกว่าเกณฑ์";
        advice = "ควรรับประทานอาหารให้เพียงพอ";
    } else if (bmi < 25.0) {
        category = "น้ำหนักปกติ (สุขภาพดี)";
        advice = "ดีมาก! รักษาไว้เช่นนี้";
    } else if (bmi < 30.0) {
        category = "น้ำหนักเกิน";
        advice = "ควรออกกำลังกายและควบคุมอาหาร";
    } else {
        category = "อ้วน";
        advice = "ควรปรึกษาแพทย์";
    }
    
    // แสดงผล
    cout << "\n=====================================" << endl;
    cout << "ผลการคำนวณ BMI สำหรับคุณ " << name << endl;
    cout << "=====================================" << endl;
    cout << fixed << setprecision(1);
    cout << "น้ำหนัก: " << weight << " กก." << endl;
    cout << "ส่วนสูง: " << height << " ซม. (" << heightM << " ม.)" << endl;
    cout << "BMI: " << setprecision(2) << bmi << endl;
    cout << "สถานะ: " << category << endl;
    cout << "คำแนะนำ: " << advice << endl;
    cout << "=====================================" << endl;
    
    return 0;
}
```

---

## สรุป Part 002

ใน Part นี้คุณได้เรียนรู้:

1. ✅ ชนิดข้อมูลพื้นฐาน (bool, char, int, float, double)
2. ✅ การประกาศและกำหนดค่าตัวแปร
3. ✅ ค่าคงที่ (const, constexpr, enum)
4. ✅ ตัวดำเนินการทางคณิตศาสตร์
5. ✅ ตัวดำเนินการเปรียบเทียบ
6. ✅ ตัวดำเนินการตรรกศาสตร์
7. ✅ ตัวดำเนินการ Bitwise
8. ✅ Ternary Operator และ Type Casting
9. ✅ การรับข้อมูลจากผู้ใช้ (cin, getline)
10. ✅ auto Type Deduction และ String operations

---

## แบบฝึกหัด Part 002

**ฝึกหัดที่ 1:** เขียนโปรแกรมแปลงอุณหภูมิ (Celsius ↔ Fahrenheit)
```
สูตร: F = C × 9/5 + 32
```

**ฝึกหัดที่ 2:** เขียนโปรแกรมคำนวณพื้นที่วงกลม
```
สูตร: A = π × r²
```

**ฝึกหัดที่ 3:** เขียนโปรแกรมตรวจสอบว่าปีที่กรอกเป็นปีอธิกสุรทิน (Leap Year) หรือไม่
```
ปีอธิกสุรทิน: หารด้วย 4 ลงตัว แต่ไม่หารด้วย 100 ลงตัว
              หรือ หารด้วย 400 ลงตัว
```

**ฝึกหัดที่ 4:** เขียนโปรแกรมที่ใช้ Bitwise Operators ตรวจสอบ Flag ต่างๆ

---

⬅️ **ก่อนหน้า:** [Part 001](part001.md) | ➡️ **ต่อไป:** [Part 003: Control Flow](part003.md)
