# Tutorial paso a paso: construir la app **cm** desde cero con Qt 6 (**CMake**, QML 6.11.2 / mínimo 6.8 LTS) y C++17

Este tutorial desarrolla el proyecto **cm** (aplicación LOB de gestión de clientes) **desde una carpeta vacía**,
siguiendo el plan de fases de [`CONSTRUCCION_POR_FASES.md`](CONSTRUCCION_POR_FASES.md) y la documentación oficial
de Qt 6 (**https://doc.qt.io/qt-6/**). Cada paso indica qué archivo crear, su contenido completo o esencial,
cómo compilarlo y dónde consultar la doc de Qt para profundizar.

> **Versiones y herramientas exigidas**:
> - Qt **6.11.2** (objetivo) o **6.8 LTS** (mínimo).
> - **CMake como único sistema de build.** En Qt 6, `qmake` está **deprecado desde Qt 6.7** y desaparece en Qt 7;
>   toda la cadena usa `qt-cmake`, `qt_add_executable`, `qt_add_library` y **`qt_add_qml_module`**.
>   Los `.pro`/`.pri` históricos del repo quedan sustituidos por `CMakeLists.txt`.
> - **C++17 como mínimo**, fijado centralmente con `qt_standard_project_setup(REQUIRES 6.8)`
>   (equivale a `set(CMAKE_CXX_STANDARD 17)` + AUTOMOC/AUTOUIC/AUTORCC + políticas Qt).
> - Imports QML **sin número de versión** (`import QtQuick`, `import QtQuick.Controls`, …).
> - Tipos C++ expuestos a QML con las macros **`QML_ELEMENT` / `QML_SINGLETON`** (no `qmlRegisterType`).
>
> El código histórico de los capítulos del repo está en Qt 5.x + qmake — este tutorial lo construye ya en Qt 6 con CMake.

Comandos de build usados en todo el tutorial (Linux/macOS; en Windows usa `qt-cmake` del kit MSVC):

```bash
~/Qt/6.11.2/gcc_64/bin/qt-cmake . -B build -G Ninja   # configura (añade CMAKE_PREFIX_PATH del kit)
cmake --build build                                    # compila
ctest --test-dir build --output-on-failure             # tests
```

---

## Índice de contenidos

0. [Antes de empezar (toolchain Qt 6 + CMake)](#paso-0--antes-de-empezar-toolchain-qt-6--cmake)
1. [Primer contacto con QML (scratchpad)](#paso-1--primer-contacto-con-qml-scratchpad)
2. [El workspace: tres targets CMake (cm-lib, cm-ui, cm-tests)](#paso-2--el-workspace-tres-targets-cmake)
3. [Navegación y estructura de vistas](#paso-3--navegación-y-estructura-de-vistas)
4. [Patrón Command + barra de comandos + estilo](#paso-4--framework-de-comandos-y-componentes)
5. [Entidades auto-conscientes con decorators](#paso-5--núcleo-de-datos-entidades-y-decorators)
6. [Tests unitarios sistemáticos (CTest)](#paso-6--tests-unitarios-sistemáticos-ctest)
7. [Persistencia SQLite](#paso-7--persistencia-sqlite)
8. [Red y RSS](#paso-8--red-y-rss)
9. [Inyección de dependencias + empaquetado](#paso-9--refactor-de-factoría-e-empaquetado)
10. [Recursos de documentación Qt 6](#recursos-de-documentación-qt-6)

---

## Paso 0 — Antes de empezar (toolchain Qt 6 + CMake)

*(Fase 0 del plan.)*

### 0.1 Instalar el toolchain

1. **Qt Maintenance Tool** → *Custom install*:
   - **Qt → Qt 6.11.2** (o 6.8 LTS): componentes *Desktop*, *Quick/QML* (incluido), *SQLite driver* (en
     *Additional Libraries → Qt SQL*), *Qt Network*, *Qt XML*, *Qt Test* (incluido en Desktop).
   - **Developer and Designer Tools**: **CMake ≥ 3.21** y **Ninja** (ambos vienen en el instalador de Qt).
2. Compilador: MSVC 2019+/Build Tools (Windows), gcc (Linux), clang/Xcode (macOS).
3. Qt Creator (opcional pero cómodo: detecta kits CMake automáticamente).

Verificación:

```bash
cmake --version        # >= 3.21
ninja --version
~/Qt/6.11.2/gcc_64/bin/qt-cmake --version   # wrapper del kit Qt 6.11.2 (o >= 6.8)
```

📚 Doc: [Get and install Qt](https://doc.qt.io/qt-6/get-and-install-qt.html) ·
[Building with CMake](https://doc.qt.io/qt-6/cmake-get-started.html) ·
[CMake for qmake users (guía de portabilidad)](https://doc.qt.io/qt-6/cmake-for-qmake-users.html)

### 0.2 Estructura de carpetas

```
cm/
├── CMakeLists.txt              # raíz: find_package(Qt6 6.8 REQUIRED ...) + subdirectorios
├── scratch/                    # playground del Paso 1
└── (Paso 2) cm-lib/  cm-ui/  cm-tests/
```

> No existen ya `qmake-target-platform.pri` ni `qmake-destination-path.pri`: **CMake detecta SO, compilador y
> arquitectura por sí solo**, y las salidas se agrupan con `CMAKE_*_OUTPUT_DIRECTORY` (ver Paso 2).

✅ **Comprobación**: `qt-cmake` disponible en el kit ≥ 6.8 y `cmake --version` ≥ 3.21.

---

## Paso 1 — Primer contacto con QML (scratchpad)

*(Fase 0 del plan: practicar QML antes de arquitecturar.)*

### 1.1 `scratch/CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.21)
project(scratch LANGUAGES CXX)

find_package(Qt6 6.8 REQUIRED COMPONENTS Quick)
qt_standard_project_setup(REQUIRES 6.8)      # C++17 + AUTOMOC + políticas Qt

qt_add_executable(scratch main.cpp)

qt_add_qml_module(scratch
    URI scratch
    VERSION 1.0
    QML_FILES main.qml
)

target_link_libraries(scratch PRIVATE Qt6::Quick)
```

Tres ideas clave de Qt 6 + CMake:

- `qt_standard_project_setup(REQUIRES 6.8)` fija **C++17** y toda la infraestructura (AUTOMOC, etc.) en una línea.
- `qt_add_qml_module` compila los `.qml` dentro del ejecutable (equivalente a un `.qrc` gestionado) y genera el
  `qmldir`; la ruta canónica de recursos pasa a ser `qrc:/qt/qml/<ruta-del-URI>/...`.
- `qt_add_executable` en lugar de `add_executable` (maneja la política WIN32/console y metadatos).

### 1.2 `scratch/main.cpp`

```cpp
#include <QGuiApplication>
#include <QQmlApplicationEngine>

int main(int argc, char *argv[])
{
    QGuiApplication app(argc, argv);          // en Qt 6 no hace falta AA_EnableHighDpiScaling

    QQmlApplicationEngine engine;
    engine.loadFromModule("scratch", "main"); // ← forma moderna: carga por (módulo, componente)

    if (engine.rootObjects().isEmpty())
        return -1;

    return app.exec();
}
```

`loadFromModule()` (Qt ≥ 6.5) sustituye a `engine.load(QUrl("qrc:/main.qml"))` y evita errores de ruta al cambiar
de esquema de recursos.

### 1.3 `scratch/main.qml`

```qml
import QtQuick
import QtQuick.Window
import QtQuick.Controls

Window {
    width: 640
    height: 480
    visible: true
    title: "cm — scratchpad"

    Label {
        anchors.centerIn: parent
        text: "Hola QML 6"
        font.pixelSize: 32
    }
}
```

Observa: imports **sin versión**. Si vienes de los capítulos del libro (`import QtQuick 2.9`) quita siempre el
número; la versión la resuelve el módulo CMake.

### 1.4 Compilar y ejecutar

```bash
cd scratch
~/Qt/6.11.2/gcc_64/bin/qt-cmake . -B build -G Ninja
cmake --build build
./build/scratch            # Linux; Windows: build\scratch.exe
```

### 1.5 Practicar layouts

Añade `AnchorsDemo.qml` y `SizingDemo.qml` al `QML_FILES` del módulo y cárgalos alternando `loadFromModule`.
Revisa `RowLayout`/`ColumnLayout` (requieren `import QtQuick.Layouts`).

📚 Doc: [Qt Quick modules con CMake](https://doc.qt.io/qt-6/qt-add-qml-module.html) ·
[QML Layouts](https://doc.qt.io/qt-6/qml-layout-element.html) ·
[QQmlApplicationEngine](https://doc.qt.io/qt-6/qqmlapplicationengine.html)

✅ **Comprobación**: ventana QML renderizada con Qt 6.11.2 (o 6.8) construida 100 % con CMake.

---

## Paso 2 — El workspace: tres targets CMake

*(Fase 1 del plan: cm-lib, cm-ui, cm-tests compilan y se enlazan.)*

### 2.1 Raíz `cm/CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.21)
project(cm LANGUAGES CXX)

find_package(Qt6 6.8 REQUIRED COMPONENTS Qml Quick Sql Network Xml Test)  # objetivo 6.11.2
qt_standard_project_setup(REQUIRES 6.8)                                   # C++17 para todos los targets

enable_testing()

# Salidas ordenadas (reemplazo de DESTINATION_PATH de los .pri históricos)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/binaries/$<CONFIG>)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/binaries/$<CONFIG>)

add_subdirectory(cm-lib)
add_subdirectory(cm-ui)
add_subdirectory(cm-tests)
```

La guarda de versión vive ahora en `find_package(Qt6 6.8 REQUIRED ...)`: si el kit configurado es Qt 5 o Qt 6.7,
CMake aborta con mensaje claro. Ya no hace falta el `error()` de qmake.

### 2.2 `cm-lib` — librería compartida (lógica de negocio)

`cm-lib/CMakeLists.txt`:

```cmake
qt_add_library(cm-lib SHARED
    cm-lib_global.h
    source/framework/command.h        source/framework/command.cpp
    source/controllers/master-controller.h source/controllers/master-controller.cpp
    source/models/client.h            source/models/client.cpp
)

target_include_directories(cm-lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/source)

target_link_libraries(cm-lib PUBLIC Qt6::Qml Qt6::Quick)

target_compile_definitions(cm-lib PRIVATE CMLIB_LIBRARY)   # dllexport al compilar la lib
```

`cm-lib/cm-lib_global.h` (igual que en el repo original):

```cpp
#ifndef CMLIB_GLOBAL_H
#define CMLIB_GLOBAL_H

#include <QtCore/qglobal.h>

#if defined(CMLIB_LIBRARY)
#  define CMLIBSHARED_EXPORT Q_DECL_EXPORT
#else
#  define CMLIBSHARED_EXPORT Q_DECL_IMPORT
#endif

#endif // CMLIB_GLOBAL_H
```

Primer modelo, `source/models/client.h` — en Qt 6 + CMake el tipo se registra **solo con la macro**, sin
`qmlRegisterType`:

```cpp
#ifndef CLIENT_H
#define CLIENT_H

#include <QObject>
#include <QString>
#include <QDate>
#include <QtQml/qqmlregistration.h>   // ← macros QML_ELEMENT etc.
#include "cm-lib_global.h"

class CMLIBSHARED_EXPORT Client : public QObject
{
    Q_OBJECT
    QML_ELEMENT                          // ← disponible en QML como "Client"
    Q_PROPERTY(QString name READ name WRITE setName NOTIFY nameChanged)
    Q_PROPERTY(QDate birthDate READ birthDate WRITE setBirthDate NOTIFY birthDateChanged)

public:
    explicit Client(QObject *parent = nullptr);

    QString name() const { return m_name; }
    void setName(const QString &v) { if (v != m_name) { m_name = v; emit nameChanged(); } }

    QDate birthDate() const { return m_birthDate; }
    void setBirthDate(const QDate &v) { if (v != m_birthDate) { m_birthDate = v; emit birthDateChanged(); } }

signals:
    void nameChanged();
    void birthDateChanged();

private:
    QString m_name;
    QDate   m_birthDate;
};

#endif // CLIENT_H
```

Y `master-controller.h` igual (`QML_ELEMENT` + `CMLIBSHARED_EXPORT`), actuando como fachada raíz expuesta a QML.

> **Equivalencias qmake → CMake** (tabla completa en
> [Porting from qmake to CMake](https://doc.qt.io/qt-6/cmake-for-qmake-users.html)):
> `TEMPLATE = lib` → `qt_add_library(SHARED)` · `QT += qml quick` → `target_link_libraries(... Qt6::Qml Qt6::Quick)` ·
> `CONFIG += c++17` → `qt_standard_project_setup(REQUIRES 6.8)` · `DEFINES += X` → `target_compile_definitions` ·
> `DESTDIR/MOC_DIR/RCC_DIR` → `CMAKE_*_OUTPUT_DIRECTORY` + AUTOMOC · `qmlRegisterType` → `QML_ELEMENT`.

### 2.3 `cm-ui` — ejecutable GUI con módulo QML

`cm-ui/CMakeLists.txt`:

```cmake
qt_add_executable(cm-ui MACOSX_BUNDLE main.cpp)

target_link_libraries(cm-ui PRIVATE cm-lib Qt6::Quick)

# Módulo QML principal: sustituye a views.qrc / components.qrc / assets.qrc del mundo qmake
qt_add_qml_module(cm-ui
    URI cm.ui
    VERSION 1.0
    QML_FILES
        views/MasterView.qml
        views/SplashView.qml
        views/DashboardView.qml
        views/CreateClientView.qml
        views/EditClientView.qml
        views/FindClientView.qml
        components/CommandBar.qml
        components/CommandButton.qml
        components/NavigationBar.qml
        components/NavigationButton.qml
        assets/Style.qml
    RESOURCES
        assets/images/logo.png
)
```

`cm-ui/main.cpp` — exponer el controller raíz y cargar `MasterView` por módulo:

```cpp
#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include <QQmlContext>
#include "controllers/master-controller.h"

int main(int argc, char *argv[])
{
    QGuiApplication app(argc, argv);

    MasterController masterController;

    QQmlApplicationEngine engine;
    engine.rootContext()->setContextProperty("masterController", &masterController);
    engine.loadFromModule("cm.ui", "MasterView");   // ← carga qrc:/qt/qml/cm/ui/MasterView.qml

    if (engine.rootObjects().isEmpty())
        return -1;

    return app.exec();
}
```

`views/MasterView.qml` (versión mínima, imports Qt 6):

```qml
import QtQuick
import QtQuick.Window
import QtQuick.Controls

Window {
    width: 1024
    height: 768
    visible: true
    title: "cm"

    Label {
        anchors.centerIn: parent
        text: "MasterView lista — controller: " + (masterController ? "OK" : "faltando")
    }
}
```

> Para instanciar `Client` u otros `QML_ELEMENT` de **cm-lib** directamente en QML, convierte `cm-lib` en un
> módulo QML propio: `qt_add_qml_module(cm-lib URI cm.lib VERSION 1.0)` y luego `import cm.lib` en los `.qml`
> (los elementos viajan con la librería). Se consolida en el Paso 9.

### 2.4 `cm-tests` — primer test con QtTest + CTest

`cm-tests/CMakeLists.txt`:

```cmake
qt_add_executable(client-tests main.cpp client-tests.cpp client-tests.h)

target_include_directories(client-tests PRIVATE ${CMAKE_SOURCE_DIR}/cm-lib/source)
target_link_libraries(client-tests PRIVATE cm-lib Qt6::Test)

add_test(NAME client-tests COMMAND client-tests)   # registrado en CTest
```

`cm-tests/main.cpp`:

```cpp
#include <QtTest>
#include "client-tests.h"

QTEST_MAIN(ClientTests)
```

`cm-tests/client-tests.h`:

```cpp
#ifndef CLIENT_TESTS_H
#define CLIENT_TESTS_H

#include <QObject>
#include "models/client.h"

class ClientTests : public QObject
{
    Q_OBJECT
private slots:
    void initTestCase();
    void name_property_works();
    void name_changed_signal_fires();
};

#endif // CLIENT_TESTS_H
```

`cm-tests/client-tests.cpp`:

```cpp
#include "client-tests.h"
#include <QSignalSpy>

void ClientTests::initTestCase() {}

void ClientTests::name_property_works()
{
    Client c;
    c.setName("Ana");
    QCOMPARE(c.name(), QStringLiteral("Ana"));
}

void ClientTests::name_changed_signal_fires()
{
    Client c;
    QSignalSpy spy(&c, &Client::nameChanged);
    c.setName("Luis");
    c.setName("Luis");           // mismo valor: no debe emitir
    QCOMPARE(spy.count(), 1);
}
```

### 2.5 Compilar el workspace

```bash
cd cm
~/Qt/6.11.2/gcc_64/bin/qt-cmake . -B build -G Ninja
cmake --build build
ctest --test-dir build --output-on-failure
./binaries/Debug/cm-ui        # o Release según configuración
```

En Qt Creator: **File → Open File or Project** sobre `CMakeLists.txt` (no `cm.pro`) y elegir kit Qt 6.11.2.

✅ **Salida**: `libcm-lib.so|cm-lib.dll`, `cm-ui`, `client-tests` en `binaries/<Config>/`; `ctest` en verde.

---

## Paso 3 — Navegación y estructura de vistas

*(Fase 2 del plan.)*

1. **`NavigationController`** en `cm-lib/source/controllers/`:

```cpp
class NavigationController : public QObject
{
    Q_OBJECT
    QML_ELEMENT
    QML_UNCREATABLE("Solo accesible via masterController")
    Q_PROPERTY(QString page READ page NOTIFY pageChanged)

public:
    enum Page { Splash, Dashboard, CreateClient, EditClient, FindClient };
    Q_ENUM(Page)

    explicit NavigationController(QObject *parent = nullptr);
    Q_INVOKABLE void navigateTo(int page);   // llamado desde QML

    QString page() const;

signals:
    void pageChanged();

private:
    int m_page = Splash;
};
```

Regla de oro del libro: **la navegación es lógica de negocio → vive en C++**; QML solo reacciona a `page`.

2. **Vistas**: `SplashView`, `DashboardView`, `CreateClientView`, `EditClientView`, `FindClientView` — ficheros
   nuevos añadidos a `QML_FILES` del `qt_add_qml_module` de `cm-ui` (Paso 2.3). No se crea ningún `.qrc`.

3. **`MasterView.qml`** como contenedor con `StackLayout` (Qt Quick Controls 6):

```qml
import QtQuick
import QtQuick.Window
import QtQuick.Controls

Window {
    width: 1024; height: 768; visible: true
    StackLayout {
        anchors.fill: parent
        currentIndex: navigation.page === "splash" ? 0
                    : navigation.page === "dashboard" ? 1
                    : navigation.page === "createClient" ? 2
                    : navigation.page === "editClient" ? 3 : 4
        SplashView {}
        DashboardView {}
        CreateClientView {}
        EditClientView {}
        FindClientView {}
    }
}
```

(`navigation` puede exponerse como propiedad de contexto en `main.cpp` o, mejor, como child de
`masterController` accesible vía `masterController.navigationController`.)

4. La splash lanza un `Timer` de 2 s que hace `navigation.navigateTo(NavigationController.Dashboard)`.

📚 Doc: [StackLayout](https://doc.qt.io/qt-6/qml-qtquick-controls-stacklayout.html) ·
[Q_ENUM y enumeraciones en QML](https://doc.qt.io/qt-6/qqmlintegration.html#enumerations).

✅ **Comprobación**: se navega entre las 4 pantallas desde la splash.

---

## Paso 4 — Framework de comandos y componentes

*(Fase 3 del plan.)*

1. **`framework/command.{h,cpp}`**: QObject con `name`, `icon`, `enabled`, `visible` y `Q_INVOKABLE execute()`
   que dispara un `std::function<void()>` (C++17) asignado por el controlador. Marcado con `QML_ELEMENT`:

```cpp
class Command : public QObject
{
    Q_OBJECT
    QML_ELEMENT
    QML_NAMED_ELEMENT(AppCommand)     // evita colisión de nombre con otros "Command"
    Q_PROPERTY(QString name READ name CONSTANT)
    Q_PROPERTY(bool enabled READ enabled WRITE setEnabled NOTIFY enabledChanged)
    Q_PROPERTY(bool visible READ visible WRITE setVisible NOTIFY visibleChanged)

public:
    explicit Command(const QString &name, std::function<void()> callback, QObject *parent = nullptr);
    Q_INVOKABLE void execute() { if (m_callback) m_callback(); }
    // getters/setters ...
signals:
    void enabledChanged();
    void visibleChanged();
private:
    QString m_name;
    bool m_enabled = true, m_visible = true;
    std::function<void()> m_callback;
};
```

2. **`CommandController`**: define comandos (Save, Cancel, Back, Find…) y expone
   `Q_PROPERTY(QVariantList commands READ commands NOTIFY commandsChanged)`.

3. **Componentes** en `cm-ui/components/` (ya listados en `QML_FILES`): `CommandBar.qml` con
   `Repeater { model: masterController.commandController.commands }` y delegate `CommandButton.qml`
   (`Controls.Button` con `text: modelData.name`, `onClicked: modelData.execute()`).

4. **Estilo singleton con CMake**: `assets/Style.qml`:

```qml
pragma Singleton
import QtQuick
QtObject {
    property color primaryColor: "#2d89ef"
    property int fontSize: 14
    property string fontFamily: "Segoe UI"
}
```

y en el `qt_add_qml_module` de `cm-ui`, márcalo así (reemplaza al viejo `qmldir` manual y a `assets.qrc`):

```cmake
set_source_files_properties(assets/Style.qml PROPERTIES QT_QML_SINGLETON_TYPE TRUE)
```

Úsalo en cualquier vista con `import cm.ui` (el singleton viaja dentro del módulo).

📚 Doc: [Singletons en QML](https://doc.qt.io/qt-6/qtqml-documents-singletons.html) ·
[`QT_QML_SINGLETON_TYPE`](https://doc.qt.io/qt-6/qt-add-qml-module.html) ·
[Delegates](https://doc.qt.io/qt-6/qml-delegates.html).

✅ **Comprobación**: barras de navegación/comandos visibles y funcionales; la lógica vive 100 % en C++.

---

## Paso 5 — Núcleo de datos: entidades y decorators

*(Fase 4 del plan.)*

1. **`data/data-decorator.{h,cpp}`**: base `QML_ELEMENT` con `dirty`, señal `dirtyChanged()` y virtual `validate()`.
2. **Decoradores concretos** (cada uno `Q_PROPERTY(QVariant value READ write NOTIFY valueChanged)` + reglas):
   `StringDecorator` (longitud min/max, regex con `QRegularExpression` — API Qt 6, no `QRegExp`),
   `IntDecorator` (rango), `DateTimeDecorator` (validez de fecha), `EnumeratorDecorator` (lista de opciones).
3. **`Entity`**: colección de decoradores registrada por nombre; `read(const QVariantMap&)`, `write()`,
   `toJson()` (con `QJsonDocument`), dirty tracking agregado; `EntityCollection` para listas.
4. **Modelos completos**: `Client`, `Address`, `Contact`, `Appointment` construidos sobre `Entity`, cada campo
   es un decorator expuesto como propiedad (`Q_PROPERTY(StringDecorator* name READ name CONSTANT)`).
5. **Formulario genérico** `components/StringEditorSingleLine.qml`:

```qml
import QtQuick
import QtQuick.Controls
import cm.ui

Item {
    required property var decorator   // StringDecorator desde C++
    TextField {
        anchors.fill: parent
        text: decorator.value
        placeholderText: decorator.label
        onTextChanged: decorator.value = text
        color: decorator.valid ? Style.primaryColor : "red"
    }
}
```

Todos los nuevos `.h/.cpp` van a la lista de fuentes de `cm-lib/CMakeLists.txt`; los nuevos `.qml`, a
`QML_FILES` de `cm-ui`. Con `QML_ELEMENT` no hay nada más que registrar.

📚 Doc: [QQmlIntegration (macros QML)](https://doc.qt.io/qt-6/qqmlintegration.html) ·
[JSON](https://doc.qt.io/qt-6/json.html) · [QRegularExpression](https://doc.qt.io/qt-6/qregularexpression.html).

✅ **Comprobación**: formularios leen/escriben entidades C++ con validación en vivo; round-trip JSON correcto.

---

## Paso 6 — Tests unitarios sistemáticos (CTest)

*(Fase 5 del plan.)*

Con CMake, la suite agregadora del libro se simplifica: **cada suite es un target registrado en CTest**
(o se mantiene el binario agregador que invoca `QTest::qExec` de varias clases). Opción recomendada en Qt 6:

```cmake
# cm-tests/CMakeLists.txt (ampliado)
foreach(suite string-decorator-tests int-decorator-tests datetime-decorator-tests
              enumerator-decorator-tests client-tests master-controller-tests)
    qt_add_executable(${suite} ${suite}.cpp main-test-runner.cpp)
    target_include_directories(${suite} PRIVATE ${CMAKE_SOURCE_DIR}/cm-lib/source)
    target_link_libraries(${suite} PRIVATE cm-lib Qt6::Test)
    add_test(NAME ${suite} COMMAND ${suite})
endforeach()
```

Ejecutar todo: `ctest --test-dir build --output-on-failure` (resultados también en JUnit XML con `--output-junit`).

Macros de mocking ligeras (`mocking.h`) e interfaces mockeables (`i-database-controller.h`, luego
`MockObjectFactory`) se heredan tal cual del repo: son C++ puro, compatible con C++17.

📚 Doc: [QtTest](https://doc.qt.io/qt-6/qtest-overview.html) · [CTest](https://doc.qt.io/qt-6/cmake-get-started.html).

✅ **Comprobación**: `ctest` ejecuta todas las suites en cada plataforma; CI-ready sin scripts extra.

---

## Paso 7 — Persistencia SQLite

*(Fase 6 del plan.)*

1. **`IDatabaseController`** (interfaz pura) + **`DatabaseController`** marcado `QML_ELEMENT`. Añade
   `Qt6::Sql` a `target_link_libraries` de `cm-lib`:

```cpp
bool DatabaseController::openOrCreate()
{
    m_db = QSqlDatabase::addDatabase("QSQLITE");
    m_db.setDatabaseName(QCoreApplication::applicationDirPath() + "/cm.sqlite");
    if (!m_db.open()) return false;
    QSqlQuery q(m_db);
    return q.exec("CREATE TABLE IF NOT EXISTS clients ("
                  "id INTEGER PRIMARY KEY AUTOINCREMENT,"
                  "name TEXT, birth_date TEXT, status INTEGER)");
}

int DatabaseController::insertClient(Client *client)
{
    QSqlQuery q(m_db);
    q.prepare("INSERT INTO clients (name, birth_date, status) VALUES (:name, :bd, :st)");
    q.bindValue(":name", client->name()->value());
    q.bindValue(":bd",   client->birthDate()->value().toString(Qt::ISODate));
    q.bindValue(":st",   client->status()->value().toInt());
    if (!q.exec()) return -1;
    return q.lastInsertId().toInt();
}
```

2. **`ClientSearch`**: objeto de criterios → `WHERE` parametrizado (nunca concatenar SQL); devuelve resultados.
3. Integrar en `MasterController`: `save`, `find`, `delete` conectados a los comandos de la UI (Paso 4).
4. Vistas: `FindClientView` con `ListView` + `SearchResultDelegate`, `AddressEditor`, `Panel` — nuevas entradas
   en `QML_FILES`.

📚 Doc: [SQL Programming](https://doc.qt.io/qt-6/sql-programming.html) ·
[driver QSQLITE](https://doc.qt.io/qt-6/sql-driver.html#qsqlite) ·
[consultas preparadas](https://doc.qt.io/qt-6/qsqlquery.html).

✅ **Comprobación**: crear cliente → cerrar app → buscarlo y editarlo; datos persistentes en `cm.sqlite`.

---

## Paso 8 — Red y RSS

*(Fase 7 del plan.)*

Añade `Qt6::Network Qt6::Xml` a `target_link_libraries` de `cm-lib`.

1. Capa `networking/`: `INetworkAccessManager`/`NetworkAccessManager` (wrapper de `QNetworkAccessManager`) e
   `IWebRequest`/`WebRequest` (URL, señal `finished`, cuerpo). En Qt 6 la descarga es asíncrona por diseño:
   conecta `QNetworkReply::finished` y llama `reply->deleteLater()`.

```cpp
void WebRequest::get(const QUrl &url)
{
    QNetworkRequest req{url};
    req.setHeader(QNetworkRequest::UserAgentHeader, "cm-app/1.0");
    m_reply = m_manager->get(req);
    connect(m_reply, &QNetworkReply::finished, this, &WebRequest::onFinished);
}

void WebRequest::onFinished()
{
    if (m_reply->error() == QNetworkReply::NoError)
        m_body = m_reply->readAll();
    emit finished(m_error.isEmpty());
    m_reply->deleteLater();
}
```

2. `utilities/xml-helper`: lectura defensiva con `QXmlStreamReader` (elementos/atributos con defaults).
3. Modelos `rss/`: `RssChannel`, `RssItem`, `RssImage` parseados desde el XML del feed.
4. UI: `RssView.qml` + `RssItemDelegate.qml` en el dashboard (`QML_FILES`); enlazar con un
   `QAbstractListModel` (más idiomático que `QVariantList` para listas grandes).

📚 Doc: [QNetworkAccessManager](https://doc.qt.io/qt-6/qnetworkaccessmanager.html) ·
[QXmlStreamReader](https://doc.qt.io/qt-6/qxmlstreamreader.html) ·
[Model/view](https://doc.qt.io/qt-6/model-view-programming.html).

✅ **Comprobación**: listado RSS en vivo en el dashboard; tests siguen verdes (las interfaces permiten mocks sin red).

---

## Paso 9 — Refactor de factoría e empaquetado

*(Fase 8 del plan.)*

1. Extraer interfaces `ICommandController`, `INavigationController`, `IObjectFactory`; implementar `ObjectFactory`
   (creación centralizada de controladores/entidades). Usa siempre sintaxis de punteros para señales/slots
   (`connect(sender, &T::signal, ...)`), nunca las macros antiguas `SIGNAL()/SLOT()`.
2. Sustituir construcciones directas por la fábrica → testeable con `MockObjectFactory` en `cm-tests`.
3. Controles avanzados: `Dropdown`/`DropdownValue`, `EnumeratorSelector(View)`, `FormButton`,
   `ContactsEditor`/`ContactDelegate` — a `QML_FILES` y/o `QML_ELEMENT` según corresponda.
4. Consolidar `cm-lib` como módulo QML real (`qt_add_qml_module(cm-lib URI cm.lib ...)`) para que las vistas
   hagan `import cm.lib` y usen `Client`, `AppCommand`, etc. como tipos propios del módulo.
5. **Empaquetado con CMake + deploy tools de Qt 6**:

```cmake
install(TARGETS cm-lib cm-ui
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
)
```

```bash
# Windows (cmd del kit):
windeployqt --qmldir cm-ui dist/bin/cm-ui.exe
# macOS:
macdeployqt dist/cm-ui.app
# Linux: linuxdeploy + AppImage, o copiar plugins platforms/sqldrivers manualmente
```

En MSVC añade `--compiler-runtime` a `windeployqt`. El bundle antiguo `installer/osx/cm-ui.app` del repo era de
Qt 5: se regenera con el flujo anterior. Ventaja extra del flujo CMake: `qt_add_qml_module` ya activa
**`qmlcachegen`** en Release (QML precompilado a C++) sin configuración adicional.

📚 Doc: [Deployment con CMake](https://doc.qt.io/qt-6/deployment-cmake.html) ·
[windeployqt](https://doc.qt.io/qt-6/windeployqt.html) · [macdeployqt](https://doc.qt.io/qt-6/macdeployqt.html).

✅ **Comprobación**: aplicación instalable en las tres plataformas, construida íntegramente con CMake, con toda
la funcionalidad (CRUD + RSS + navegación + comandos) y `ctest` en verde.

---

## Recursos de documentación Qt 6

| Tema | Enlace |
|---|---|
| **CMake manual (central en Qt 6)** | https://doc.qt.io/qt-6/cmake-manual.html |
| Empezar con CMake + Qt | https://doc.qt.io/qt-6/cmake-get-started.html |
| Porting desde qmake (referencia histórica) | https://doc.qt.io/qt-6/cmake-for-qmake-users.html |
| `qt_add_qml_module` | https://doc.qt.io/qt-6/qt-add-qml-module.html |
| Macros QML / QQmlIntegration | https://doc.qt.io/qt-6/qqmlintegration.html |
| Qt Quick / Controls | https://doc.qt.io/qt-6/qtquick-index.html |
| SQL / SQLite | https://doc.qt.io/qt-6/sql-driver.html#qsqlite |
| Network | https://doc.qt.io/qt-6/qtnetwork-programming.html |
| Qt Test + CTest | https://doc.qt.io/qt-6/qtest-overview.html |
| Despliegue | https://doc.qt.io/qt-6/deployment.html |

### Checklist de errores típicos de la migración qmake→CMake / Qt5→Qt6

1. ❌ Seguir creando/editando `.pro`/`.pri` → ✅ todo el build en `CMakeLists.txt` invocado con `qt-cmake` del kit.
2. ❌ `CONFIG += c++14` (o no fijar estándar) → ✅ `qt_standard_project_setup(REQUIRES 6.8)` (C++17 garantizado).
3. ❌ `import QtQuick 2.9` / `QtQuick.Controls 2.2` → ✅ imports sin versión.
4. ❌ `qmlRegisterType<T>("uri", 1, 0, "T")` en `main.cpp` → ✅ macro `QML_ELEMENT` (+ `QML_NAMED_ELEMENT`,
   `QML_SINGLETON`, `QML_UNCREATABLE`) dentro de la clase y módulo declarado en CMake; `qmlRegisterType` está
   deprecado desde Qt 6.9.
5. ❌ `.qrc` manuales para QML (`views.qrc`, `assets.qrc`) → ✅ `QML_FILES`/`RESOURCES` de `qt_add_qml_module`;
   cargar con `engine.loadFromModule("cm.ui", "MasterView")`.
6. ❌ Ruta `qrc:/views/MasterView.qml` → ✅ ruta canónica del módulo: `qrc:/qt/qml/cm/ui/MasterView.qml`.
7. ❌ Singleton registrado con `qmlRegisterSingletonType`/`qmldir` manual → ✅ `pragma Singleton` +
   `set_source_files_properties(... PROPERTIES QT_QML_SINGLETON_TYPE TRUE)`.
8. ❌ `QRegExp`, `QString::sprintf`, `QTextCodec` → ✅ `QRegularExpression`, `QString::arg`, codecs eliminados en Qt 6.
9. ❌ `AA_EnableHighDpiScaling`/`AA_UseHighDpiPixmaps` → ✅ redundantes: DPI alto siempre activo en Qt 6.
10. ❌ Flags de plataforma artesanales (`.pri` con `win32/linux/macx`) → ✅ generadores multiplataforma de CMake
    (`$<CONFIG>`, toolchain files, `qt-cmake`).
11. ❌ Invocar `qmake`/`qmake6` del PATH → ✅ `qt-cmake` del kit (o `cmake` con `CMAKE_PREFIX_PATH` al directorio Qt).
12. ⚠️ MSVC: `qt-cmake` debe correr en la *Developer Command Prompt* (necesita las env vars del compilador).
