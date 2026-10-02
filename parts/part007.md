# Part 007: Inheritance และ Polymorphism

## ขั้นตอนที่ 71-85

---

## ขั้นตอนที่ 71: Inheritance พื้นฐาน

```cpp
#include <iostream>
#include <string>
using namespace std;

// === Base Class ===
class Animal {
protected:
    string name;
    int age;
    double weight;
    
public:
    Animal(const string& n, int a, double w)
        : name(n), age(a), weight(w) {
        cout << "Animal created: " << name << endl;
    }
    
    virtual ~Animal() {
        cout << "Animal destroyed: " << name << endl;
    }
    
    // Virtual function = สามารถ Override ได้
    virtual void speak() const {
        cout << name << ": (ไม่มีเสียง)" << endl;
    }
    
    virtual void move() const {
        cout << name << ": เคลื่อนที่" << endl;
    }
    
    virtual string getType() const {
        return "Animal";
    }
    
    void eat(const string& food) const {
        cout << name << " กิน " << food << endl;
    }
    
    void display() const {
        cout << "[" << getType() << "] " << name 
             << " | อายุ: " << age 
             << " | น้ำหนัก: " << weight << " กก." << endl;
    }
    
    string getName() const { return name; }
    int getAge() const { return age; }
    double getWeight() const { return weight; }
};

// === Derived Classes ===
class Dog : public Animal {
private:
    string breed;
    bool trained;
    
public:
    Dog(const string& n, int a, double w, const string& b, bool t = false)
        : Animal(n, a, w), breed(b), trained(t) {
        cout << "Dog created: " << name << endl;
    }
    
    ~Dog() override {
        cout << "Dog destroyed: " << name << endl;
    }
    
    void speak() const override {
        cout << name << ": โฮ่ง! โฮ่ง!" << endl;
    }
    
    void move() const override {
        cout << name << ": วิ่งด้วย 4 ขา" << endl;
    }
    
    string getType() const override {
        return "Dog";
    }
    
    // Dog-specific methods
    void fetch(const string& item) const {
        cout << name << " วิ่งไปหยิบ " << item << endl;
    }
    
    void doTrick() const {
        if (trained) {
            cout << name << " ทำ trick: นั่ง, ยืน, กลิ้ง!" << endl;
        } else {
            cout << name << " ยังไม่ได้ฝึก" << endl;
        }
    }
    
    string getBreed() const { return breed; }
};

class Cat : public Animal {
private:
    bool indoor;
    
public:
    Cat(const string& n, int a, double w, bool i = true)
        : Animal(n, a, w), indoor(i) {}
    
    void speak() const override {
        cout << name << ": เมี้ยว~ เมี้ยว~" << endl;
    }
    
    void move() const override {
        cout << name << ": เดินเบาๆ อย่างสง่า" << endl;
    }
    
    string getType() const override {
        return "Cat";
    }
    
    void purr() const {
        cout << name << ": Purrrr..." << endl;
    }
};

class Bird : public Animal {
private:
    bool canFly;
    double wingspan;
    
public:
    Bird(const string& n, int a, double w, bool fly, double ws)
        : Animal(n, a, w), canFly(fly), wingspan(ws) {}
    
    void speak() const override {
        cout << name << ": จิ้บๆ หรือ แซ่บ!" << endl;
    }
    
    void move() const override {
        if (canFly) {
            cout << name << ": บินด้วย wingspan " << wingspan << " ม." << endl;
        } else {
            cout << name << ": เดิน (บินไม่ได้)" << endl;
        }
    }
    
    string getType() const override {
        return "Bird";
    }
};

// === Polymorphism ===
void makeAnimalSpeak(const Animal& animal) {
    animal.speak();  // เรียก virtual function ที่ถูกต้อง
}

void displayAnimal(Animal* animal) {
    if (animal) {
        animal->display();
        animal->speak();
        animal->move();
        cout << "---" << endl;
    }
}

int main() {
    // === สร้าง Objects ===
    cout << "=== สร้าง Objects ===" << endl;
    
    Dog dog("บัดดี้", 3, 15.5, "โกลเด้น รีทรีฟเวอร์", true);
    Cat cat("มิ้นต์", 2, 4.0, true);
    Bird bird("พีคอก", 5, 6.0, true, 1.8);
    
    cout << "\n=== แสดงข้อมูล ===" << endl;
    dog.display();
    cat.display();
    bird.display();
    
    cout << "\n=== Polymorphism ===" << endl;
    
    // Array ของ pointers
    Animal* animals[] = { &dog, &cat, &bird };
    
    for (Animal* a : animals) {
        displayAnimal(a);
    }
    
    // === Dog-specific ===
    cout << "=== Dog Methods ===" << endl;
    dog.fetch("ลูกบอล");
    dog.doTrick();
    
    // Cat-specific
    cout << "\n=== Cat Methods ===" << endl;
    cat.purr();
    
    // === Upcasting / Downcasting ===
    cout << "\n=== Casting ===" << endl;
    
    Animal* ptr = &dog;       // Upcast: Dog* -> Animal*
    ptr->speak();              // เรียก Dog::speak()
    
    // Downcast (Dynamic)
    Dog* dogPtr = dynamic_cast<Dog*>(ptr);
    if (dogPtr) {
        dogPtr->fetch("กระดูก");  // เรียก method เฉพาะของ Dog
    }
    
    Cat* catPtr = dynamic_cast<Cat*>(ptr);  // จะเป็น nullptr
    if (!catPtr) {
        cout << "ptr ไม่ใช่ Cat" << endl;
    }
    
    return 0;
}
```

