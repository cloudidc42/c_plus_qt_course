# Part 064: Advanced Testing, Code Quality & CI/CD

## ขั้นตอนที่ 926-940

---

## ขั้นตอนที่ 926: Advanced QTest Patterns

```cpp
// tests/test_repository.cpp
#include <QtTest>
#include <QSqlDatabase>
#include "core/database/database.h"
#include "modules/customer/customerrepository.h"

class TestCustomerRepository : public QObject {
    Q_OBJECT
    
private slots:
    void initTestCase() {
        // ใช้ in-memory SQLite สำหรับ tests
        QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE", "test_connection");
        db.setDatabaseName(":memory:");
        QVERIFY(db.open());
        
        Database::instance().connectWithDatabase(db);
        Database::instance().runMigrations();
    }
    
    void cleanupTestCase() {
        QSqlDatabase::removeDatabase("test_connection");
    }
    
    void init() {
        // ล้าง data ก่อนแต่ละ test
        QSqlQuery q(Database::instance().database());
        q.exec("DELETE FROM customers");
    }
    
    // === ทดสอบ create ===
    void test_create_validCustomer_returnsId() {
        CustomerRepository repo;
        Customer c;
        c.name = "สมชาย ใจดี";
        c.email = "somchai@example.com";
        c.phone = "0812345678";
        
        int id = repo.create(c);
        
        QVERIFY(id > 0);
    }
    
    void test_create_duplicateEmail_returnsMinusOne() {
        CustomerRepository repo;
        Customer c1;
        c1.name = "ลูกค้า 1";
        c1.email = "test@example.com";
        repo.create(c1);
        
        Customer c2;
        c2.name = "ลูกค้า 2";
        c2.email = "test@example.com";  // duplicate
        
        int id = repo.create(c2);
        QCOMPARE(id, -1);
    }
    
    // === ทดสอบ find ===
    void test_findById_existing_returnsCustomer() {
        CustomerRepository repo;
        Customer c;
        c.name = "ทดสอบ";
        c.email = "test@test.com";
        int id = repo.create(c);
        
        auto found = repo.findById(id);
        QVERIFY(found.has_value());
        QCOMPARE(found->name, c.name);
        QCOMPARE(found->email, c.email);
    }
    
    void test_findById_nonExisting_returnsNullopt() {
        CustomerRepository repo;
        auto found = repo.findById(99999);
        QVERIFY(!found.has_value());
    }
    
    // === ทดสอบ search ===
    void test_search_byName_returnsMatches() {
        CustomerRepository repo;
        
        Customer c1; c1.name = "สมชาย"; c1.email = "a@a.com";
        Customer c2; c2.name = "สมหญิง"; c2.email = "b@b.com";
        Customer c3; c3.name = "วิชัย"; c3.email = "c@c.com";
        repo.create(c1); repo.create(c2); repo.create(c3);
        
        auto results = repo.search("สม");
        QCOMPARE(results.size(), 2);
    }
    
    // === Benchmark ===
    void test_benchmark_insert1000() {
        CustomerRepository repo;
        QBENCHMARK {
            for (int i = 0; i < 1000; ++i) {
                Customer c;
                c.name = QString("ลูกค้า %1").arg(i);
                c.email = QString("customer%1@test.com").arg(i);
                repo.create(c);
            }
        }
    }
    
    // === Data-driven test ===
    void test_email_validation_data() {
        QTest::addColumn<QString>("email");
        QTest::addColumn<bool>("valid");
        
        QTest::newRow("valid") << "user@example.com" << true;
        QTest::newRow("valid-subdomain") << "user@mail.example.co.th" << true;
        QTest::newRow("no-at") << "userexample.com" << false;
        QTest::newRow("no-domain") << "user@" << false;
        QTest::newRow("empty") << "" << false;
        QTest::newRow("thai-domain") << "สมชาย@ไทย.th" << false;
    }
    
    void test_email_validation() {
        QFETCH(QString, email);
        QFETCH(bool, valid);
        
        static QRegularExpression rx(R"([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})");
        QCOMPARE(rx.match(email).hasMatch(), valid);
    }
};

QTEST_MAIN(TestCustomerRepository)
#include "test_repository.moc"
```

---

## ขั้นตอนที่ 927: QSignalSpy & Async Testing

