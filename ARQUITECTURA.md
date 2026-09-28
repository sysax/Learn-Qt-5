# Arquitectura del proyecto **cm**

Aplicación LOB (*Line of Business*) de gestión de clientes construida con **QML + C++17**, documentada en
[`CONSTRUCCION_POR_FASES.md`](CONSTRUCCION_POR_FASES.md) y [`TUTORIAL_DESDE_CERO.md`](TUTORIAL_DESDE_CERO.md).

> **Versiones objetivo**: Qt **6.11.2** (mínimo **Qt 6.8 LTS**), **CMake** como sistema de build oficial
> (qmake deprecado desde Qt 6.7), **C++17 mínimo**. El código histórico de los capítulos del repositorio
> (`Chapter01`–`Chapter09`) está escrito para **Qt 5.x + QMake**; la guía de fases describe la migración
> completa a Qt 6 (imports QML sin versión, `.pro`/`.pri` → `CMakeLists.txt`, `qmlRegisterType` → `QML_ELEMENT`).

---

## 1. Resumen ejecutivo

| Aspecto | Decisión |
|---|---|
| Lenguaje | **C++17 (mínimo)** + QML / Qt Quick |
| Framework | **Qt 6.11.2** (objetivo) / **Qt 6.8 LTS** (mínimo) — módulos `qml`, `quick`, `sql`, `network`, `xml`, `test` |
| Sistema de build | **CMake** con `qt_standard_project_setup` + `qt_add_qml_module` (histórico: QMake `.pro`/`.pri`) |
| Persistencia | **SQLite** (`QSQLITE`), fichero `cm.sqlite` junto al ejecutable |
| Pruebas | **Qt Test** (`testlib`) con suite agregadora + mocks, registrados en **CTest** |
| Estilo arquitectónico | **MVVM ligero**: lógica 100 % en C++, QML sólo presenta y enlaza |
| Plataformas | Windows (MSVC/MinGW), Linux (gcc/clang), macOS (clang) — x86_64/arm64, debug/release |

### Funcionalidad
- Splash inicial + dashboard con feed RSS en vivo.
- Crear / editar / buscar / borrar clientes (nombre, fecha de nacimiento, estado, contactos, direcciones).
- Formularios validados enlazados a entidades C++ auto-conscientes (serialización JSON).
- Búsqueda con criterios combinados sobre base de datos.
- Consumo de un canal RSS vía red con parseo XML defensivo.
- Paquete instalable por plataforma (`windeployqt` / `macdeployqt` / `linuxdeploy`).

---

## 2. Estructura general: tres targets

Regla de oro: **el UI no contiene lógica de negocio**; todo lo testeable vive en `cm-lib`.

```
CMakeLists.txt (raíz: find_package(Qt6 6.8) + add_subdirectory)   # histórico: cm.pro subdirs
├── cm-lib    → librería compartida (qt_add_library SHARED; toda la lógica de negocio)
├── cm-ui     → ejecutable GUI (qt_add_executable + qt_add_qml_module); enlaza contra cm-lib
└── cm-tests  → ejecutable de consola con tests unitarios (Qt6::Test + CTest); enlaza contra cm-lib
```

Diagrama de dependencias:

```
            ┌─────────────┐        ┌─────────────┐
            │    cm-ui    │        │   cm-tests  │
            │ (QML + main)│        │  (Qt Test)  │
            └──────┬──────┘        └──────┬──────┘
                   │  enlaza              │  enlaza (+ mocks)
                   ▼                      ▼
            ┌──────────────────────────────────┐
            │             cm-lib               │
            │  controllers · data · models     │
            │  framework · networking · rss    │
            │  utilities                       │
            └──────────────────────────────────┘
                   │ Qt6::Qml/Quick/Sql/Network/Xml
                   ▼
                 Qt 6 SDK
```

Salidas ordenadas por configuración (reemplazo de los `.pri` `DESTINATION_PATH` de qmake):

```cmake
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/binaries/$<CONFIG>)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/binaries/$<CONFIG>)
```

---

## 3. `cm-lib` — núcleo de negocio

Estructura interna de `cm-lib/source`:

