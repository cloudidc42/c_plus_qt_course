# Part 020: Qt Testing (QTest)

## ขั้นตอนที่ 266-280

---

## ขั้นตอนที่ 266: Qt Testing Overview

```qmake
# เพิ่มใน .pro
QT += testlib
```

```
QTest Features:
  ├── Unit Testing
  ├── Data-Driven Testing (QFETCH)
  ├── Benchmarking (QBENCHMARK)
  ├── GUI Testing (QTest::mouseClick, QTest::keyClick)
  ├── Async Testing (QSignalSpy)
  └── Integration with CTest/CMake
```

---

## ขั้นตอนที่ 267: Basic Unit Tests

```cpp
// === testbasic.h ===
#pragma once
#include <QTest>
#include <QString>
#include <QList>
#include <algorithm>

// Class ที่จะ test
class Calculator {
public:
    static double add(double a, double b) { return a + b; }
    static double subtract(double a, double b) { return a - b; }
    static double multiply(double a, double b) { return a * b; }
    
    static double divide(double a, double b) {
        if (b == 0) throw std::invalid_argument("Division by zero");
        return a / b;
    }
    
    static bool isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++) {
            if (n % i == 0) return false;
        }
        return true;
    }
    
    static double average(const QList<double>& nums) {
        if (nums.isEmpty()) return 0;
        double sum = 0;
        for (double n : nums) sum += n;
        return sum / nums.size();
    }
    
    static QList<int> fibonacci(int n) {
        QList<int> fib;
        if (n <= 0) return fib;
        fib << 0;
        if (n == 1) return fib;
        fib << 1;
        for (int i = 2; i < n; i++) {
            fib << fib[i-1] + fib[i-2];
        }
        return fib;
    }
};

class StringUtils {
public:
    static QString reverse(const QString& s) {
        QString r = s;
        std::reverse(r.begin(), r.end());
        return r;
    }
    
    static bool isPalindrome(const QString& s) {
        QString clean = s.toLower().remove(QRegularExpression("[^a-z0-9]"));
        return clean == reverse(clean);
    }
    
    static int countWords(const QString& text) {
        if (text.trimmed().isEmpty()) return 0;
        return text.trimmed().split(QRegularExpression("\\s+")).size();
    }
    
    static QString capitalize(const QString& s) {
        if (s.isEmpty()) return s;
        return s[0].toUpper() + s.mid(1).toLower();
    }
};

// === testbasic.cpp ===
class TestCalculator : public QObject {
    Q_OBJECT
    
private slots:
    // called before each test function
    void init() {
        // Setup
    }
    
    // called after each test function
    void cleanup() {
        // Teardown
    }
    
    // called before all tests
    void initTestCase() {
        qDebug() << "Starting Calculator Tests";
    }
    
    // called after all tests
    void cleanupTestCase() {
        qDebug() << "Calculator Tests Done";
    }
    
    // === Test functions (ชื่อต้องขึ้นต้นด้วย test) ===
    void test_add() {
        QCOMPARE(Calculator::add(2, 3), 5.0);
        QCOMPARE(Calculator::add(-1, 1), 0.0);
        QCOMPARE(Calculator::add(0, 0), 0.0);
        QCOMPARE(Calculator::add(1.5, 2.5), 4.0);
    }
    
    void test_subtract() {
        QCOMPARE(Calculator::subtract(5, 3), 2.0);
        QCOMPARE(Calculator::subtract(0, 5), -5.0);
        QCOMPARE(Calculator::subtract(-3, -2), -1.0);
    }
    
    void test_multiply() {
        QCOMPARE(Calculator::multiply(3, 4), 12.0);
        QCOMPARE(Calculator::multiply(-2, 3), -6.0);
        QCOMPARE(Calculator::multiply(0, 100), 0.0);
    }
    
    void test_divide() {
        QCOMPARE(Calculator::divide(10, 2), 5.0);
        QCOMPARE(Calculator::divide(7, 2), 3.5);
        QCOMPARE(Calculator::divide(-6, 3), -2.0);
    }
    
    void test_divide_by_zero() {
        QVERIFY_THROWS_EXCEPTION(std::invalid_argument, 
                                  Calculator::divide(5, 0));
    }
    
    void test_isPrime_data() {
        QTest::addColumn<int>("number");
        QTest::addColumn<bool>("expected");
        
        QTest::newRow("2 is prime")      << 2  << true;
        QTest::newRow("3 is prime")      << 3  << true;
        QTest::newRow("5 is prime")      << 5  << true;
        QTest::newRow("17 is prime")     << 17 << true;
        QTest::newRow("97 is prime")     << 97 << true;
        QTest::newRow("1 not prime")     << 1  << false;
        QTest::newRow("4 not prime")     << 4  << false;
        QTest::newRow("9 not prime")     << 9  << false;
        QTest::newRow("100 not prime")   << 100 << false;
        QTest::newRow("negative not prime") << -5 << false;
    }
    
    void test_isPrime() {
        QFETCH(int, number);
        QFETCH(bool, expected);
        
        QCOMPARE(Calculator::isPrime(number), expected);
    }
    
    void test_average() {
        QCOMPARE(Calculator::average({1, 2, 3, 4, 5}), 3.0);
        QCOMPARE(Calculator::average({10}), 10.0);
        QCOMPARE(Calculator::average({}), 0.0);
        QVERIFY(qAbs(Calculator::average({1.5, 2.5}) - 2.0) < 1e-10);
    }
    
    void test_fibonacci() {
        QCOMPARE(Calculator::fibonacci(0), QList<int>{});
        QCOMPARE(Calculator::fibonacci(1), QList<int>{0});
        QCOMPARE(Calculator::fibonacci(5), QList<int>({0, 1, 1, 2, 3}));
        QCOMPARE(Calculator::fibonacci(8), QList<int>({0, 1, 1, 2, 3, 5, 8, 13}));
    }
};

class TestStringUtils : public QObject {
    Q_OBJECT
    
private slots:
    void test_reverse() {
        QCOMPARE(StringUtils::reverse("hello"), QString("olleh"));
        QCOMPARE(StringUtils::reverse(""),      QString(""));
        QCOMPARE(StringUtils::reverse("a"),     QString("a"));
        QCOMPARE(StringUtils::reverse("racecar"), QString("racecar"));
    }
    
    void test_isPalindrome_data() {
        QTest::addColumn<QString>("input");
        QTest::addColumn<bool>("expected");
        
        QTest::newRow("racecar")                   << "racecar" << true;
        QTest::newRow("A man a plan a canal Panama") << "A man a plan a canal Panama" << true;
        QTest::newRow("hello")                     << "hello" << false;
        QTest::newRow("Was it a car or a cat I saw") << "Was it a car or a cat I saw" << true;
        QTest::newRow("empty")                     << "" << true;
    }
    
    void test_isPalindrome() {
        QFETCH(QString, input);
        QFETCH(bool, expected);
        QCOMPARE(StringUtils::isPalindrome(input), expected);
    }
    
    void test_countWords() {
        QCOMPARE(StringUtils::countWords("hello world"),       2);
        QCOMPARE(StringUtils::countWords("  spaces  "),        1);
        QCOMPARE(StringUtils::countWords("one two three"),     3);
        QCOMPARE(StringUtils::countWords(""),                  0);
        QCOMPARE(StringUtils::countWords("   "),               0);
    }
    
    void test_capitalize() {
        QCOMPARE(StringUtils::capitalize("hello"), QString("Hello"));
        QCOMPARE(StringUtils::capitalize("WORLD"), QString("World"));
        QCOMPARE(StringUtils::capitalize(""),      QString(""));
        QCOMPARE(StringUtils::capitalize("a"),     QString("A"));
    }
    
    // Benchmark
    void bench_isPalindrome() {
        QBENCHMARK {
            StringUtils::isPalindrome("A man a plan a canal Panama");
        }
    }
};
```