---

## ขั้นตอนที่ 72: Virtual Functions และ Abstract Classes

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <cmath>
using namespace std;

// === Abstract Class ===
class Shape {
protected:
    string color;
    
public:
    Shape(const string& c = "black") : color(c) {}
    
    virtual ~Shape() = default;
    
    // Pure virtual functions = abstract
    virtual double area() const = 0;
    virtual double perimeter() const = 0;
    virtual string getShapeName() const = 0;
    virtual void draw() const = 0;
    
    // Concrete methods
    void setColor(const string& c) { color = c; }
    string getColor() const { return color; }
    
    void describe() const {
        cout << getShapeName() << " [" << color << "]"
             << " | พื้นที่: " << area()
             << " | เส้นรอบรูป: " << perimeter() << endl;
    }
};

// === Concrete Classes ===
class Circle : public Shape {
private:
    double radius;
    
public:
    Circle(double r, const string& c = "red")
        : Shape(c), radius(r) {}
    
    double area() const override {
        return M_PI * radius * radius;
    }
    
    double perimeter() const override {
        return 2 * M_PI * radius;
    }
    
    string getShapeName() const override {
        return "วงกลม";
    }
    
    void draw() const override {
        int r = static_cast<int>(radius);
        cout << "วาดวงกลม รัศมี=" << radius << ":" << endl;
        for (int y = -r; y <= r; y++) {
            for (int x = -r * 2; x <= r * 2; x++) {
                double dist = sqrt((double)x*x/4 + (double)y*y);
                if (abs(dist - r) < 0.5) cout << "*";
                else cout << " ";
            }
            cout << endl;
        }
    }
};

class Rectangle : public Shape {
private:
    double width, height;
    
public:
    Rectangle(double w, double h, const string& c = "blue")
        : Shape(c), width(w), height(h) {}
    
    double area() const override {
        return width * height;
    }
    
    double perimeter() const override {
        return 2 * (width + height);
    }
    
    string getShapeName() const override {
        return "สี่เหลี่ยมผืนผ้า";
    }
    
    void draw() const override {
        int w = static_cast<int>(width);
        int h = static_cast<int>(height);
        cout << "วาดสี่เหลี่ยม " << w << "x" << h << ":" << endl;
        for (int i = 0; i < h; i++) {
            for (int j = 0; j < w; j++) {
                if (i == 0 || i == h-1 || j == 0 || j == w-1) cout << "*";
                else cout << " ";
            }
            cout << endl;
        }
    }
};

class Triangle : public Shape {
private:
    double a, b, c;  // ด้าน 3 ด้าน
    
public:
    Triangle(double a, double b, double c, const string& col = "green")
        : Shape(col), a(a), b(b), c(c) {}
    
    double area() const override {
        double s = (a + b + c) / 2;  // Heron's formula
        return sqrt(s * (s-a) * (s-b) * (s-c));
    }
    
    double perimeter() const override {
        return a + b + c;
    }
    
