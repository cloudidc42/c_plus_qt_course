# Part 006: การเขียนโปรแกรมเชิงวัตถุ (OOP) พื้นฐาน

## ขั้นตอนที่ 56-70

---

## ขั้นตอนที่ 56: Class และ Object

```cpp
#include <iostream>
#include <string>
#include <cmath>
using namespace std;

// === Class พื้นฐาน ===
class Circle {
// Private members (default)
private:
    double radius;
    string color;
    
// Public members
public:
    // Constructor
    Circle(double r = 1.0, string c = "red") 
        : radius(r), color(c) {}
    
    // Methods
    double area() const {
        return M_PI * radius * radius;
    }
    
    double circumference() const {
        return 2 * M_PI * radius;
    }
    
    // Getters (Accessors)
    double getRadius() const { return radius; }
    string getColor() const { return color; }
    
    // Setters (Mutators)
    void setRadius(double r) {
        if (r > 0) radius = r;
    }
    
    void setColor(const string& c) {
        color = c;
    }
    
    // Display
    void display() const {
        cout << "Circle { radius=" << radius 
             << ", color=" << color 
             << ", area=" << area() << " }" << endl;
    }
};

// === Class ที่ซับซ้อนขึ้น ===
class BankAccount {
private:
    string accountNumber;
    string ownerName;
    double balance;
    
    // Private helper method
    bool isValidAmount(double amount) const {
        return amount > 0;
    }
    
public:
    // Constructor
    BankAccount(string accNum, string owner, double initialBalance = 0)
        : accountNumber(accNum), ownerName(owner), balance(initialBalance) {
        cout << "สร้างบัญชี " << accountNumber << " สำหรับ " << ownerName << endl;
    }
    
    // Destructor
    ~BankAccount() {
        cout << "ปิดบัญชี " << accountNumber << endl;
    }
    
    // Methods
    bool deposit(double amount) {
        if (!isValidAmount(amount)) {
            cout << "จำนวนเงินไม่ถูกต้อง" << endl;
            return false;
        }
        balance += amount;
        cout << "ฝากเงิน " << amount << " บาท | ยอดคงเหลือ: " << balance << endl;
        return true;
    }
    
    bool withdraw(double amount) {
        if (!isValidAmount(amount)) {
            cout << "จำนวนเงินไม่ถูกต้อง" << endl;
            return false;
        }
        if (amount > balance) {
            cout << "ยอดเงินไม่เพียงพอ" << endl;
            return false;
        }
        balance -= amount;
        cout << "ถอนเงิน " << amount << " บาท | ยอดคงเหลือ: " << balance << endl;
        return true;
    }
    
    // Getters
    double getBalance() const { return balance; }
    string getAccountNumber() const { return accountNumber; }
    string getOwnerName() const { return ownerName; }
    
    void printStatement() const {
        cout << "===========================" << endl;
        cout << "บัญชี: " << accountNumber << endl;
        cout << "เจ้าของ: " << ownerName << endl;
        cout << "ยอดคงเหลือ: " << balance << " บาท" << endl;
        cout << "===========================" << endl;
    }
};

int main() {
    // === Circle Objects ===
    cout << "=== Circle Objects ===" << endl;
    
    Circle c1;                    // Default constructor
    Circle c2(5.0);               // radius=5
    Circle c3(3.0, "blue");       // radius=3, color=blue
    
    c1.display();
    c2.display();
    c3.display();
    
    // เปลี่ยนค่า
    c1.setRadius(7.5);
    c1.setColor("green");
    cout << "\nหลังแก้ไข:" << endl;
    c1.display();
    
    // === BankAccount Objects ===
    cout << "\n=== BankAccount Objects ===" << endl;
    
    {
        BankAccount acc("TH001", "สมชาย ใจดี", 1000);
        
        acc.deposit(5000);
        acc.withdraw(2000);
        acc.withdraw(10000);  // ไม่เพียงพอ
        acc.deposit(-100);    // ไม่ถูกต้อง
        
        acc.printStatement();
    }
    // destructor ถูกเรียกเมื่อออกจาก scope
    
    return 0;
}
```

---

## ขั้นตอนที่ 57: Constructors และ Destructors

