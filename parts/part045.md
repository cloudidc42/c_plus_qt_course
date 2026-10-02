# Part 045: Professional Level — Custom Widget Engine

## ขั้นตอนที่ 641-655

---

## ขั้นตอนที่ 641: Custom Widget จาก QWidget

```cpp
// === Professional Custom Widget ===
// Paint everything manually with QPainter

class ToggleSwitch : public QWidget {
    Q_OBJECT
    Q_PROPERTY(bool checked READ isChecked WRITE setChecked NOTIFY toggled)
    Q_PROPERTY(QColor onColor READ onColor WRITE setOnColor)
    Q_PROPERTY(QColor offColor READ offColor WRITE setOffColor)
    
public:
    explicit ToggleSwitch(QWidget* parent = nullptr) : QWidget(parent) {
        setFixedSize(50, 26);
        setCursor(Qt::PointingHandCursor);
        setFocusPolicy(Qt::StrongFocus);
    }
    
    bool isChecked() const { return m_checked; }
    QColor onColor()  const { return m_onColor; }
    QColor offColor() const { return m_offColor; }
    
    void setOnColor(const QColor& c)  { m_onColor = c; update(); }
    void setOffColor(const QColor& c) { m_offColor = c; update(); }
    
public slots:
    void setChecked(bool checked) {
        if (m_checked == checked) return;
        m_checked = checked;
        
        // Animate thumb position
        m_animation->stop();
        m_animation->setStartValue(m_thumbPos);
        m_animation->setEndValue(checked ? 1.0 : 0.0);
        m_animation->start();
        
        emit toggled(checked);
    }
    
    void toggle() { setChecked(!m_checked); }
    
signals:
    void toggled(bool checked);
    
protected:
    void paintEvent(QPaintEvent*) override {
        QPainter p(this);
        p.setRenderHint(QPainter::Antialiasing);
        
        QRect r = rect().adjusted(1, 1, -1, -1);
        int h = r.height();
        int radius = h / 2;
        
        // Track
        QColor trackColor = mix(m_offColor, m_onColor, m_thumbPos);
        p.setBrush(trackColor);
        p.setPen(Qt::NoPen);
        p.drawRoundedRect(r, radius, radius);
        
        // Thumb
        double thumbX = r.left() + radius + m_thumbPos * (r.width() - h);
        double thumbY = r.top() + radius;
        double thumbR = radius - 2;
        
        // Shadow
        p.setBrush(QColor(0, 0, 0, 40));
        p.drawEllipse(QPointF(thumbX + 1, thumbY + 1), thumbR, thumbR);
        
        // Thumb itself
        p.setBrush(Qt::white);
        p.drawEllipse(QPointF(thumbX, thumbY), thumbR, thumbR);
        
        // Focus indicator
        if (hasFocus()) {
            p.setPen(QPen(m_onColor.lighter(120), 2));
            p.setBrush(Qt::NoBrush);
            p.drawRoundedRect(rect().adjusted(0, 0, -1, -1), radius + 1, radius + 1);
        }
    }
    
    void mousePressEvent(QMouseEvent*) override { toggle(); }
    
    void keyPressEvent(QKeyEvent* event) override {
        if (event->key() == Qt::Key_Space || event->key() == Qt::Key_Return) {
            toggle();
        } else {
            QWidget::keyPressEvent(event);
        }
    }
    
    QSize sizeHint() const override { return {50, 26}; }
    
private:
    static QColor mix(const QColor& a, const QColor& b, double t) {
        return QColor(
            a.red()   + (b.red()   - a.red())   * t,
            a.green() + (b.green() - a.green()) * t,
            a.blue()  + (b.blue()  - a.blue())  * t
        );
    }
    
    bool m_checked = false;
    double m_thumbPos = 0.0;
    QColor m_onColor = QColor("#27ae60");
    QColor m_offColor = QColor("#ccc");
    
    QPropertyAnimation* m_animation = [this]() {
        auto* anim = new QPropertyAnimation(this);
        anim->setTargetObject(this);
        anim->setPropertyName("thumbPosition");
        anim->setDuration(180);
        anim->setEasingCurve(QEasingCurve::OutCubic);
        connect(anim, &QPropertyAnimation::valueChanged, [this]() { update(); });
        return anim;
    }();
    
    // Internal property for animation
    Q_PROPERTY(double thumbPosition READ thumbPosition WRITE setThumbPosition)
    double thumbPosition() const { return m_thumbPos; }
    void setThumbPosition(double p) { m_thumbPos = p; update(); }
};
```

---

## ขั้นตอนที่ 642: Rating Widget

