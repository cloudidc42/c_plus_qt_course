# Part 035: Qt Quick 3D

## ขั้นตอนที่ 491-505

---

## ขั้นตอนที่ 491: Qt Quick 3D Overview

```
Qt Quick 3D:
  - 3D scene graph integrated with Qt Quick 2D
  - เพิ่มใน Qt 5.15, ปรับปรุงใน Qt 6
  - Built on Qt Rendering Hardware Interface (RHI)
    ├── Vulkan (Linux, Android)
    ├── Metal (macOS, iOS)
    ├── Direct3D 11/12 (Windows)
    └── OpenGL (cross-platform fallback)

Key Types:
  ├── View3D    - 3D viewport
  ├── Node      - scene graph node (position/rotation/scale)
  ├── Model     - 3D mesh renderer
  ├── Camera    - perspective/orthographic camera
  ├── Light     - DirectionalLight, PointLight, SpotLight
  ├── Materials - PrincipledMaterial, DefaultMaterial, CustomMaterial
  └── Texture   - texture maps

CMakeLists.txt:
  find_package(Qt6 REQUIRED COMPONENTS Quick Quick3D)
  target_link_libraries(app PRIVATE Qt6::Quick Qt6::Quick3D)
```

---

## ขั้นตอนที่ 492: Hello 3D World

```qml
import QtQuick
import QtQuick3D

ApplicationWindow {
    visible: true
    width: 800; height: 600
    title: "Qt Quick 3D"
    
    View3D {
        anchors.fill: parent
        
        // Camera
        PerspectiveCamera {
            id: camera
            position: Qt.vector3d(0, 200, 400)
            eulerRotation.x: -20
        }
        
        // Lighting
        DirectionalLight {
            eulerRotation.x: -30
            eulerRotation.y: -70
            brightness: 1.5
        }
        
        // Ambient
        SceneEnvironment {
            id: sceneEnv
            clearColor: "#1a1a2e"
            backgroundMode: SceneEnvironment.Color
            antialiasingMode: SceneEnvironment.MSAA
            antialiasingQuality: SceneEnvironment.VeryHigh
        }
        
        environment: sceneEnv
        
        // 3D Objects
        Model {
            id: cube
            source: "#Cube"
            position: Qt.vector3d(0, 0, 0)
            scale: Qt.vector3d(100, 100, 100)
            
            materials: PrincipledMaterial {
                baseColor: "#3498db"
                metalness: 0.0
                roughness: 0.3
                emissiveFactor: Qt.vector3d(0, 0, 0)
            }
            
            // Rotation animation
            RotationAnimation on eulerRotation.y {
                running: true
                loops: Animation.Infinite
                from: 0; to: 360
                duration: 3000
            }
        }
        
        Model {
            source: "#Sphere"
            position: Qt.vector3d(200, 0, 0)
            scale: Qt.vector3d(80, 80, 80)
            
            materials: PrincipledMaterial {
                baseColor: "#e74c3c"
                metalness: 0.8
                roughness: 0.2
            }
        }
        
        Model {
            source: "#Cylinder"
            position: Qt.vector3d(-200, 0, 0)
            scale: Qt.vector3d(80, 80, 80)
            
            materials: PrincipledMaterial {
                baseColor: "#27ae60"
                metalness: 0.3
                roughness: 0.5
            }
        }
        
        // Floor
        Model {
            source: "#Rectangle"
            scale: Qt.vector3d(600, 1, 600)
            position: Qt.vector3d(0, -52, 0)
            
            materials: PrincipledMaterial {
                baseColor: "#34495e"
                roughness: 0.8
            }
        }
    }
    
    // 2D UI overlay
    Row {
        anchors.bottom: parent.bottom
        anchors.horizontalCenter: parent.horizontalCenter
        anchors.margins: 16
        spacing: 12
        
        Button {
            text: "Top"
            onClicked: {
                camera.position = Qt.vector3d(0, 500, 0)
                camera.eulerRotation.x = -90
            }
        }
        Button {
            text: "Front"
            onClicked: {
                camera.position = Qt.vector3d(0, 0, 500)
                camera.eulerRotation.x = 0
            }
        }
        Button {
            text: "Perspective"
            onClicked: {
                camera.position = Qt.vector3d(200, 300, 400)
                camera.eulerRotation.x = -25
            }
        }
    }
}
```

