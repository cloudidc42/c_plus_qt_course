# Part 047: Professional Level — Qt Testing & CI/CD

## ขั้นตอนที่ 671-685

---

## ขั้นตอนที่ 671: Qt Test Framework

```
Qt Test:
  - QTest namespace: QCOMPARE, QVERIFY, QFAIL, QSKIP
  - QSignalSpy: check signals were emitted
  - QBENCHMARK: performance testing
  - QFETCH / QTEST_APPLESS_MAIN: data-driven tests
  - Test discovery: qmake or CMake CTest integration

CMakeLists.txt:
  find_package(Qt6 REQUIRED COMPONENTS Test)
  
  add_executable(TaskServiceTest tests/TaskServiceTest.cpp)
  target_link_libraries(TaskServiceTest PRIVATE Qt6::Test Qt6::Core MyApp_lib)
  
  add_test(NAME TaskServiceTest COMMAND TaskServiceTest)
```

---

## ขั้นตอนที่ 672: Unit Test — Model

```cpp
#include <QTest>
#include <QSignalSpy>

class TaskModelTest : public QObject {
    Q_OBJECT
    
private slots:
    // Called before every test
    void init() {
        model = new TaskListModel();
    }
    
    // Called after every test
    void cleanup() {
        delete model;
        model = nullptr;
    }
    
    // Test: empty model
    void testEmptyModel() {
        QCOMPARE(model->rowCount(), 0);
        QVERIFY(!model->index(0, 0).isValid());
    }
    
    // Test: add task
    void testAddTask() {
        Task task;
        task.id = 1;
        task.title = "Test Task";
        task.priority = Task::Priority::High;
        task.status = Task::Status::Todo;
        
        QSignalSpy insertSpy(model, &TaskListModel::rowsInserted);
        
        model->addTask(task);
        
        QCOMPARE(model->rowCount(), 1);
        QCOMPARE(insertSpy.count(), 1);
        
        QModelIndex idx = model->index(0);
        QCOMPARE(idx.data(TaskListModel::TitleRole).toString(), "Test Task");
        QCOMPARE(idx.data(TaskListModel::PriorityRole).toInt(), static_cast<int>(Task::Priority::High));
    }
    
    // Test: update task
    void testUpdateTask() {
        Task task{1, "Original", "", Task::Priority::Low, Task::Status::Todo};
        model->addTask(task);
        
        QSignalSpy changeSpy(model, &QAbstractItemModel::dataChanged);
        
        task.title = "Updated";
        task.status = Task::Status::Done;
        model->updateTask(task);
        
        QCOMPARE(changeSpy.count(), 1);
        
        QModelIndex idx = model->index(0);
        QCOMPARE(idx.data(TaskListModel::TitleRole).toString(), "Updated");
        QCOMPARE(idx.data(TaskListModel::StatusRole).toInt(), static_cast<int>(Task::Status::Done));
    }
    
    // Test: remove task
    void testRemoveTask() {
        model->addTask({1, "Task 1"});
        model->addTask({2, "Task 2"});
        model->addTask({3, "Task 3"});
        
        QSignalSpy removeSpy(model, &QAbstractItemModel::rowsRemoved);
        
        model->removeTask(2);
        
        QCOMPARE(model->rowCount(), 2);
        QCOMPARE(removeSpy.count(), 1);
        
        // Verify task 2 is gone
        for (int i = 0; i < model->rowCount(); ++i) {
            QVERIFY(model->index(i).data(TaskListModel::IdRole).toInt() != 2);
        }
    }
    
    // Data-driven test
    void testOverdue_data() {
        QTest::addColumn<QDate>("dueDate");
        QTest::addColumn<Task::Status>("status");
        QTest::addColumn<bool>("expectOverdue");
        
        QTest::newRow("past date, todo")     << QDate::currentDate().addDays(-1) << Task::Status::Todo       << true;
        QTest::newRow("past date, done")     << QDate::currentDate().addDays(-1) << Task::Status::Done       << false;
        QTest::newRow("future date, todo")   << QDate::currentDate().addDays(1)  << Task::Status::Todo       << false;
        QTest::newRow("no date, todo")       << QDate()                          << Task::Status::InProgress << false;
    }
    
    void testOverdue() {
        QFETCH(QDate,        dueDate);
        QFETCH(Task::Status, status);
        QFETCH(bool,         expectOverdue);
        
        Task task;
        task.id = 1;
        task.dueDate = dueDate;
        task.status = status;
        
        QCOMPARE(task.isOverdue(), expectOverdue);
    }
    
    // Test role names
    void testRoleNames() {
        QHash<int, QByteArray> roles = model->roleNames();
        QVERIFY(roles.contains(TaskListModel::TitleRole));
        QCOMPARE(roles[TaskListModel::TitleRole], QByteArray("title"));
    }
    
private:
    TaskListModel* model = nullptr;
};

QTEST_MAIN(TaskModelTest)
#include "TaskModelTest.moc"
```

