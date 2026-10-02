# Part 050: World-Class Level — Qt OpenGL & Custom Renderer

## ขั้นตอนที่ 716-730

---

## ขั้นตอนที่ 716: Qt + OpenGL

```
Qt OpenGL Integration:
  - QOpenGLWidget: render OpenGL in a QWidget
  - QOpenGLFunctions: safe cross-platform OpenGL functions
  - QOpenGLShaderProgram: compile/link GLSL shaders
  - QOpenGLBuffer: VBO, IBO management
  - QOpenGLVertexArrayObject: VAO management
  - QOpenGLTexture: texture management

CMakeLists.txt:
  find_package(Qt6 REQUIRED COMPONENTS OpenGL OpenGLWidgets)
  target_link_libraries(app PRIVATE Qt6::OpenGL Qt6::OpenGLWidgets)
```

---

## ขั้นตอนที่ 717: Basic OpenGL Widget

```cpp
#include <QOpenGLWidget>
#include <QOpenGLFunctions>
#include <QOpenGLShaderProgram>
#include <QOpenGLBuffer>
#include <QOpenGLVertexArrayObject>

class OpenGLTriangle : public QOpenGLWidget, protected QOpenGLFunctions {
    Q_OBJECT
    
public:
    explicit OpenGLTriangle(QWidget* parent = nullptr) : QOpenGLWidget(parent) {}
    
protected:
    void initializeGL() override {
        initializeOpenGLFunctions();
        glClearColor(0.1f, 0.1f, 0.15f, 1.0f);
        
        // Compile shaders
        program = new QOpenGLShaderProgram(this);
        
        program->addShaderFromSourceCode(QOpenGLShader::Vertex, R"(
            #version 330 core
            layout(location = 0) in vec3 position;
            layout(location = 1) in vec3 color;
            
            out vec3 vColor;
            
            uniform mat4 mvpMatrix;
            
            void main() {
                gl_Position = mvpMatrix * vec4(position, 1.0);
                vColor = color;
            }
        )");
        
        program->addShaderFromSourceCode(QOpenGLShader::Fragment, R"(
            #version 330 core
            in vec3 vColor;
            out vec4 fragColor;
            
            void main() {
                fragColor = vec4(vColor, 1.0);
            }
        )");
        
        if (!program->link()) {
            qCritical() << "Shader link error:" << program->log();
            return;
        }
        
        // Vertex data: position (xyz) + color (rgb)
        float vertices[] = {
            // Position          // Color
            -0.5f, -0.5f, 0.0f,  1.0f, 0.0f, 0.0f,  // Red
             0.5f, -0.5f, 0.0f,  0.0f, 1.0f, 0.0f,  // Green
             0.0f,  0.5f, 0.0f,  0.0f, 0.0f, 1.0f   // Blue
        };
        
        vao.create();
        vao.bind();
        
        vbo.create();
        vbo.bind();
        vbo.allocate(vertices, sizeof(vertices));
        
        // Position attribute
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), nullptr);
        
        // Color attribute
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float),
                              reinterpret_cast<void*>(3 * sizeof(float)));
        
        vao.release();
        vbo.release();
    }
    
    void resizeGL(int w, int h) override {
        glViewport(0, 0, w, h);
        
        float aspect = static_cast<float>(w) / static_cast<float>(h ? h : 1);
        
        m_projection.setToIdentity();
        m_projection.perspective(45.0f, aspect, 0.1f, 100.0f);
    }
    
    void paintGL() override {
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        glEnable(GL_DEPTH_TEST);
        
        QMatrix4x4 view, model;
        view.translate(0, 0, -3);
        model.rotate(m_angle, 0, 0, 1);
        
        QMatrix4x4 mvp = m_projection * view * model;
        
        program->bind();
        program->setUniformValue("mvpMatrix", mvp);
        
        vao.bind();
        glDrawArrays(GL_TRIANGLES, 0, 3);
        vao.release();
        
        program->release();
        
        // Auto-animate
        m_angle += 1.0f;
        if (m_angle >= 360.0f) m_angle -= 360.0f;
        update();  // Trigger next frame
    }
    
private:
    QOpenGLShaderProgram* program = nullptr;
    QOpenGLBuffer vbo{QOpenGLBuffer::VertexBuffer};
    QOpenGLVertexArrayObject vao;
    QMatrix4x4 m_projection;
    float m_angle = 0;
};
```