```cpp
#include <iostream>
#include <string>
using namespace std;

class Student {
private:
    string name;
    int age;
    double gpa;
    int* scores;
    int numScores;
    static int count;  // นับจำนวน object ทั้งหมด
    
public:
    // === Default Constructor ===
    Student() 
        : name("ไม่ทราบ"), age(0), gpa(0.0), scores(nullptr), numScores(0) {
        count++;
        cout << "[Default Constructor] นักเรียน #" << count << endl;
    }
    
    // === Parameterized Constructor ===
    Student(string n, int a, double g)
        : name(n), age(a), gpa(g), scores(nullptr), numScores(0) {
        count++;
        cout << "[Param Constructor] " << name << " #" << count << endl;
    }
    
    // === Constructor กับ Dynamic Array ===
    Student(string n, int a, int* sc, int num)
        : name(n), age(a), gpa(0.0), numScores(num) {
        count++;
        scores = new int[numScores];
        for (int i = 0; i < numScores; i++) {
            scores[i] = sc[i];
            gpa += sc[i];
        }
        gpa /= numScores;
        cout << "[Array Constructor] " << name << " #" << count << endl;
    }
    
    // === Copy Constructor ===
    Student(const Student& other)
        : name(other.name), age(other.age), gpa(other.gpa), 
          numScores(other.numScores) {
        count++;
        if (other.scores) {
            scores = new int[numScores];
            for (int i = 0; i < numScores; i++) {
                scores[i] = other.scores[i];
            }
        } else {
            scores = nullptr;
        }
        cout << "[Copy Constructor] " << name << " #" << count << endl;
    }
    
    // === Move Constructor (C++11) ===
    Student(Student&& other) noexcept
        : name(move(other.name)), age(other.age), gpa(other.gpa),
          scores(other.scores), numScores(other.numScores) {
        count++;
        other.scores = nullptr;  // โอน ownership
        other.numScores = 0;
        cout << "[Move Constructor] " << name << " #" << count << endl;
    }
    
    // === Destructor ===
    ~Student() {
        delete[] scores;
        count--;
        cout << "[Destructor] " << name << " | remaining: " << count << endl;
    }
    
    // === Assignment Operator ===
    Student& operator=(const Student& other) {
        if (this != &other) {  // Self-assignment check
            delete[] scores;
            
            name = other.name;
            age = other.age;
            gpa = other.gpa;
            numScores = other.numScores;
            
            if (other.scores) {
                scores = new int[numScores];
                for (int i = 0; i < numScores; i++) {
                    scores[i] = other.scores[i];
                }
            } else {
                scores = nullptr;
            }
        }
        return *this;
    }
    
    // Getters
    string getName() const { return name; }
    int getAge() const { return age; }
    double getGpa() const { return gpa; }
    static int getCount() { return count; }
    
    void display() const {
        cout << "นักเรียน: " << name << " | อายุ: " << age << " | GPA: " << gpa;
        if (scores && numScores > 0) {
            cout << " | คะแนน: [";
            for (int i = 0; i < numScores; i++) {
                cout << scores[i];
                if (i < numScores - 1) cout << ", ";
            }
            cout << "]";
        }
        cout << endl;
    }
};

// กำหนดค่าเริ่มต้น static member
int Student::count = 0;

// === Initializer List ===
class Point3D {
    double x, y, z;
public:
    Point3D(double x, double y, double z) : x(x), y(y), z(z) {}
    
    double distanceTo(const Point3D& other) const {
        return sqrt(pow(x - other.x, 2) + 
                    pow(y - other.y, 2) + 
                    pow(z - other.z, 2));
    }
    
    void print() const {
        cout << "(" << x << ", " << y << ", " << z << ")" << endl;
    }
};

int main() {
    cout << "=== Constructors & Destructors ===" << endl;
    cout << "จำนวน Student: " << Student::getCount() << endl;
    
    {
        Student s1;                           // Default
        Student s2("สมชาย", 20, 3.5);        // Param
        
        int sc[] = {80, 90, 85, 95};
        Student s3("สมหญิง", 19, sc, 4);      // Array
        
        cout << "\n--- ข้อมูลนักเรียน ---" << endl;
        s1.display();
        s2.display();
        s3.display();
        
        cout << "\nจำนวน Student: " << Student::getCount() << endl;
        
        Student s4 = s3;                      // Copy Constructor
        s4.display();
        
        cout << "\nจำนวน Student: " << Student::getCount() << endl;
    }  // Destructors ถูกเรียกที่นี่
    
    cout << "\nจำนวน Student หลัง scope: " << Student::getCount() << endl;
    
    // === Point3D ===
    cout << "\n=== Point3D ===" << endl;
    
    Point3D p1(1.0, 2.0, 3.0);
    Point3D p2(4.0, 6.0, 3.0);
    
    p1.print();
    p2.print();
    cout << "ระยะห่าง: " << p1.distanceTo(p2) << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 58: Encapsulation และ Access Modifiers

```cpp
#include <iostream>
#include <string>
#include <regex>
using namespace std;

