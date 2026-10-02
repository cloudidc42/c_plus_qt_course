# Part 001: แนะนำ C++ และการติดตั้ง

## ขั้นตอนที่ 1-10: เริ่มต้นกับ C++

---

## ขั้นตอนที่ 1: C++ คืออะไร?

C++ เป็นภาษาโปรแกรมมิ่งที่พัฒนาโดย **Bjarne Stroustrup** ที่ Bell Labs ในช่วงต้นทศวรรษ 1980 โดยเป็นการขยายความสามารถของภาษา C ให้รองรับการเขียนโปรแกรมเชิงวัตถุ (Object-Oriented Programming)

### ประวัติโดยย่อ

```
1972 - ภาษา C ถูกพัฒนาโดย Dennis Ritchie
1979 - Bjarne Stroustrup เริ่มพัฒนา "C with Classes"
1983 - เปลี่ยนชื่อเป็น C++
1985 - C++ เวอร์ชันแรกออกวางจำหน่าย
1998 - C++98 (มาตรฐานแรก ISO/IEC 14882:1998)
2003 - C++03 (แก้ไขบัก)
2011 - C++11 (การเปลี่ยนแปลงครั้งใหญ่)
2014 - C++14 (ปรับปรุงเล็กน้อย)
2017 - C++17 (ฟีเจอร์ใหม่หลายอย่าง)
2020 - C++20 (Concepts, Ranges, Coroutines)
2023 - C++23 (ล่าสุด)
```

### ทำไมต้องเรียน C++?

**1. ประสิทธิภาพสูง (High Performance)**
- ทำงานใกล้เคียงกับ Hardware
- ควบคุม Memory ได้โดยตรง
- ไม่มี Garbage Collector ทำให้ Response Time คงที่

**2. ใช้งานได้หลากหลาย (Versatile)**
- ระบบปฏิบัติการ (OS): Windows, Linux, macOS
- เกม: Unreal Engine ใช้ C++
- ฐานข้อมูล: MySQL, MongoDB เขียนด้วย C++
- เบราว์เซอร์: Chrome, Firefox มีส่วนที่เขียนด้วย C++
- ระบบ Embedded: อุปกรณ์ IoT, Robotics
- การเงิน: High-Frequency Trading Systems

**3. ข้ามแพลตฟอร์ม (Cross-Platform)**
- เขียนครั้งเดียว Compile ได้บนหลายระบบปฏิบัติการ

**4. ชุมชนขนาดใหญ่ (Large Community)**
- Library และ Framework มากมาย
- ทรัพยากรการเรียนรู้มากมาย

---

## ขั้นตอนที่ 2: ความแตกต่างระหว่าง C++ และภาษาอื่น

### เปรียบเทียบกับ Python

```
| คุณสมบัติ       | C++           | Python        |
|----------------|---------------|---------------|
| ความเร็ว        | เร็วมาก        | ช้ากว่า 10-100x|
| Syntax         | ซับซ้อน        | ง่าย           |
| Memory Control | Manual        | Automatic     |
| Type System    | Static        | Dynamic       |
| ใช้งาน         | Systems, Games| Scripts, ML   |
```

### เปรียบเทียบกับ Java

```
| คุณสมบัติ       | C++           | Java          |
|----------------|---------------|---------------|
| Memory         | Manual        | GC (Auto)     |
| Platform       | Native Binary | JVM           |
| OOP            | Multi-paradigm| Pure OOP      |
| ความเร็ว        | เร็วกว่า       | ช้ากว่า       |
| Pointers       | มี            | ไม่มี (Ref)   |
```

---

## ขั้นตอนที่ 3: การติดตั้ง C++ Compiler บน Windows

### วิธีที่ 1: MinGW-w64 (แนะนำสำหรับ Windows)

**ขั้นตอนการติดตั้ง:**

1. ดาวน์โหลด MSYS2 จาก https://www.msys2.org/
2. รัน Installer และติดตั้งที่ `C:\msys64`
3. เปิด MSYS2 UCRT64 Terminal
4. รันคำสั่ง:

```bash
# อัปเดต Package Database
pacman -Syu

# ติดตั้ง MinGW-w64 Toolchain
pacman -S mingw-w64-ucrt-x86_64-gcc
pacman -S mingw-w64-ucrt-x86_64-gdb

# ตรวจสอบการติดตั้ง
gcc --version
g++ --version
```

5. เพิ่ม Path ใน System Environment Variables:
   - เปิด `System Properties` > `Advanced` > `Environment Variables`
   - เพิ่ม `C:\msys64\ucrt64\bin` ใน `Path`

**ตรวจสอบ:**
```cmd
# เปิด Command Prompt
g++ --version
# ควรแสดง: g++ (Rev1, Built by MSYS2 project) 13.x.x
```

### วิธีที่ 2: Microsoft Visual C++ (MSVC)

1. ดาวน์โหลด Visual Studio Community (ฟรี) จาก https://visualstudio.microsoft.com/
2. เลือก Workload: "Desktop development with C++"
3. ติดตั้งและรอ

---

