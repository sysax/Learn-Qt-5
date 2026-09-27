# Proyecto **cm** — Guía paso a paso de construcción por fases

Aplicación LOB (Line of Business) de gestión de clientes construida con **QML / C++17** y **CMake (Qt 6)**.

> **Requisito de versión (actualizado)**: esta guía apunta a **Qt 6.11.2** como versión objetivo
> (**Qt 6.8 LTS** como versión mínima aceptable), **C++17 como mínimo** y **CMake como sistema de
> build** (`qt_standard_project_setup` + `qt_add_qml_module`): en Qt 6.11 qmake está deprecado
> (desde 6.7) y no es la vía soportada. El código histórico del repositorio está escrito para
> Qt 5.x + qmake; en la **Fase 0** se detalla la migración completa (imports QML sin versión,
> `.pro`/`.pri` → `CMakeLists.txt`, `qmlRegisterType` → `QML_ELEMENT`). Donde aparezca una
> discrepancia entre el código de los capítulos y esta guía, prevalecen Qt 6.8+, C++17 y CMake.

---

## 1. Visión general

| Aspecto | Decisión |
|---|---|
| Lenguaje | **C++17 (mínimo)** + QML/Qt Quick (`set(CMAKE_CXX_STANDARD 17)` vía `qt_standard_project_setup`) |
| Framework | **Qt 6.11.2 (objetivo) / Qt 6.8 LTS (mínimo)** — módulos: `qml`, `quick`, `sql`, `network`, `xml` |
| Build system | **CMake (Qt 6)** con `qt_standard_project_setup` + `qt_add_qml_module` (obligatorio en Qt 6.11; qmake deprecado desde Qt 6.7 y eliminado en Qt 7). El repo histórico usa QMake (`.pro`/`.pri`): se porta en la Fase 0 |
| Imports QML | Sintaxis Qt 6 sin versión: `import QtQuick`, `import QtQuick.Controls`, `import QtQuick.Window` |
| Persistencia | SQLite (`QSQLITE`), fichero `cm.sqlite` junto al ejecutable |
| Pruebas | Qt Test (`testlib`) con suite agregadora y mocks |
| Plataformas | Windows (msvc2019+/mingw), Linux (gcc/clang), macOS (clang) — x86_64/arm64, debug/release |

### Funcionalidad final
- Splash inicial + dashboard con feed RSS en vivo.
- Crear / editar / buscar / borrar clientes (nombre, fecha de nacimiento, estado, contactos, direcciones).
- Alta y edición de clientes mediante formularios validados enlazados a entidades C++.
- Búsqueda con criterios combinados sobre base de datos.
- Consumo de un canal RSS vía red con parseo XML.
- Paquete instalable por plataforma.

---

## 2. Arquitectura

### 2.1 Los tres proyectos

```
CMakeLists.txt (raíz: find_package(Qt6 6.8) + add_subdirectory)   # histórico: cm.pro subdirs
├── cm-lib    → librería compartida (qt_add_library SHARED; toda la lógica de negocio)
├── cm-ui     → ejecutable GUI (qt_add_executable + qt_add_qml_module); enlaza contra cm-lib
└── cm-tests  → ejecutable de consola con tests unitarios (Qt6::Test + CTest); enlaza contra cm-lib
```

Regla de oro: **el UI no contiene lógica de negocio**; sólo presenta y enlaza. Todo lo testeable vive en `cm-lib`.

### 2.2 Estructura interna de `cm-lib/source`

```
cm-lib_global.h   Macro de exportación CMLIB_EXPORT (windows dllimport/dllexport)
framework/        Command (patrón Command), ObjectFactory + IObjectFactory (fábrica/inversión de dependencias)
controllers/      MasterController (agregador/fachada raíz)
                  NavigationController (estado de navegación)
                  CommandController  (comandos Save/Edit/Delete/Back/Find...)
                  DatabaseController + IDatabaseController (persistencia)
data/             Entity (entidad auto-consciente) + DataDecorator
                  StringDecorator, IntDecorator, DateTimeDecorator, EnumeratorDecorator
                  Dropdown / DropdownValue, EntityCollection
models/           Client, Contact, Address, Appointment, ClientSearch
networking/       NetworkAccessManager + INetworkAccessManager, WebRequest + IWebRequest
rss/              RssChannel, RssItem, RssImage
utilities/        XmlHelper (lectura tolerante de nodos XML)
```

