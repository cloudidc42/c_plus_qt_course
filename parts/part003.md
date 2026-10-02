# Part 003: การควบคุมการทำงาน (Control Flow)

## ขั้นตอนที่ 21-30

---

## ขั้นตอนที่ 21: คำสั่ง if-else

การควบคุมการทำงานเป็นสิ่งสำคัญในการเขียนโปรแกรม ช่วยให้โปรแกรมตัดสินใจและทำงานแตกต่างกันตามเงื่อนไข

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    // === if เบื้องต้น ===
    cout << "=== if Statement ===" << endl;
    
    int temperature = 35;
    
    if (temperature > 30) {
        cout << "อากาศร้อนมาก" << endl;
    }
    
    // === if-else ===
    cout << "\n=== if-else ===" << endl;
    
    int score = 75;
    
    if (score >= 60) {
        cout << "ผ่านการสอบ" << endl;
    } else {
        cout << "ไม่ผ่านการสอบ" << endl;
    }
    
    // === if-else if-else ===
    cout << "\n=== if-else if-else ===" << endl;
    
    int marks = 85;
    
    if (marks >= 90) {
        cout << "เกรด A (ดีเยี่ยม)" << endl;
    } else if (marks >= 80) {
        cout << "เกรด B (ดีมาก)" << endl;
    } else if (marks >= 70) {
        cout << "เกรด C (ดี)" << endl;
    } else if (marks >= 60) {
        cout << "เกรด D (พอใช้)" << endl;
    } else {
        cout << "เกรด F (ตก)" << endl;
    }
    
    // === Nested if ===
    cout << "\n=== Nested if ===" << endl;
    
    int age = 20;
    bool hasID = true;
    
    if (age >= 18) {
        cout << "อายุผ่านเกณฑ์" << endl;
        if (hasID) {
            cout << "สามารถเข้าได้" << endl;
        } else {
            cout << "ต้องมีบัตรประชาชน" << endl;
        }
    } else {
        cout << "อายุไม่ถึงเกณฑ์ (ต้องอายุ 18+)" << endl;
    }
    
    // === โปรแกรมตัวอย่าง: ตรวจสอบสามเหลี่ยม ===
    cout << "\n=== ตรวจสอบสามเหลี่ยม ===" << endl;
    
    double a = 3, b = 4, c = 5;
    
    // ตรวจสอบว่าเป็นสามเหลี่ยมได้หรือไม่
    if (a + b > c && b + c > a && a + c > b) {
        cout << "เป็นสามเหลี่ยม!" << endl;
        
        // ตรวจสอบประเภทสามเหลี่ยม
        if (a == b && b == c) {
            cout << "ประเภท: สามเหลี่ยมด้านเท่า" << endl;
        } else if (a == b || b == c || a == c) {
            cout << "ประเภท: สามเหลี่ยมหน้าจั่ว" << endl;
        } else {
            cout << "ประเภท: สามเหลี่ยมด้านไม่เท่า" << endl;
        }
        
        // ตรวจสอบมุม
        if (a*a + b*b == c*c || b*b + c*c == a*a || a*a + c*c == b*b) {
            cout << "มุม: สามเหลี่ยมมุมฉาก" << endl;
        } else if (a*a + b*b > c*c && b*b + c*c > a*a && a*a + c*c > b*b) {
            cout << "มุม: สามเหลี่ยมมุมแหลม" << endl;
        } else {
            cout << "มุม: สามเหลี่ยมมุมป้าน" << endl;
        }
    } else {
        cout << "ไม่สามารถเป็นสามเหลี่ยมได้!" << endl;
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 22: คำสั่ง switch-case

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    // === switch-case พื้นฐาน ===
    cout << "=== switch-case ===" << endl;
    
    int day = 3;
    
    switch (day) {
        case 1:
            cout << "วันจันทร์" << endl;
            break;
        case 2:
            cout << "วันอังคาร" << endl;
            break;
        case 3:
            cout << "วันพุธ" << endl;
            break;
        case 4:
            cout << "วันพฤหัสบดี" << endl;
            break;
        case 5:
            cout << "วันศุกร์" << endl;
            break;
        case 6:
            cout << "วันเสาร์" << endl;
            break;
        case 7:
            cout << "วันอาทิตย์" << endl;
            break;
        default:
            cout << "วันที่ไม่ถูกต้อง" << endl;
    }
    
    // === switch กับ char ===
    cout << "\n=== switch กับ char ===" << endl;
    
    char grade = 'B';
    
    switch (grade) {
        case 'A':
        case 'a':
            cout << "ดีเยี่ยม (90-100)" << endl;
            break;
        case 'B':
        case 'b':
            cout << "ดีมาก (80-89)" << endl;
            break;
        case 'C':
        case 'c':
            cout << "ดี (70-79)" << endl;
            break;
        case 'D':
        case 'd':
            cout << "พอใช้ (60-69)" << endl;
            break;
        case 'F':
        case 'f':
            cout << "ตก (ต่ำกว่า 60)" << endl;
            break;
        default:
            cout << "เกรดไม่ถูกต้อง" << endl;
    }
    
    // === Fall-through (ไม่มี break) ===
    cout << "\n=== Fall-through ===" << endl;
    
    int month = 4;
    int daysInMonth;
    
    switch (month) {
        case 2:
            daysInMonth = 28;  // ไม่คำนึงถึงปีอธิกสุรทิน
            break;
        case 4:
        case 6:
        case 9:
        case 11:
            daysInMonth = 30;
            break;
        default:
            daysInMonth = 31;
    }
    
    cout << "เดือนที่ " << month << " มี " << daysInMonth << " วัน" << endl;
    
    // === switch กับ string (C++17 ใช้ if-else หรือ map) ===
    cout << "\n=== เมนูโปรแกรม ===" << endl;
    
    int choice;
    cout << "1. เพิ่มข้อมูล\n2. ลบข้อมูล\n3. แก้ไขข้อมูล\n4. แสดงข้อมูล\n5. ออก" << endl;
    cout << "เลือก: ";
    cin >> choice;
    
    switch (choice) {
        case 1:
            cout << "เพิ่มข้อมูล..." << endl;
            break;
        case 2:
            cout << "ลบข้อมูล..." << endl;
            break;
        case 3:
            cout << "แก้ไขข้อมูล..." << endl;
            break;
        case 4:
            cout << "แสดงข้อมูล..." << endl;
            break;
        case 5:
            cout << "กำลังออกจากโปรแกรม..." << endl;
            break;
        default:
            cout << "ตัวเลือกไม่ถูกต้อง!" << endl;
    }
    
    // === switch กับ enum class ===
    cout << "\n=== switch กับ enum class ===" << endl;
    
    enum class Season { SPRING, SUMMER, AUTUMN, WINTER };
    Season currentSeason = Season::SUMMER;
    
    switch (currentSeason) {
        case Season::SPRING:
            cout << "ฤดูใบไม้ผลิ - อากาศอบอุ่น" << endl;
            break;
        case Season::SUMMER:
            cout << "ฤดูร้อน - อากาศร้อน" << endl;
            break;
        case Season::AUTUMN:
            cout << "ฤดูใบไม้ร่วง - อากาศเย็น" << endl;
            break;
        case Season::WINTER:
            cout << "ฤดูหนาว - อากาศหนาว" << endl;
            break;
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 23: วนซ้ำด้วย while

```cpp
#include <iostream>
using namespace std;

int main() {
    // === while loop พื้นฐาน ===
    cout << "=== while loop ===" << endl;
    
    int count = 1;
    while (count <= 5) {
        cout << "นับ: " << count << endl;
        count++;
    }
    
    // === while กับ Accumulator ===
    cout << "\n=== Sum กับ while ===" << endl;
    
    int sum = 0;
    int i = 1;
    while (i <= 100) {
        sum += i;
        i++;
    }
    cout << "ผลรวม 1-100 = " << sum << endl;  // 5050
    
    // === while กับ User Input ===
    cout << "\n=== รับข้อมูลจนกว่าจะ 0 ===" << endl;
    
    int num;
    int total = 0;
    int count2 = 0;
    
    cout << "กรอกตัวเลข (กรอก 0 เพื่อหยุด): " << endl;
    while (true) {
        cout << "ตัวเลข #" << (count2 + 1) << ": ";
        cin >> num;
        
        if (num == 0) break;  // หยุดถ้าเป็น 0
        
        total += num;
        count2++;
    }
    
    if (count2 > 0) {
        cout << "จำนวนตัวเลข: " << count2 << endl;
        cout << "ผลรวม: " << total << endl;
        cout << "ค่าเฉลี่ย: " << (double)total / count2 << endl;
    }
    
    // === do-while loop ===
    cout << "\n=== do-while loop ===" << endl;
    
    // do-while รันอย่างน้อย 1 ครั้ง แม้เงื่อนไขจะเป็นเท็จตั้งแต่ต้น
    int x = 10;
    do {
        cout << "x = " << x << endl;
        x++;
    } while (x < 5);  // เงื่อนไขเป็นเท็จแต่แสดงครั้งนึง
    
    // === ตัวอย่าง do-while กับ Menu ===
    cout << "\n=== Menu ด้วย do-while ===" << endl;
    
    int menuChoice;
    do {
        cout << "\n--- เมนูหลัก ---" << endl;
        cout << "1. ดูข้อมูล" << endl;
        cout << "2. เพิ่มข้อมูล" << endl;
        cout << "0. ออก" << endl;
        cout << "เลือก: ";
        cin >> menuChoice;
        
        switch (menuChoice) {
            case 1: cout << "แสดงข้อมูล..." << endl; break;
            case 2: cout << "เพิ่มข้อมูล..." << endl; break;
            case 0: cout << "ลาก่อน!" << endl; break;
            default: cout << "ตัวเลือกไม่ถูกต้อง" << endl;
        }
    } while (menuChoice != 0);
    
    return 0;
}
```

---

## ขั้นตอนที่ 24: วนซ้ำด้วย for

```cpp
#include <iostream>
#include <vector>
using namespace std;

int main() {
    // === for loop พื้นฐาน ===
    cout << "=== for loop ===" << endl;
    
    for (int i = 0; i < 5; i++) {
        cout << "i = " << i << endl;
    }
    
    // === for นับถอยหลัง ===
    cout << "\n=== นับถอยหลัง ===" << endl;
    
    for (int i = 10; i >= 1; i--) {
        cout << i;
        if (i > 1) cout << " ";
    }
    cout << " ปล่อยจรวด!" << endl;
    
    // === for วนซ้ำ n ครั้ง ===
    cout << "\n=== ตารางสูตรคูณ 5 ===" << endl;
    
    for (int i = 1; i <= 10; i++) {
        cout << "5 x " << i << " = " << (5 * i) << endl;
    }
    
    // === Nested for loops ===
    cout << "\n=== ตารางสูตรคูณ 1-10 ===" << endl;
    
    for (int i = 1; i <= 10; i++) {
        for (int j = 1; j <= 10; j++) {
            cout.width(4);  // จัดช่องว่าง
            cout << (i * j);
        }
        cout << endl;
    }
    
    // === Range-based for loop (C++11) ===
    cout << "\n=== Range-based for ===" << endl;
    
    // กับ Array
    int numbers[] = {10, 20, 30, 40, 50};
    for (int n : numbers) {
        cout << n << " ";
    }
    cout << endl;
    
    // กับ string
    string text = "Hello";
    for (char c : text) {
        cout << c << "-";
    }
    cout << endl;
    
    // กับ vector
    vector<string> fruits = {"แอปเปิ้ล", "ส้ม", "กล้วย", "มะม่วง"};
    for (const string& fruit : fruits) {  // const& เพื่อไม่ copy
        cout << fruit << endl;
    }
    
    // แก้ไขค่าด้วย reference
    vector<int> vals = {1, 2, 3, 4, 5};
    for (int& v : vals) {
        v *= 2;
    }
    for (int v : vals) {
        cout << v << " ";  // 2 4 6 8 10
    }
    cout << endl;
    
    // === for loop หลายตัวแปร ===
    cout << "\n=== for หลายตัวแปร ===" << endl;
    
    for (int i = 0, j = 10; i < j; i++, j--) {
        cout << "i=" << i << ", j=" << j << endl;
    }
    
    // === Infinite for loop ===
    // for (;;) { ... }  // ไม่มีเงื่อนไข = ไม่มีสิ้นสุด (ต้องใช้ break)
    
    return 0;
}
```

---

## ขั้นตอนที่ 25: คำสั่ง break, continue, goto

```cpp
#include <iostream>
using namespace std;

int main() {
    // === break ===
    cout << "=== break ===" << endl;
    
    // หาตัวเลขแรกที่หารด้วย 7 ลงตัว ในช่วง 1-100
    for (int i = 1; i <= 100; i++) {
        if (i % 7 == 0) {
            cout << "ตัวเลขแรกที่หารด้วย 7 ลงตัว: " << i << endl;
            break;  // ออกจาก loop
        }
    }
    
    // break ใน nested loop
    cout << "\n=== break ใน nested loop ===" << endl;
    
    bool found = false;
    int targetRow, targetCol;
    
    int matrix[3][3] = {
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9}
    };
    
    int target = 5;
    
    for (int row = 0; row < 3 && !found; row++) {
        for (int col = 0; col < 3; col++) {
            if (matrix[row][col] == target) {
                targetRow = row;
                targetCol = col;
                found = true;
                break;  // break แค่ inner loop
            }
        }
    }
    
    if (found) {
        cout << "พบ " << target << " ที่ [" << targetRow << "][" << targetCol << "]" << endl;
    }
    
    // === continue ===
    cout << "\n=== continue ===" << endl;
    
    // แสดงเลขคี่ 1-20
    cout << "เลขคี่ 1-20: ";
    for (int i = 1; i <= 20; i++) {
        if (i % 2 == 0) continue;  // ข้ามถ้าเป็นเลขคู่
        cout << i << " ";
    }
    cout << endl;
    
    // แสดงตัวเลขยกเว้น 5 และ 10
    cout << "1-15 ยกเว้น 5 และ 10: ";
    for (int i = 1; i <= 15; i++) {
        if (i == 5 || i == 10) continue;
        cout << i << " ";
    }
    cout << endl;
    
    // === goto (ไม่แนะนำ แต่ควรรู้) ===
    cout << "\n=== goto (ตัวอย่าง - ไม่แนะนำใช้) ===" << endl;
    
    int n = 1;
    
    loopStart:  // Label
    if (n <= 5) {
        cout << "n = " << n << endl;
        n++;
        goto loopStart;  // กระโดดไปที่ label
    }
    
    cout << "(goto ควรหลีกเลี่ยง ใช้ loop แทน)" << endl;
    
    // === การออกจาก Nested Loops ===
    cout << "\n=== ออกจาก Nested Loops อย่างถูกต้อง ===" << endl;
    
    // วิธีที่ 1: ใช้ flag variable
    bool stop = false;
    for (int i = 0; i < 5 && !stop; i++) {
        for (int j = 0; j < 5; j++) {
            if (i == 2 && j == 3) {
                stop = true;
                break;
            }
            cout << "(" << i << "," << j << ") ";
        }
        if (stop) break;
    }
    cout << endl;
    
    // วิธีที่ 2: ใช้ Lambda Function (C++11)
    auto searchAndBreak = [&]() {
        for (int i = 0; i < 5; i++) {
            for (int j = 0; j < 5; j++) {
                if (i == 2 && j == 3) {
                    cout << "พบที่ (" << i << "," << j << ")" << endl;
                    return;  // ออกจากทั้งหมด
                }
            }
        }
    };
    
    searchAndBreak();
    
    return 0;
}
```

---

## ขั้นตอนที่ 26: Pattern Programs - โปรแกรมวาดรูปแบบ

```cpp
#include <iostream>
using namespace std;

int main() {
    int n = 5;
    
    // === รูปแบบที่ 1: สามเหลี่ยมดาว ===
    cout << "=== สามเหลี่ยมดาว ===" << endl;
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= i; j++) {
            cout << "* ";
        }
        cout << endl;
    }
    
    // === รูปแบบที่ 2: สามเหลี่ยมกลับหัว ===
    cout << "\n=== สามเหลี่ยมกลับหัว ===" << endl;
    for (int i = n; i >= 1; i--) {
        for (int j = 1; j <= i; j++) {
            cout << "* ";
        }
        cout << endl;
    }
    
    // === รูปแบบที่ 3: พีระมิด ===
    cout << "\n=== พีระมิด ===" << endl;
    for (int i = 1; i <= n; i++) {
        // ช่องว่าง
        for (int j = 1; j <= n - i; j++) {
            cout << "  ";
        }
        // ดาว
        for (int j = 1; j <= 2*i - 1; j++) {
            cout << "* ";
        }
        cout << endl;
    }
    
    // === รูปแบบที่ 4: เพชร ===
    cout << "\n=== เพชร ===" << endl;
    // ครึ่งบน
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= n - i; j++) cout << " ";
        for (int j = 1; j <= 2*i - 1; j++) cout << "*";
        cout << endl;
    }
    // ครึ่งล่าง
    for (int i = n - 1; i >= 1; i--) {
        for (int j = 1; j <= n - i; j++) cout << " ";
        for (int j = 1; j <= 2*i - 1; j++) cout << "*";
        cout << endl;
    }
    
    // === รูปแบบที่ 5: Pascal's Triangle ===
    cout << "\n=== Pascal's Triangle ===" << endl;
    
    int triangle[10][10] = {};
    
    for (int i = 0; i < n; i++) {
        triangle[i][0] = 1;
        for (int j = 1; j <= i; j++) {
            triangle[i][j] = triangle[i-1][j-1] + triangle[i-1][j];
        }
    }
    
    for (int i = 0; i < n; i++) {
        // ช่องว่าง
        for (int j = 0; j < n - i - 1; j++) cout << "  ";
        // ตัวเลข
        for (int j = 0; j <= i; j++) {
            cout << triangle[i][j] << "   ";
        }
        cout << endl;
    }
    
    // === รูปแบบที่ 6: ตัวเลข Floyd's Triangle ===
    cout << "\n=== Floyd's Triangle ===" << endl;
    
    int num = 1;
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= i; j++) {
            cout.width(3);
            cout << num++;
        }
        cout << endl;
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 27: โปรแกรมตรรกะขั้นสูง

```cpp
#include <iostream>
#include <cmath>
using namespace std;

int main() {
    // === ตรวจสอบจำนวนเฉพาะ (Prime Number) ===
    cout << "=== จำนวนเฉพาะ 1-50 ===" << endl;
    
    for (int n = 2; n <= 50; n++) {
        bool isPrime = true;
        
        if (n < 2) {
            isPrime = false;
        } else {
            for (int i = 2; i <= sqrt(n); i++) {
                if (n % i == 0) {
                    isPrime = false;
                    break;
                }
            }
        }
        
        if (isPrime) cout << n << " ";
    }
    cout << endl;
    
    // === Fibonacci Series ===
    cout << "\n=== Fibonacci 20 ตัวแรก ===" << endl;
    
    long long a = 0, b = 1;
    cout << a << " " << b << " ";
    
    for (int i = 2; i < 20; i++) {
        long long next = a + b;
        cout << next << " ";
        a = b;
        b = next;
    }
    cout << endl;
    
    // === แฟกทอเรียล ===
    cout << "\n=== Factorial 1-10 ===" << endl;
    
    for (int n = 1; n <= 10; n++) {
        long long factorial = 1;
        for (int i = 2; i <= n; i++) {
            factorial *= i;
        }
        cout << n << "! = " << factorial << endl;
    }
    
    // === ห.ร.ม. และ ค.ร.น. ===
    cout << "\n=== ห.ร.ม. และ ค.ร.น. ===" << endl;
    
    auto gcd = [](int a, int b) {
        while (b != 0) {
            int temp = b;
            b = a % b;
            a = temp;
        }
        return a;
    };
    
    auto lcm = [&gcd](int a, int b) {
        return (a / gcd(a, b)) * b;
    };
    
    int x = 48, y = 36;
    cout << "ห.ร.ม.(" << x << ", " << y << ") = " << gcd(x, y) << endl;  // 12
    cout << "ค.ร.น.(" << x << ", " << y << ") = " << lcm(x, y) << endl;  // 144
    
    // === Armstrong Number ===
    cout << "\n=== Armstrong Numbers 1-1000 ===" << endl;
    
    for (int n = 1; n <= 1000; n++) {
        int temp = n;
        int digits = 0;
        while (temp > 0) {
            digits++;
            temp /= 10;
        }
        
        temp = n;
        long long sum = 0;
        while (temp > 0) {
            int digit = temp % 10;
            sum += pow(digit, digits);
            temp /= 10;
        }
        
        if (sum == n) {
            cout << n << " ";
        }
    }
    cout << endl;
    
    // === Palindrome ===
    cout << "\n=== ตรวจสอบ Palindrome ===" << endl;
    
    int nums[] = {121, 123, 1221, 12321, 12345};
    
    for (int n : nums) {
        int original = n;
        int reversed = 0;
        
        while (n > 0) {
            reversed = reversed * 10 + n % 10;
            n /= 10;
        }
        
        if (original == reversed) {
            cout << original << " เป็น Palindrome" << endl;
        } else {
            cout << original << " ไม่ใช่ Palindrome" << endl;
        }
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 28: โปรแกรม Number Systems Conversion

```cpp
#include <iostream>
#include <string>
#include <algorithm>
using namespace std;

// แปลง Decimal เป็น Binary
string decimalToBinary(int n) {
    if (n == 0) return "0";
    
    string result = "";
    bool negative = (n < 0);
    if (negative) n = -n;
    
    while (n > 0) {
        result += (char)('0' + n % 2);
        n /= 2;
    }
    
    if (negative) result += "-";
    reverse(result.begin(), result.end());
    return result;
}

// แปลง Decimal เป็น Hexadecimal
string decimalToHex(int n) {
    if (n == 0) return "0";
    
    string hex_chars = "0123456789ABCDEF";
    string result = "";
    
    while (n > 0) {
        result += hex_chars[n % 16];
        n /= 16;
    }
    
    reverse(result.begin(), result.end());
    return result;
}

// แปลง Binary เป็น Decimal
int binaryToDecimal(string binary) {
    int result = 0;
    int power = 1;
    
    for (int i = binary.length() - 1; i >= 0; i--) {
        if (binary[i] == '1') {
            result += power;
        }
        power *= 2;
    }
    
    return result;
}

int main() {
    cout << "=== ระบบเลขฐาน ===" << endl;
    
    // ตาราง 0-20
    cout << "\nDecimal\tBinary\t\tOctal\tHexadecimal" << endl;
    cout << "-----------------------------------------------" << endl;
    
    for (int i = 0; i <= 20; i++) {
        cout << i << "\t";
        cout << decimalToBinary(i) << "\t\t";
        cout << oct << i << "\t";  // Octal
        cout << hex << uppercase << i << endl;  // Hexadecimal
    }
    
    cout << dec;  // Reset to decimal
    
    // การแปลงค่า
    cout << "\n=== การแปลงค่า ===" << endl;
    cout << "255 (Dec) = " << decimalToBinary(255) << " (Bin)" << endl;
    cout << "255 (Dec) = " << decimalToHex(255) << " (Hex)" << endl;
    cout << "11111111 (Bin) = " << binaryToDecimal("11111111") << " (Dec)" << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 29: โปรแกรมสถิติ

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <cmath>
#include <iomanip>
using namespace std;

int main() {
    cout << "=== โปรแกรมคำนวณสถิติ ===" << endl;
    
    // รับข้อมูล
    int n;
    cout << "กรอกจำนวนข้อมูล: ";
    cin >> n;
    
    vector<double> data(n);
    cout << "กรอกข้อมูล " << n << " ตัว:" << endl;
    for (int i = 0; i < n; i++) {
        cout << "ข้อมูลที่ " << (i+1) << ": ";
        cin >> data[i];
    }
    
    // === คำนวณค่าสถิติ ===
    
    // Sum
    double sum = 0;
    for (double x : data) sum += x;
    
    // Mean (ค่าเฉลี่ย)
    double mean = sum / n;
    
    // Median (ค่ากลาง)
    vector<double> sorted = data;
    sort(sorted.begin(), sorted.end());
    
    double median;
    if (n % 2 == 0) {
        median = (sorted[n/2 - 1] + sorted[n/2]) / 2.0;
    } else {
        median = sorted[n/2];
    }
    
    // Mode (ฐานนิยม)
    double mode = sorted[0];
    int maxCount = 1, currentCount = 1;
    for (int i = 1; i < n; i++) {
        if (sorted[i] == sorted[i-1]) {
            currentCount++;
            if (currentCount > maxCount) {
                maxCount = currentCount;
                mode = sorted[i];
            }
        } else {
            currentCount = 1;
        }
    }
    
    // Variance และ Standard Deviation
    double variance = 0;
    for (double x : data) {
        variance += pow(x - mean, 2);
    }
    variance /= n;
    double stdDev = sqrt(variance);
    
    // Min, Max, Range
    double minVal = *min_element(data.begin(), data.end());
    double maxVal = *max_element(data.begin(), data.end());
    double range = maxVal - minVal;
    
    // แสดงผล
    cout << fixed << setprecision(2);
    cout << "\n=== ผลการคำนวณ ===" << endl;
    cout << "จำนวนข้อมูล: " << n << endl;
    cout << "ผลรวม: " << sum << endl;
    cout << "ค่าเฉลี่ย (Mean): " << mean << endl;
    cout << "ค่ากลาง (Median): " << median << endl;
    cout << "ฐานนิยม (Mode): " << mode << endl;
    cout << "ค่าต่ำสุด: " << minVal << endl;
    cout << "ค่าสูงสุด: " << maxVal << endl;
    cout << "พิสัย (Range): " << range << endl;
    cout << "ความแปรปรวน (Variance): " << variance << endl;
    cout << "ส่วนเบี่ยงเบนมาตรฐาน (Std Dev): " << stdDev << endl;
    
    // Histogram แบบง่าย
    cout << "\n=== Histogram ===" << endl;
    for (double x : sorted) {
        cout << setw(8) << x << " | ";
        int bars = (int)(x / maxVal * 20);
        for (int i = 0; i < bars; i++) cout << "█";
        cout << endl;
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 30: โปรแกรม Mini Bank System

```cpp
#include <iostream>
#include <string>
#include <iomanip>
using namespace std;

int main() {
    // === Mini Bank System ===
    cout << "=====================================" << endl;
    cout << "         ระบบธนาคารอย่างง่าย          " << endl;
    cout << "=====================================" << endl;
    
    string accountName;
    double balance = 0.0;
    
    // เปิดบัญชี
    cout << "\nกรอกชื่อเจ้าของบัญชี: ";
    getline(cin, accountName);
    
    cout << "กรอกเงินฝากเริ่มต้น: ";
    cin >> balance;
    
    cout << "\nเปิดบัญชีสำเร็จ สวัสดี คุณ" << accountName << "!" << endl;
    
    int choice;
    do {
        cout << "\n--- เมนูธนาคาร ---" << endl;
        cout << fixed << setprecision(2);
        cout << "ยอดคงเหลือ: " << balance << " บาท" << endl;
        cout << "1. ฝากเงิน" << endl;
        cout << "2. ถอนเงิน" << endl;
        cout << "3. โอนเงิน" << endl;
        cout << "4. ดูยอดคงเหลือ" << endl;
        cout << "0. ออกจากระบบ" << endl;
        cout << "เลือก: ";
        cin >> choice;
        
        switch (choice) {
            case 1: {
                double amount;
                cout << "กรอกจำนวนเงินที่ต้องการฝาก: ";
                cin >> amount;
                
                if (amount <= 0) {
                    cout << "จำนวนเงินต้องมากกว่า 0" << endl;
                } else {
                    balance += amount;
                    cout << "ฝากเงิน " << amount << " บาท สำเร็จ" << endl;
                    cout << "ยอดคงเหลือใหม่: " << balance << " บาท" << endl;
                }
                break;
            }
            
            case 2: {
                double amount;
                cout << "กรอกจำนวนเงินที่ต้องการถอน: ";
                cin >> amount;
                
                if (amount <= 0) {
                    cout << "จำนวนเงินต้องมากกว่า 0" << endl;
                } else if (amount > balance) {
                    cout << "ยอดเงินไม่เพียงพอ!" << endl;
                    cout << "ยอดคงเหลือปัจจุบัน: " << balance << " บาท" << endl;
                } else {
                    balance -= amount;
                    cout << "ถอนเงิน " << amount << " บาท สำเร็จ" << endl;
                    cout << "ยอดคงเหลือใหม่: " << balance << " บาท" << endl;
                }
                break;
            }
            
            case 3: {
                string targetAccount;
                double amount;
                
                cout << "กรอกชื่อบัญชีปลายทาง: ";
                cin.ignore();
                getline(cin, targetAccount);
                
                cout << "กรอกจำนวนเงินที่ต้องการโอน: ";
                cin >> amount;
                
                if (amount <= 0) {
                    cout << "จำนวนเงินต้องมากกว่า 0" << endl;
                } else if (amount > balance) {
                    cout << "ยอดเงินไม่เพียงพอ!" << endl;
                } else {
                    balance -= amount;
                    cout << "โอนเงิน " << amount << " บาท ไปยัง " 
                         << targetAccount << " สำเร็จ" << endl;
                    cout << "ยอดคงเหลือใหม่: " << balance << " บาท" << endl;
                }
                break;
            }
            
            case 4:
                cout << "\n--- ข้อมูลบัญชี ---" << endl;
                cout << "ชื่อ: " << accountName << endl;
                cout << "ยอดคงเหลือ: " << balance << " บาท" << endl;
                break;
            
            case 0:
                cout << "\nขอบคุณที่ใช้บริการ คุณ" << accountName << endl;
                cout << "ลาก่อน!" << endl;
                break;
            
            default:
                cout << "ตัวเลือกไม่ถูกต้อง กรุณาลองใหม่" << endl;
        }
        
    } while (choice != 0);
    
    return 0;
}
```

---

## สรุป Part 003

ใน Part นี้คุณได้เรียนรู้:

1. ✅ คำสั่ง if, if-else, if-else if-else
2. ✅ คำสั่ง switch-case
3. ✅ วนซ้ำด้วย while และ do-while
4. ✅ วนซ้ำด้วย for และ range-based for
5. ✅ คำสั่ง break, continue
6. ✅ Pattern Programs (สามเหลี่ยม, พีระมิด, เพชร)
7. ✅ โปรแกรมตรรกะขั้นสูง (Prime, Fibonacci, Factorial)
8. ✅ การแปลงระบบเลข
9. ✅ โปรแกรมคำนวณสถิติ
10. ✅ โปรแกรม Mini Bank System

---

## แบบฝึกหัด Part 003

**ฝึกหัดที่ 1:** เขียนโปรแกรมแสดงตารางสูตรคูณ 1-12 แบบสวยงาม

**ฝึกหัดที่ 2:** เขียนโปรแกรมเกมทายตัวเลข (1-100) พร้อมนับจำนวนครั้งที่ทาย

**ฝึกหัดที่ 3:** เขียนโปรแกรมตรวจสอบจำนวนเฉพาะโดยใช้ Sieve of Eratosthenes

**ฝึกหัดที่ 4:** เขียนโปรแกรมแปลงเลขฐานสอง ฐานแปด ฐานสิบ ฐานสิบหก

---

⬅️ **ก่อนหน้า:** [Part 002](part002.md) | ➡️ **ต่อไป:** [Part 004: ฟังก์ชัน](part004.md)