## ขั้นตอนที่ 4: การติดตั้ง C++ Compiler บน macOS

```bash
# วิธีที่ 1: Xcode Command Line Tools (แนะนำ)
xcode-select --install

# ตรวจสอบ
clang++ --version

# วิธีที่ 2: Homebrew + GCC
brew install gcc

# ตรวจสอบ
g++-13 --version
```

---

## ขั้นตอนที่ 5: การติดตั้ง C++ Compiler บน Linux (Ubuntu/Debian)

```bash
# อัปเดต Package List
sudo apt update

# ติดตั้ง GCC/G++
sudo apt install build-essential

# ติดตั้ง GDB (Debugger)
sudo apt install gdb

# ตรวจสอบ
gcc --version
g++ --version
gdb --version

# ติดตั้ง CMake (Build System)
sudo apt install cmake

# ตรวจสอบ CMake
cmake --version
```

---

## ขั้นตอนที่ 6: การติดตั้ง Qt Framework

Qt Framework เป็น Cross-Platform Application Framework ที่ใช้กับ C++ เป็นหลัก

### ดาวน์โหลดและติดตั้ง Qt

**ขั้นตอน:**

1. ไปที่ https://www.qt.io/download-open-source
2. คลิก "Download the Qt Online Installer"
3. รัน Installer
4. สร้างบัญชี Qt ฟรี (จำเป็น)
5. เลือก Components:

```
Qt 6.7.x
  ├── MSVC 2022 64-bit (Windows)
  ├── MinGW 11.2.0 64-bit (Windows)
  ├── macOS (macOS)
  ├── gcc 64-bit (Linux)
  └── Qt Creator IDE
  
Qt Creator X.X.X
  └── Qt Creator (IDE)
  
Developer and Designer Tools
  ├── Qt Creator X.X.X CDB Debugger Support
  ├── Debugging Tools for Windows
  ├── Qt Design Studio X.X.X
  └── MinGW 11.2.0 64-bit
```

---

## ขั้นตอนที่ 7: การติดตั้ง Qt บน Linux (Ubuntu)

```bash
# วิธีที่ 1: ผ่าน Package Manager (เร็ว แต่อาจได้เวอร์ชันเก่า)
sudo apt install qt6-base-dev
sudo apt install qt6-tools-dev
sudo apt install qtcreator

# หรือ Qt5
sudo apt install qt5-default
sudo apt install qtcreator

# ตรวจสอบ
qmake --version
qtcreator --version
```

---

## ขั้นตอนที่ 8: การตั้งค่า IDE - Qt Creator

Qt Creator เป็น IDE ที่ออกแบบมาสำหรับการพัฒนา Qt โดยเฉพาะ

### การตั้งค่าเบื้องต้น

1. เปิด Qt Creator
2. ไปที่ `Edit` > `Preferences` (Windows/Linux) หรือ `Qt Creator` > `Preferences` (macOS)
3. ตั้งค่า:
   - **Kits**: ตรวจสอบว่า Qt Version และ Compiler ถูกต้อง
   - **Text Editor**: เลือก Font size (แนะนำ 13-14pt)
   - **Build & Run**: ตั้งค่า Default Build Directory

### Shortcut สำคัญใน Qt Creator

```
Ctrl+N          - สร้างไฟล์ใหม่
Ctrl+Shift+N    - สร้างโปรเจกต์ใหม่
Ctrl+O          - เปิดไฟล์
Ctrl+S          - บันทึก
Ctrl+B          - Build
Ctrl+R          - Build & Run
F5              - Debug
F9              - Toggle Breakpoint
Ctrl+F          - Find
Ctrl+H          - Find & Replace
F1              - Help
Ctrl+/          - Toggle Comment
Ctrl+Space      - Auto Complete
Alt+Enter       - Quick Fix
Ctrl+K          - Locator (หาไฟล์, ฟังก์ชัน)
```

---

## ขั้นตอนที่ 9: โปรแกรมแรก - Hello World

เปิด Qt Creator และทำตามขั้นตอน:

### สร้างโปรเจกต์ C++ ใหม่

1. คลิก `New Project...`
2. เลือก `Non-Qt Project` > `Plain C++ Application`
3. ตั้งชื่อโปรเจกต์: `HelloWorld`
4. เลือก Build System: `CMake`
5. คลิก `Next` และ `Finish`

### โค้ด Hello World

**ไฟล์: main.cpp**

```cpp
#include <iostream>

int main() {
    std::cout << "Hello, World!" << std::endl;
    return 0;
}
```

**อธิบายทีละบรรทัด:**

```cpp
#include <iostream>
```
- `#include` คือ Preprocessor Directive ที่บอกให้ Compiler นำไฟล์ Header มาใช้
- `<iostream>` คือ Header ที่มีฟังก์ชัน Input/Output เช่น `cout`, `cin`

```cpp
int main() {
```
- `int` คือ Return Type ของฟังก์ชัน (คืนค่าเป็น Integer)
- `main` คือชื่อฟังก์ชันหลัก ทุกโปรแกรม C++ ต้องมีฟังก์ชันนี้
- `()` คือ Parameter List (ว่างแปลว่าไม่รับ Parameter)

