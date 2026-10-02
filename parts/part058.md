# Part 058: World-Class — Security & Authentication

## ขั้นตอนที่ 836-850

---

## ขั้นตอนที่ 836: Password Hashing & Security Fundamentals

```cpp
// security/passwordhasher.h
#pragma once
#include <QCryptographicHash>
#include <QRandomGenerator>
#include <QByteArray>

class PasswordHasher {
public:
    struct HashedPassword {
        QByteArray hash;
        QByteArray salt;
        int iterations;
        
        QString toString() const {
            return QString("pbkdf2:%1:%2:%3")
                .arg(iterations)
                .arg(QString::fromLatin1(salt.toHex()))
                .arg(QString::fromLatin1(hash.toHex()));
        }
        
        static HashedPassword fromString(const QString& encoded) {
            auto parts = encoded.split(':');
            if (parts.size() != 4 || parts[0] != "pbkdf2") return {};
            
            return {
                QByteArray::fromHex(parts[3].toLatin1()),
                QByteArray::fromHex(parts[2].toLatin1()),
                parts[1].toInt()
            };
        }
    };
    
    // PBKDF2-HMAC-SHA256 password hashing
    static HashedPassword hash(const QString& password, int iterations = 260000) {
        // Generate cryptographically random salt
        QByteArray salt(32, 0);
        auto* gen = QRandomGenerator::securelySeeded();
        for (auto& byte : salt) {
            byte = static_cast<char>(gen->generate() & 0xFF);
        }
        
        return {pbkdf2(password.toUtf8(), salt, iterations), salt, iterations};
    }
    
    static bool verify(const QString& password, const QString& encodedHash) {
        auto stored = HashedPassword::fromString(encodedHash);
        if (stored.hash.isEmpty()) return false;
        
        auto derived = pbkdf2(password.toUtf8(), stored.salt, stored.iterations);
        
        // Constant-time comparison (prevent timing attacks)
        return constantTimeCompare(derived, stored.hash);
    }
    
private:
    static QByteArray pbkdf2(const QByteArray& password,
                              const QByteArray& salt,
                              int iterations,
                              int keyLength = 32)
    {
        QByteArray result;
        int blockCount = (keyLength + 31) / 32;
        
        for (int i = 1; i <= blockCount; ++i) {
            QByteArray u = hmacSha256(password,
                salt + QByteArray::fromHex(
                    QString("%1").arg(i, 8, 16, QChar('0')).toLatin1()));
            
            QByteArray t = u;
            
            for (int j = 1; j < iterations; ++j) {
                u = hmacSha256(password, u);
                for (int k = 0; k < t.size(); ++k) {
                    t[k] = t[k] ^ u[k];
                }
            }
            
            result.append(t);
        }
        
        return result.left(keyLength);
    }
    
    static QByteArray hmacSha256(const QByteArray& key, const QByteArray& message) {
        const int blockSize = 64;
        QByteArray k = key;
        
        if (k.size() > blockSize) {
            k = QCryptographicHash::hash(k, QCryptographicHash::Sha256);
        }
        
        k = k.leftJustified(blockSize, '\0');
        
        QByteArray ipad(blockSize, '\x36');
        QByteArray opad(blockSize, '\x5c');
        
        for (int i = 0; i < blockSize; ++i) {
            ipad[i] = ipad[i] ^ k[i];
            opad[i] = opad[i] ^ k[i];
        }
        
        return QCryptographicHash::hash(
            opad + QCryptographicHash::hash(ipad + message, QCryptographicHash::Sha256),
            QCryptographicHash::Sha256
        );
    }
    
    static bool constantTimeCompare(const QByteArray& a, const QByteArray& b) {
        if (a.size() != b.size()) return false;
        
        unsigned char diff = 0;
        for (int i = 0; i < a.size(); ++i) {
            diff |= static_cast<unsigned char>(a[i]) ^
                    static_cast<unsigned char>(b[i]);
        }
        return diff == 0;
    }
};
```

---

## ขั้นตอนที่ 837: JWT Token System

