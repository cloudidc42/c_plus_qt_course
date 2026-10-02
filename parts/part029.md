# Part 029: Advanced Error Handling & std::expected

## ขั้นตอนที่ 401-415

---

## ขั้นตอนที่ 401: Exception Hierarchy

```cpp
#include <stdexcept>
#include <system_error>
#include <exception>
#include <string>
#include <optional>

// === Custom Exception Hierarchy ===
class AppException : public std::runtime_error {
public:
    explicit AppException(const std::string& msg) : std::runtime_error(msg) {}
    
    virtual int code() const { return 0; }
    virtual std::string category() const { return "app"; }
};

class DatabaseException : public AppException {
public:
    explicit DatabaseException(const std::string& msg, int sqlCode = 0)
        : AppException("[DB] " + msg), m_sqlCode(sqlCode) {}
    
    int code() const override { return m_sqlCode; }
    std::string category() const override { return "database"; }
    int sqlCode() const { return m_sqlCode; }
    
private:
    int m_sqlCode;
};

class NetworkException : public AppException {
public:
    enum class ErrorType { Timeout, Refused, NotFound, Unauthorized };
    
    NetworkException(const std::string& msg, ErrorType type)
        : AppException("[NET] " + msg), m_type(type) {}
    
    int code() const override { return static_cast<int>(m_type); }
    std::string category() const override { return "network"; }
    ErrorType errorType() const { return m_type; }
    
private:
    ErrorType m_type;
};

class ValidationException : public AppException {
public:
    explicit ValidationException(const std::string& field, const std::string& msg)
        : AppException("[VALID] " + field + ": " + msg), m_field(field) {}
    
    int code() const override { return 422; }
    std::string category() const override { return "validation"; }
    const std::string& field() const { return m_field; }
    
private:
    std::string m_field;
};

// Multi-exception handler
void handleException(const std::exception_ptr& eptr) {
    try {
        if (eptr) std::rethrow_exception(eptr);
    } catch (const ValidationException& e) {
        std::cerr << "Validation error on '" << e.field() << "': " << e.what() << '\n';
    } catch (const DatabaseException& e) {
        std::cerr << "DB error (SQL " << e.sqlCode() << "): " << e.what() << '\n';
    } catch (const NetworkException& e) {
        std::cerr << "Network error type " << e.code() << ": " << e.what() << '\n';
    } catch (const AppException& e) {
        std::cerr << "App error [" << e.category() << "]: " << e.what() << '\n';
    } catch (const std::exception& e) {
        std::cerr << "Standard exception: " << e.what() << '\n';
    } catch (...) {
        std::cerr << "Unknown exception\n";
    }
}

void demoExceptions() {
    auto eptr = std::make_exception_ptr(ValidationException("email", "invalid format"));
    handleException(eptr);
    
    auto dbPtr = std::make_exception_ptr(DatabaseException("Connection failed", 1045));
    handleException(dbPtr);
}
```

---

## ขั้นตอนที่ 402: std::optional

```cpp
#include <optional>
#include <vector>

struct User {
    int id;
    std::string name, email;
    std::optional<std::string> phone; // optional field
    std::optional<int> age;
};

// Functions that might fail
std::optional<User> findUser(int id, const std::vector<User>& users) {
    for (const auto& u : users) {
        if (u.id == id) return u;
    }
    return std::nullopt;
}

std::optional<double> safeDivide(double a, double b) {
    if (b == 0) return std::nullopt;
    return a / b;
}

std::optional<int> parseInteger(const std::string& s) {
    try {
        std::size_t pos;
        int val = std::stoi(s, &pos);
        if (pos != s.size()) return std::nullopt; // trailing chars
        return val;
    } catch (...) {
        return std::nullopt;
    }
}

void demoOptional() {
    std::vector<User> users = {
        {1, "Alice", "alice@example.com", "555-1234", 28},
        {2, "Bob",   "bob@example.com",   std::nullopt, std::nullopt},
        {3, "Carol", "carol@example.com",  "555-5678", 32}
    };
    
    // Using optional
    auto user = findUser(2, users);
    if (user) {
        std::cout << "Found: " << user->name << '\n';
        
        // Optional chaining
        if (user->phone) {
            std::cout << "Phone: " << *user->phone << '\n';
        } else {
            std::cout << "No phone number\n";
        }
        
        // value_or
        std::string phone = user->phone.value_or("N/A");
        int age = user->age.value_or(0);
        std::cout << "Phone: " << phone << ", Age: " << age << '\n';
    }
    
    // Chaining with and_then / transform (C++23)
    // or just manual if chains:
    auto result = findUser(1, users);
    std::string info = result
        ? result->name + " (" + result->email + ")"
        : "User not found";
    std::cout << info << '\n';
    
    // Optional arithmetic
    auto div1 = safeDivide(10, 2);   // {5.0}
    auto div2 = safeDivide(10, 0);   // nullopt
    
    std::cout << "10/2 = " << div1.value_or(0) << '\n';
    std::cout << "10/0 = " << (div2 ? std::to_string(*div2) : "undefined") << '\n';
    
    // Parse
    auto num = parseInteger("42");
    auto bad = parseInteger("3.14");
    auto err = parseInteger("abc");
    
    std::cout << "42 parsed: " << num.value_or(-1) << '\n'; // 42
    std::cout << "3.14 parsed: " << bad.value_or(-1) << '\n'; // -1
    std::cout << "abc parsed: " << err.value_or(-1) << '\n'; // -1
}
```

