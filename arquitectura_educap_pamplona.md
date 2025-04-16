# Plan de Arquitectura Refinado - Educap.Pamplona

## 1. Visión General

*   **Tipo:** Aplicación de escritorio local para Windows.
*   **Propósito:** Gestión integral del centro de asesorías Educap.Pamplona.
*   **Funcionalidades Clave:**
    *   Gestión de Estudiantes (regulares, ocasionales).
    *   Gestión de Pagos (matrículas, mensualidades, sesiones ocasionales, servicios adicionales).
    *   Gestión de Gastos operativos.
    *   Generación de Documentos (recibos, facturas, certificados con numeración consecutiva).
    *   Reportes básicos (financieros, pagos pendientes, historial).
*   **Almacenamiento:** Base de datos local y persistente (SQLite).
*   **Interfaz:** Gráfica de usuario (GUI) intuitiva y moderna.
*   **Extras:** Funcionalidad de backup y restauración de la base de datos.

## 2. Arquitectura Propuesta (Capas)

Se propone una arquitectura de capas estándar para aplicaciones de escritorio, utilizando un ORM para facilitar el acceso a datos.

```mermaid
graph TD
    A[Interfaz de Usuario (UI)<br>(PyQt/PySide: Ventanas, Botones, Tablas)] --> B(Lógica de Negocio<br>(Cálculos, Validaciones, Reglas,<br>Numeración Documentos, Backups));
    B --> C(Acceso a Datos<br>(ORM: SQLAlchemy));
    C --> D(Base de Datos<br>(SQLite: Archivo .db local));

    subgraph "Aplicación Educap.Pamplona"
        A
        B
        C
    end

    subgraph "Almacenamiento Local"
        D
    end
```

*   **Interfaz de Usuario (UI):** Responsable de la presentación y la interacción con el usuario. Se usará PyQt o PySide.
*   **Lógica de Negocio:** Contiene las reglas, cálculos, validaciones y procesos centrales de la aplicación (ej: cálculo de saldos, generación de números de recibo, lógica de backup).
*   **Acceso a Datos:** Capa que interactúa con la base de datos, abstraída mediante un ORM (SQLAlchemy) para simplificar las operaciones CRUD.
*   **Base de Datos:** Almacén persistente de los datos (archivo SQLite).

## 3. Modelo de Datos (SQLite - Refinado)

Modelo Entidad-Relación detallado, incorporando tablas para tarifas y secuencias de numeración.