```
cm-lib_global.h   Macro de exportación CMLIB_EXPORT (windows dllimport/dllexport)
framework/        Command (patrón Command), ObjectFactory + IObjectFactory (fábrica / inversión de dependencias)
controllers/      MasterController (agregador / fachada raíz)
                  NavigationController (estado de navegación)
                  CommandController  (comandos Save/Edit/Delete/Back/Find…)
                  DatabaseController + IDatabaseController (persistencia)
data/             Entity (entidad auto-consciente) + DataDecorator
                  StringDecorator, IntDecorator, DateTimeDecorator, EnumeratorDecorator
                  Dropdown / DropdownValue, EntityCollection
models/           Client, Contact, Address, Appointment, ClientSearch
networking/       NetworkAccessManager + INetworkAccessManager, WebRequest + IWebRequest
rss/              RssChannel, RssItem, RssImage
utilities/        XmlHelper (lectura tolerante de nodos XML)
```

Componentes clave:

- **Patrón Command**: las acciones son objetos (`nombre`, `icono`, `enabled/visible`, slot `execute()`)
  expuestos directamente a QML; habilitar/deshabilitar y reordenar botones es trivial sin tocar la UI.
- **Entidades auto-conscientes**: cada campo de una entidad es un **decorator** (`StringDecorator`,
  `IntDecorator`, `DateTimeDecorator`, `EnumeratorDecorator`) expuesto a QML con `value`, `dirty`,
  señales de cambio y reglas de validación → formularios genéricos y reutilizables.
  `Entity::write(QVariantMap)` / `toJson()` permiten que las entidades se traduzcan a filas SQL y JSON
  (round-trip) sin ORM.
- **Inversión de dependencias**: interfaces (`i-*.h`: `IDatabaseController`, `INavigationController`,
  `ICommandController`, `IObjectFactory`) + `ObjectFactory` para creación centralizada; los tests usan
  `MockObjectFactory` y mocks ligeros (`mocking.h`).
- **Fachada raíz**: `MasterController` agrega a los demás controladores y es el único objeto expuesto
  al contexto QML.

---

## 4. `cm-ui` — capa de presentación (QML)

```
source/main.cpp          Arranque: QQmlApplicationEngine + registro de tipos + contexto
views/                   MasterView (shell), Splash, Dashboard, CreateClient, EditClient, FindClient, Rss
components/              CommandBar/Button, NavigationBar/Button, Panel, StringEditorSingleLine,
                         AddressEditor, SearchResultDelegate, ContactsEditor/ContactDelegate,
                         EnumeratorSelector(View), FormButton, RssItemDelegate
assets/Style.qml         Singleton de estilo (colores, tamaños, fuentes)
views.qrc / components.qrc / assets.qrc   Recursos embebidos (histórico Qt 5)
```

- En Qt 6, los `.qrc` manuales se sustituyen por `QML_FILES`/`RESOURCES` dentro de
  `qt_add_qml_module(TARGET cm-ui URI cm.ui ...)`; la vista raíz se carga desde la ruta canónica
  del módulo (`qrc:/qt/qml/cm/ui/MasterView.qml`).
- Los tipos C++ se exponen con macros **`QML_ELEMENT` / `QML_SINGLETON`** (no `qmlRegisterType`).
- Imports QML **sin número de versión** (`import QtQuick`, `import QtQuick.Controls`, …).
- `Style.qml` como singleton centraliza el tema visual.

---

## 5. Flujo de datos (MVVM ligero)

```
QML View  ⇄  property/signal  ⇄  Controller (cm-lib)  ⇄  Entity/Decorators  ⇄  SQLite
                                        │
                                        └── Command objects → CommandBar QML
```

1. `main.cpp` crea `MasterController` y lo expone como propiedad de contexto del `QQmlApplicationEngine`.
2. Las vistas QML se enlazan a propiedades de los decorators (`value`, `dirty`, `valid`) mediante bindings.
3. Los comandos (`Save`, `Find`, `Delete`…) viajan de `CommandController` a la `CommandBar`; su ejecución
   invoca lógica pura C++ (validación → entidad → `DatabaseController` → SQLite).
4. El dashboard consume red: `WebRequest` → `XmlHelper` → `RssChannel/RssItem` → `RssView`.

---

## 6. Build y toolchain (CMake sobre Qt 6)

```cmake
cmake_minimum_required(VERSION 3.21)
project(cm LANGUAGES CXX)

find_package(Qt6 6.8 REQUIRED COMPONENTS Qml Quick Sql Network Xml Test)  # objetivo: 6.11.2
qt_standard_project_setup(REQUIRES 6.8)   # C++17 + AUTOMOC/AUTORCC + políticas Qt
```