---

## ขั้นตอนที่ 673: Unit Test — Service

```cpp
class TaskServiceTest : public QObject {
    Q_OBJECT
    
private slots:
    void initTestCase() {
        // Use temp dir for test data
        m_tempDir = new QTemporaryDir();
        QVERIFY(m_tempDir->isValid());
    }
    
    void cleanupTestCase() {
        delete m_tempDir;
    }
    
    void init() {
        service = new TaskService();
        // In a real test, inject a mock/temp data path
    }
    
    void cleanup() {
        delete service;
    }
    
    void testCreateTask() {
        QSignalSpy spy(service, &TaskService::taskCreated);
        
        Task task;
        task.title = "New Task";
        task.priority = Task::Priority::Medium;
        
        int id = service->create(task);
        
        QVERIFY(id > 0);
        QCOMPARE(spy.count(), 1);
        
        const Task& emittedTask = spy.first().first().value<Task>();
        QCOMPARE(emittedTask.id, id);
        QCOMPARE(emittedTask.title, "New Task");
    }
    
    void testFindById() {
        Task t;
        t.title = "Find Me";
        int id = service->create(t);
        
        auto found = service->findById(id);
        QVERIFY(found.has_value());
        QCOMPARE(found->title, "Find Me");
        QCOMPARE(found->id, id);
        
        auto notFound = service->findById(9999);
        QVERIFY(!notFound.has_value());
    }
    
    void testUpdateStatus() {
        Task t;
        t.title = "Status Test";
        t.status = Task::Status::Todo;
        int id = service->create(t);
        
        QSignalSpy spy(service, &TaskService::taskUpdated);
        
        bool ok = service->updateStatus(id, Task::Status::InProgress);
        QVERIFY(ok);
        QCOMPARE(spy.count(), 1);
        
        auto updated = service->findById(id);
        QVERIFY(updated.has_value());
        QCOMPARE(updated->status, Task::Status::InProgress);
    }
    
    void testDeleteTask() {
        Task t;
        t.title = "Delete Me";
        int id = service->create(t);
        
        QVERIFY(service->findById(id).has_value());
        
        QSignalSpy spy(service, &TaskService::taskDeleted);
        bool ok = service->remove(id);
        
        QVERIFY(ok);
        QCOMPARE(spy.count(), 1);
        QCOMPARE(spy.first().first().toInt(), id);
        QVERIFY(!service->findById(id).has_value());
    }
    
    void testFilter() {
        service->create(makeTask("Frontend Bug", Task::Status::Todo, Task::Priority::High));
        service->create(makeTask("Backend Feature", Task::Status::InProgress, Task::Priority::Medium));
        service->create(makeTask("Write Docs", Task::Status::Done, Task::Priority::Low));
        
        // Filter by query
        auto byQuery = service->filtered("Bug");
        QCOMPARE(byQuery.size(), 1);
        QCOMPARE(byQuery.first().title, "Frontend Bug");
        
        // Filter by status
        auto inProgress = service->filtered("", Task::Status::InProgress);
        QCOMPARE(inProgress.size(), 1);
        
        // All active
        auto all = service->filtered();
        QVERIFY(all.size() >= 3);
    }
    
    void testCountByStatus() {
        service->create(makeTask("T1", Task::Status::Todo));
        service->create(makeTask("T2", Task::Status::Todo));
        service->create(makeTask("T3", Task::Status::InProgress));
        
        auto counts = service->countByStatus();
        QVERIFY(counts[Task::Status::Todo] >= 2);
        QVERIFY(counts[Task::Status::InProgress] >= 1);
    }
    
private:
    Task makeTask(const QString& title,
                  Task::Status status = Task::Status::Todo,
                  Task::Priority priority = Task::Priority::Medium) {
        Task t;
        t.title = title;
        t.status = status;
        t.priority = priority;
        return t;
    }
    
    TaskService* service = nullptr;
    QTemporaryDir* m_tempDir = nullptr;
};
```