---

## ขั้นตอนที่ 718: 3D Mesh Renderer

```cpp
class MeshRenderer : public QOpenGLWidget, protected QOpenGLFunctions {
    Q_OBJECT
    
public:
    struct Vertex {
        float position[3];
        float normal[3];
        float texCoord[2];
    };
    
    struct Mesh {
        QVector<Vertex> vertices;
        QVector<unsigned int> indices;
        QString name;
        
        static Mesh createCube() {
            Mesh m;
            m.name = "Cube";
            
            // 6 faces, 4 vertices each
            struct Face { float nx, ny, nz; float v[4][3]; };
            Face faces[] = {
                {0,0,1,  {{-1,-1,1},{1,-1,1},{1,1,1},{-1,1,1}}},   // Front
                {0,0,-1, {{1,-1,-1},{-1,-1,-1},{-1,1,-1},{1,1,-1}}}, // Back
                {0,1,0,  {{-1,1,1},{1,1,1},{1,1,-1},{-1,1,-1}}},    // Top
                {0,-1,0, {{-1,-1,-1},{1,-1,-1},{1,-1,1},{-1,-1,1}}}, // Bottom
                {1,0,0,  {{1,-1,1},{1,-1,-1},{1,1,-1},{1,1,1}}},    // Right
                {-1,0,0, {{-1,-1,-1},{-1,-1,1},{-1,1,1},{-1,1,-1}}} // Left
            };
            
            for (auto& face : faces) {
                unsigned int base = m.vertices.size();
                
                for (auto& v : face.v) {
                    Vertex vert;
                    vert.position[0] = v[0]; vert.position[1] = v[1]; vert.position[2] = v[2];
                    vert.normal[0] = face.nx; vert.normal[1] = face.ny; vert.normal[2] = face.nz;
                    vert.texCoord[0] = 0; vert.texCoord[1] = 0;
                    m.vertices.append(vert);
                }
                
                // Two triangles per face
                m.indices << base << base+1 << base+2 << base << base+2 << base+3;
            }
            
            return m;
        }
    };
    
    explicit MeshRenderer(QWidget* parent = nullptr) : QOpenGLWidget(parent) {
        setFocusPolicy(Qt::StrongFocus);
    }
    
    void setMesh(const Mesh& mesh) {
        m_mesh = mesh;
        if (isValid()) uploadMesh();
    }
    
protected:
    void initializeGL() override {
        initializeOpenGLFunctions();
        glClearColor(0.15f, 0.15f, 0.2f, 1.0f);
        
        // Phong shading
        program = new QOpenGLShaderProgram(this);
        
        program->addShaderFromSourceCode(QOpenGLShader::Vertex, R"(
            #version 330 core
            layout(location = 0) in vec3 aPosition;
            layout(location = 1) in vec3 aNormal;
            
            uniform mat4 modelMatrix;
            uniform mat4 viewMatrix;
            uniform mat4 projMatrix;
            uniform mat3 normalMatrix;
            
            out vec3 fragPos;
            out vec3 fragNormal;
            
            void main() {
                vec4 worldPos = modelMatrix * vec4(aPosition, 1.0);
                fragPos = worldPos.xyz;
                fragNormal = normalMatrix * aNormal;
                gl_Position = projMatrix * viewMatrix * worldPos;
            }
        )");
        
        program->addShaderFromSourceCode(QOpenGLShader::Fragment, R"(
            #version 330 core
            in vec3 fragPos;
            in vec3 fragNormal;
            out vec4 fragColor;
            
            uniform vec3 lightPos;
            uniform vec3 lightColor;
            uniform vec3 objectColor;
            uniform vec3 viewPos;
            
            void main() {
                // Ambient
                float ambientStrength = 0.15;
                vec3 ambient = ambientStrength * lightColor;
                
                // Diffuse
                vec3 normal = normalize(fragNormal);
                vec3 lightDir = normalize(lightPos - fragPos);
                float diff = max(dot(normal, lightDir), 0.0);
                vec3 diffuse = diff * lightColor;
                
                // Specular
                float specStrength = 0.5;
                vec3 viewDir = normalize(viewPos - fragPos);
                vec3 reflectDir = reflect(-lightDir, normal);
                float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32.0);
                vec3 specular = specStrength * spec * lightColor;
                
                vec3 result = (ambient + diffuse + specular) * objectColor;
                fragColor = vec4(result, 1.0);
            }
        )");
        
        program->link();
        
        vao.create();
        vbo.create();
        ibo.create();
        
        m_mesh = Mesh::createCube();
        uploadMesh();
    }
    
    void resizeGL(int w, int h) override {
        glViewport(0, 0, w, h);
        
        float aspect = static_cast<float>(w) / std::max(1, h);
        m_projection.setToIdentity();
        m_projection.perspective(60.0f, aspect, 0.1f, 1000.0f);
    }
    
    void paintGL() override {
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        glEnable(GL_DEPTH_TEST);
        glEnable(GL_CULL_FACE);
        
        QMatrix4x4 model, view;
        model.rotate(m_rotX, 1, 0, 0);
        model.rotate(m_rotY, 0, 1, 0);
        
        view.translate(0, 0, -m_zoom);
        view.translate(m_panX, m_panY, 0);
        
        QMatrix3x3 normalMat = model.normalMatrix();
        
        program->bind();
        program->setUniformValue("modelMatrix", model);
        program->setUniformValue("viewMatrix", view);
        program->setUniformValue("projMatrix", m_projection);
        program->setUniformValue("normalMatrix", normalMat);
        program->setUniformValue("lightPos", QVector3D(5, 5, 5));
        program->setUniformValue("lightColor", QVector3D(1, 1, 1));
        program->setUniformValue("objectColor", QVector3D(0.5f, 0.7f, 1.0f));
        program->setUniformValue("viewPos", QVector3D(0, 0, m_zoom));
        
        vao.bind();
        glDrawElements(GL_TRIANGLES, m_mesh.indices.size(), GL_UNSIGNED_INT, nullptr);
        vao.release();
        
        program->release();
        
        // Auto-rotate
        m_rotY += 0.5f;
        update();
    }
    
    void mousePressEvent(QMouseEvent* e) override {
        m_lastMousePos = e->pos();
    }
    
    void mouseMoveEvent(QMouseEvent* e) override {
        QPoint delta = e->pos() - m_lastMousePos;
        
        if (e->buttons() & Qt::LeftButton) {
            m_rotX += delta.y() * 0.3f;
            m_rotY += delta.x() * 0.3f;
        } else if (e->buttons() & Qt::RightButton) {
            m_panX += delta.x() * 0.01f;
            m_panY -= delta.y() * 0.01f;
        }
        
        m_lastMousePos = e->pos();
        update();
    }
    
    void wheelEvent(QWheelEvent* e) override {
        m_zoom -= e->angleDelta().y() * 0.01f;
        m_zoom = qBound(1.0f, m_zoom, 50.0f);
        update();
    }
    
private:
    void uploadMesh() {
        if (!vao.isCreated()) return;
        
        vao.bind();
        
        vbo.bind();
        vbo.allocate(m_mesh.vertices.constData(),
                     m_mesh.vertices.size() * sizeof(Vertex));
        
        // Position
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), nullptr);
        
        // Normal
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex),
                              reinterpret_cast<void*>(3 * sizeof(float)));
        
        ibo.bind();
        ibo.allocate(m_mesh.indices.constData(),
                     m_mesh.indices.size() * sizeof(unsigned int));
        
        vao.release();
    }
    
    QOpenGLShaderProgram* program = nullptr;
    QOpenGLBuffer vbo{QOpenGLBuffer::VertexBuffer};
    QOpenGLBuffer ibo{QOpenGLBuffer::IndexBuffer};
    QOpenGLVertexArrayObject vao;
    Mesh m_mesh;
    QMatrix4x4 m_projection;
    
    float m_rotX = 0, m_rotY = 30;
    float m_zoom = 5.0f;
    float m_panX = 0, m_panY = 0;
    QPoint m_lastMousePos;
};
```

---

## สรุป Part 050

ใน Part นี้คุณได้เรียนรู้:

1. ✅ QOpenGLWidget + QOpenGLFunctions พื้นฐาน
2. ✅ GLSL Vertex/Fragment shaders ด้วย QOpenGLShaderProgram
3. ✅ VAO + VBO + IBO management
4. ✅ Phong shading model (ambient + diffuse + specular)
5. ✅ Mouse interaction: rotate, pan, zoom

---

⬅️ [Part 049](part049.md) | ➡️ [Part 051: World-Class — Packaging & Deployment](part051.md)