```
cm/
├── CMakeLists.txt          # add_subdirectory(cm-lib cm-ui cm-tests), guardas de versión
├── cm-lib/CMakeLists.txt   # qt_add_library(cm-lib SHARED) + QML elements
├── cm-ui/CMakeLists.txt    # qt_add_executable + qt_add_qml_module
└── cm-tests/CMakeLists.txt # qt_add_executable + suites Qt Test registradas en CTest
```

Comandos de referencia:

```bash
~/Qt/6.11.2/gcc_64/bin/qt-cmake . -B build -G Ninja   # configura (falla si Qt < 6.8)
cmake --build build                                    # compila cm-lib → cm-ui/cm-tests
./binaries/Release/cm-ui                               # lanza la app
ctest --test-dir build --output-on-failure             # ejecuta los tests
```

CMake detecta SO/compilador/arquitectura automáticamente, por lo que los `.pri` históricos de
detección de plataforma (`qmake-target-platform.pri`, `qmake-destination-path.pri`) quedan obsoletos.

---

## 7. Estrategia de pruebas

- `cm-tests`: binario de consola con **suite agregadora** (`test-suite.{h,cpp}`) que ejecuta varias
  suites en un solo proceso (resultados XML): suites por decorator, `client-tests`,
  `master-controller-tests`.
- **Mocks** sobre las interfaces (`MockObjectFactory`, mock de `IDatabaseController`,
  `INetworkAccessManager`) → persistencia y red testables sin entorno real.
- Integración con **CTest** (`enable_testing()` + `add_test`): verde en las tres plataformas antes de
  tocar fases de riesgo (persistencia/red).

---

## 8. Empaquetado y despliegue

- Reglas `install(TARGETS ...)` de CMake + stage de binarios + recursos + plugins Qt
  (`sqldrivers`, `platforms`) por plataforma; alternativa: `cmake --install build --prefix dist`.
- Generadores por SO: `windeployqt --qmldir cm-ui` (Windows), `macdeployqt cm-ui.app` (macOS),
  `linuxdeploy`/AppImage (Linux).
- Ejemplo incluido en el repo: `Chapter09/chapter9/cm/installer/osx/cm-ui.app` (bundle macOS,
  generado con Qt 5 → regenerar con `macdeployqt` bajo Qt 6).

---

## 9. Decisiones arquitectónicas clave

1. **Librería + UI + tests separados**: obliga a que la lógica sea independiente de la GUI y testeable.
2. **Patrón Command**: acciones como objetos tipados expuestos a QML; la UI queda declarativa.
3. **Decorators sobre miembros de entidad**: validación, tracking de suciedad y binding granular sin ORM.
4. **Interfaces (`i-*.h`) + ObjectFactory**: inversión de dependencias para mocks y evolución futura.
5. **Build único multiplataforma con CMake**: un mismo árbol fuente produce salidas por configuración
   (`binaries/<Config>`); abstrae SO, compilador y arquitectura.
6. **Recursos embebidos** (`.qrc` en Qt 5 → `qt_add_qml_module` en Qt 6): despliegue de un único
   ejecutable + librería.
7. **Versionado moderno**: Qt 6.11.2 objetivo (6.8 LTS mínimo), CMake obligatorio, C++17 mínimo vía
   `qt_standard_project_setup(REQUIRES 6.8)`, imports QML sin versión, tipos expuestos con
   `QML_ELEMENT`, y guarda `find_package(Qt6 6.8 REQUIRED ...)` que rechaza toolchains antiguos.

---

## 10. Correspondencia con los capítulos del repositorio

| Fase | Capítulo | Contenido arquitectónico |
|---|---|---|
| 0 | Chapter01 | Entorno Qt 6 + CMake, migración desde Qt 5/qmake, scratchpad QML |
| 1 | Chapter02 | Workspace: tres targets (`cm-lib`, `cm-ui`, `cm-tests`) |
| 2 | Chapter03 | Navegación (`NavigationController`) y shell de vistas |
| 3 | Chapter04 | Patrón Command, barras de comandos/navegación, `Style.qml` |
| 4 | Chapter05 | Entidades auto-conscientes y decorators |
| 5 | Chapter06 | Suite de tests unitarios + mocking |
| 6 | Chapter07 | Persistencia SQLite (`DatabaseController`, `ClientSearch`) |
| 7 | Chapter08 | Capa de red y parseo RSS/XML |
| 8 | Chapter09 | Inyección de dependencias (`ObjectFactory`) y empaquetado |