```cpp
// security/jwtmanager.h
#pragma once
#include <QByteArray>
#include <QJsonObject>
#include <QJsonDocument>
#include <QDateTime>
#include <QMessageAuthenticationCode>

class JwtManager {
public:
    struct Claims {
        QString userId;
        QString username;
        QStringList roles;
        QDateTime issuedAt;
        QDateTime expiresAt;
        
        bool isExpired() const {
            return QDateTime::currentDateTimeUtc() > expiresAt;
        }
        
        bool hasRole(const QString& role) const {
            return roles.contains(role);
        }
    };
    
    explicit JwtManager(const QByteArray& secretKey) : m_secret(secretKey) {}
    
    QString createToken(const Claims& claims) const {
        // Header
        QJsonObject header;
        header["alg"] = "HS256";
        header["typ"] = "JWT";
        
        // Payload
        QJsonObject payload;
        payload["sub"] = claims.userId;
        payload["name"] = claims.username;
        payload["roles"] = QJsonArray::fromStringList(claims.roles);
        payload["iat"] = claims.issuedAt.toSecsSinceEpoch();
        payload["exp"] = claims.expiresAt.toSecsSinceEpoch();
        
        QString headerB64 = base64UrlEncode(
            QJsonDocument(header).toJson(QJsonDocument::Compact));
        QString payloadB64 = base64UrlEncode(
            QJsonDocument(payload).toJson(QJsonDocument::Compact));
        
        QString signingInput = headerB64 + "." + payloadB64;
        
        QMessageAuthenticationCode mac(QCryptographicHash::Sha256, m_secret);
        mac.addData(signingInput.toUtf8());
        
        QString signatureB64 = base64UrlEncode(mac.result());
        
        return signingInput + "." + signatureB64;
    }
    
    std::optional<Claims> verifyToken(const QString& token) const {
        auto parts = token.split('.');
        if (parts.size() != 3) return std::nullopt;
        
        // Verify signature
        QString signingInput = parts[0] + "." + parts[1];
        
        QMessageAuthenticationCode mac(QCryptographicHash::Sha256, m_secret);
        mac.addData(signingInput.toUtf8());
        
        QString expectedSig = base64UrlEncode(mac.result());
        
        // Constant-time comparison
        if (parts[2].size() != expectedSig.size()) return std::nullopt;
        
        unsigned char diff = 0;
        for (int i = 0; i < parts[2].size(); ++i) {
            diff |= parts[2][i].toLatin1() ^ expectedSig[i].toLatin1();
        }
        if (diff != 0) return std::nullopt;
        
        // Parse payload
        QByteArray payloadBytes = base64UrlDecode(parts[1]);
        QJsonObject payload = QJsonDocument::fromJson(payloadBytes).object();
        
        Claims claims;
        claims.userId   = payload["sub"].toString();
        claims.username = payload["name"].toString();
        claims.issuedAt = QDateTime::fromSecsSinceEpoch(payload["iat"].toInteger());
        claims.expiresAt = QDateTime::fromSecsSinceEpoch(payload["exp"].toInteger());
        
        for (const auto& role : payload["roles"].toArray()) {
            claims.roles << role.toString();
        }
        
        if (claims.isExpired()) return std::nullopt;
        
        return claims;
    }
    
    Claims createUserClaims(const QString& userId, const QString& username,
                             const QStringList& roles,
                             int expiresInMinutes = 480)
    {
        Claims c;
        c.userId    = userId;
        c.username  = username;
        c.roles     = roles;
        c.issuedAt  = QDateTime::currentDateTimeUtc();
        c.expiresAt = c.issuedAt.addSecs(expiresInMinutes * 60);
        return c;
    }
    
private:
    static QString base64UrlEncode(const QByteArray& data) {
        return data.toBase64(QByteArray::Base64UrlEncoding | QByteArray::OmitTrailingEquals);
    }
    
    static QByteArray base64UrlDecode(const QString& s) {
        QByteArray padded = s.toLatin1();
        while (padded.size() % 4 != 0) padded += '=';
        return QByteArray::fromBase64(padded, QByteArray::Base64UrlEncoding);
    }
    
    QByteArray m_secret;
};
```

---

## ขั้นตอนที่ 838: User Authentication Service