```cpp
// tests/test_websocket.cpp
#include <QtTest>
#include <QSignalSpy>
#include "network/websocketclient.h"

class TestWebSocketClient : public QObject {
    Q_OBJECT
    
private slots:
    void test_connect_emitsConnected() {
        WebSocketClient client;
        
        QSignalSpy connectedSpy(&client, &WebSocketClient::connected);
        QSignalSpy disconnectedSpy(&client, &WebSocketClient::disconnected);
        
        client.connectToServer(QUrl("ws://localhost:8080"));
        
        // รอ signal ไม่เกิน 5 วินาที
        QVERIFY(connectedSpy.wait(5000));
        QCOMPARE(connectedSpy.count(), 1);
        QCOMPARE(disconnectedSpy.count(), 0);
    }
    
    void test_send_message_receivedBack() {
        WebSocketClient client;
        client.connectToServer(QUrl("ws://echo.websocket.org"));
        
        QSignalSpy spy(&client, &WebSocketClient::messageReceived);
        QVERIFY(QTest::qWaitFor([&] { return client.isConnected(); }, 5000));
        
        client.send(R"({"type":"ping","data":"hello"})");
        
        QVERIFY(spy.wait(3000));
        
        auto args = spy.takeFirst();
        QString received = args[0].toString();
        QVERIFY(received.contains("hello"));
    }
    
    void test_reconnect_afterDisconnect() {
        WebSocketClient client;
        client.setAutoReconnect(true);
        client.connectToServer(QUrl("ws://localhost:8080"));
        
        QSignalSpy reconnectSpy(&client, &WebSocketClient::reconnecting);
        
        // Force disconnect
        client.disconnect();
        
        QVERIFY(reconnectSpy.wait(10000));  // wait for reconnect attempt
        QVERIFY(reconnectSpy.count() >= 1);
    }
};

// tests/test_model.cpp — Test QAbstractItemModel
class TestContactModel : public QObject {
    Q_OBJECT
    
private slots:
    void test_addContact_rowsInserted() {
        ContactModel model;
        
        QSignalSpy spy(&model, &QAbstractItemModel::rowsInserted);
        
        model.addContact(Contact{.name = "ทดสอบ", .email = "t@t.com"});
        
        QCOMPARE(spy.count(), 1);
        auto args = spy.takeFirst();
        int first = args[1].toInt();
        int last  = args[2].toInt();
        QCOMPARE(first, 0);
        QCOMPARE(last, 0);
        QCOMPARE(model.rowCount(), 1);
    }
    
    void test_removeContact_rowsRemoved() {
        ContactModel model;
        model.addContact(Contact{.name = "A"});
        model.addContact(Contact{.name = "B"});
        
        QSignalSpy spy(&model, &QAbstractItemModel::rowsRemoved);
        
        model.removeContact(0);
        
        QCOMPARE(spy.count(), 1);
        QCOMPARE(model.rowCount(), 1);
        QCOMPARE(model.data(model.index(0), Qt::DisplayRole).toString(), "B");
    }
    
    void test_data_roles() {
        ContactModel model;
        model.addContact(Contact{.name = "สมชาย", .email = "s@s.com", .isFavorite = true});
        
        QModelIndex idx = model.index(0);
        QCOMPARE(model.data(idx, ContactModel::NameRole).toString(), "สมชาย");
        QCOMPARE(model.data(idx, ContactModel::EmailRole).toString(), "s@s.com");
        QCOMPARE(model.data(idx, ContactModel::IsFavoriteRole).toBool(), true);
    }
};
```

---

## ขั้นตอนที่ 928: Fuzzing & Property-Based Testing

```cpp
// tests/fuzz/fuzz_parser.cpp
// Compile with: clang++ -fsanitize=fuzzer,address -o fuzz_parser fuzz_parser.cpp

#include <cstdint>
#include <cstring>
#include "core/jsonparser.h"

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    QString input = QString::fromUtf8(reinterpret_cast<const char*>(data),
                                       static_cast<int>(size));
    
    // ต้องไม่ crash ไม่ว่าจะ input อะไรก็ตาม
    try {
        auto result = JsonParser::parse(input);
        (void)result;
    } catch (const std::exception&) {
        // Expected — invalid JSON throws
    }
    
    return 0;  // 0 = not interesting input
}

// Property-based testing with QTest
class TestOrderCalculations : public QObject {
    Q_OBJECT
    
private slots:
    // Property: ผลรวมของ items ต้องเท่ากับ order total
    void test_property_orderTotal_equalsItemSum() {
        // Run 1000 random test cases
        for (int trial = 0; trial < 1000; ++trial) {
            int numItems = QRandomGenerator::global()->bounded(1, 20);
            
            SalesOrder order;
            double expectedTotal = 0.0;
            
            for (int i = 0; i < numItems; ++i) {
                OrderItem item;
                item.quantity = QRandomGenerator::global()->bounded(1, 100);
                item.unitPrice = QRandomGenerator::global()->generateDouble() * 10000;
                item.discountPercent = QRandomGenerator::global()->bounded(0, 50);
                item.recalculate();
                
                order.items.append(item);
                expectedTotal += item.lineTotal;
            }
            
            order.recalculate();
            
            QVERIFY2(qAbs(order.subtotal - expectedTotal) < 0.01,
                     qPrintable(QString("Trial %1: expected %2, got %3")
                         .arg(trial).arg(expectedTotal).arg(order.subtotal)));
        }
    }
    
    // Property: หลัง sort แล้ว array ต้องเรียงลำดับถูกต้อง
    void test_property_sort_isOrdered() {
        for (int trial = 0; trial < 100; ++trial) {
            int n = QRandomGenerator::global()->bounded(1, 1000);
            std::vector<int> data(n);
            for (auto& x : data) x = QRandomGenerator::global()->generate();
            
            parallelSort(data);
            
            QVERIFY(std::is_sorted(data.begin(), data.end()));
        }
    }
};
```

