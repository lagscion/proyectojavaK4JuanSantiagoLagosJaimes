# CrediYa - Sistema de Gestión de Préstamos y Pagos

**CrediYa** es una aplicación de consola desarrollada en **Java 17** con base de datos **MySQL** para administrar el ciclo de vida de préstamos personales: registro de clientes y empleados, emisión de préstamos con cálculo de cuotas, control de abonos y reportes de cartera.

---

## ¿Qué hace el sistema?

* **Autenticación y permisos:** el acceso es con nombre de usuario y una contraseña numérica de hasta 5 dígitos. El sistema busca primero en `clientes` y luego en `empleados`, solo entre los registros activos. Cada usuario tiene un indicador `permiso_modificacion` que habilita las opciones de edición del menú.
* **Gestión de personas:** registra, lista, actualiza y desactiva clientes y empleados. La desactivación es **lógica** (`activo = FALSE`); no se borran filas.
* **Administración de préstamos:** crea préstamos asociados a un cliente y a un empleado. Cada préstamo guarda el capital, la tasa mensual (en %), el número de cuotas y la fecha de inicio. La cuota, el total, los pagos acumulados y el saldo **no se almacenan**: se calculan a partir de esos datos y de la tabla `pagos`.
* **Control de pagos:** registra abonos parciales o totales en la tabla `pagos`. Los abonos **no modifican** `prestamos.monto`; el saldo es el total calculado menos la suma de los pagos. Cuando el saldo llega a cero, el préstamo pasa a `PAGADO`. El registro se hace en una transacción que bloquea la fila del préstamo para evitar abonos simultáneos inconsistentes.
* **Reportes de cartera:** préstamos pendientes, préstamos vencidos y clientes morosos, filtrados con **Lambda y Stream API**.
* **Validación de reglas de negocio:** se rechazan montos o tasas negativas, cuotas menores o iguales a cero, abonos no positivos, abonos mayores al saldo y préstamos inexistentes o ya pagados. Los errores se informan con `IllegalArgumentException` (datos inválidos) y `SQLException` (base de datos); el menú los captura y muestra un mensaje.
* **Persistencia:** toda la información se guarda en MySQL mediante JDBC. Las clases `ArchivoUtil` y `ArchivoRepository` permiten leer y escribir archivos de texto, pero **no están conectadas al menú**, por lo que hoy el sistema no exporta reportes a archivo.

---

## Cálculo financiero

Se usa el **sistema francés de amortización** (cuota fija), centralizado en `CalculoPrestamo`:

* `prestamos.monto`: capital original.
* `prestamos.interes`: tasa **mensual** en porcentaje (`2.00` = 2 % mensual). Para calcular se convierte a fracción (0.02).
* Cuota mensual: `Q = C · i / (1 − (1 + i)^(−n))`, redondeada a 2 decimales. Si la tasa es 0, `Q = C / n`.
* Total a pagar: `Q × n`.
* Saldo pendiente: `total − suma de pagos`.

## Estados de un préstamo

| Estado | Significado |
|---|---|
| `PENDIENTE` | Todavía tiene saldo por pagar. |
| `PAGADO` | Los pagos registrados cubrieron el total. |

El **vencimiento no es un estado almacenado**: se calcula como `fecha_inicio + cuotas meses` y se compara con la fecha actual mientras el préstamo siga `PENDIENTE`. Un cliente es **moroso** si tiene al menos un préstamo pendiente vencido.

---

## Menú principal

El menú depende del tipo de usuario y de su permiso de modificación.

| Usuario | Siempre disponible | Con `permiso_modificacion` |
|---|---|---|
| **Empleado** | Registrar empleado, consultar empleados, consultar préstamos relacionados, cerrar sesión | Actualizar y desactivar empleado, crear préstamo, registrar abono, histórico de pagos, reportes de cartera |
| **Cliente** | Registrar cliente, listar clientes, consultar mis préstamos, cerrar sesión | Actualizar y desactivar cliente |

---

## Arquitectura

El proyecto sigue una **arquitectura por capas**:

```text
Main  ->  Servicio  ->  DAO  ->  ConexionBD  ->  MySQL
```