```cpp
// security/authservice.h
#pragma once
#include <QObject>
#include "passwordhasher.h"
#include "jwtmanager.h"

class AuthService : public QObject {
    Q_OBJECT
    
    Q_PROPERTY(bool isLoggedIn READ isLoggedIn NOTIFY loginStateChanged)
    Q_PROPERTY(QString currentUser READ currentUser NOTIFY loginStateChanged)
    Q_PROPERTY(QStringList userRoles READ userRoles NOTIFY loginStateChanged)
    
public:
    struct User {
        int id;
        QString username;
        QString passwordHash;
        QStringList roles;
        bool isActive = true;
        int failedAttempts = 0;
        QDateTime lockedUntil;
        
        bool isLocked() const {
            return lockedUntil.isValid() && lockedUntil > QDateTime::currentDateTime();
        }
    };
    
    enum LoginResult {
        Success,
        InvalidCredentials,
        AccountLocked,
        AccountDisabled,
        TooManyAttempts
    };
    Q_ENUM(LoginResult)
    
    static AuthService& instance() {
        static AuthService inst;
        return inst;
    }
    
    bool isLoggedIn() const { return !m_currentToken.isEmpty(); }
    QString currentUser() const { return m_currentClaims ? m_currentClaims->username : ""; }
    QStringList userRoles() const { return m_currentClaims ? m_currentClaims->roles : QStringList{}; }
    bool hasRole(const QString& role) const {
        return m_currentClaims && m_currentClaims->hasRole(role);
    }
    
    LoginResult login(const QString& username, const QString& password) {
        auto* user = findUser(username);
        
        if (!user) return InvalidCredentials;
        if (!user->isActive) return AccountDisabled;
        if (user->isLocked()) return AccountLocked;
        
        if (!PasswordHasher::verify(password, user->passwordHash)) {
            user->failedAttempts++;
            
            if (user->failedAttempts >= 5) {
                user->lockedUntil = QDateTime::currentDateTime().addSecs(15 * 60);
                user->failedAttempts = 0;
                emit loginFailed(username, AccountLocked);
                return AccountLocked;
            }
            
            emit loginFailed(username, InvalidCredentials);
            return InvalidCredentials;
        }
        
        // Success — reset failure counter
        user->failedAttempts = 0;
        user->lockedUntil = {};
        
        auto claims = m_jwt.createUserClaims(
            QString::number(user->id), username, user->roles);
        
        m_currentToken = m_jwt.createToken(claims);
        m_currentClaims = claims;
        
        saveTokenToStorage();
        
        emit loginStateChanged();
        emit loginSucceeded(username);
        
        return Success;
    }
    
    void logout() {
        m_currentToken.clear();
        m_currentClaims.reset();
        clearStoredToken();
        emit loginStateChanged();
        emit loggedOut();
    }
    
    bool tryAutoLogin() {
        QString token = loadTokenFromStorage();
        if (token.isEmpty()) return false;
        
        auto claims = m_jwt.verifyToken(token);
        if (!claims) return false;
        
        m_currentToken = token;
        m_currentClaims = *claims;
        emit loginStateChanged();
        return true;
    }
    
    void createUser(const QString& username, const QString& password,
                    const QStringList& roles = {"user"})
    {
        auto hashed = PasswordHasher::hash(password);
        
        int newId = m_users.isEmpty() ? 1 : m_users.last().id + 1;
        m_users.append({newId, username, hashed.toString(), roles});
        
        // Persist to DB
        Database::instance().exec(
            "INSERT OR REPLACE INTO users(id,username,password_hash,roles) VALUES(?,?,?,?)",
            {newId, username, hashed.toString(), roles.join(",")}
        );
    }
    
signals:
    void loginStateChanged();
    void loginSucceeded(const QString& username);
    void loginFailed(const QString& username, LoginResult reason);
    void loggedOut();
    
private:
    AuthService()
        : m_jwt(generateOrLoadSecret())
    {
        loadUsersFromDb();
        createDefaultAdminIfNeeded();
    }
    
    QByteArray generateOrLoadSecret() {
        QSettings s;
        if (!s.contains("security/jwt_secret")) {
            QByteArray secret(64, 0);
            auto* gen = QRandomGenerator::securelySeeded();
            for (auto& b : secret) b = static_cast<char>(gen->generate() & 0xFF);
            s.setValue("security/jwt_secret", secret.toHex());
        }
        return QByteArray::fromHex(s.value("security/jwt_secret").toByteArray());
    }
    
    void loadUsersFromDb() {
        Database::instance().exec(R"(
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY,
                username TEXT NOT NULL UNIQUE,
                password_hash TEXT NOT NULL,
                roles TEXT DEFAULT 'user',
                is_active INTEGER DEFAULT 1,
                failed_attempts INTEGER DEFAULT 0,
                locked_until TEXT
            )
        )");
        
        for (const auto& row : Database::instance().fetchAll("SELECT * FROM users")) {
            User u;
            u.id           = row["id"].toInt();
            u.username     = row["username"].toString();
            u.passwordHash = row["password_hash"].toString();
            u.roles        = row["roles"].toString().split(',');
            u.isActive     = row["is_active"].toBool();
            m_users.append(u);
        }
    }
    
    void createDefaultAdminIfNeeded() {
        if (m_users.isEmpty()) {
            createUser("admin", "Admin@1234", {"admin", "user"});
        }
    }
    
    User* findUser(const QString& username) {
        for (auto& u : m_users) {
            if (u.username == username) return &u;
        }
        return nullptr;
    }
    
    void saveTokenToStorage() {
        QSettings s;
        s.setValue("auth/token", m_currentToken);
    }
    
    void clearStoredToken() {
        QSettings s;
        s.remove("auth/token");
    }
    
    QString loadTokenFromStorage() {
        return QSettings().value("auth/token").toString();
    }
    
    JwtManager m_jwt;
    QList<User> m_users;
    QString m_currentToken;
    std::optional<JwtManager::Claims> m_currentClaims;
};
```