---

## ขั้นตอนที่ 929: Static Analysis Integration

```cmake
# CMakeLists.txt — Static analysis tools

# clang-tidy
set(CMAKE_CXX_CLANG_TIDY
    clang-tidy;
    -checks=bugprone-*,modernize-*,performance-*,readability-*;
    -warnings-as-errors=*;
    --extra-arg=-std=c++20
)

# cppcheck
find_program(CPPCHECK cppcheck)
if(CPPCHECK)
    set(CMAKE_CXX_CPPCHECK
        ${CPPCHECK};
        --std=c++20;
        --enable=all;
        --suppress=missingIncludeSystem;
        --suppress=unmatchedSuppression;
        --error-exitcode=1
    )
endif()

# AddressSanitizer for debug builds
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(ErpPro PRIVATE
        -fsanitize=address,undefined
        -fno-omit-frame-pointer
    )
    target_link_options(ErpPro PRIVATE
        -fsanitize=address,undefined
    )
endif()
```

```yaml
# .clang-tidy
---
Checks: >
  bugprone-*,
  cert-*,
  cppcoreguidelines-*,
  modernize-*,
  performance-*,
  readability-*,
  -modernize-use-trailing-return-type,
  -readability-magic-numbers,
  -cppcoreguidelines-avoid-magic-numbers

WarningsAsErrors: '*'

CheckOptions:
  - key: modernize-use-default-member-init.UseAssignment
    value: true
  - key: readability-identifier-naming.ClassCase
    value: CamelCase
  - key: readability-identifier-naming.MemberPrefix
    value: m_
  - key: readability-identifier-naming.ParameterCase
    value: camelCase
```

---

## ขั้นตอนที่ 930: Complete CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop, 'feature/**']
  pull_request:
    branches: [main, develop]

env:
  QT_VERSION: '6.6.0'

jobs:
  # ===== Static Analysis =====
  static-analysis:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      
      - name: Install clang-tidy
        run: sudo apt-get install -y clang-tidy cppcheck
      
      - name: Install Qt
        uses: jurplel/install-qt-action@v3
        with:
          version: ${{ env.QT_VERSION }}
          modules: 'qtcharts qtsql qtwebsockets'
      
      - name: Configure
        run: cmake -B build -DCMAKE_BUILD_TYPE=Debug -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
      
      - name: Run clang-tidy
        run: |
          run-clang-tidy -p build -header-filter='.*' \
            $(find src -name '*.cpp' | tr '\n' ' ')
      
      - name: Run cppcheck
        run: cppcheck --enable=all --std=c++20 --error-exitcode=1 src/

  # ===== Build Matrix =====
  build:
    strategy:
      matrix:
        os: [ubuntu-22.04, windows-latest, macos-14]
        build_type: [Debug, Release]
        
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Qt
        uses: jurplel/install-qt-action@v3
        with:
          version: ${{ env.QT_VERSION }}
          modules: 'qtcharts qtsql qtwebsockets qtpdf'
      
      - name: Configure
        run: |
          cmake -B build \
            -DCMAKE_BUILD_TYPE=${{ matrix.build_type }} \
            -DBUILD_TESTING=ON
      
      - name: Build
        run: cmake --build build --parallel
      
      - name: Test
        working-directory: build
        run: ctest --output-on-failure --parallel 4
        env:
          QT_QPA_PLATFORM: offscreen
      
      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.os }}-${{ matrix.build_type }}
          path: build/Testing/

  # ===== Code Coverage =====
  coverage:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      
      - name: Install dependencies
        run: |
          sudo apt-get install -y lcov
      
      - name: Install Qt
        uses: jurplel/install-qt-action@v3
        with:
          version: ${{ env.QT_VERSION }}
      
      - name: Build with coverage
        run: |
          cmake -B build \
            -DCMAKE_BUILD_TYPE=Debug \
            -DBUILD_TESTING=ON \
            -DCMAKE_CXX_FLAGS="--coverage"
          cmake --build build --parallel
      
      - name: Run tests
        working-directory: build
        run: ctest --output-on-failure
        env:
          QT_QPA_PLATFORM: offscreen
      
      - name: Generate coverage report
        run: |
          lcov --capture --directory build --output-file coverage.info
          lcov --remove coverage.info '/usr/*' '*/Qt/*' '*/tests/*' \
               --output-file coverage.info
          genhtml coverage.info --output-directory coverage-report
      
      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: coverage.info
          fail_ci_if_error: true

  # ===== Security Scan =====
  security:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/cpp
            p/security-audit