// === Encapsulation ตัวอย่าง ===
class Person {
private:
    string name;
    int age;
    string email;
    double salary;
    
    // Validation helpers
    bool isValidAge(int a) const {
        return a >= 0 && a <= 150;
    }
    
    bool isValidEmail(const string& e) const {
        // Simple email validation
        return e.find('@') != string::npos && e.find('.') != string::npos;
    }
    
    bool isValidSalary(double s) const {
        return s >= 0;
    }
    
protected:
    string department;  // เข้าถึงได้จาก derived class
    
public:
    // Constructor
    Person(const string& n, int a, const string& e, double s)
        : department("ทั่วไป") {
        setName(n);
        setAge(a);
        setEmail(e);
        setSalary(s);
    }
    
    // Getters
    string getName() const { return name; }
    int getAge() const { return age; }
    string getEmail() const { return email; }
    double getSalary() const { return salary; }
    string getDepartment() const { return department; }
    
    // Setters กับ Validation
    void setName(const string& n) {
        if (!n.empty()) {
            name = n;
        } else {
            cout << "Error: ชื่อต้องไม่ว่าง" << endl;
        }
    }
    
    void setAge(int a) {
        if (isValidAge(a)) {
            age = a;
        } else {
            cout << "Error: อายุไม่ถูกต้อง (0-150)" << endl;
            age = 0;
        }
    }
    
    void setEmail(const string& e) {
        if (isValidEmail(e)) {
            email = e;
        } else {
            cout << "Error: Email ไม่ถูกต้อง: " << e << endl;
            email = "";
        }
    }
    
    void setSalary(double s) {
        if (isValidSalary(s)) {
            salary = s;
        } else {
            cout << "Error: เงินเดือนต้องไม่ติดลบ" << endl;
            salary = 0;
        }
    }
    
    void giveRaise(double percent) {
        if (percent > 0 && percent <= 100) {
            salary *= (1 + percent / 100);
            cout << name << " ได้รับการขึ้นเงินเดือน " << percent << "%" << endl;
        }
    }
    
    void display() const {
        cout << "=== ข้อมูลพนักงาน ===" << endl;
        cout << "ชื่อ: " << name << endl;
        cout << "อายุ: " << age << " ปี" << endl;
        cout << "Email: " << email << endl;
        cout << "แผนก: " << department << endl;
        cout << "เงินเดือน: " << salary << " บาท" << endl;
    }
};

// === Struct vs Class ===
// Struct: default public
struct Point {
    double x, y;  // public by default
    
    double distanceTo(const Point& other) const {
        return sqrt(pow(x - other.x, 2) + pow(y - other.y, 2));
    }
};

// Class: default private
class Vector2D {
public:
    double x, y;  // explicit public
    
    Vector2D(double x = 0, double y = 0) : x(x), y(y) {}
    
    Vector2D operator+(const Vector2D& other) const {
        return Vector2D(x + other.x, y + other.y);
    }
    
    double magnitude() const {
        return sqrt(x * x + y * y);
    }
    
    void normalize() {
        double mag = magnitude();
        if (mag > 0) {
            x /= mag;
            y /= mag;
        }
    }
    
    void print() const {
        cout << "(" << x << ", " << y << ")" << endl;
    }
};