    string getShapeName() const override {
        return "สามเหลี่ยม";
    }
    
    void draw() const override {
        int h = static_cast<int>(a);
        cout << "วาดสามเหลี่ยม (" << a << ", " << b << ", " << c << "):" << endl;
        for (int i = 1; i <= h; i++) {
            int spaces = h - i;
            for (int j = 0; j < spaces; j++) cout << " ";
            for (int j = 0; j < 2*i - 1; j++) {
                if (j == 0 || j == 2*i-2 || i == h) cout << "*";
                else cout << " ";
            }
            cout << endl;
        }
    }
};

// Canvas ที่รับ Shape ใดก็ได้
class Canvas {
private:
    vector<Shape*> shapes;
    
public:
    void addShape(Shape* shape) {
        shapes.push_back(shape);
    }
    
    void drawAll() const {
        cout << "=== Canvas ===" << endl;
        for (Shape* s : shapes) {
            s->draw();
            cout << endl;
        }
    }
    
    void describeAll() const {
        cout << "=== รูปร่างทั้งหมด ===" << endl;
        for (Shape* s : shapes) {
            s->describe();
        }
    }
    
    double totalArea() const {
        double total = 0;
        for (Shape* s : shapes) total += s->area();
        return total;
    }
};

int main() {
    // Cannot instantiate abstract class
    // Shape s;  // Error!
    
    // === Shapes ===
    Circle c(5, "red");
    Rectangle r(8, 4, "blue");
    Triangle t(5, 5, 6, "green");
    
    // === Canvas ===
    Canvas canvas;
    canvas.addShape(&c);
    canvas.addShape(&r);
    canvas.addShape(&t);
    
    canvas.describeAll();
    cout << "\nพื้นที่รวม: " << canvas.totalArea() << endl;
    
    cout << "\n";
    canvas.drawAll();
    
    return 0;
}
```

---

## ขั้นตอนที่ 73: Multiple Inheritance

```cpp
#include <iostream>
#include <string>
using namespace std;

// === Multiple Inheritance ===

class Flyable {
public:
    virtual void fly() const {
        cout << "กำลังบิน..." << endl;
    }
    
    virtual double getMaxAltitude() const = 0;
    
    virtual ~Flyable() = default;
};

class Swimmable {
public:
    virtual void swim() const {
        cout << "กำลังว่ายน้ำ..." << endl;
    }
    
    virtual double getMaxDepth() const = 0;
    
    virtual ~Swimmable() = default;
};

class Walkable {
public:
    virtual void walk() const {
        cout << "กำลังเดิน..." << endl;
    }
    
    virtual double getSpeed() const = 0;
    
    virtual ~Walkable() = default;
};

// === Classes ที่สืบทอดจากหลาย Interface ===
class Duck : public Walkable, public Flyable, public Swimmable {
private:
    string name;
    
public:
    Duck(const string& n) : name(n) {}
    
    void fly() const override {
        cout << name << ": บินด้วยปีก..." << endl;
    }
    
    void swim() const override {
        cout << name << ": ว่ายน้ำด้วยเท้าพัง..." << endl;
    }
    
    void walk() const override {
        cout << name << ": เดินแบบเป็ด ปิ้งๆ" << endl;
    }
    
    double getMaxAltitude() const override { return 100.0; }
    double getMaxDepth() const override { return 5.0; }
    double getSpeed() const override { return 5.0; }
    
    void quack() const {
        cout << name << ": Quack! Quack!" << endl;
    }
};

class FlyingFish : public Flyable, public Swimmable {
private:
    string name;
    
public:
    FlyingFish(const string& n) : name(n) {}
    
    void fly() const override {
        cout << name << ": กระโดดขึ้นฟ้า..." << endl;
    }
    
    void swim() const override {
        cout << name << ": ว่ายน้ำได้เร็ว..." << endl;
    }
    
    double getMaxAltitude() const override { return 2.0; }
    double getMaxDepth() const override { return 50.0; }
};

// === Diamond Problem ===
class Base {
public:
    int value;
    Base(int v = 0) : value(v) {
        cout << "Base constructor, value=" << v << endl;
    }
    
    virtual void show() {
        cout << "Base::show() value=" << value << endl;
    }
};

class Left : virtual public Base {
public:
    Left() : Base(10) {
        cout << "Left constructor" << endl;
    }
    