---

## ขั้นตอนที่ 493: 3D Solar System

```qml
import QtQuick
import QtQuick3D
import QtQuick.Controls

ApplicationWindow {
    visible: true
    width: 1024; height: 768
    title: "Solar System"
    
    View3D {
        anchors.fill: parent
        
        PerspectiveCamera {
            id: camera
            position: Qt.vector3d(0, 300, 800)
            eulerRotation.x: -15
            clipNear: 1
            clipFar: 10000
        }
        
        // Sun light
        PointLight {
            position: Qt.vector3d(0, 0, 0)
            brightness: 5.0
            linearFade: 0.01
            quadraticFade: 0
        }
        
        SceneEnvironment {
            clearColor: "#000010"
            backgroundMode: SceneEnvironment.Color
        }
        
        // === Sun ===
        Model {
            id: sun
            source: "#Sphere"
            scale: Qt.vector3d(80, 80, 80)
            
            materials: PrincipledMaterial {
                baseColor: "#ffcc00"
                emissiveFactor: Qt.vector3d(1.5, 1.0, 0)
                metalness: 0
                roughness: 1
            }
            
            // Slow rotation
            RotationAnimation on eulerRotation.y {
                running: true; loops: -1
                from: 0; to: 360
                duration: 20000
            }
        }
        
        // === Planet component ===
        component Planet : Node {
            property real orbitRadius: 200
            property real orbitSpeed: 5000
            property real planetRadius: 30
            property color planetColor: "blue"
            property real selfRotateSpeed: 3000
            property real startAngle: 0
            
            id: orbitNode
            
            // Orbit animation
            RotationAnimation on eulerRotation.y {
                running: true; loops: -1
                from: orbitNode.startAngle
                to: orbitNode.startAngle + 360
                duration: orbitNode.orbitSpeed
            }
            
            // Planet
            Model {
                source: "#Sphere"
                position: Qt.vector3d(orbitNode.orbitRadius, 0, 0)
                scale: Qt.vector3d(orbitNode.planetRadius, orbitNode.planetRadius, orbitNode.planetRadius)
                
                materials: PrincipledMaterial {
                    baseColor: orbitNode.planetColor
                    metalness: 0.1
                    roughness: 0.6
                }
                
                RotationAnimation on eulerRotation.y {
                    running: true; loops: -1
                    from: 0; to: 360
                    duration: orbitNode.selfRotateSpeed
                }
            }
            
            // Orbit ring (dotted)
            Model {
                source: "#Ring"
                scale: Qt.vector3d(orbitNode.orbitRadius, 1, orbitNode.orbitRadius)
                eulerRotation.x: 90
                
                materials: DefaultMaterial {
                    diffuseColor: Qt.rgba(1, 1, 1, 0.1)
                    opacity: 0.2
                }
            }
        }
        
        // Planets
        Planet { orbitRadius: 150; orbitSpeed: 3000;  planetRadius: 15; planetColor: "#b5b5b5"; startAngle: 0   } // Mercury
        Planet { orbitRadius: 230; orbitSpeed: 5000;  planetRadius: 22; planetColor: "#d4a27a"; startAngle: 60  } // Venus
        Planet { orbitRadius: 320; orbitSpeed: 8000;  planetRadius: 24; planetColor: "#4488ff"; startAngle: 120 } // Earth
        Planet { orbitRadius: 420; orbitSpeed: 14000; planetRadius: 20; planetColor: "#cc4422"; startAngle: 200 } // Mars
        Planet { orbitRadius: 600; orbitSpeed: 40000; planetRadius: 50; planetColor: "#d4aa60"; startAngle: 30  } // Jupiter
        Planet { orbitRadius: 780; orbitSpeed: 90000; planetRadius: 42; planetColor: "#c8a05a"; startAngle: 170 } // Saturn
        
        // Starfield
        Model {
            source: "#Sphere"
            scale: Qt.vector3d(5000, 5000, 5000)
            
            materials: DefaultMaterial {
                diffuseColor: "#020220"
                doubleSided: true
            }
        }
    }
    
    // Camera control
    DragHandler {
        onTranslationChanged: {
            camera.eulerRotation.y += delta.x * 0.2
            camera.eulerRotation.x = Math.max(-90, Math.min(0, camera.eulerRotation.x + delta.y * 0.1))
        }
    }
    
    WheelHandler {
        onWheel: camera.z -= event.angleDelta.y * 0.5
    }
    
    // Speed control
    Slider {
        anchors.bottom: parent.bottom
        anchors.horizontalCenter: parent.horizontalCenter
        anchors.margins: 20
        width: 300
        from: 0.1; to: 5.0; value: 1.0
    }
}
```