int main() {
    // === Person ===
    cout << "=== Encapsulation ===" << endl;
    
    Person emp("สมชาย ใจดี", 35, "somchai@example.com", 45000);
    emp.display();
    
    cout << "\n--- ทดสอบ Validation ---" << endl;
    emp.setAge(-5);         // Error
    emp.setAge(200);        // Error
    emp.setEmail("invalid"); // Error
    emp.setSalary(-1000);   // Error
    
    cout << "\n--- ขึ้นเงินเดือน ---" << endl;
    emp.giveRaise(10);
    emp.display();
    
    // === Struct ===
    cout << "\n=== Struct Point ===" << endl;
    
    Point p1 = {0, 0};
    Point p2 = {3, 4};
    
    cout << "ระยะห่าง: " << p1.distanceTo(p2) << endl;
    
    // === Vector2D ===
    cout << "\n=== Vector2D ===" << endl;
    
    Vector2D v1(3, 4);
    Vector2D v2(1, 2);
    
    cout << "v1 = "; v1.print();
    cout << "v2 = "; v2.print();
    cout << "|v1| = " << v1.magnitude() << endl;
    
    Vector2D v3 = v1 + v2;
    cout << "v1 + v2 = "; v3.print();
    
    v1.normalize();
    cout << "v1 normalized = "; v1.print();
    cout << "|v1 normalized| = " << v1.magnitude() << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 59: Operator Overloading

```cpp
#include <iostream>
#include <string>
#include <cmath>
using namespace std;

// === Complex Number Class ===
class Complex {
private:
    double real, imag;
    
public:
    Complex(double r = 0, double i = 0) : real(r), imag(i) {}
    
    // Getters
    double getReal() const { return real; }
    double getImag() const { return imag; }
    
    // Arithmetic Operators
    Complex operator+(const Complex& other) const {
        return Complex(real + other.real, imag + other.imag);
    }
    
    Complex operator-(const Complex& other) const {
        return Complex(real - other.real, imag - other.imag);
    }
    
    Complex operator*(const Complex& other) const {
        return Complex(
            real * other.real - imag * other.imag,
            real * other.imag + imag * other.real
        );
    }
    
    Complex operator/(const Complex& other) const {
        double denom = other.real * other.real + other.imag * other.imag;
        return Complex(
            (real * other.real + imag * other.imag) / denom,
            (imag * other.real - real * other.imag) / denom
        );
    }
    
    // Unary operators
    Complex operator-() const {
        return Complex(-real, -imag);
    }
    
    Complex operator+() const {
        return *this;
    }
    
    // Comparison
    bool operator==(const Complex& other) const {
        return real == other.real && imag == other.imag;
    }
    
    bool operator!=(const Complex& other) const {
        return !(*this == other);
    }
    
    // Assignment operators
    Complex& operator+=(const Complex& other) {
        real += other.real;
        imag += other.imag;
        return *this;
    }
    
    // Magnitude
    double magnitude() const {
        return sqrt(real * real + imag * imag);
    }
    
    Complex conjugate() const {
        return Complex(real, -imag);
    }
    
    // Stream operators (friend functions)
    friend ostream& operator<<(ostream& os, const Complex& c) {
        os << c.real;
        if (c.imag >= 0) os << "+";
        os << c.imag << "i";
        return os;
    }
    
    friend istream& operator>>(istream& is, Complex& c) {
        is >> c.real >> c.imag;
        return is;
    }
};

// === Matrix Class ===
class Matrix {
private:
    int rows, cols;
    double** data;
    
public:
    Matrix(int r, int c, double initVal = 0) : rows(r), cols(c) {
        data = new double*[rows];
        for (int i = 0; i < rows; i++) {
            data[i] = new double[cols];
            fill(data[i], data[i] + cols, initVal);
        }
    }
    
    Matrix(const Matrix& other) : rows(other.rows), cols(other.cols) {
        data = new double*[rows];
        for (int i = 0; i < rows; i++) {
            data[i] = new double[cols];
            copy(other.data[i], other.data[i] + cols, data[i]);
        }
    }
    
    ~Matrix() {
        for (int i = 0; i < rows; i++) delete[] data[i];
        delete[] data;
    }
    
    // [] operator
    double* operator[](int i) { return data[i]; }
    const double* operator[](int i) const { return data[i]; }
    
    // + operator
    Matrix operator+(const Matrix& other) const {
        if (rows != other.rows || cols != other.cols) {
            throw invalid_argument("Matrix sizes don't match");
        }
        Matrix result(rows, cols);
        for (int i = 0; i < rows; i++)
            for (int j = 0; j < cols; j++)
                result[i][j] = data[i][j] + other.data[i][j];
        return result;
    }
    
    // * operator (matrix multiplication)
    Matrix operator*(const Matrix& other) const {
        if (cols != other.rows) {
            throw invalid_argument("Invalid matrix dimensions for multiplication");
        }
        Matrix result(rows, other.cols);
        for (int i = 0; i < rows; i++)
            for (int j = 0; j < other.cols; j++)
                for (int k = 0; k < cols; k++)
                    result[i][j] += data[i][k] * other.data[k][j];
        return result;
    }
    
    // Stream operator
    friend ostream& operator<<(ostream& os, const Matrix& m) {
        for (int i = 0; i < m.rows; i++) {
            os << "[ ";
            for (int j = 0; j < m.cols; j++) {
                os.width(6);
                os << m.data[i][j];
            }
            os << " ]" << endl;
        }
        return os;
    }
    
    int getRows() const { return rows; }
    int getCols() const { return cols; }
};

int main() {
    // === Complex Numbers ===
    cout << "=== Complex Numbers ===" << endl;
    
    Complex c1(3, 4);
    Complex c2(1, -2);
    
    cout << "c1 = " << c1 << endl;
    cout << "c2 = " << c2 << endl;
    cout << "c1 + c2 = " << (c1 + c2) << endl;
    cout << "c1 - c2 = " << (c1 - c2) << endl;
    cout << "c1 * c2 = " << (c1 * c2) << endl;
    cout << "c1 / c2 = " << (c1 / c2) << endl;
    cout << "|c1| = " << c1.magnitude() << endl;
    cout << "conj(c1) = " << c1.conjugate() << endl;
    cout << "c1 == c2: " << boolalpha << (c1 == c2) << endl;
    
    // === Matrix ===
    cout << "\n=== Matrix ===" << endl;
    
    Matrix A(2, 3);
    A[0][0] = 1; A[0][1] = 2; A[0][2] = 3;
    A[1][0] = 4; A[1][1] = 5; A[1][2] = 6;
    
    Matrix B(2, 3);
    B[0][0] = 7; B[0][1] = 8; B[0][2] = 9;
    B[1][0] = 10; B[1][1] = 11; B[1][2] = 12;
    
    cout << "A:" << endl << A;
    cout << "B:" << endl << B;
    cout << "A + B:" << endl << (A + B);
    
    Matrix C(3, 2);
    C[0][0] = 1; C[0][1] = 2;
    C[1][0] = 3; C[1][1] = 4;
    C[2][0] = 5; C[2][1] = 6;
    
    cout << "A * C:" << endl << (A * C);
    
    return 0;
}
```

---

## ขั้นตอนที่ 60: Static Members และ Friend

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

// === Static Members ===
class Counter {
private:
    static int totalObjects;
    static int maxObjects;
    int id;
    string name;
    
public:
    Counter(const string& n) : name(n) {
        if (totalObjects >= maxObjects) {
            cout << "Error: ถึงจำนวนสูงสุดแล้ว!" << endl;
            return;
        }
        id = ++totalObjects;
        cout << "สร้าง " << name << " (ID=" << id << ", total=" << totalObjects << ")" << endl;
    }
    
    ~Counter() {
        totalObjects--;
        cout << "ลบ " << name << " (ID=" << id << ", total=" << totalObjects << ")" << endl;
    }
    
    // Static methods
    static int getTotal() { return totalObjects; }
    static void setMaxObjects(int max) { maxObjects = max; }
    
    int getID() const { return id; }
    string getName() const { return name; }
};

// กำหนดค่า static members
int Counter::totalObjects = 0;
int Counter::maxObjects = 5;

// === Friend Functions ===
class Rectangle;  // Forward declaration

class Square {
private:
    double side;
    
public:
    Square(double s) : side(s) {}
    double getArea() const { return side * side; }
    double getSide() const { return side; }
    
    // friend function สามารถเข้าถึง private members
    friend bool isLarger(const Square& sq, const Rectangle& rect);
    friend void swapSizes(Square& sq, Rectangle& rect);
};

class Rectangle {
private:
    double width, height;
    
public:
    Rectangle(double w, double h) : width(w), height(h) {}
    double getArea() const { return width * height; }
    double getWidth() const { return width; }
    double getHeight() const { return height; }
    
    friend bool isLarger(const Square& sq, const Rectangle& rect);
    friend void swapSizes(Square& sq, Rectangle& rect);
    
    // Friend class
    friend class ShapeCalculator;
};

// Friend functions
bool isLarger(const Square& sq, const Rectangle& rect) {
    return sq.side * sq.side > rect.width * rect.height;
}

void swapSizes(Square& sq, Rectangle& rect) {
    double sqSide = sq.side;
    double rectW = rect.width;
    double rectH = rect.height;
    
    sq.side = sqrt(rectW * rectH);
    rect.width = sqSide;
    rect.height = sqSide;
}

// Friend class
class ShapeCalculator {
public:
    static double totalArea(const Rectangle& r1, const Rectangle& r2) {
        // เข้าถึง private members ของ Rectangle
        return (r1.width * r1.height) + (r2.width * r2.height);
    }
    
    static double combinedPerimeter(const Rectangle& r) {
        return 2 * (r.width + r.height);
    }
};

// === Singleton Pattern ===
class Database {
private:
    static Database* instance;
    string connectionString;
    bool connected;
    
    // Private constructor
    Database() : connected(false) {
        cout << "Database instance created" << endl;
    }
    
    // Delete copy/move
    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;
    
public:
    static Database* getInstance() {
        if (!instance) {
            instance = new Database();
        }
        return instance;
    }
    
    void connect(const string& connStr) {
        connectionString = connStr;
        connected = true;
        cout << "เชื่อมต่อ: " << connectionString << endl;
    }
    
    void disconnect() {
        connected = false;
        cout << "ตัดการเชื่อมต่อ" << endl;
    }
    
    bool isConnected() const { return connected; }
    
    void query(const string& sql) {
        if (connected) {
            cout << "Query: " << sql << endl;
        } else {
            cout << "Error: ไม่ได้เชื่อมต่อ!" << endl;
        }
    }
    
    static void destroyInstance() {
        delete instance;
        instance = nullptr;
    }
};

Database* Database::instance = nullptr;

int main() {
    // === Static Members ===
    cout << "=== Static Members ===" << endl;
    Counter::setMaxObjects(3);
    
    cout << "Total: " << Counter::getTotal() << endl;
    
    {
        Counter c1("Object1");
        Counter c2("Object2");
        Counter c3("Object3");
        Counter c4("Object4");  // เกินจำนวนสูงสุด
        
        cout << "Total in scope: " << Counter::getTotal() << endl;
    }
    
    cout << "Total after scope: " << Counter::getTotal() << endl;
    
    // === Friend Functions ===
    cout << "\n=== Friend Functions ===" << endl;
    
    Square sq(5);
    Rectangle rect(4, 7);
    
    cout << "Square area: " << sq.getArea() << endl;
    cout << "Rectangle area: " << rect.getArea() << endl;
    cout << "Square is larger: " << boolalpha << isLarger(sq, rect) << endl;
    
    swapSizes(sq, rect);
    cout << "\nหลัง swap:" << endl;
    cout << "Square area: " << sq.getArea() << endl;
    cout << "Rectangle area: " << rect.getArea() << endl;
    
    // === Friend Class ===
    cout << "\n=== Friend Class ===" << endl;
    
    Rectangle r1(3, 4), r2(5, 6);
    cout << "Total area: " << ShapeCalculator::totalArea(r1, r2) << endl;
    cout << "r1 perimeter: " << ShapeCalculator::combinedPerimeter(r1) << endl;
    
    // === Singleton ===
    cout << "\n=== Singleton Pattern ===" << endl;
    
    Database* db1 = Database::getInstance();
    Database* db2 = Database::getInstance();
    
    cout << "db1 == db2: " << (db1 == db2 ? "true" : "false") << endl;
    
    db1->connect("mysql://localhost/mydb");
    db2->query("SELECT * FROM users");
    
    Database::destroyInstance();
    
    return 0;
}
```

---

## สรุป Part 006

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Class และ Object พื้นฐาน
2. ✅ Constructors (Default, Parameterized, Copy, Move)
3. ✅ Destructors
4. ✅ Encapsulation และ Access Modifiers
5. ✅ Getters และ Setters พร้อม Validation
6. ✅ Operator Overloading
7. ✅ Static Members และ Methods
8. ✅ Friend Functions และ Friend Classes
9. ✅ Singleton Pattern

---

## แบบฝึกหัด Part 006

**ฝึกหัดที่ 1:** สร้าง Class `Temperature` ที่รองรับ Celsius, Fahrenheit, Kelvin พร้อม Operator Overloading

**ฝึกหัดที่ 2:** สร้าง Class `Stack<T>` ที่ใช้ Dynamic Array

**ฝึกหัดที่ 3:** สร้าง Class `Date` พร้อม Operator `+`, `-`, `<<`, `>>`

---

⬅️ **ก่อนหน้า:** [Part 005](part005.md) | ➡️ **ต่อไป:** [Part 007: Inheritance & Polymorphism](part007.md)
