# Part 005: Arrays, Strings และ Pointers

## ขั้นตอนที่ 41-55

---

## ขั้นตอนที่ 41: Arrays พื้นฐาน

```cpp
#include <iostream>
#include <array>
#include <algorithm>
#include <numeric>
using namespace std;

int main() {
    // === Array Declaration ===
    
    // C-style array
    int scores[5];                       // ไม่กำหนดค่าเริ่มต้น
    int grades[5] = {90, 85, 78, 92, 88}; // กำหนดค่า
    int zeros[5] = {};                   // ค่าเป็น 0 ทั้งหมด
    int partial[5] = {1, 2};             // {1, 2, 0, 0, 0}
    
    // ขนาดจาก initializer
    int primes[] = {2, 3, 5, 7, 11, 13};  // ขนาด = 6 อัตโนมัติ
    
    // === เข้าถึงสมาชิก ===
    cout << "=== Array Access ===" << endl;
    
    for (int i = 0; i < 5; i++) {
        cout << "grades[" << i << "] = " << grades[i] << endl;
    }
    
    // Range-based for
    cout << "\nprimes: ";
    for (int p : primes) cout << p << " ";
    cout << endl;
    
    // === Array ขนาด ===
    int primeCount = sizeof(primes) / sizeof(primes[0]);
    cout << "จำนวน primes: " << primeCount << endl;
    
    // === 2D Array ===
    cout << "\n=== 2D Array ===" << endl;
    
    int matrix[3][4] = {
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12}
    };
    
    for (int row = 0; row < 3; row++) {
        for (int col = 0; col < 4; col++) {
            cout.width(4);
            cout << matrix[row][col];
        }
        cout << endl;
    }
    
    // === std::array (C++11) - แนะนำ ===
    cout << "\n=== std::array ===" << endl;
    
    array<int, 5> arr = {3, 1, 4, 1, 5};
    
    cout << "size: " << arr.size() << endl;
    cout << "front: " << arr.front() << endl;
    cout << "back: " << arr.back() << endl;
    
    sort(arr.begin(), arr.end());
    
    cout << "sorted: ";
    for (int x : arr) cout << x << " ";
    cout << endl;
    
    // at() มีการตรวจสอบ bounds
    try {
        cout << arr.at(10) << endl;  // จะ throw exception
    } catch (out_of_range& e) {
        cout << "Error: " << e.what() << endl;
    }
    
    // === Array Operations ===
    cout << "\n=== Array Operations ===" << endl;
    
    int numbers[] = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    int n = sizeof(numbers) / sizeof(numbers[0]);
    
    // Sum
    int sum = 0;
    for (int x : numbers) sum += x;
    cout << "Sum: " << sum << endl;
    
    // Min/Max
    int minVal = *min_element(numbers, numbers + n);
    int maxVal = *max_element(numbers, numbers + n);
    cout << "Min: " << minVal << ", Max: " << maxVal << endl;
    
    // Sort
    sort(numbers, numbers + n);
    cout << "Sorted: ";
    for (int x : numbers) cout << x << " ";
    cout << endl;
    
    // Binary Search (ต้อง sort ก่อน)
    int target = 7;
    bool found = binary_search(numbers, numbers + n, target);
    cout << "Search " << target << ": " << (found ? "found" : "not found") << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 42: Vector - Dynamic Array

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
using namespace std;

int main() {
    // === Vector Declaration ===
    cout << "=== Vector ===" << endl;
    
    vector<int> v1;                          // ว่าง
    vector<int> v2(5);                       // 5 ตัว ค่า 0
    vector<int> v3(5, 10);                   // 5 ตัว ค่า 10
    vector<int> v4 = {1, 2, 3, 4, 5};       // Initializer list
    vector<int> v5(v4);                      // Copy constructor
    
    // === Push/Pop ===
    cout << "\n=== Push/Pop ===" << endl;
    
    vector<string> names;
    names.push_back("สมชาย");
    names.push_back("สมหญิง");
    names.push_back("สมศรี");
    
    cout << "Size: " << names.size() << endl;
    cout << "Capacity: " << names.capacity() << endl;
    
    for (const string& name : names) {
        cout << name << endl;
    }
    
    names.pop_back();  // ลบตัวสุดท้าย
    cout << "After pop_back: " << names.size() << " elements" << endl;
    
    // === Insert/Erase ===
    cout << "\n=== Insert/Erase ===" << endl;
    
    vector<int> nums = {1, 2, 3, 4, 5};
    
    // Insert ที่ตำแหน่งที่ 2
    nums.insert(nums.begin() + 2, 99);
    cout << "After insert: ";
    for (int n : nums) cout << n << " ";
    cout << endl;
    
    // Erase ที่ตำแหน่ง 0
    nums.erase(nums.begin());
    cout << "After erase(0): ";
    for (int n : nums) cout << n << " ";
    cout << endl;
    
    // Erase range
    nums.erase(nums.begin() + 1, nums.begin() + 3);
    cout << "After erase range: ";
    for (int n : nums) cout << n << " ";
    cout << endl;
    
    // === emplace_back (C++11) - เร็วกว่า push_back ===
    cout << "\n=== emplace_back ===" << endl;
    
    vector<pair<int, string>> students;
    students.emplace_back(1, "Alice");  // สร้าง pair ตรงใน vector
    students.emplace_back(2, "Bob");
    students.emplace_back(3, "Charlie");
    
    for (const auto& [id, name] : students) {
        cout << id << ": " << name << endl;
    }
    
    // === Vector 2D ===
    cout << "\n=== 2D Vector ===" << endl;
    
    int rows = 3, cols = 4;
    vector<vector<int>> matrix(rows, vector<int>(cols, 0));
    
    // กรอกข้อมูล
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            matrix[i][j] = i * cols + j + 1;
        }
    }
    
    // แสดงผล
    for (const auto& row : matrix) {
        for (int val : row) {
            cout.width(4);
            cout << val;
        }
        cout << endl;
    }
    
    // === Algorithms ===
    cout << "\n=== Vector Algorithms ===" << endl;
    
    vector<int> data = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};
    
    // Sort
    sort(data.begin(), data.end());
    cout << "Sorted: ";
    for (int x : data) cout << x << " ";
    cout << endl;
    
    // Unique (ลบ duplicates หลัง sort)
    auto last = unique(data.begin(), data.end());
    data.erase(last, data.end());
    cout << "Unique: ";
    for (int x : data) cout << x << " ";
    cout << endl;
    
    // Reverse
    reverse(data.begin(), data.end());
    cout << "Reversed: ";
    for (int x : data) cout << x << " ";
    cout << endl;
    
    // Accumulate
    int total = accumulate(data.begin(), data.end(), 0);
    cout << "Sum: " << total << endl;
    
    // Count
    vector<int> v = {1, 2, 2, 3, 2, 4, 2};
    int count2 = count(v.begin(), v.end(), 2);
    cout << "Count of 2: " << count2 << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 43: String Operations ขั้นสูง

```cpp
#include <iostream>
#include <string>
#include <sstream>
#include <algorithm>
#include <vector>
using namespace std;