```

---

## ขั้นตอนที่ 931: Code Coverage Widget

```cpp
// tests/coverageviewer.h — แสดง coverage report ใน Qt
#pragma once
#include <QWidget>
#include <QTreeWidget>

struct FileCoverage {
    QString filePath;
    int totalLines;
    int coveredLines;
    double percentage() const {
        return totalLines > 0 ? (coveredLines * 100.0 / totalLines) : 0.0;
    }
};

class CoverageViewer : public QWidget {
    Q_OBJECT
    
public:
    explicit CoverageViewer(QWidget* parent = nullptr) : QWidget(parent) {
        auto* layout = new QVBoxLayout(this);
        
        // Summary bar
        m_summaryLabel = new QLabel(this);
        m_summaryLabel->setFont(QFont("Segoe UI", 14, QFont::Bold));
        
        m_overallBar = new QProgressBar(this);
        m_overallBar->setMinimum(0);
        m_overallBar->setMaximum(100);
        m_overallBar->setTextVisible(true);
        m_overallBar->setFormat("%p% overall coverage");
        m_overallBar->setMinimumHeight(24);
        
        // File tree
        m_tree = new QTreeWidget(this);
        m_tree->setHeaderLabels({"File", "Lines", "Covered", "Coverage"});
        m_tree->setColumnWidth(0, 300);
        m_tree->setSortingEnabled(true);
        
        layout->addWidget(m_summaryLabel);
        layout->addWidget(m_overallBar);
        layout->addWidget(m_tree);
    }
    
    void loadFromLcov(const QString& infoFilePath) {
        QFile file(infoFilePath);
        if (!file.open(QFile::ReadOnly)) return;
        
        QMap<QString, FileCoverage> coverageMap;
        FileCoverage* current = nullptr;
        
        QTextStream in(&file);
        while (!in.atEnd()) {
            QString line = in.readLine().trimmed();
            
            if (line.startsWith("SF:")) {
                QString path = line.mid(3);
                coverageMap[path] = FileCoverage{path, 0, 0};
                current = &coverageMap[path];
            } else if (current && line.startsWith("LF:")) {
                current->totalLines = line.mid(3).toInt();
            } else if (current && line.startsWith("LH:")) {
                current->coveredLines = line.mid(3).toInt();
            }
        }
        
        populateTree(coverageMap.values());
    }
    
private:
    void populateTree(const QList<FileCoverage>& files) {
        m_tree->clear();
        
        int totalLines = 0, totalCovered = 0;
        
        for (const auto& fc : files) {
            auto* item = new QTreeWidgetItem(m_tree);
            item->setText(0, QFileInfo(fc.filePath).fileName());
            item->setToolTip(0, fc.filePath);
            item->setText(1, QString::number(fc.totalLines));
            item->setText(2, QString::number(fc.coveredLines));
            
            double pct = fc.percentage();
            item->setText(3, QString("%1%").arg(pct, 0, 'f', 1));
            
            // Color code
            if      (pct >= 90) item->setForeground(3, QColor("#27AE60"));
            else if (pct >= 70) item->setForeground(3, QColor("#F39C12"));
            else                item->setForeground(3, QColor("#E74C3C"));
            
            totalLines += fc.totalLines;
            totalCovered += fc.coveredLines;
        }
        
        double overall = totalLines > 0 ? (totalCovered * 100.0 / totalLines) : 0;
        m_overallBar->setValue((int)overall);
        m_summaryLabel->setText(
            QString("Coverage: %1 / %2 lines (%3%)")
                .arg(totalCovered).arg(totalLines).arg(overall, 0, 'f', 1));
    }
    
    QLabel* m_summaryLabel;
    QProgressBar* m_overallBar;
    QTreeWidget* m_tree;
};
```

---

## สรุป Part 064

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Advanced QTest patterns: data-driven, benchmark, init/cleanup
2. ✅ QSignalSpy สำหรับ async signal testing
3. ✅ Fuzzing ด้วย LLVM libFuzzer
4. ✅ Property-based testing ด้วย random inputs
5. ✅ Static analysis: clang-tidy + cppcheck + ASan/UBSan
6. ✅ Complete GitHub Actions CI pipeline: build matrix, coverage, security
7. ✅ Coverage viewer widget แสดงผล lcov

---

⬅️ [Part 063](part063.md) | ➡️ [Part 065: Qt for Embedded & IoT](part065.md)