```cpp
    std::cout << "Hello, World!" << std::endl;
```
- `std::cout` คือ Standard Output Stream (จอแสดงผล)
- `<<` คือ Stream Insertion Operator
- `"Hello, World!"` คือ String Literal
- `std::endl` คือ End of Line (ขึ้นบรรทัดใหม่)

```cpp
    return 0;
```
- คืนค่า 0 กลับไปให้ Operating System (0 แปลว่าโปรแกรมทำงานสำเร็จ)

### Compile และ Run ด้วย Terminal

```bash
# Compile
g++ -o hello main.cpp

# Run บน Windows
hello.exe

# Run บน Linux/macOS
./hello

# Output:
# Hello, World!
```

---

## ขั้นตอนที่ 10: โครงสร้างโปรแกรม C++ เบื้องต้น

```cpp
// ============================================
// โครงสร้างโปรแกรม C++ พื้นฐาน
// ============================================

// 1. Preprocessor Directives
#include <iostream>    // I/O operations
#include <string>      // String operations
#include <vector>      // Dynamic array

// 2. Namespace Declaration (Optional)
using namespace std;

// 3. Function Declarations (Prototypes)
void greetUser(string name);
int addNumbers(int a, int b);

// 4. Main Function
int main() {
    // 5. Variable Declarations
    string userName = "ผู้เรียน";
    int num1 = 10;
    int num2 = 20;
    
    // 6. Function Calls
    greetUser(userName);
    
    int result = addNumbers(num1, num2);
    cout << "ผลรวม: " << result << endl;
    
    return 0;  // 7. Return Statement
}

// 8. Function Definitions
void greetUser(string name) {
    cout << "สวัสดี, " << name << "!" << endl;
    cout << "ยินดีต้อนรับสู่หลักสูตร C++ และ Qt" << endl;
}

int addNumbers(int a, int b) {
    return a + b;
}
```

**Output:**
```
สวัสดี, ผู้เรียน!
ยินดีต้อนรับสู่หลักสูตร C++ และ Qt
ผลรวม: 30
```

### ส่วนประกอบหลักของโปรแกรม C++

```
โปรแกรม C++
├── Preprocessor Directives (#include, #define)
├── Global Declarations (ตัวแปร, ฟังก์ชัน ระดับ Global)
├── main() Function (จุดเริ่มต้นโปรแกรม)
│   ├── Local Variable Declarations
│   ├── Statements (คำสั่ง)
│   └── return statement
└── Other Function Definitions
```

---

## ตัวอย่างโปรแกรมแรกด้วย Qt

```cpp
// ============================================
// โปรแกรม Qt แรก: Hello Qt
// ============================================

#include <QApplication>
#include <QLabel>

int main(int argc, char *argv[]) {
    // สร้าง Qt Application
    QApplication app(argc, argv);
    
    // สร้าง Label Widget แสดงข้อความ
    QLabel label("สวัสดี Qt!");
    label.setWindowTitle("โปรแกรม Qt แรก");
    label.resize(300, 100);
    label.show();
    
    // เริ่ม Event Loop
    return app.exec();
}
```

**การ Compile:**
```bash
# สร้าง CMakeLists.txt
cmake_minimum_required(VERSION 3.16)
project(HelloQt)

find_package(Qt6 REQUIRED COMPONENTS Widgets)

qt_add_executable(HelloQt main.cpp)

target_link_libraries(HelloQt PRIVATE Qt6::Widgets)
```

---

## สรุป Part 001

ใน Part นี้คุณได้เรียนรู้:

1. ✅ C++ คืออะไรและประวัติความเป็นมา
2. ✅ เปรียบเทียบ C++ กับภาษาอื่น
3. ✅ การติดตั้ง Compiler บน Windows, macOS, Linux
4. ✅ การติดตั้ง Qt Framework
5. ✅ การตั้งค่า Qt Creator IDE
6. ✅ เขียนและรันโปรแกรม Hello World แรก
7. ✅ โครงสร้างโปรแกรม C++ เบื้องต้น
8. ✅ ตัวอย่างโปรแกรม Qt เบื้องต้น

---

## แบบฝึกหัด Part 001

**ฝึกหัดที่ 1:** ติดตั้ง C++ Compiler และ Qt บนเครื่องของคุณ แล้วตรวจสอบด้วยคำสั่ง `g++ --version`

**ฝึกหัดที่ 2:** เขียนโปรแกรมที่แสดงชื่อของคุณออกทางหน้าจอ

**ฝึกหัดที่ 3:** แก้ไขโปรแกรม Hello World ให้แสดงข้อความภาษาไทยได้ถูกต้อง

**ฝึกหัดที่ 4:** ลองเขียนโปรแกรม Qt ที่แสดง Label ที่มีข้อความ "Hello Qt!"

---

➡️ **ต่อไป:** [Part 002: ตัวแปร ชนิดข้อมูล และตัวดำเนินการ](part002.md)