### 2.3 Estructura de `cm-ui`

```
source/main.cpp          Arranque: QQmlApplicationEngine + registro de tipos + contexto
views/                   MasterView (shell), Splash, Dashboard, CreateClient, EditClient, FindClient, Rss
components/              CommandBar/Button, NavigationBar/Button, Panel, StringEditorSingleLine,
                         AddressEditor, SearchResultDelegate, ContactsEditor/ContactDelegate,
                         EnumeratorSelector(View), FormButton, RssItemDelegate
assets/Style.qml         Singleton de estilo (colores, tamaños, fuentes)
views.qrc / components.qrc / assets.qrc   Recursos embebidos
```

### 2.4 Flujo de datos (MVVM ligero)

```
QML View  ⇄  property/signal  ⇄  Controller (cm-lib)  ⇄  Entity/Decorators  ⇄  SQLite
                                        │
                                        └── Command objects → CommandBar QML
```

- `MasterController` se expone al contexto QML como propiedad de contexto; los demás controllers cuelgan de él.
- Cada campo de una entidad es un **decorator** (`StringDecorator`, `IntDecorator`, …) expuesto a QML con
  `value`, `dirty`, señales de cambio y reglas de validación. Esto permite formularios genéricos y reutilizables.
- `Entity::write(QVariantMap)` / serialización JSON permiten que las entidades "se conozcan" a sí mismas y
  se traduzcan a filas SQL y viceversa.

### 2.5 Configuración de build (CMake sobre Qt 6)

**Qt 6.11 usa CMake como sistema de build oficial** (`qmake` está deprecado desde Qt 6.7 y desaparece en Qt 7).
Requisitos mínimos del `CMakeLists.txt` raíz:

```cmake
cmake_minimum_required(VERSION 3.21)
project(cm LANGUAGES CXX)

find_package(Qt6 6.8 REQUIRED COMPONENTS Qml Quick Sql Network Xml Test)  # objetivo: 6.11.2
qt_standard_project_setup(REQUIRES 6.8)   # CMAKE_CXX_STANDARD 17 + AUTOMOC/AUTORCC + políticas Qt
```

- `set(CMAKE_CXX_STANDARD 17)` / `qt_standard_project_setup()` garantizan **C++17 mínimo** en todos los targets
  (equivale a `CONFIG += c++17`; en MSVC añade `/std:c++17`).
- La versión mínima se declara con `find_package(Qt6 6.8 REQUIRED ...)`: si el kit es más antiguo, CMake falla
  con mensaje claro (equivale a la guarda `error()` que se usaba en qmake).
- Los tipos C++ se exponen a QML con la macro **`QML_ELEMENT`** (+ `QML_SINGLETON`, `QML_NAMED_ELEMENT`) y el
  comando **`qt_add_qml_module`**, que genera `qmldir` y compila los `.qml` sin necesidad de `.qrc` manual.

Estructura CMake equivalente al workspace histórico (sustituye a `cm.pro subdirs` + `.pri`):

```
cm/
├── CMakeLists.txt          # add_subdirectory(cm-lib cm-ui cm-tests), find_package, guardas
├── cm-lib/CMakeLists.txt   # qt_add_library(cm-lib SHARED) + QML elements
├── cm-ui/CMakeLists.txt    # qt_add_executable + qt_add_qml_module(TARGET cm-ui URI cm.ui ...)
└── cm-tests/CMakeLists.txt # qt_add_executable(testlib) + qt_add_qml_module de recursos de test
```

Salidas ordenadas (reemplazo de `DESTINATION_PATH` de los `.pri`):

```cmake
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/binaries/$<CONFIG>)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/binaries/$<CONFIG>)
```

Resultado: artefactos limpios por configuración en `binaries/<Debug|Release>/`; CMake detecta SO, compilador
y arquitectura automáticamente (ya no hacen falta los `.pri` de detección de plataforma).

---

## 3. Plan de construcción por fases