```text
Proyectok4JuanlLgos/
├── pom.xml                                   # Maven: JDK 17 y driver mysql-connector-j 9.3.0
├── README.md
├── README_PROBLEMA_2.md                      # Notas del cálculo financiero unificado
├── README_PROBLEMA_3.md                      # Notas de la integración DAO / menú
├── classes/                                  # Salida compilada (.class); se regenera con Maven
└── src/main/
    ├── resources/
    │   ├── schema.sql                        # Crea la base crediya_db y sus 4 tablas
    │   ├── migracion_estados_prestamos.sql   # Normaliza estados de una base existente
    │   └── db.properties.example             # Plantilla de configuración de conexión
    └── java/
        ├── main/
        │   └── Main.java                     # Interfaz de consola y menú (solo llama servicios)
        └── com/mycompany/proyectok4juanllgos/
            ├── servicio/                     # Reglas de aplicación; no contienen SQL
            │   ├── AutenticacionServicio.java
            │   ├── ClienteServicio.java
            │   ├── EmpleadoServicio.java
            │   ├── PrestamoServicio.java
            │   ├── PagoServicio.java
            │   └── ReporteServicio.java      # Reportes con Lambda y Stream API
            ├── persistencia/                 # Acceso a datos (JDBC)
            │   ├── ConexionBD.java           # Única fuente de conexiones a MySQL
            │   ├── AutenticacionDAO.java
            │   ├── ClienteDAO.java
            │   ├── EmpleadoDAO.java
            │   ├── PrestamoDAOJDBC.java      # Consulta y crea préstamos (DAO activo)
            │   ├── PagoDAO.java              # Registra abonos y consulta el histórico
            │   ├── ArchivoUtil.java          # Lectura/escritura de archivos (sin uso en el menú)
            │   └── ArchivoRepository.java    # Idem
            └── modelo/
                ├── Persona.java              # Clase abstracta: id, nombre, teléfono, mostrarRol()
                ├── Cliente.java              # Hereda de Persona; añade documento y correo
                ├── Empleado.java             # Hereda de Persona; añade documento, rol, correo y salario
                ├── Prestamo.java             # Crédito: capital, tasa, cuotas, fechas, estado (enum PENDIENTE/PAGADO)
                └── CalculoPrestamo.java      # Fórmula del sistema francés (única implementación)
```

### Clases heredadas

Se conservan para no borrar código original, pero el menú **ya no las invoca**:

* `modelo/clientes.java`, `modelo/empleados.java`, `modelo/prestamos.java`: lógica antigua con consola y SQL mezclados.
* `controlador/ingreso.java`: autenticación antigua, reemplazada por `AutenticacionServicio` y `AutenticacionDAO`.
* `persistencia/PrestamoDAO.java`: DAO antiguo, reemplazado por `PrestamoDAOJDBC`.
* `persistencia/dbConnection.java`: adaptador que delega en `ConexionBD`.
* `persistencia/PagoDAO.java.tmp`: archivo temporal sin uso.

---

## Base de datos

`src/main/resources/schema.sql` crea la base `crediya_db` con estas tablas:

| Tabla | Campos principales |
|---|---|
| `empleados` | `id`, `nombre`, `documento`, `rol`, `correo`, `contrasena`, `salario`, `permiso_modificacion`, `activo` |
| `clientes` | `id`, `nombre`, `documento`, `correo`, `telefono`, `contrasena`, `permiso_modificacion`, `activo` |
| `prestamos` | `id`, `cliente_id`, `empleado_id`, `monto`, `interes`, `cuotas`, `fecha_inicio`, `estado` |
| `pagos` | `id`, `prestamo_id`, `fecha_pago`, `monto` |

`prestamos` referencia a `clientes` y `empleados`; `pagos` referencia a `prestamos`.

### Migrar una base existente

Si ya tienes datos con los estados antiguos, ejecuta **una sola vez** `src/main/resources/migracion_estados_prestamos.sql` en MySQL Workbench. Convierte `Activo`, `Mora` y `Pendiente` en `PENDIENTE`, y `Pagado` en `PAGADO`. Revisa el resultado de los `SELECT` incluidos antes de continuar. El script no modifica columnas ni relaciones.

---

## Configuración y ejecución

Requiere **JDK 17**, **Maven** y un servidor **MySQL**.

1. Crea la base de datos ejecutando `src/main/resources/schema.sql`.
2. Copia `src/main/resources/db.properties.example` como `src/main/resources/db.properties` y ajusta la URL, el usuario y la clave. También puedes usar las variables de entorno `DB_URL`, `DB_USER` y `DB_PASSWORD`.
3. Compila y ejecuta:

```bash
mvn clean package
mvn exec:java
```

La clase principal es `main.Main` (propiedad `exec.mainClass` del `pom.xml`).

> **Nota:** esta versión del `pom.xml` no genera un jar ejecutable con dependencias, por lo que `java -jar` no funciona; usa `mvn exec:java` o ejecuta desde tu IDE.