    void show() override {
        cout << "Left::show()" << endl;
        Base::show();
    }
};

class Right : virtual public Base {
public:
    Right() : Base(20) {
        cout << "Right constructor" << endl;
    }
    
    void show() override {
        cout << "Right::show()" << endl;
        Base::show();
    }
};

class Diamond : public Left, public Right {
public:
    Diamond() : Base(30) {  // ระบุ Base ตรงๆ
        cout << "Diamond constructor" << endl;
    }
    
    void show() override {
        cout << "Diamond::show()" << endl;
        Base::show();
    }
};

int main() {
    // === Duck (Multiple Inheritance) ===
    cout << "=== Duck ===" << endl;
    
    Duck duck("โดนัลด์");
    duck.fly();
    duck.swim();
    duck.walk();
    duck.quack();
    
    cout << "Max altitude: " << duck.getMaxAltitude() << " ม." << endl;
    cout << "Max depth: " << duck.getMaxDepth() << " ม." << endl;
    
    // ใช้งานผ่าน Interface
    Flyable* flyable = &duck;
    Swimmable* swimmable = &duck;
    
    flyable->fly();
    swimmable->swim();
    
    // === FlyingFish ===
    cout << "\n=== FlyingFish ===" << endl;
    
    FlyingFish fish("ปลาบิน");
    fish.fly();
    fish.swim();
    
    // === Diamond Problem ===
    cout << "\n=== Diamond Problem ===" << endl;
    
    Diamond d;
    d.show();
    cout << "value = " << d.value << endl;
    
    return 0;
}
```

---

## ขั้นตอนที่ 74: Polymorphism ขั้นสูง - Design Patterns

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
using namespace std;

// === Strategy Pattern ===
class SortStrategy {
public:
    virtual void sort(vector<int>& data) = 0;
    virtual string getName() const = 0;
    virtual ~SortStrategy() = default;
};

class BubbleSortStrategy : public SortStrategy {
public:
    void sort(vector<int>& data) override {
        int n = data.size();
        for (int i = 0; i < n-1; i++) {
            for (int j = 0; j < n-i-1; j++) {
                if (data[j] > data[j+1]) {
                    swap(data[j], data[j+1]);
                }
            }
        }
    }
    
    string getName() const override { return "Bubble Sort"; }
};

class QuickSortStrategy : public SortStrategy {
private:
    int partition(vector<int>& data, int low, int high) {
        int pivot = data[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++) {
            if (data[j] <= pivot) {
                i++;
                swap(data[i], data[j]);
            }
        }
        swap(data[i+1], data[high]);
        return i + 1;
    }
    
    void quickSort(vector<int>& data, int low, int high) {
        if (low < high) {
            int pi = partition(data, low, high);
            quickSort(data, low, pi - 1);
            quickSort(data, pi + 1, high);
        }
    }
    
public:
    void sort(vector<int>& data) override {
        if (!data.empty()) {
            quickSort(data, 0, data.size() - 1);
        }
    }
    
    string getName() const override { return "Quick Sort"; }
};

class Sorter {
private:
    unique_ptr<SortStrategy> strategy;
    
public:
    void setStrategy(unique_ptr<SortStrategy> s) {
        strategy = move(s);
    }
    
    void sort(vector<int>& data) {
        if (strategy) {
            cout << "Sorting with " << strategy->getName() << "..." << endl;
            strategy->sort(data);
        }
    }
};

// === Observer Pattern ===
class Observer {
public:
    virtual void update(const string& event, int value) = 0;
    virtual ~Observer() = default;
};

class Subject {
private:
    vector<Observer*> observers;
    int state;
    
public:
    Subject() : state(0) {}
    
    void addObserver(Observer* obs) {
        observers.push_back(obs);
    }
    
    void removeObserver(Observer* obs) {
        observers.erase(remove(observers.begin(), observers.end(), obs), 
                        observers.end());
    }
    
    void setState(int s) {
        state = s;
        notifyAll("stateChanged", s);
    }
    
    int getState() const { return state; }
    
    void notifyAll(const string& event, int value) {
        for (Observer* obs : observers) {
            obs->update(event, value);
        }
    }
};

class Logger : public Observer {
public:
    void update(const string& event, int value) override {
        cout << "[Logger] Event: " << event << " | Value: " << value << endl;
    }
};

class Display : public Observer {
private:
    string name;
    
public:
    Display(const string& n) : name(n) {}
    
    void update(const string& event, int value) override {
        cout << "[Display " << name << "] " << event << " = " << value << endl;
    }
};

class Alarm : public Observer {
private:
    int threshold;
    
public:
    Alarm(int t) : threshold(t) {}
    
    void update(const string& event, int value) override {
        if (value > threshold) {
            cout << "[ALARM! ⚠️] " << event << " exceeded threshold: " 
                 << value << " > " << threshold << endl;
        }
    }
};

int main() {
    // === Strategy Pattern ===
    cout << "=== Strategy Pattern ===" << endl;
    
    vector<int> data1 = {64, 34, 25, 12, 22, 11, 90};
    vector<int> data2 = data1;
    
    Sorter sorter;
    
    sorter.setStrategy(make_unique<BubbleSortStrategy>());
    sorter.sort(data1);
    cout << "Bubble sorted: ";
    for (int x : data1) cout << x << " ";
    cout << endl;
    
    sorter.setStrategy(make_unique<QuickSortStrategy>());
    sorter.sort(data2);
    cout << "Quick sorted: ";
    for (int x : data2) cout << x << " ";
    cout << endl;
    
    // === Observer Pattern ===
    cout << "\n=== Observer Pattern ===" << endl;
    
    Subject sensor;
    Logger logger;
    Display display1("Monitor A");
    Display display2("Monitor B");
    Alarm alarm(80);
    
    sensor.addObserver(&logger);
    sensor.addObserver(&display1);
    sensor.addObserver(&display2);
    sensor.addObserver(&alarm);
    
    cout << "--- ตั้งค่า 50 ---" << endl;
    sensor.setState(50);
    
    cout << "\n--- ตั้งค่า 85 ---" << endl;
    sensor.setState(85);
    
    // ลบ observer
    sensor.removeObserver(&display2);
    
    cout << "\n--- ตั้งค่า 30 (Display B ถูกลบแล้ว) ---" << endl;
    sensor.setState(30);
    
    return 0;
}
```