```mermaid
erDiagram
    ESTUDIANTES ||--o{ PAGOS_MENSUALES : "tiene"
    ESTUDIANTES ||--o{ SESIONES_OCASIONALES : "tiene"
    ESTUDIANTES ||--o{ SERVICIOS_ADICIONALES : "recibe"
    ESTUDIANTES ||--o{ RECIBOS : "puede generar (Certificado)"
    ESTUDIANTES }|--|| TARIFAS : "tiene asignada"

    PAGOS_MENSUALES ||--o{ RECIBOS : "genera"
    SESIONES_OCASIONALES ||--o{ RECIBOS : "genera"
    SERVICIOS_ADICIONALES ||--o{ RECIBOS : "genera"

    TARIFAS {
        INT id_tarifa PK
        VARCHAR nombre_tarifa
        TEXT descripcion NULL
        DECIMAL monto
        VARCHAR tipo "('Mensual', 'PorHora', 'Otro')"
    }

    ESTUDIANTES {
        INT id_estudiante PK
        VARCHAR nombres
        VARCHAR apellidos
        VARCHAR documento_identidad NULL
        VARCHAR telefono NULL
        VARCHAR email NULL
        DATE fecha_inscripcion
        DATE fecha_retiro NULL
        VARCHAR modalidad "('Presencial', 'Virtual')"
        VARCHAR tipo "('Mensual', 'Ocasional')"
        VARCHAR estado "('Activo', 'Inactivo', 'Retirado')"
        INT id_tarifa FK "(Refiere a TARIFAS)"
        TEXT notas_adicionales NULL
    }

    PAGOS_MENSUALES {
        INT id_pago_mensual PK
        INT id_estudiante FK
        INT anio
        INT mes
        DECIMAL monto_esperado
        DECIMAL monto_pagado DEFAULT 0
        DATE fecha_pago NULL
        DECIMAL saldo_pendiente
        VARCHAR estado_pago "('Pagado', 'Pendiente', 'Vencido', 'Abonado')"
        TEXT notas NULL
    }

    SESIONES_OCASIONALES {
        INT id_sesion PK
        INT id_estudiante FK
        DATE fecha_sesion
        DECIMAL duracion_horas
        DECIMAL tarifa_aplicada
        DECIMAL monto_cobrado
        DATE fecha_pago NULL
        VARCHAR estado_pago "('Pagado', 'Pendiente')"
    }

    SERVICIOS_ADICIONALES {
        INT id_servicio PK
        INT id_estudiante FK NULL
        VARCHAR descripcion_servicio
        DATE fecha_servicio
        DECIMAL monto_acordado
        DECIMAL monto_pagado DEFAULT 0
        DATE fecha_pago NULL
        VARCHAR estado_pago "('Pagado', 'Pendiente', 'Abonado')"
    }

    GASTOS {
        INT id_gasto PK
        DATE fecha_gasto
        VARCHAR categoria "('Arriendo', 'Materiales', 'Servicios Públicos', 'Salarios', 'Otros')"
        VARCHAR descripcion
        DECIMAL monto
    }

    RECIBOS {
        INT id_recibo PK
        VARCHAR numero_consecutivo UK "Gestionado por SECUENCIAS_DOCUMENTOS"
        VARCHAR tipo_documento "('Recibo Pago Mensual', 'Recibo Sesion', 'Recibo Servicio', 'Factura Venta', 'Recibo Devolución', 'Certificado')"
        DATE fecha_generacion
        INT id_pago_mensual FK NULL "(Refiere a PAGOS_MENSUALES)"
        INT id_sesion FK NULL "(Refiere a SESIONES_OCASIONALES)"
        INT id_servicio FK NULL "(Refiere a SERVICIOS_ADICIONALES)"
        INT id_estudiante FK NULL "(Refiere a ESTUDIANTES, para Certificados)"
        DECIMAL monto NULL "(Para recibos/facturas)"
        TEXT datos_adicionales NULL "(Ej: texto certificado)"
    }

    SECUENCIAS_DOCUMENTOS {
        VARCHAR tipo_documento PK "('Recibo Pago Mensual', 'Recibo Sesion', ...)"
        VARCHAR prefijo NULL "(Ej: 'F-', 'R-', 'C-')"
        INT ultimo_numero_usado DEFAULT 0
    }
```

## 4. Módulos Principales

*   **Dashboard:** Vista inicial con KPIs (Estudiantes activos, Pagos pendientes mes actual, Ingresos vs Gastos mes actual).
*   **Gestión de Estudiantes:** CRUD completo para estudiantes, incluyendo asignación de tarifas.
*   **Gestión de Pagos:**
    *   Registro y seguimiento de Pagos Mensuales.
    *   Registro y seguimiento de Sesiones Ocasionales.
    *   Registro y seguimiento de Servicios Adicionales.
*   **Gestión de Gastos:** CRUD para gastos operativos.
*   **Generación de Documentos:**
    *   Creación de Recibos/Facturas (basado en pagos, con numeración automática desde `SECUENCIAS_DOCUMENTOS`).
    *   Generación de Certificados (plantillas personalizables).
    *   Historial y opción de reimpresión/exportación a PDF.
*   **Reportes:**
    *   Informe Financiero (Ingresos vs Gastos por período).
    *   Listado de Estudiantes con Pagos Pendientes.
    *   Historial detallado de Pagos por Estudiante.
*   **Configuración:**
    *   Gestión de Tarifas (CRUD).
    *   Gestión de Categorías de Gastos.
    *   Gestión de Secuencias de Documentos (prefijos, próximo número).
    *   Configuración de plantillas básicas (Certificados).
*   **Utilidades:**
    *   Backup de Base de Datos (Crear copia de seguridad del archivo `.db`).
    *   Restauración de Base de Datos (Restaurar desde una copia).

## 5. Tecnologías Sugeridas

*   **Lenguaje de Programación:** Python 3.x
*   **Base de Datos:** SQLite 3
*   **Interfaz Gráfica (GUI):** PyQt 6 o PySide 6
*   **Acceso a Datos (ORM):** SQLAlchemy
*   **Generación de PDF:** ReportLab
*   **Empaquetado (Creación de `.exe`):** PyInstaller