```cpp
class StarRating : public QWidget {
    Q_OBJECT
    Q_PROPERTY(int value READ value WRITE setValue NOTIFY valueChanged)
    Q_PROPERTY(int maximum READ maximum WRITE setMaximum)
    
public:
    explicit StarRating(int max = 5, QWidget* parent = nullptr)
        : QWidget(parent), m_max(max) {
        setFixedHeight(24);
        setMinimumWidth(max * 28);
        setCursor(Qt::PointingHandCursor);
        setMouseTracking(true);
    }
    
    int value()   const { return m_value; }
    int maximum() const { return m_max; }
    
    void setMaximum(int max) {
        m_max = max;
        setMinimumWidth(max * 28);
        update();
    }
    
public slots:
    void setValue(int v) {
        v = qBound(0, v, m_max);
        if (m_value == v) return;
        m_value = v;
        emit valueChanged(v);
        update();
    }
    
signals:
    void valueChanged(int value);
    
protected:
    void paintEvent(QPaintEvent*) override {
        QPainter p(this);
        p.setRenderHint(QPainter::Antialiasing);
        
        int starSize = height() - 4;
        
        for (int i = 0; i < m_max; ++i) {
            QRect starRect(i * (starSize + 4) + 2, 2, starSize, starSize);
            bool filled = (i < m_hovered) ? true : (m_hovered == 0 && i < m_value);
            
            drawStar(p, starRect, filled);
        }
    }
    
    void mouseMoveEvent(QMouseEvent* e) override {
        int starW = (height() - 4 + 4);
        m_hovered = qMin(m_max, static_cast<int>((e->pos().x() + starW/2) / starW) + 1);
        update();
    }
    
    void leaveEvent(QEvent*) override {
        m_hovered = 0;
        update();
    }
    
    void mousePressEvent(QMouseEvent*) override {
        if (m_hovered > 0) setValue(m_hovered == m_value ? 0 : m_hovered);
    }
    
private:
    void drawStar(QPainter& p, const QRect& rect, bool filled) {
        QPolygonF star;
        double cx = rect.center().x();
        double cy = rect.center().y();
        double outerR = rect.width() / 2.0;
        double innerR = outerR * 0.4;
        
        for (int i = 0; i < 10; ++i) {
            double angle = (i * 36 - 90) * M_PI / 180.0;
            double r = (i % 2 == 0) ? outerR : innerR;
            star << QPointF(cx + r * std::cos(angle), cy + r * std::sin(angle));
        }
        
        QColor color = filled ? QColor("#f39c12") : QColor("#ddd");
        
        p.setBrush(color);
        p.setPen(QPen(filled ? color.darker(120) : QColor("#ccc"), 0.5));
        p.drawPolygon(star);
    }
    
    int m_value = 0;
    int m_max;
    int m_hovered = 0;
};
```

---

## ขั้นตอนที่ 643: Progress Ring

```cpp
class ProgressRing : public QWidget {
    Q_OBJECT
    Q_PROPERTY(double value READ value WRITE setValue)
    Q_PROPERTY(QColor ringColor READ ringColor WRITE setRingColor)
    
public:
    explicit ProgressRing(QWidget* parent = nullptr) : QWidget(parent) {
        setMinimumSize(80, 80);
    }
    
    double value() const { return m_value; }
    QColor ringColor() const { return m_color; }
    
    void setRingColor(const QColor& c) { m_color = c; update(); }
    
    void animateTo(double target, int durationMs = 1000) {
        auto* anim = new QPropertyAnimation(this, "value", this);
        anim->setDuration(durationMs);
        anim->setStartValue(m_value);
        anim->setEndValue(qBound(0.0, target, 100.0));
        anim->setEasingCurve(QEasingCurve::OutCubic);
        anim->start(QAbstractAnimation::DeleteWhenStopped);
    }
    
public slots:
    void setValue(double v) {
        m_value = qBound(0.0, v, 100.0);
        update();
    }
    
protected:
    void paintEvent(QPaintEvent*) override {
        QPainter p(this);
        p.setRenderHint(QPainter::Antialiasing);
        
        int size = qMin(width(), height()) - 8;
        int x = (width() - size) / 2;
        int y = (height() - size) / 2;
        QRectF rect(x, y, size, size);
        
        int thickness = size / 8;
        rect.adjust(thickness/2, thickness/2, -thickness/2, -thickness/2);
        
        // Background ring
        p.setPen(QPen(QColor("#e0e0e0"), thickness, Qt::SolidLine, Qt::FlatCap));
        p.drawEllipse(rect);
        
        // Progress arc
        if (m_value > 0) {
            double span = m_value / 100.0 * 360.0;
            
            // Gradient pen
            QConicalGradient grad(rect.center(), 90);
            grad.setColorAt(0, m_color.lighter(130));
            grad.setColorAt(1, m_color);
            
            p.setPen(QPen(QBrush(grad), thickness, Qt::SolidLine, Qt::RoundCap));
            p.drawArc(rect, 90 * 16, static_cast<int>(-span * 16));
        }
        
        // Text
        p.setPen(QPen(Qt::black));
        QFont f = font();
        f.setPixelSize(size / 4);
        f.setBold(true);
        p.setFont(f);
        p.drawText(rect, Qt::AlignCenter, QString("%1%").arg(static_cast<int>(m_value)));
    }
    
private:
    double m_value = 0;
    QColor m_color = QColor("#3498db");
};
```