---

## ขั้นตอนที่ 494: 3D Data Visualization

```qml
import QtQuick
import QtQuick3D
import QtQuick.Controls

ApplicationWindow {
    visible: true
    width: 900; height: 700
    title: "3D Data Visualization"
    
    // Data model
    ListModel {
        id: dataModel
        
        ListElement { name: "Q1"; value: 85; color: "#3498db" }
        ListElement { name: "Q2"; value: 120; color: "#27ae60" }
        ListElement { name: "Q3"; value: 95; color: "#e74c3c" }
        ListElement { name: "Q4"; value: 140; color: "#f39c12" }
        ListElement { name: "Q5"; value: 75; color: "#9b59b6" }
        ListElement { name: "Q6"; value: 110; color: "#1abc9c" }
    }
    
    View3D {
        anchors.fill: parent
        anchors.bottomMargin: 50
        
        PerspectiveCamera {
            id: cam
            position: Qt.vector3d(0, 300, 600)
            eulerRotation.x: -25
        }
        
        DirectionalLight {
            eulerRotation.x: -40
            eulerRotation.y: -20
            brightness: 2.0
        }
        
        SceneEnvironment {
            clearColor: "#f0f2f5"
            backgroundMode: SceneEnvironment.Color
            antialiasingMode: SceneEnvironment.MSAA
        }
        
        // 3D Bar Chart
        Node {
            id: chart
            
            Repeater3D {
                model: dataModel
                
                Node {
                    x: (index - (dataModel.count - 1) / 2.0) * 120
                    
                    // Bar
                    Model {
                        id: bar
                        source: "#Cube"
                        
                        property real targetHeight: value
                        property real animatedHeight: 0
                        
                        scale: Qt.vector3d(80, animatedHeight, 80)
                        y: animatedHeight * 50 / 2  // Center at bottom
                        
                        materials: PrincipledMaterial {
                            baseColor: color
                            metalness: 0.1
                            roughness: 0.4
                        }
                        
                        // Entrance animation
                        NumberAnimation on animatedHeight {
                            from: 0; to: bar.targetHeight
                            duration: 800 + index * 100
                            easing.type: Easing.OutBack
                        }
                    }
                }
            }
        }
        
        // Floor grid
        Model {
            source: "#Rectangle"
            scale: Qt.vector3d(800, 1, 400)
            position: Qt.vector3d(0, 0, 0)
            eulerRotation.x: -90
            
            materials: DefaultMaterial {
                diffuseColor: "#dddddd"
            }
        }
    }
    
    // Labels (2D overlay)
    Row {
        anchors.bottom: parent.bottom
        anchors.horizontalCenter: parent.horizontalCenter
        spacing: 0
        
        Repeater {
            model: dataModel
            
            Rectangle {
                width: 120
                height: 40
                color: "transparent"
                
                Column {
                    anchors.centerIn: parent
                    spacing: 2
                    
                    Rectangle {
                        width: 20; height: 10
                        radius: 2
                        color: model.color
                        anchors.horizontalCenter: parent.horizontalCenter
                    }
                    
                    Text {
                        text: model.name
                        font.pixelSize: 12
                        anchors.horizontalCenter: parent.horizontalCenter
                    }
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 495-505: Interactive 3D Scene

```qml
import QtQuick
import QtQuick3D
import QtQuick3D.Helpers