---

## ขั้นตอนที่ 268: GUI Testing

```cpp
#include <QTest>
#include <QSignalSpy>
#include <QPushButton>
#include <QLineEdit>
#include <QLabel>

class Counter : public QWidget {
    Q_OBJECT
    
public:
    int count = 0;
    
    QLabel* display;
    QPushButton* incBtn;
    QPushButton* decBtn;
    QLineEdit* input;
    
    Counter(QWidget* parent = nullptr) : QWidget(parent) {
        display = new QLabel("0", this);
        incBtn  = new QPushButton("+", this);
        decBtn  = new QPushButton("-", this);
        input   = new QLineEdit(this);
        
        connect(incBtn, &QPushButton::clicked, [this]() {
            count++;
            display->setText(QString::number(count));
            emit countChanged(count);
        });
        
        connect(decBtn, &QPushButton::clicked, [this]() {
            count--;
            display->setText(QString::number(count));
            emit countChanged(count);
        });
    }
    
signals:
    void countChanged(int count);
};

class TestGui : public QObject {
    Q_OBJECT
    
private slots:
    void test_counter_increment() {
        Counter counter;
        
        // Verify initial state
        QCOMPARE(counter.count, 0);
        QCOMPARE(counter.display->text(), QString("0"));
        
        // Click button
        QTest::mouseClick(counter.incBtn, Qt::LeftButton);
        
        QCOMPARE(counter.count, 1);
        QCOMPARE(counter.display->text(), QString("1"));
        
        // Click 5 more times
        for (int i = 0; i < 5; i++) {
            QTest::mouseClick(counter.incBtn, Qt::LeftButton);
        }
        
        QCOMPARE(counter.count, 6);
    }
    
    void test_counter_decrement() {
        Counter counter;
        counter.count = 3;
        counter.display->setText("3");
        
        QTest::mouseClick(counter.decBtn, Qt::LeftButton);
        QCOMPARE(counter.count, 2);
    }
    
    void test_counter_signal() {
        Counter counter;
        QSignalSpy spy(&counter, &Counter::countChanged);
        
        QTest::mouseClick(counter.incBtn, Qt::LeftButton);
        QTest::mouseClick(counter.incBtn, Qt::LeftButton);
        QTest::mouseClick(counter.decBtn, Qt::LeftButton);
        
        // Should emit 3 signals
        QCOMPARE(spy.count(), 3);
        
        // Check last value
        QList<QVariant> lastSignal = spy.takeLast();
        QCOMPARE(lastSignal.at(0).toInt(), 1);  // 2-1 = 1
    }
    
    void test_text_input() {
        QLineEdit edit;
        
        // Type text
        edit.show();
        edit.setFocus();
        QTest::keyClicks(&edit, "Hello World");
        
        QCOMPARE(edit.text(), QString("Hello World"));
        
        // Clear and type
        QTest::keyClick(&edit, Qt::Key_A, Qt::ControlModifier);  // Ctrl+A
        QTest::keyClick(&edit, Qt::Key_Delete);
        
        QVERIFY(edit.text().isEmpty());
        
        // Type with special keys
        QTest::keyClicks(&edit, "abc");
        QTest::keyClick(&edit, Qt::Key_Backspace);
        
        QCOMPARE(edit.text(), QString("ab"));
    }
    
    void test_signal_spy_async() {
        Counter counter;
        QSignalSpy spy(&counter, &Counter::countChanged);
        
        // Simulate async operation
        QTimer::singleShot(100, [&counter]() {
            counter.incBtn->click();
        });
        
        // Wait for signal (max 1000ms)
        spy.wait(1000);
        
        QCOMPARE(spy.count(), 1);
    }
};
```