int main() {
    // === String Methods ===
    cout << "=== String Methods ===" << endl;
    
    string s = "  Hello, World! C++  ";
    
    cout << "Original: \"" << s << "\"" << endl;
    cout << "Length: " << s.length() << endl;
    cout << "Empty: " << boolalpha << s.empty() << endl;
    
    // Trim (C++ ไม่มี built-in trim ต้องเขียนเอง)
    auto trimLeft = [](string str) {
        str.erase(str.begin(), find_if(str.begin(), str.end(), [](unsigned char c) {
            return !isspace(c);
        }));
        return str;
    };
    
    auto trimRight = [](string str) {
        str.erase(find_if(str.rbegin(), str.rend(), [](unsigned char c) {
            return !isspace(c);
        }).base(), str.end());
        return str;
    };
    
    auto trim = [&](string str) {
        return trimLeft(trimRight(str));
    };
    
    string trimmed = trim(s);
    cout << "Trimmed: \"" << trimmed << "\"" << endl;
    
    // Upper / Lower case
    string upper = trimmed;
    transform(upper.begin(), upper.end(), upper.begin(), ::toupper);
    
    string lower = trimmed;
    transform(lower.begin(), lower.end(), lower.begin(), ::tolower);
    
    cout << "Upper: " << upper << endl;
    cout << "Lower: " << lower << endl;
    
    // === Find and Replace ===
    cout << "\n=== Find and Replace ===" << endl;
    
    string text = "ฉันชอบ C++ เพราะ C++ เร็วมาก";
    cout << "Original: " << text << endl;
    
    // Find
    size_t pos = text.find("C++");
    cout << "Find 'C++' at: " << pos << endl;
    
    // Replace all occurrences
    string replaceAll = text;
    string from = "C++";
    string to = "Python";
    
    size_t start = 0;
    while ((start = replaceAll.find(from, start)) != string::npos) {
        replaceAll.replace(start, from.length(), to);
        start += to.length();
    }
    cout << "Replace all: " << replaceAll << endl;
    
    // === Split String ===
    cout << "\n=== Split String ===" << endl;
    
    auto split = [](const string& str, char delimiter) {
        vector<string> tokens;
        string token;
        istringstream tokenStream(str);
        while (getline(tokenStream, token, delimiter)) {
            if (!token.empty()) tokens.push_back(token);
        }
        return tokens;
    };
    
    string csv = "สมชาย,25,กรุงเทพ,วิศวกร";
    vector<string> parts = split(csv, ',');
    
    cout << "CSV: " << csv << endl;
    cout << "Parts:" << endl;
    for (const string& part : parts) {
        cout << "  - " << part << endl;
    }
    
    // === Join String ===
    cout << "\n=== Join String ===" << endl;
    
    auto join = [](const vector<string>& v, const string& sep) {
        string result;
        for (size_t i = 0; i < v.size(); i++) {
            if (i > 0) result += sep;
            result += v[i];
        }
        return result;
    };
    
    vector<string> words = {"Hello", "World", "C++", "Qt"};
    cout << "Join with ', ': " << join(words, ", ") << endl;
    cout << "Join with ' | ': " << join(words, " | ") << endl;
    
    // === String Stream ===
    cout << "\n=== String Stream ===" << endl;
    
    // ostringstream: สร้าง string
    ostringstream oss;
    oss << "ชื่อ: " << "สมชาย" << ", อายุ: " << 25 << ", คะแนน: " << 95.5;
    string info = oss.str();
    cout << info << endl;
    
    // istringstream: แยกข้อมูล
    string data = "100 3.14 Hello true";
    istringstream iss(data);
    
    int i;
    double d;
    string str;
    bool b;
    
    iss >> i >> d >> str >> b;
    cout << "int: " << i << ", double: " << d 
         << ", string: " << str << ", bool: " << b << endl;
    
    // === String Operations ===
    cout << "\n=== String Operations ===" << endl;
    
    string s1 = "Hello";
    string s2 = "World";
    
    // Concatenate
    string concat = s1 + " " + s2;
    cout << "Concat: " << concat << endl;
    
    // Compare
    cout << "s1 == s2: " << (s1 == s2) << endl;
    cout << "s1 < s2: " << (s1 < s2) << endl;
    cout << "compare: " << s1.compare(s2) << endl;  // < 0 ถ้า s1 < s2
    
    // Starts/Ends with (C++20)
    // cout << s1.starts_with("He") << endl;
    // cout << s1.ends_with("lo") << endl;
    
    // Manual starts_with / ends_with
    auto startsWith = [](const string& str, const string& prefix) {
        return str.find(prefix) == 0;
    };
    
    auto endsWith = [](const string& str, const string& suffix) {
        if (str.length() < suffix.length()) return false;
        return str.substr(str.length() - suffix.length()) == suffix;
    };
    
    cout << "startsWith 'He': " << startsWith(s1, "He") << endl;
    cout << "endsWith 'lo': " << endsWith(s1, "lo") << endl;
    
    // === Palindrome Check ===
    cout << "\n=== Palindrome Check ===" << endl;
    
    auto isPalindrome = [](string s) {
        transform(s.begin(), s.end(), s.begin(), ::tolower);
        // ลบช่องว่างและเครื่องหมาย
        s.erase(remove_if(s.begin(), s.end(), [](char c) {
            return !isalnum(c);
        }), s.end());
        
        string rev = s;
        reverse(rev.begin(), rev.end());
        return s == rev;
    };
    
    vector<string> tests = {"racecar", "hello", "A man a plan a canal Panama", "level"};
    for (const string& test : tests) {
        cout << "\"" << test << "\": " << (isPalindrome(test) ? "palindrome" : "not palindrome") << endl;
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 44: Pointers พื้นฐาน

```cpp
#include <iostream>
using namespace std;

int main() {
    // === Pointer Basics ===
    cout << "=== Pointer Basics ===" << endl;
    
    int value = 42;
    int* ptr = &value;  // ptr เก็บ address ของ value
    
    cout << "value = " << value << endl;
    cout << "address of value = " << &value << endl;
    cout << "ptr = " << ptr << endl;         // address
    cout << "*ptr = " << *ptr << endl;       // dereference: ค่าที่ ptr ชี้ไป
    cout << "address of ptr = " << &ptr << endl;
    
    // เปลี่ยนค่าผ่าน pointer
    *ptr = 100;
    cout << "\nหลัง *ptr = 100:" << endl;
    cout << "value = " << value << endl;    // 100!
    cout << "*ptr = " << *ptr << endl;      // 100
    
    // === Pointer Types ===
    cout << "\n=== Pointer Types ===" << endl;
    
    int* intPtr;
    double* dblPtr;
    char* charPtr;
    bool* boolPtr;
    void* voidPtr;  // สามารถชี้ไปหา type ใดก็ได้
    
    int i = 10;
    double d = 3.14;
    char c = 'A';
    
    intPtr = &i;
    dblPtr = &d;
    charPtr = &c;
    
    cout << "*intPtr = " << *intPtr << endl;
    cout << "*dblPtr = " << *dblPtr << endl;
    cout << "*charPtr = " << *charPtr << endl;
    
    // Void pointer
    voidPtr = &i;
    cout << "*((int*)voidPtr) = " << *(static_cast<int*>(voidPtr)) << endl;
    
    // === Null Pointer ===
    cout << "\n=== Null Pointer ===" << endl;
    
    int* nullPtr = nullptr;  // C++11
    int* nullPtr2 = NULL;    // เก่า
    int* nullPtr3 = 0;       // เก่า
    
    if (nullPtr == nullptr) {
        cout << "nullPtr เป็น nullptr" << endl;
    }
    
    // ตรวจสอบก่อน dereference
    if (intPtr != nullptr) {
        cout << "*intPtr = " << *intPtr << endl;
    }
    
    // === Pointer Arithmetic ===
    cout << "\n=== Pointer Arithmetic ===" << endl;
    
    int arr[] = {10, 20, 30, 40, 50};
    int* p = arr;  // ชี้ไปที่ตัวแรก
    
    cout << "p = " << *p << endl;       // 10
    cout << "p+1 = " << *(p+1) << endl; // 20
    cout << "p+2 = " << *(p+2) << endl; // 30
    
    // Increment pointer
    p++;
    cout << "*p (after ++) = " << *p << endl;  // 20
    
    p += 2;
    cout << "*p (after +=2) = " << *p << endl;  // 40
    
    // ความแตกต่างระหว่าง pointer
    int* p1 = &arr[0];
    int* p2 = &arr[4];
    cout << "p2 - p1 = " << (p2 - p1) << endl;  // 4
    
    // วนซ้ำด้วย pointer
    cout << "\nวนซ้ำด้วย pointer: ";
    for (int* ptr = arr; ptr < arr + 5; ptr++) {
        cout << *ptr << " ";
    }
    cout << endl;
    
    // === Pointer กับ const ===
    cout << "\n=== Pointer กับ const ===" << endl;
    
    int x = 10, y = 20;
    
    // pointer to const: ค่าที่ชี้ไปเปลี่ยนไม่ได้
    const int* cp = &x;
    // *cp = 100;  // Error!
    cp = &y;       // ชี้ pointer ได้
    cout << "*cp = " << *cp << endl;  // 20
    
    // const pointer: pointer เองเปลี่ยนไม่ได้
    int* const pc = &x;
    *pc = 100;     // เปลี่ยนค่าได้
    // pc = &y;    // Error! pointer เปลี่ยนไม่ได้
    cout << "x = " << x << endl;  // 100
    
    // const pointer to const: ทั้ง pointer และค่าเปลี่ยนไม่ได้
    const int* const cpc = &y;
    // *cpc = 200;  // Error!
    // cpc = &x;    // Error!
    cout << "*cpc = " << *cpc << endl;  // 20
    
    return 0;
}
```

---

## ขั้นตอนที่ 45: Pointers ขั้นสูง - Dynamic Memory

```cpp
#include <iostream>
using namespace std;

int main() {
    // === Dynamic Memory Allocation ===
    cout << "=== Dynamic Memory (new/delete) ===" << endl;
    
    // Allocate single variable
    int* ptr = new int(42);
    cout << "*ptr = " << *ptr << endl;
    delete ptr;  // ต้อง free หน่วยความจำ!
    ptr = nullptr;  // ป้องกัน dangling pointer
    
    // Allocate array
    int n = 5;
    int* arr = new int[n];
    
    for (int i = 0; i < n; i++) {
        arr[i] = (i + 1) * 10;
    }
    
    cout << "Dynamic array: ";
    for (int i = 0; i < n; i++) {
        cout << arr[i] << " ";
    }
    cout << endl;
    
    delete[] arr;  // ต้องใช้ delete[] สำหรับ array!
    arr = nullptr;
    
    // === Dynamic 2D Array ===
    cout << "\n=== Dynamic 2D Array ===" << endl;
    
    int rows = 3, cols = 4;
    
    // Allocate
    int** matrix = new int*[rows];
    for (int i = 0; i < rows; i++) {
        matrix[i] = new int[cols];
    }
    
    // กรอกข้อมูล
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            matrix[i][j] = i * cols + j + 1;
        }
    }
    
    // แสดงผล
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            cout.width(4);
            cout << matrix[i][j];
        }
        cout << endl;
    }
    
    // Deallocate
    for (int i = 0; i < rows; i++) {
        delete[] matrix[i];
    }
    delete[] matrix;
    matrix = nullptr;
    
    // === Memory Leaks ===
    cout << "\n=== Common Pointer Problems ===" << endl;
    
    // Problem 1: Memory Leak (ไม่ delete)
    // int* leak = new int(100);
    // // ลืม delete leak; -> Memory Leak!
    
    // Problem 2: Dangling Pointer
    int* dangling = new int(50);
    delete dangling;
    // dangling = nullptr;  // ถ้าไม่ทำ:
    // *dangling = 100;  // Undefined Behavior!
    
    // Problem 3: Double Delete
    int* dd = new int(30);
    delete dd;
    // delete dd;  // Error: Double delete!
    dd = nullptr;
    
    // Problem 4: Buffer Overflow
    int* buf = new int[5];
    // buf[10] = 100;  // Undefined Behavior!
    delete[] buf;
    
    cout << "Pointer problems demonstrated (with comments)" << endl;
    
    // === Smart Pointers Preview (C++11) ===
    cout << "\n=== Smart Pointers Preview ===" << endl;
    #include <memory>
    
    // unique_ptr: ถ้าออกจาก scope จะ delete อัตโนมัติ
    unique_ptr<int> uptr = make_unique<int>(42);
    cout << "*uptr = " << *uptr << endl;
    // ไม่ต้อง delete!
    
    // shared_ptr: นับการอ้างอิง
    shared_ptr<int> sptr1 = make_shared<int>(100);
    shared_ptr<int> sptr2 = sptr1;  // share ได้
    cout << "*sptr1 = " << *sptr1 << endl;
    cout << "use_count = " << sptr1.use_count() << endl;  // 2
    
    // weak_ptr: ไม่นับ reference count
    weak_ptr<int> wptr = sptr1;
    cout << "weak use_count = " << sptr1.use_count() << endl;  // ยังเป็น 2
    
    return 0;
}
```

---

## ขั้นตอนที่ 46-55: โปรแกรมรวม Arrays, Strings, Pointers

```cpp
// ============================================
// โปรแกรม: Library Management System
// ใช้ Arrays, Strings, Pointers, Vectors
// ============================================
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <iomanip>
#include <sstream>
using namespace std;

// === Struct ===
struct Book {
    int id;
    string title;
    string author;
    string isbn;
    int year;
    bool available;
    double price;
    
    string toString() const {
        ostringstream oss;
        oss << "ID: " << id
            << " | " << left << setw(30) << title
            << " | " << setw(20) << author
            << " | " << year
            << " | " << (available ? "พร้อมยืม" : "ถูกยืมไปแล้ว")
            << " | " << fixed << setprecision(2) << price << " บาท";
        return oss.str();
    }
};

// === ฟังก์ชัน ===
void displayBooks(const vector<Book>& books) {
    cout << string(100, '-') << endl;
    cout << setw(5) << "ID"
         << setw(32) << left << " ชื่อหนังสือ"
         << setw(22) << "ผู้แต่ง"
         << setw(6) << right << "ปี"
         << setw(15) << "สถานะ"
         << setw(10) << "ราคา" << endl;
    cout << string(100, '-') << endl;
    
    for (const Book& book : books) {
        cout << setw(5) << book.id
             << "  " << left << setw(30) << book.title
             << setw(20) << book.author
             << right << setw(6) << book.year
             << setw(15) << (book.available ? "พร้อมยืม" : "ถูกยืมแล้ว")
             << setw(10) << fixed << setprecision(2) << book.price << endl;
    }
    cout << string(100, '-') << endl;
    cout << "รวมทั้งหมด: " << books.size() << " เล่ม" << endl;
}

Book* findBookById(vector<Book>& books, int id) {
    for (Book& book : books) {
        if (book.id == id) return &book;
    }
    return nullptr;
}

vector<Book> searchByTitle(const vector<Book>& books, const string& keyword) {
    vector<Book> results;
    string kwLower = keyword;
    transform(kwLower.begin(), kwLower.end(), kwLower.begin(), ::tolower);
    
    for (const Book& book : books) {
        string titleLower = book.title;
        transform(titleLower.begin(), titleLower.end(), titleLower.begin(), ::tolower);
        
        if (titleLower.find(kwLower) != string::npos) {
            results.push_back(book);
        }
    }
    return results;
}

vector<Book> filterAvailable(const vector<Book>& books) {
    vector<Book> available;
    copy_if(books.begin(), books.end(), back_inserter(available),
            [](const Book& b) { return b.available; });
    return available;
}

void sortBooksByTitle(vector<Book>& books) {
    sort(books.begin(), books.end(), [](const Book& a, const Book& b) {
        return a.title < b.title;
    });
}

void sortBooksByYear(vector<Book>& books) {
    sort(books.begin(), books.end(), [](const Book& a, const Book& b) {
        return a.year > b.year;  // ใหม่สุดก่อน
    });
}

double calculateTotalValue(const vector<Book>& books) {
    double total = 0;
    for (const Book& book : books) total += book.price;
    return total;
}

int main() {
    cout << "===============================" << endl;
    cout << "     ระบบจัดการห้องสมุด        " << endl;
    cout << "===============================" << endl;
    
    // ข้อมูลเริ่มต้น
    vector<Book> library = {
        {1, "Clean Code", "Robert C. Martin", "978-0132350884", 2008, true, 850.00},
        {2, "The Pragmatic Programmer", "David Thomas", "978-0135957059", 2019, true, 920.00},
        {3, "Design Patterns", "Gang of Four", "978-0201633610", 1994, false, 980.00},
        {4, "Introduction to Algorithms", "Thomas H. Cormen", "978-0262046305", 2022, true, 1250.00},
        {5, "C++ Primer", "Stanley B. Lippman", "978-0321714114", 2012, true, 780.00},
        {6, "Effective Modern C++", "Scott Meyers", "978-1491903995", 2014, false, 850.00},
        {7, "The C++ Programming Language", "Bjarne Stroustrup", "978-0321958327", 2013, true, 950.00},
        {8, "Qt5 C++ GUI", "Lee Zhi Eng", "978-1784397883", 2016, true, 650.00},
    };
    
    int choice;
    do {
        cout << "\n--- เมนู ---" << endl;
        cout << "1. แสดงหนังสือทั้งหมด" << endl;
        cout << "2. ค้นหาหนังสือ" << endl;
        cout << "3. ยืมหนังสือ" << endl;
        cout << "4. คืนหนังสือ" << endl;
        cout << "5. เรียงลำดับตามชื่อ" << endl;
        cout << "6. เรียงลำดับตามปี" << endl;
        cout << "7. หนังสือที่พร้อมยืม" << endl;
        cout << "8. สถิติ" << endl;
        cout << "0. ออก" << endl;
        cout << "เลือก: ";
        cin >> choice;
        
        switch (choice) {
            case 1:
                displayBooks(library);
                break;
                
            case 2: {
                cin.ignore();
                string keyword;
                cout << "ค้นหาด้วยชื่อหนังสือ: ";
                getline(cin, keyword);
                
                auto results = searchByTitle(library, keyword);
                cout << "\nผลการค้นหา '" << keyword << "': " << results.size() << " รายการ" << endl;
                if (!results.empty()) displayBooks(results);
                break;
            }
                
            case 3: {
                int id;
                cout << "กรอก ID หนังสือที่ต้องการยืม: ";
                cin >> id;
                
                Book* book = findBookById(library, id);
                if (!book) {
                    cout << "ไม่พบหนังสือ ID: " << id << endl;
                } else if (!book->available) {
                    cout << "หนังสือ '" << book->title << "' ถูกยืมไปแล้ว!" << endl;
                } else {
                    book->available = false;
                    cout << "ยืมหนังสือ '" << book->title << "' สำเร็จ!" << endl;
                }
                break;
            }
                
            case 4: {
                int id;
                cout << "กรอก ID หนังสือที่ต้องการคืน: ";
                cin >> id;
                
                Book* book = findBookById(library, id);
                if (!book) {
                    cout << "ไม่พบหนังสือ ID: " << id << endl;
                } else if (book->available) {
                    cout << "หนังสือ '" << book->title << "' ยังไม่ถูกยืม!" << endl;
                } else {
                    book->available = true;
                    cout << "คืนหนังสือ '" << book->title << "' สำเร็จ!" << endl;
                }
                break;
            }
                
            case 5:
                sortBooksByTitle(library);
                cout << "เรียงตามชื่อแล้ว" << endl;
                displayBooks(library);
                break;
                
            case 6:
                sortBooksByYear(library);
                cout << "เรียงตามปีแล้ว" << endl;
                displayBooks(library);
                break;
                
            case 7: {
                auto available = filterAvailable(library);
                cout << "หนังสือที่พร้อมยืม: " << available.size() << " เล่ม" << endl;
                displayBooks(available);
                break;
            }
                
            case 8: {
                int totalBooks = library.size();
                int availableCount = count_if(library.begin(), library.end(),
                    [](const Book& b) { return b.available; });
                int borrowedCount = totalBooks - availableCount;
                double totalValue = calculateTotalValue(library);
                
                cout << "\n=== สถิติ ===" << endl;
                cout << "หนังสือทั้งหมด: " << totalBooks << " เล่ม" << endl;
                cout << "พร้อมยืม: " << availableCount << " เล่ม" << endl;
                cout << "ถูกยืม: " << borrowedCount << " เล่ม" << endl;
                cout << "มูลค่ารวม: " << fixed << setprecision(2) << totalValue << " บาท" << endl;
                cout << "มูลค่าเฉลี่ย: " << totalValue / totalBooks << " บาท/เล่ม" << endl;
                break;
            }
                
            case 0:
                cout << "ขอบคุณที่ใช้บริการ!" << endl;
                break;
                
            default:
                cout << "ตัวเลือกไม่ถูกต้อง" << endl;
        }
        
    } while (choice != 0);
    
    return 0;
}
```

---

## สรุป Part 005

ใน Part นี้คุณได้เรียนรู้:

1. ✅ C-style Arrays และ std::array
2. ✅ Vector - Dynamic Array
3. ✅ String Operations ขั้นสูง (trim, split, join, find & replace)
4. ✅ Pointer Basics (declaration, dereference, arithmetic)
5. ✅ Pointer กับ const
6. ✅ Dynamic Memory Allocation (new/delete)
7. ✅ 2D Arrays และ Dynamic 2D Arrays
8. ✅ Smart Pointers Preview
9. ✅ โปรแกรม Library Management System

---

## แบบฝึกหัด Part 005

**ฝึกหัดที่ 1:** เขียนฟังก์ชัน `reverseArray` ที่กลับอาร์เรย์โดยใช้ Pointer

**ฝึกหัดที่ 2:** เขียนโปรแกรม Stack ด้วย Dynamic Array

**ฝึกหัดที่ 3:** เขียนฟังก์ชัน String Tokenizer ที่รองรับ Multiple Delimiters

---

⬅️ **ก่อนหน้า:** [Part 004](part004.md) | ➡️ **ต่อไป:** [Part 006: OOP](part006.md)