ApplicationWindow {
    visible: true
    width: 1024; height: 768
    title: "Interactive 3D"
    
    View3D {
        id: view3d
        anchors.fill: parent
        
        // Camera with helper for mouse navigation
        Node {
            id: cameraNode
            
            PerspectiveCamera {
                id: camera
                position: Qt.vector3d(0, 0, 600)
            }
        }
        
        // Lighting
        DirectionalLight {
            eulerRotation: Qt.vector3d(-45, -45, 0)
            brightness: 1.5
            castsShadow: true
        }
        
        PointLight {
            id: movingLight
            position: Qt.vector3d(0, 200, 0)
            brightness: 3.0
            color: "#4488ff"
            
            // Moving light animation
            SequentialAnimation on x {
                running: true; loops: -1
                NumberAnimation { from: -300; to: 300; duration: 3000; easing.type: Easing.InOutSine }
                NumberAnimation { from: 300; to: -300; duration: 3000; easing.type: Easing.InOutSine }
            }
        }
        
        SceneEnvironment {
            clearColor: "#0d1117"
            backgroundMode: SceneEnvironment.Color
            antialiasingMode: SceneEnvironment.MSAA
            antialiasingQuality: SceneEnvironment.High
        }
        
        // Interactive objects
        Repeater3D {
            model: 9
            
            Node {
                property int row: Math.floor(index / 3)
                property int col: index % 3
                property bool selected: false
                
                x: (col - 1) * 150
                y: 0
                z: (row - 1) * 150
                
                Model {
                    id: obj
                    source: "#Cube"
                    scale: Qt.vector3d(
                        selected ? 60 : 50,
                        selected ? 60 : 50,
                        selected ? 60 : 50
                    )
                    
                    property bool hovered: false
                    
                    materials: PrincipledMaterial {
                        baseColor: selected ? "#f39c12"
                                 : hovered  ? "#3498db"
                                 : Qt.hsla(index / 9, 0.7, 0.5, 1.0)
                        metalness: selected ? 0.8 : 0.2
                        roughness: 0.4
                        emissiveFactor: selected ? Qt.vector3d(0.3, 0.2, 0) : Qt.vector3d(0, 0, 0)
                    }
                    
                    Behavior on scale {
                        Vector3dAnimation { duration: 200; easing.type: Easing.OutBack }
                    }
                }
                
                // Hover detection
                // (In real app, use View3D.pick for precise detection)
                
                RotationAnimation on eulerRotation.y {
                    running: true; loops: -1
                    from: 0; to: 360
                    duration: 5000 + index * 500
                }
            }
        }
        
        // Floor
        Model {
            source: "#Rectangle"
            eulerRotation.x: -90
            scale: Qt.vector3d(600, 600, 1)
            y: -60
            
            materials: PrincipledMaterial {
                baseColor: "#1a1a2e"
                roughness: 0.9
                metalness: 0.0
            }
        }
    }
    
    // Mouse picking
    MouseArea {
        anchors.fill: view3d
        
        onClicked: (mouse) => {
            var hit = view3d.pick(mouse.x, mouse.y)
            if (hit.objectHit) {
                var node = hit.objectHit.parent
                node.selected = !node.selected
                console.log("Picked object:", hit.objectHit)
            }
        }
    }
    
    // Orbital camera controller using drag
    property real cameraRotX: -25
    property real cameraRotY: 0
    property real cameraDistance: 600
    
    DragHandler {
        target: null
        onTranslationChanged: {
            parent.cameraRotY += delta.x * 0.3
            parent.cameraRotX = Math.max(-80, Math.min(0, parent.cameraRotX + delta.y * 0.2))
            
            cameraNode.eulerRotation = Qt.vector3d(parent.cameraRotX, parent.cameraRotY, 0)
        }
    }
    
    WheelHandler {
        target: null
        onWheel: (e) => {
            parent.cameraDistance = Math.max(200, parent.cameraDistance - e.angleDelta.y)
            camera.z = parent.cameraDistance
        }
    }
    
    // HUD
    Column {
        anchors.top: parent.top
        anchors.right: parent.right
        anchors.margins: 16
        spacing: 8
        
        Text {
            text: "Qt Quick 3D"
            color: "white"
            font.pixelSize: 20
            font.bold: true
        }
        
        Text {
            text: "Drag to rotate\nScroll to zoom\nClick to select"
            color: "#aaaaaa"
            font.pixelSize: 13
        }
    }
}
```

---

## สรุป Part 035

ใน Part นี้คุณได้เรียนรู้:

1. ✅ Qt Quick 3D architecture overview
2. ✅ View3D, Camera, Light, Model, PrincipledMaterial
3. ✅ 3D Solar System simulation
4. ✅ 3D Bar Chart visualization
5. ✅ Interactive 3D scene (picking, orbit camera, lighting)

---

⬅️ [Part 034](part034.md) | ➡️ [Part 036: Qt Bluetooth & Serial Port](part036.md)