### Fase 0 — Entorno, Qt 6, CMake y migración desde Qt 5 (Capítulo 1)
**Objetivo**: toolchain Qt 6.8+ con CMake funcionando, código actualizado a QML 6 y C++17, primer contacto con QML.
1. Instalar **Qt 6.11.2** (o, como mínimo, **Qt 6.8 LTS**) vía Maintenance Tool: componentes *Qt 6 → Quick/QML*,
   *Sql (SQLite driver)*, *Network*, *Xml* + kit de compilador (MSVC 2019+/gcc/clang) + Qt Creator.
   Verificar con `cmake --version` (≥ 3.21) y `qt-cmake --version` del kit instalado.
2. **Portar el build de qmake a CMake** (obligatorio en Qt 6.11): crear los `CMakeLists.txt` descritos en §2.5;
   `qt_standard_project_setup(REQUIRES 6.8)` fija **C++17 mínimo** en todos los targets (sustituye a
   `CONFIG += c++14/c++17` de los `.pro`).
3. **Migrar las importaciones QML** (los capítulos usan `QtQuick 2.9` / `Controls 2.2` / `Window 2.2|2.3`):
   en Qt 6 se escriben **sin número de versión** — `import QtQuick`, `import QtQuick.Controls`,
   `import QtQuick.Window`, `import QtQuick.Layouts`. Un `sed` basta:
   `grep -rl "import QtQuick" views components | xargs sed -i -E 's/(import Qt[A-Za-z.]+) [0-9]+\.[0-9]+/\1/'`.
4. Ajustes de API Qt 6 relevantes para este proyecto:
   - `QRegExp` → `QRegularExpression` (no se usa aquí, pero vigilar validaciones futuras).
   - `QString::sprintf`/`toAscii` eliminados; señales/slots con sintaxis de punteros ya presente.
   - **Registro de tipos**: sustituir `qmlRegisterType`/`qmlRegisterSingletonType` manuales por macros
     `QML_ELEMENT` / `QML_SINGLETON` + `qt_add_qml_module` (con Qt ≥ 6.9 `qmlRegisterType` está deprecado).
   - Los `.qrc` de vistas/componentes pasan a ser `QML_FILES`/`RESOURCES` dentro de `qt_add_qml_module`.
5. Compilar el *scratchpad*: `main.cpp` (QGuiApplication + QQmlApplicationEngine) + `main.qml` (sin versión
   en el import), construido con `qt-cmake` (ejemplo completo en el tutorial).
6. Practicar layouts QML: `AnchorsDemo.qml`, `SizingDemo.qml`.

✅ Salida: ventana QML renderizada con Qt 6.8+; `cmake` configura sin avisos y `Version of Qt = 6.11.2` (o ≥ 6.8)
en el log de configure; ningún `.qml` con import versionado; todos los targets en C++17 vía CMake.

### Fase 1 — Esqueleto del workspace (Capítulo 2)
**Objetivo**: los tres proyectos compilan y se enlazan entre sí (sobre Qt 6.8+ / C++17 / CMake).
1. Crear el `CMakeLists.txt` raíz (`project(cm)`, `find_package(Qt6 6.8 REQUIRED ...)`,
   `qt_standard_project_setup(REQUIRES 6.8)`, `add_subdirectory(cm-lib cm-ui cm-tests)`) — reemplaza a `cm.pro subdirs`.
2. No hacen falta los `.pri` de plataforma: CMake detecta SO/compilador/arquitectura; las salidas se agrupan
   con `CMAKE_*_OUTPUT_DIRECTORY` (§2.5).
3. `cm-lib`: `qt_add_library(cm-lib SHARED ...)` con `DEFINES CMLIB_LIBRARY` + macro de exportación,
   `target_link_libraries(cm-lib PUBLIC Qt6::Qml Qt6::Quick)`, primer `Client` y `MasterController` marcados
   con `QML_ELEMENT`.
4. `cm-ui`: `qt_add_executable` + `qt_add_qml_module(TARGET cm-ui URI cm.ui ...)` que empaqueta `MasterView.qml`;
   `main.cpp` carga `qrc:/qt/qml/cm/ui/MasterView.qml` (ruta canónica del módulo) y expone el controller.