---

## ขั้นตอนที่ 403: std::expected (C++23)

```cpp
// C++23 std::expected - better than optional for error-bearing functions
#include <expected>
#include <string>
#include <vector>

// Error type
struct Error {
    int code;
    std::string message;
    
    static Error notFound(const std::string& what) {
        return {404, what + " not found"};
    }
    
    static Error invalidArg(const std::string& msg) {
        return {400, "Invalid argument: " + msg};
    }
    
    static Error ioError(const std::string& msg) {
        return {500, "I/O Error: " + msg};
    }
    
    operator std::string() const {
        return "[" + std::to_string(code) + "] " + message;
    }
};

using Result = std::expected<std::string, Error>;
using IntResult = std::expected<int, Error>;

// Functions returning expected
IntResult parsePositiveInt(const std::string& s) {
    try {
        int val = std::stoi(s);
        if (val <= 0) return std::unexpected(Error::invalidArg("must be positive"));
        return val;
    } catch (const std::invalid_argument&) {
        return std::unexpected(Error::invalidArg("'" + s + "' is not a number"));
    } catch (const std::out_of_range&) {
        return std::unexpected(Error::invalidArg("out of range"));
    }
}

std::expected<std::string, Error> readFile(const std::string& path) {
    std::ifstream file(path);
    if (!file) {
        return std::unexpected(Error::ioError("Cannot open " + path));
    }
    
    std::string content((std::istreambuf_iterator<char>(file)),
                         std::istreambuf_iterator<char>());
    return content;
}

// Chaining operations
std::expected<double, Error> calculate(const std::string& numStr, double divisor) {
    auto num = parsePositiveInt(numStr);
    if (!num) return std::unexpected(num.error());
    
    if (divisor == 0) return std::unexpected(Error{500, "Division by zero"});
    
    return *num / divisor;
}

void demoExpected() {
    // Success case
    auto r1 = parsePositiveInt("42");
    if (r1) {
        std::cout << "Parsed: " << *r1 << '\n';
    }
    
    // Error case
    auto r2 = parsePositiveInt("abc");
    if (!r2) {
        std::cout << "Error: " << std::string(r2.error()) << '\n';
    }
    
    // value_or
    int val = parsePositiveInt("10").value_or(-1); // 10
    int bad = parsePositiveInt("-5").value_or(-1); // -1
    
    // and_then / transform / or_else (monadic operations)
    auto result = parsePositiveInt("5")
        .and_then([](int n) -> IntResult {
            if (n > 100) return std::unexpected(Error{400, "too large"});
            return n * 2;
        })
        .transform([](int n) { return std::to_string(n); });
    
    if (result) std::cout << "Result: " << *result << '\n'; // "10"
    
    // or_else: provide fallback on error
    auto withFallback = parsePositiveInt("bad")
        .or_else([](const Error&) -> IntResult { return 0; });
    
    std::cout << "With fallback: " << *withFallback << '\n'; // 0
}
```

---

## ขั้นตอนที่ 404: Error Handling Patterns in Qt

```cpp
#include <QException>
#include <QFuture>
#include <QPromise>

// === Qt Exception ===
class QtAppException : public QException {
public:
    explicit QtAppException(const QString& msg) : m_msg(msg) {}
    
    void raise() const override { throw *this; }
    QException* clone() const override { return new QtAppException(*this); }
    
    QString message() const { return m_msg; }
    
private:
    QString m_msg;
};

// === Result type for Qt ===
template<typename T>
struct QtResult {
    std::optional<T> value;
    QString error;
    
    bool ok() const { return value.has_value(); }
    
    static QtResult<T> success(T val) { return {val, {}}; }
    static QtResult<T> failure(const QString& err) { return {std::nullopt, err}; }
};

// Database operation with Qt result types
class SafeDatabase {
    QSqlDatabase db;
    
public:
    bool connect(const QString& host, const QString& dbName) {
        db = QSqlDatabase::addDatabase("QMYSQL");
        db.setHostName(host);
        db.setDatabaseName(dbName);
        return db.open();
    }
    
    QtResult<QList<QVariantMap>> query(const QString& sql, 
                                        const QVariantList& params = {}) {
        QSqlQuery q(db);
        q.prepare(sql);
        for (int i = 0; i < params.size(); i++) {
            q.addBindValue(params[i]);
        }
        
        if (!q.exec()) {
            return QtResult<QList<QVariantMap>>::failure(
                "Query failed: " + q.lastError().text()
            );
        }
        
        QList<QVariantMap> rows;
        QSqlRecord record = q.record();
        
        while (q.next()) {
            QVariantMap row;
            for (int i = 0; i < record.count(); i++) {
                row[record.fieldName(i)] = q.value(i);
            }
            rows.append(row);
        }
        
        return QtResult<QList<QVariantMap>>::success(rows);
    }
    
    QtResult<int> execute(const QString& sql, const QVariantList& params = {}) {
        QSqlQuery q(db);
        q.prepare(sql);
        for (const auto& p : params) q.addBindValue(p);
        
        if (!q.exec()) {
            return QtResult<int>::failure("Execute failed: " + q.lastError().text());
        }
        
        return QtResult<int>::success(q.numRowsAffected());
    }
};

// === QPromise for async error propagation ===
class AsyncFileReader : public QObject {
    Q_OBJECT
    
public:
    QFuture<QString> readAsync(const QString& path) {
        QPromise<QString> promise;
        QFuture<QString> future = promise.future();
        
        QtConcurrent::run([path, p = std::move(promise)]() mutable {
            QFile file(path);
            if (!file.open(QIODevice::ReadOnly)) {
                p.setException(std::make_exception_ptr(
                    QtAppException("Cannot open: " + path)
                ));
                return;
            }
            
            p.addResult(QString::fromUtf8(file.readAll()));
        });
        
        return future;
    }
};
```