---

## ขั้นตอนที่ 644: Color Picker Widget

```cpp
class ColorPicker : public QWidget {
    Q_OBJECT
    Q_PROPERTY(QColor color READ color WRITE setColor NOTIFY colorChanged)
    
public:
    explicit ColorPicker(QWidget* parent = nullptr) : QWidget(parent) {
        setMinimumSize(200, 200);
        setMouseTracking(true);
    }
    
    QColor color() const { return m_color; }
    
public slots:
    void setColor(const QColor& color) {
        m_color = color;
        m_hue = color.hsvHueF();
        m_saturation = color.hsvSaturationF();
        m_value = color.valueF();
        update();
        emit colorChanged(color);
    }
    
signals:
    void colorChanged(const QColor& color);
    
protected:
    void paintEvent(QPaintEvent*) override {
        QPainter p(this);
        p.setRenderHint(QPainter::Antialiasing);
        
        QRect sq = squareRect();
        
        // Saturation-Value square
        // Horizontal: saturation (0 to 1), Vertical: value (1 to 0)
        QLinearGradient satGrad(sq.topLeft(), sq.topRight());
        satGrad.setColorAt(0, Qt::white);
        satGrad.setColorAt(1, QColor::fromHsvF(m_hue, 1, 1));
        p.fillRect(sq, satGrad);
        
        QLinearGradient valGrad(sq.topLeft(), sq.bottomLeft());
        valGrad.setColorAt(0, Qt::transparent);
        valGrad.setColorAt(1, Qt::black);
        p.fillRect(sq, valGrad);
        
        // Cursor
        double cx = sq.left() + m_saturation * sq.width();
        double cy = sq.top() + (1.0 - m_value) * sq.height();
        
        p.setPen(QPen(Qt::white, 2));
        p.setBrush(Qt::NoBrush);
        p.drawEllipse(QPointF(cx, cy), 6, 6);
        
        // Hue strip
        QRect hueRect = hueStripRect();
        QLinearGradient hueGrad(hueRect.topLeft(), hueRect.bottomLeft());
        for (int i = 0; i <= 360; i += 10) {
            hueGrad.setColorAt(i / 360.0, QColor::fromHsvF(i / 360.0, 1, 1));
        }
        p.fillRect(hueRect, hueGrad);
        
        // Hue cursor
        double hy = hueRect.top() + m_hue * hueRect.height();
        p.setPen(QPen(Qt::white, 2));
        p.drawRect(hueRect.left() - 2, static_cast<int>(hy) - 4,
                   hueRect.width() + 4, 8);
        
        // Color preview
        QRect preview(width() - 30, 0, 30, 30);
        p.fillRect(preview, m_color);
        p.setPen(QPen(Qt::black));
        p.drawRect(preview);
    }
    
    void mousePressEvent(QMouseEvent* e) override {
        updateColor(e->pos());
    }
    
    void mouseMoveEvent(QMouseEvent* e) override {
        if (e->buttons() & Qt::LeftButton) {
            updateColor(e->pos());
        }
    }
    
private:
    QRect squareRect() const {
        int s = qMin(width() - 40, height() - 10);
        return QRect(0, 5, s, s);
    }
    
    QRect hueStripRect() const {
        QRect sq = squareRect();
        return QRect(sq.right() + 8, sq.top(), 20, sq.height());
    }
    
    void updateColor(const QPoint& pos) {
        QRect sq = squareRect();
        QRect hue = hueStripRect();
        
        if (sq.contains(pos)) {
            m_saturation = qBound(0.0, (pos.x() - sq.left()) / static_cast<double>(sq.width()), 1.0);
            m_value = qBound(0.0, 1.0 - (pos.y() - sq.top()) / static_cast<double>(sq.height()), 1.0);
        } else if (hue.contains(pos)) {
            m_hue = qBound(0.0, (pos.y() - hue.top()) / static_cast<double>(hue.height()), 1.0);
        }
        
        m_color = QColor::fromHsvF(m_hue, m_saturation, m_value);
        emit colorChanged(m_color);
        update();
    }
    
    QColor m_color = Qt::red;
    double m_hue = 0, m_saturation = 1, m_value = 1;
};
```

---

## สรุป Part 045

ใน Part นี้คุณได้เรียนรู้:

1. ✅ ToggleSwitch ด้วย QPropertyAnimation + QPainter
2. ✅ StarRating widget ด้วย hover + click
3. ✅ ProgressRing ด้วย QConicalGradient + animation
4. ✅ ColorPicker ด้วย HSV model + manual painting
5. ✅ แนวทาง Custom Widget พร้อม Q_PROPERTY animation

---

⬅️ [Part 044](part044.md) | ➡️ [Part 046: Professional Level — Signals Architecture](part046.md)