5. `cm-tests`: ejecutable con `Qt6::Test`; primer test unitario con `QtTest` registrado en CTest
   (`enable_testing()` + `add_test`).

✅ Salida: `cm-lib.(dll|a|dylib)`, `cm-ui(.exe)` y `client-tests` generados en `binaries/<Config>/`; tests en verde vía `ctest`.

### Fase 2 — Navegación y estructura de vistas (Capítulo 3)
**Objetivo**: shell navegable de la aplicación.
1. `NavigationController` con propiedad `page` y señal de cambio.
2. Vistas: `SplashView` (arranque), `DashboardView`, `CreateClientView`, `EditClientView`, `FindClientView`.
3. `MasterView` como contenedor que intercambia páginas según la navegación.
4. Añadir todas las vistas a `QML_FILES` del `qt_add_qml_module` de `cm-ui` (sustituye a `views.qrc`).

✅ Salida: se puede navegar entre las 4 pantallas desde la splash.

### Fase 3 — Framework de comandos y componentes (Capítulo 4)
**Objetivo**: acciones tipadas y reutilizables (patrón Command) + barra de herramientas.
1. `framework/command.{h,cpp}`: QObject con nombre, icono, enabled/visible y slot `execute()` (callback).
2. `CommandController`: define comandos (Save, Cancel, Back, Find…) y expone listas a QML.
3. Componentes: `CommandBar`, `CommandButton`, `NavigationBar`, `NavigationButton`.
4. `assets/Style.qml` singleton declarado con `QML_SINGLETON` + `QML_ELEMENT` y registrado como
   `QT_QML_GENERATE_QMLLS_INI`-friendly resource dentro del módulo (`qt_add_qml_module(... QML_FILES assets/Style.qml)`):
   tema visual centralizado sin `.qrc` manual.

✅ Salida: barras de navegación/comandos funcionales y estilizadas; sin lógica en QML.

### Fase 4 — Núcleo de datos: entidades auto-conscientes (Capítulo 5)
**Objetivo**: modelo de dominio con validación, binding y serialización JSON.
1. `DataDecorator` base + `StringDecorator`, `IntDecorator`, `DateTimeDecorator`, `EnumeratorDecorator`.
2. `Entity`: colección de decoradores, `read/write`, `toJson()`, dirty tracking; `EntityCollection`.
3. Modelos completos: `Client` (sobre `Entity`), `Address`, `Contact`, `Appointment`.
4. Componente de formulario: `StringEditorSingleLine.qml` vinculado a decorators.

✅ Salida: formularios leen/escriben entidades C++ con validación; round-trip JSON correcto.

### Fase 5 — Tests unitarios sistemáticos (Capítulo 6)
**Objetivo**: seguridad de refactorización antes de tocar persistencia/red.
1. `test-suite.{h,cpp}`: agregador que ejecuta varias suites en un solo binario (XML de resultados).
2. Suites por decorator (`string/int/datetime/enumerator-decorator-tests`), `client-tests`, `master-controller-tests`.
3. Macros de mocking ligeras (`mocking.h` → luego `mocks/mock-object-factory.h`).

✅ Salida: `client-tests` pasa todas las suites en cada plataforma.

### Fase 6 — Persistencia SQLite (Capítulo 7)
**Objetivo**: CRUD real de clientes.
1. `IDatabaseController` (interfaz para mockear) + `DatabaseController`: abre `cm.sqlite`, crea tablas,
   `insert/update/delete/find` mapeando entidades ⇄ filas.
2. `ClientSearch`: objeto de criterios → `WHERE` parametrizado; lista de resultados.
3. Integración en `MasterController` (save/find/delete conectados a comandos de la UI).
4. Vistas: `FindClientView` con `ListView` + `SearchResultDelegate`, `AddressEditor`, `Panel`.

✅ Salida: crear cliente → cerrar app → buscarlo y editarlo; datos persistentes.

### Fase 7 — Red y RSS (Capítulo 8)
**Objetivo**: consumir un feed externo en el dashboard.
1. Capa `networking/`: `INetworkAccessManager`/`NetworkAccessManager` (wrapper de `QNetworkAccessManager`)
   e `IWebRequest`/`WebRequest` (petición con URL, señal `finished`, cuerpo).