---

## ขั้นตอนที่ 674: QSignalSpy Advanced

```cpp
class SignalSpyTest : public QObject {
    Q_OBJECT
    
private slots:
    // Test signal content
    void testSignalArgs() {
        TaskService service;
        
        QSignalSpy spy(&service, &TaskService::taskCreated);
        QVERIFY(spy.isValid());
        
        Task t;
        t.title = "Spy Test";
        service.create(t);
        
        QCOMPARE(spy.count(), 1);
        
        // Access signal arguments
        QList<QVariant> args = spy.takeFirst();
        Task emitted = args.first().value<Task>();
        QCOMPARE(emitted.title, "Spy Test");
    }
    
    // Test signal is emitted exactly once
    void testExactlyOnce() {
        TaskService service;
        QSignalSpy spy(&service, &TaskService::taskCreated);
        
        Task t;
        t.title = "Once";
        service.create(t);
        
        QCOMPARE(spy.count(), 1);
        QVERIFY(!spy.isEmpty());
    }
    
    // Wait for async signal
    void testAsyncSignal() {
        TaskService service;
        QSignalSpy spy(&service, &TaskService::taskCreated);
        
        // Start async operation
        QTimer::singleShot(100, [&service]() {
            Task t;
            t.title = "Async";
            service.create(t);
        });
        
        // Wait up to 1000ms for signal
        bool received = spy.wait(1000);
        QVERIFY(received);
        QCOMPARE(spy.count(), 1);
    }
    
    // Signal should NOT fire
    void testNoSignal() {
        TaskService service;
        QSignalSpy spy(&service, &TaskService::taskDeleted);
        
        service.remove(9999); // Non-existent ID
        
        // Give event loop a cycle
        QCoreApplication::processEvents();
        QCOMPARE(spy.count(), 0);
    }
};
```

---

## ขั้นตอนที่ 675: Benchmark Tests

```cpp
class BenchmarkTest : public QObject {
    Q_OBJECT
    
private slots:
    void benchmarkAddTasks() {
        TaskListModel model;
        
        QBENCHMARK {
            model.setTasks({});  // Reset
            
            QList<Task> tasks;
            for (int i = 0; i < 1000; ++i) {
                Task t;
                t.id = i;
                t.title = QString("Task %1").arg(i);
                tasks << t;
            }
            model.setTasks(tasks);
        }
    }
    
    void benchmarkSearch() {
        TaskService service;
        
        // Setup: 10000 tasks
        for (int i = 0; i < 10000; ++i) {
            Task t;
            t.title = QString("Task number %1 with some keywords").arg(i);
            service.create(t);
        }
        
        QBENCHMARK {
            auto results = service.filtered("number 500");
            Q_UNUSED(results)
        }
    }
    
    void benchmarkJsonSerialization() {
        Task t;
        t.id = 1;
        t.title = "Benchmark Task";
        t.description = "Some description with text";
        t.priority = Task::Priority::High;
        t.status = Task::Status::InProgress;
        t.tags = {"qt", "cpp", "benchmark"};
        
        QBENCHMARK {
            auto json = t.toJson();
            auto t2 = Task::fromJson(json);
            Q_UNUSED(t2)
        }
    }
};
```

---

## ขั้นตอนที่ 676: Integration Test