---

## ขั้นตอนที่ 405-415: RAII Error Safety

```cpp
// === Transaction RAII ===
class DbTransaction {
public:
    explicit DbTransaction(QSqlDatabase& db) : m_db(db), m_committed(false) {
        m_active = m_db.transaction();
        if (!m_active) throw DatabaseException("Cannot start transaction");
    }
    
    ~DbTransaction() {
        if (m_active && !m_committed) {
            m_db.rollback();
            qWarning() << "Transaction rolled back (destructor)";
        }
    }
    
    void commit() {
        if (!m_active) throw DatabaseException("No active transaction");
        if (!m_db.commit()) {
            m_db.rollback();
            throw DatabaseException("Commit failed: " + m_db.lastError().text().toStdString());
        }
        m_committed = true;
        m_active = false;
    }
    
    void rollback() {
        if (m_active) {
            m_db.rollback();
            m_active = false;
        }
    }
    
private:
    QSqlDatabase& m_db;
    bool m_committed;
    bool m_active;
};

// === Scope guard with custom cleanup ===
template<typename F>
class ScopeGuard {
public:
    explicit ScopeGuard(F&& f) : m_func(std::forward<F>(f)), m_active(true) {}
    
    ~ScopeGuard() {
        if (m_active) {
            try { m_func(); }
            catch (...) { /* suppress in destructor */ }
        }
    }
    
    void dismiss() { m_active = false; } // Don't run cleanup
    
    // Disable copy
    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
    
private:
    F m_func;
    bool m_active;
};

template<typename F>
auto makeScopeGuard(F&& f) { return ScopeGuard<F>(std::forward<F>(f)); }

// Usage
void transferFunds(int fromId, int toId, double amount) {
    QSqlDatabase db = QSqlDatabase::database();
    DbTransaction tx(db);
    
    // RAII logging
    auto logGuard = makeScopeGuard([&]() {
        qDebug() << "Transfer attempted:" << fromId << "->" << toId << amount;
    });
    
    QSqlQuery q(db);
    
    // Deduct from sender
    q.prepare("UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?");
    q.addBindValue(amount);
    q.addBindValue(fromId);
    q.addBindValue(amount);
    
    if (!q.exec() || q.numRowsAffected() == 0) {
        // tx destructor will rollback automatically
        throw DatabaseException("Insufficient funds or sender not found");
    }
    
    // Add to receiver
    q.prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?");
    q.addBindValue(amount);
    q.addBindValue(toId);
    
    if (!q.exec() || q.numRowsAffected() == 0) {
        throw DatabaseException("Receiver not found");
    }
    
    // Log transaction
    q.prepare("INSERT INTO transactions (from_id, to_id, amount) VALUES (?, ?, ?)");
    q.addBindValue(fromId);
    q.addBindValue(toId);
    q.addBindValue(amount);
    
    if (!q.exec()) {
        throw DatabaseException("Failed to log transaction");
    }
    
    tx.commit(); // Everything succeeded
    qDebug() << "Transfer completed successfully";
}

// Error-safe version with Qt result type
QtResult<bool> safeTrasferFunds(int from, int to, double amount) {
    try {
        transferFunds(from, to, amount);
        return QtResult<bool>::success(true);
    } catch (const DatabaseException& e) {
        return QtResult<bool>::failure(QString::fromStdString(e.what()));
    } catch (const std::exception& e) {
        return QtResult<bool>::failure(e.what());
    }
}
```

---

## สรุป Part 029

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Custom exception hierarchy
2. ✅ std::optional สำหรับ nullable values
3. ✅ std::expected (C++23) สำหรับ error-bearing functions
4. ✅ Monadic operations (and_then, transform, or_else)
5. ✅ Qt exception handling
6. ✅ RAII Transaction + ScopeGuard

---

⬅️ [Part 028](part028.md) | ➡️ [Part 030: Performance Optimization](part030.md)