---

## ขั้นตอนที่ 75: Virtual Inheritance และ vtable

```cpp
#include <iostream>
#include <string>
#include <typeinfo>
using namespace std;

// === vtable Demonstration ===
class Vehicle {
protected:
    string name;
    int year;
    double speed;
    
public:
    Vehicle(const string& n, int y, double s)
        : name(n), year(y), speed(s) {}
    
    virtual ~Vehicle() = default;
    
    virtual void start() {
        cout << name << ": เครื่องติด!" << endl;
    }
    
    virtual void stop() {
        cout << name << ": หยุดแล้ว" << endl;
    }
    
    virtual void accelerate(double amount) {
        speed += amount;
        cout << name << ": เร่งเครื่อง ความเร็ว=" << speed << " km/h" << endl;
    }
    
    virtual string getInfo() const {
        return name + " (" + to_string(year) + ")";
    }
    
    string getName() const { return name; }
    double getSpeed() const { return speed; }
};

class Car : public Vehicle {
private:
    int numDoors;
    string fuelType;
    
public:
    Car(const string& n, int y, double s, int d, const string& ft)
        : Vehicle(n, y, s), numDoors(d), fuelType(ft) {}
    
    void start() override {
        cout << name << ": 🚗 เครื่องยนต์ " << fuelType << " ติด!" << endl;
    }
    
    void honk() const {
        cout << name << ": Beep Beep!" << endl;
    }
    
    string getInfo() const override {
        return Vehicle::getInfo() + " | " + fuelType + " | " + to_string(numDoors) + " ประตู";
    }
};

class ElectricCar : public Car {
private:
    int batteryLevel;
    
public:
    ElectricCar(const string& n, int y, double s, int d)
        : Car(n, y, s, d, "Electric"), batteryLevel(100) {}
    
    void start() override {
        cout << name << ": ⚡ Electric car เปิดเงียบๆ... ready!" << endl;
    }
    
    void charge(int percent) {
        batteryLevel = min(100, batteryLevel + percent);
        cout << name << ": ชาร์จแบต... " << batteryLevel << "%" << endl;
    }
    
    void accelerate(double amount) override {
        batteryLevel -= static_cast<int>(amount * 0.5);
        Vehicle::accelerate(amount);
        cout << name << ": แบต=" << batteryLevel << "%" << endl;
    }
    
    string getInfo() const override {
        return Car::getInfo() + " | Battery: " + to_string(batteryLevel) + "%";
    }
};

class Motorcycle : public Vehicle {
private:
    string type;
    
public:
    Motorcycle(const string& n, int y, double s, const string& t)
        : Vehicle(n, y, s), type(t) {}
    
    void start() override {
        cout << name << ": 🏍️ " << type << " รถมอเตอร์ไซค์ พร้อมแล้ว!" << endl;
    }
    
    void wheelie() const {
        cout << name << ": !! Wheelie !!" << endl;
    }
};

// Fleet Management
class Fleet {
private:
    vector<Vehicle*> vehicles;
    
public:
    void addVehicle(Vehicle* v) {
        vehicles.push_back(v);
    }
    
    void startAll() {
        cout << "--- เริ่มยานพาหนะทั้งหมด ---" << endl;
        for (Vehicle* v : vehicles) v->start();
    }
    
    void stopAll() {
        cout << "--- หยุดยานพาหนะทั้งหมด ---" << endl;
        for (Vehicle* v : vehicles) v->stop();
    }
    
    void showFleet() const {
        cout << "=== ยานพาหนะในกองยาน ===" << endl;
        for (Vehicle* v : vehicles) {
            cout << "  " << v->getInfo() << endl;
        }
    }
    
    // แสดง vtable info
    void showTypes() const {
        for (Vehicle* v : vehicles) {
            cout << typeid(*v).name() << ": " << v->getName() << endl;
        }
    }
};

int main() {
    cout << "=== Vehicle Hierarchy ===" << endl;
    
    Car car("Toyota Camry", 2023, 0, 4, "Petrol");
    ElectricCar ev("Tesla Model 3", 2024, 0, 4);
    Motorcycle moto("Honda CBR", 2023, 0, "Sport");
    
    Fleet fleet;
    fleet.addVehicle(&car);
    fleet.addVehicle(&ev);
    fleet.addVehicle(&moto);
    
    fleet.showFleet();
    cout << endl;
    fleet.startAll();
    
    cout << "\n--- Accelerate ---" << endl;
    for (Vehicle* v : {(Vehicle*)&car, (Vehicle*)&ev, (Vehicle*)&moto}) {
        v->accelerate(50);
    }
    
    cout << "\n--- EV Specific ---" << endl;
    ev.charge(30);
    
    cout << "\n--- Type Info ---" << endl;
    fleet.showTypes();
    
    // Dynamic cast
    cout << "\n--- Dynamic Cast ---" << endl;
    
    Vehicle* ptr = &ev;
    
    if (ElectricCar* ecPtr = dynamic_cast<ElectricCar*>(ptr)) {
        cout << "Dynamic cast สำเร็จ -> ElectricCar" << endl;
        ecPtr->charge(20);
    }
    
    if (!dynamic_cast<Motorcycle*>(ptr)) {
        cout << "ptr ไม่ใช่ Motorcycle" << endl;
    }
    
    fleet.stopAll();
    
    return 0;
}
```

---

## สรุป Part 007

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Inheritance พื้นฐาน (Base/Derived Class)
2. ✅ Virtual Functions และ Override
3. ✅ Abstract Classes และ Pure Virtual Functions
4. ✅ Polymorphism ผ่าน Pointers และ References
5. ✅ Multiple Inheritance
6. ✅ Diamond Problem และ Virtual Inheritance
7. ✅ Dynamic Cast
8. ✅ Strategy Pattern ด้วย Polymorphism
9. ✅ Observer Pattern
10. ✅ Fleet Management System

---

## แบบฝึกหัด Part 007

**ฝึกหัดที่ 1:** สร้าง Animal hierarchy ที่มี Mammals, Reptiles, Fish

**ฝึกหัดที่ 2:** สร้าง GUI Component hierarchy (Button, Label, TextBox, etc.)

**ฝึกหัดที่ 3:** Implement Factory Pattern สำหรับสร้าง Shapes

---

⬅️ **ก่อนหน้า:** [Part 006](part006.md) | ➡️ **ต่อไป:** [Part 008: Templates & STL](part008.md)