```cpp
class IntegrationTest : public QObject {
    Q_OBJECT
    
private slots:
    void testEndToEndWorkflow() {
        // Simulate full user workflow
        TaskService service;
        TaskListModel model;
        
        // Step 1: Create tasks
        QSignalSpy createSpy(&service, &TaskService::taskCreated);
        
        Task t1, t2, t3;
        t1.title = "Design UI";
        t2.title = "Implement Backend";
        t3.title = "Write Tests";
        
        int id1 = service.create(t1);
        int id2 = service.create(t2);
        int id3 = service.create(t3);
        
        QCOMPARE(createSpy.count(), 3);
        
        // Step 2: Load into model
        model.setTasks(service.tasks());
        QCOMPARE(model.rowCount(), 3);
        
        // Step 3: Update status
        service.updateStatus(id1, Task::Status::Done);
        service.updateStatus(id2, Task::Status::InProgress);
        
        auto counts = service.countByStatus();
        QCOMPARE(counts[Task::Status::Done], 1);
        QCOMPARE(counts[Task::Status::InProgress], 1);
        QCOMPARE(counts[Task::Status::Todo], 1);
        
        // Step 4: Delete
        service.remove(id3);
        QCOMPARE(service.tasks().size(), 2);
        
        // Step 5: Verify no overdue
        QVERIFY(service.overdueTasks().isEmpty());
    }
};
```

---

## ขั้นตอนที่ 677-685: CMake CTest Integration

```cmake
# CMakeLists.txt — Test configuration
cmake_minimum_required(VERSION 3.21)
project(TaskManagerApp)

find_package(Qt6 REQUIRED COMPONENTS Core Widgets Test)

# Build app as a library (for reuse in tests)
add_library(TaskManagerLib STATIC
    models/Task.cpp
    models/TaskListModel.cpp
    services/TaskService.cpp
)
target_link_libraries(TaskManagerLib PUBLIC Qt6::Core Qt6::Widgets)

# Main executable
add_executable(TaskManager main.cpp)
target_link_libraries(TaskManager PRIVATE TaskManagerLib Qt6::Widgets)

# Test executable
enable_testing()

add_executable(TaskModelTest tests/TaskModelTest.cpp)
target_link_libraries(TaskModelTest PRIVATE TaskManagerLib Qt6::Test)
add_test(NAME TaskModelTest COMMAND TaskModelTest)

add_executable(TaskServiceTest tests/TaskServiceTest.cpp)
target_link_libraries(TaskServiceTest PRIVATE TaskManagerLib Qt6::Test)
add_test(NAME TaskServiceTest COMMAND TaskServiceTest)

add_executable(BenchmarkTest tests/BenchmarkTest.cpp)
target_link_libraries(BenchmarkTest PRIVATE TaskManagerLib Qt6::Test)
add_test(NAME BenchmarkTest COMMAND BenchmarkTest)
```

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Install Qt
      uses: jurplel/install-qt-action@v3
      with:
        version: '6.6.0'
        modules: 'qtcharts qtnetworkauth'
    
    - name: Configure
      run: cmake -B build -DCMAKE_BUILD_TYPE=Release
    
    - name: Build
      run: cmake --build build --parallel
    
    - name: Test
      run: ctest --test-dir build --output-on-failure
    
    - name: Coverage
      if: success()
      run: |
        lcov --capture --directory build --output-file coverage.info
        lcov --remove coverage.info '/usr/*' '*/Qt*' --output-file coverage.info
        genhtml coverage.info --output-directory coverage_html
    
    - name: Upload Coverage
      uses: codecov/codecov-action@v3
      with:
        files: coverage.info
```

```bash
# Run tests locally
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --target all

# Run all tests with verbose output
ctest --test-dir build -V

# Run specific test
./build/TaskModelTest

# Run with test output
./build/TaskServiceTest -v2

# Benchmark
./build/BenchmarkTest -benchmark
```

---

## สรุป Part 047

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QTest: QCOMPARE, QVERIFY, QFETCH, data-driven tests
2. ✅ QSignalSpy: ตรวจสอบ signals ถูก emit
3. ✅ QBENCHMARK สำหรับ performance testing
4. ✅ Integration test สำหรับ end-to-end workflow
5. ✅ CMake CTest + GitHub Actions CI configuration

---

⬅️ [Part 046](part046.md) | ➡️ [Part 048: World-Class Level — Advanced C++ Metaprogramming](part048.md)