2. `utilities/xml-helper`: lectura defensiva de elementos/atributos XML.
3. Modelos `rss/`: `RssChannel`, `RssItem`, `RssImage` parseados desde XML.
4. UI: `RssView.qml` + `RssItemDelegate.qml` en el dashboard.

✅ Salida: listado RSS en vivo en la pantalla principal; tests siguen verdes (interfaces mockeables).

### Fase 8 — Refactor de inyección de dependencias + empaquetado (Capítulo 9)
**Objetivo**: cierre arquitectónico y distribución.
1. Extraer interfaces `ICommandController`, `INavigationController`, `IObjectFactory`;
   implementar `ObjectFactory` (creación centralizada de controladores/entidades).
2. Sustituir construcciones directas por la fábrica → testable con `MockObjectFactory`.
3. Controles avanzados: `Dropdown`/`DropdownValue`, `EnumeratorSelector(View)`, `FormButton`,
   `ContactsEditor`/`ContactDelegate`.
4. Instalador: reglas `install(TARGETS ...)` de CMake + stage de binarios + recursos + plugins Qt
   (`sqldrivers`, `platforms`) por plataforma; generar con `windeployqt --qmldir cm-ui` (Windows),
   `macdeployqt cm-ui.app` (macOS) o `linuxdeploy`/AppImage (Linux). Con CMake también sirve
   `cmake --install build --prefix dist`.
   Ejemplo incluido en el repo: `installer/osx/cm-ui.app` (bundle macOS, generado con Qt 5 → regenerar con `macdeployqt`).

✅ Salida: aplicación instalable en las tres plataformas con toda la funcionalidad.

---

## 4. Cómo compilar y ejecutar (referencia rápida)

```bash
cd Chapter09/chapter9/cm      # o el capítulo que te interese (ya portado a CMake)
# Usar el qt-cmake del kit Qt 6.11.2 (o 6.8 LTS): añade -DCMAKE_PREFIX_PATH automáticamente
~/Qt/6.11.2/gcc_64/bin/qt-cmake . -B build -G Ninja       # configura (falla si Qt < 6.8)
cmake --build build                                        # compila cm-lib → cm-ui/cm-tests (C++17)
./binaries/Release/cm-ui                                   # lanzar la app
ctest --test-dir build --output-on-failure                 # ejecutar tests
```

Con Qt Creator: abrir `CMakeLists.txt` y seleccionar un **kit Qt 6.8+** (6.11.2 recomendado).
Referencia oficial: [CMake Manual for Qt 6](https://doc.qt.io/qt-6/cmake-manual.html) ·
[qt_add_qml_module](https://doc.qt.io/qt-6/qt-add-qml-module.html).

---

## 5. Decisiones arquitectónicas clave (resumen)

1. **Librería + UI + tests separados**: obliga a que la lógica sea independiente de la GUI y testeable.
2. **Patrón Command**: las acciones son objetos expuestos a QML; habilitar/deshabilitar y reordenar es trivial.
3. **Decorators sobre miembros de entidad**: validación, suciedad y binding granular sin ORM.
4. **Interfaces (`i-*.h`) + ObjectFactory**: inversión de dependencias para mocks y evolución.
5. **Build único multiplataforma con CMake**: un mismo árbol fuente produce salidas por configuración
   (`binaries/<Config>`); CMake/Qt abstraen SO, compilador y arquitectura (los `.pri` de qmake quedan obsoletos).
6. **Recursos `.qrc`**: QML, componentes y assets embebidos → despliegue de un único ejecutable + librería.
7. **Versionado moderno**: Qt 6.11.2 como objetivo (Qt 6.8 LTS como mínimo), **CMake como sistema de build**
   (qmake deprecado desde Qt 6.7), C++17 mínimo vía `qt_standard_project_setup(REQUIRES 6.8)`, imports QML sin
   versión, tipos expuestos con `QML_ELEMENT`/`qt_add_qml_module` y guarda de versión en
   `find_package(Qt6 6.8 REQUIRED ...)` que rechaza toolchains antiguos.