---

## ขั้นตอนที่ 839: Login Dialog

```cpp
// ui/auth/logindialog.h
#pragma once
#include <QDialog>
#include "../../security/authservice.h"

class LoginDialog : public QDialog {
    Q_OBJECT
    
public:
    explicit LoginDialog(QWidget* parent = nullptr) : QDialog(parent) {
        setWindowTitle("เข้าสู่ระบบ");
        setWindowFlags(Qt::Dialog | Qt::FramelessWindowHint);
        setFixedSize(400, 480);
        setModal(true);
        
        setupUi();
        
        // Try auto-login from saved token
        if (AuthService::instance().tryAutoLogin()) {
            QTimer::singleShot(0, this, &QDialog::accept);
        }
    }
    
private slots:
    void attemptLogin() {
        QString username = m_userEdit->text().trimmed();
        QString password = m_passEdit->text();
        
        if (username.isEmpty() || password.isEmpty()) {
            showError("กรุณากรอกชื่อผู้ใช้และรหัสผ่าน");
            return;
        }
        
        setEnabled(false);
        m_loginBtn->setText("กำลังเข้าสู่ระบบ...");
        
        // Simulate async (in real app use QFuture)
        QTimer::singleShot(100, this, [=] {
            auto result = AuthService::instance().login(username, password);
            
            setEnabled(true);
            m_loginBtn->setText("เข้าสู่ระบบ");
            
            switch (result) {
                case AuthService::Success:
                    accept();
                    break;
                    
                case AuthService::InvalidCredentials:
                    showError("ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง");
                    m_passEdit->clear();
                    m_passEdit->setFocus();
                    shakeAnimation();
                    break;
                    
                case AuthService::AccountLocked:
                    showError("บัญชีถูกล็อค กรุณาลองใหม่ใน 15 นาที");
                    break;
                    
                case AuthService::AccountDisabled:
                    showError("บัญชีนี้ถูกปิดใช้งาน");
                    break;
                    
                default:
                    showError("เกิดข้อผิดพลาด กรุณาลองใหม่");
                    break;
            }
        });
    }
    
private:
    void setupUi() {
        auto* layout = new QVBoxLayout(this);
        layout->setContentsMargins(40, 40, 40, 40);
        layout->setSpacing(16);
        
        setStyleSheet(R"(
            QDialog { background: white; border-radius: 16px; }
            QLineEdit {
                border: 2px solid #E0E0E0;
                border-radius: 8px;
                padding: 12px;
                font-size: 14px;
            }
            QLineEdit:focus { border-color: #3498DB; }
            QPushButton#loginBtn {
                background: #3498DB;
                color: white;
                border: none;
                border-radius: 8px;
                padding: 14px;
                font-size: 15px;
                font-weight: bold;
            }
            QPushButton#loginBtn:hover { background: #2980B9; }
            QPushButton#loginBtn:disabled { background: #BDC3C7; }
        )");
        
        // Logo
        auto* logo = new QLabel("🏢", this);
        logo->setAlignment(Qt::AlignCenter);
        logo->setStyleSheet("font-size: 48px;");
        
        auto* title = new QLabel("ERP Pro", this);
        title->setAlignment(Qt::AlignCenter);
        title->setStyleSheet("font-size: 24px; font-weight: bold; color: #2C3E50;");
        
        auto* subtitle = new QLabel("กรุณาเข้าสู่ระบบเพื่อดำเนินการต่อ", this);
        subtitle->setAlignment(Qt::AlignCenter);
        subtitle->setStyleSheet("color: #7F8C8D;");
        
        m_userEdit = new QLineEdit(this);
        m_userEdit->setPlaceholderText("ชื่อผู้ใช้");
        m_userEdit->setMinimumHeight(48);
        
        m_passEdit = new QLineEdit(this);
        m_passEdit->setPlaceholderText("รหัสผ่าน");
        m_passEdit->setEchoMode(QLineEdit::Password);
        m_passEdit->setMinimumHeight(48);
        
        m_errorLabel = new QLabel(this);
        m_errorLabel->setStyleSheet("color: #E74C3C; font-size: 13px;");
        m_errorLabel->setAlignment(Qt::AlignCenter);
        m_errorLabel->setVisible(false);
        
        m_loginBtn = new QPushButton("เข้าสู่ระบบ", this);
        m_loginBtn->setObjectName("loginBtn");
        m_loginBtn->setMinimumHeight(48);
        m_loginBtn->setCursor(Qt::PointingHandCursor);
        
        auto* rememberCheck = new QCheckBox("จดจำการเข้าสู่ระบบ", this);
        rememberCheck->setStyleSheet("color: #7F8C8D;");
        
        layout->addWidget(logo);
        layout->addWidget(title);
        layout->addWidget(subtitle);
        layout->addSpacing(16);
        layout->addWidget(m_userEdit);
        layout->addWidget(m_passEdit);
        layout->addWidget(m_errorLabel);
        layout->addWidget(m_loginBtn);
        layout->addWidget(rememberCheck, 0, Qt::AlignCenter);
        layout->addStretch();
        
        connect(m_loginBtn, &QPushButton::clicked, this, &LoginDialog::attemptLogin);
        connect(m_passEdit, &QLineEdit::returnPressed, this, &LoginDialog::attemptLogin);
        connect(m_userEdit, &QLineEdit::returnPressed, m_passEdit, &QWidget::setFocus);
    }
    
    void showError(const QString& msg) {
        m_errorLabel->setText(msg);
        m_errorLabel->setVisible(true);
        
        // Auto-hide after 5 seconds
        QTimer::singleShot(5000, m_errorLabel, &QLabel::hide);
    }
    
    void shakeAnimation() {
        auto* anim = new QPropertyAnimation(this, "geometry", this);
        anim->setDuration(300);
        
        QRect r = geometry();
        anim->setKeyValueAt(0.0, r);
        anim->setKeyValueAt(0.1, r.translated(-10, 0));
        anim->setKeyValueAt(0.3, r.translated(10, 0));
        anim->setKeyValueAt(0.5, r.translated(-8, 0));
        anim->setKeyValueAt(0.7, r.translated(8, 0));
        anim->setKeyValueAt(0.9, r.translated(-4, 0));
        anim->setKeyValueAt(1.0, r);
        
        anim->start(QAbstractAnimation::DeleteWhenStopped);
    }
    
    QLineEdit* m_userEdit;
    QLineEdit* m_passEdit;
    QLabel* m_errorLabel;
    QPushButton* m_loginBtn;
};
```

---

## สรุป Part 058

ใน Part นี้คุณได้เรียนรู้:

1. ✅ PBKDF2-HMAC-SHA256 password hashing ด้วย QCryptographicHash
2. ✅ JWT token สร้าง/verify ด้วย HMAC-SHA256 + constant-time compare
3. ✅ AuthService — login, logout, auto-login, role checking, account locking
4. ✅ LoginDialog ด้วย form validation + shake animation + error messages
5. ✅ Security best practices (timing attack prevention, random salt)

---

⬅️ [Part 057](part057.md) | ➡️ [Part 059: World-Class — REST API Client](part059.md)