---

## ขั้นตอนที่ 269-280: Test Runner & CMake Integration

```cpp
// === main_test.cpp ===
#include "testbasic.h"
#include "testgui.h"
#include <QTest>

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    
    int status = 0;
    
    {
        TestCalculator testCalc;
        status |= QTest::qExec(&testCalc, argc, argv);
    }
    
    {
        TestStringUtils testString;
        status |= QTest::qExec(&testString, argc, argv);
    }
    
    {
        TestGui testGui;
        status |= QTest::qExec(&testGui, argc, argv);
    }
    
    return status;
}
```

```cmake
# CMakeLists.txt สำหรับ Tests
cmake_minimum_required(VERSION 3.16)
project(MyProject)

enable_testing()

find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets Test)

# Main app
add_executable(myapp
    src/main.cpp
    src/mainwindow.cpp
)

target_link_libraries(myapp PRIVATE Qt6::Core Qt6::Gui Qt6::Widgets)

# Tests
function(add_qt_test name)
    add_executable(${name} tests/${name}.cpp)
    target_link_libraries(${name} PRIVATE Qt6::Core Qt6::Gui Qt6::Widgets Qt6::Test)
    add_test(NAME ${name} COMMAND ${name})
endfunction()

add_qt_test(TestCalculator)
add_qt_test(TestStringUtils)
add_qt_test(TestGui)

# Run all tests: cmake --build . --target test
# หรือ: ctest --output-on-failure
```

---

## สรุป Part 020

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QTest framework
2. ✅ Unit testing ด้วย QCOMPARE, QVERIFY
3. ✅ Data-driven testing ด้วย QFETCH
4. ✅ GUI testing ด้วย QTest::mouseClick, keyClicks
5. ✅ Signal testing ด้วย QSignalSpy
6. ✅ Benchmarking ด้วย QBENCHMARK
7. ✅ CMake integration สำหรับ tests

---

⬅️ [Part 019](part019.md) | ➡️ [Part 021: Design Patterns in Qt](part021.md)
