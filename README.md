# CrediYa - Sistema de Gestión de Préstamos y Pagos

**CrediYa** es una aplicación de consola desarrollada en **Java** diseñada para administrar de forma integral el ciclo de vida de microcréditos y préstamos personales. Permite registrar clientes y empleados, emitir préstamos con cálculo de cuotas, controlar el abono de pagos, gestionar errores de negocio y exportar reportes de movimientos[cite: 1].

---

## ¿Qué hace el sistema?

* **Gestión de Personas (Clientes y Empleados):** Registra y consulta la información de los clientes que solicitan créditos y de los empleados encargados de aprobarlos o gestionarlos.
* **Administración de Préstamos:** Crea nuevos préstamos asociados a un cliente y un empleado, calculando cuotas mensuales, tasas de interés, plazos y manteniendo actualizado el estado del crédito (ej. *Activo*, *Pagado*, *Vencido*).
* **Control de Pagos:** Registra abonos parciales o totales a un préstamo específico y descuenta automáticamente el valor del saldo pendiente.
* **Validación y Reglas de Negocio:** Evita inconsistencias (como montos negativos, clientes inexistentes o pagos superiores a la deuda) mediante excepciones personalizadas (`CrediYaException`).
* **Persistencia Dual:** Guarda la información de forma permanente en una base de datos relacional (`database.sql` / JDBC) y permite leer o exportar reportes en archivos locales (`ArchivoManager`).

---

## ¿Cómo funciona por dentro? (Arquitectura)

El proyecto sigue una **arquitectura por capas** para separar responsabilidades y facilitar su mantenimiento:

```text
CrediYa/
├── database.sql                        # Script SQL para crear tablas y estructura inicial
└── src/
    ├── vista/
    │   └── Main.java                   # Interfaz de usuario en consola y menú principal
    ├── servicio/
    │   └── CrediYaService.java         # Lógica de negocio, cálculos y validaciones
    ├── modelo/
    │   ├── Persona.java                # Clase base con datos comunes (id, nombre, documento, teléfono)
    │   ├── Cliente.java                # Hereda de Persona; añade datos del solicitante
    │   ├── Empleado.java               # Hereda de Persona; añade datos del gestor/asesor
    │   ├── Prestamo.java               # Representa el crédito (monto, tasa, plazo, estado)
    │   └── Pago.java                   # Representa cada abono realizado a un préstamo
    ├── persistencia/
    │   ├── dbConnection.java           # Conexión JDBC con la base de datos
    │   └── ArchivoManager.java         # Lectura y escritura de archivos/reportes
    └── excepciones/
        └── CrediYaException.java       # Manejo de errores propios del sistema
