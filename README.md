# Talleres de Laboratorio: Oracle Autonomous Database + AI Vector Search + Select AI

**Formato:** guía académica y técnica (paso a paso)  
**Audiencia:** estudiantes, docentes e investigadores que deseen implementar búsqueda semántica (vectores) y consultas en lenguaje natural sobre datos relacionales.  
**Plataforma objetivo:** **Oracle Autonomous Database (ADB)** con **Database Actions** y soporte de **AI Vector Search** y **Select AI**.

---

## Tabla de contenido

1. [Visión general](#visión-general)  
2. [Tecnologías utilizadas](#tecnologías-utilizadas)  
3. [Taller 1: AI Vector Search (Estudiantes ↔ Vacantes)](#taller-1-ai-vector-search-estudiantes--vacantes)  
   1. [Objetivos de aprendizaje](#objetivos-de-aprendizaje)  
   2. [Prerequisitos](#prerequisitos)  
   3. [Arquitectura de datos](#arquitectura-de-datos)  
   4. [Paso a paso](#paso-a-paso)  
4. [Taller 2: Chat con tus datos usando Select AI](#taller-2-chat-con-tus-datos-usando-select-ai)  
   1. [Objetivos de aprendizaje](#objetivos-de-aprendizaje-1)  
   2. [Prerequisitos](#prerequisitos-1)  
   3. [Paso a paso](#paso-a-paso-1)  
5. [Anexos](#anexos)  
   1. [Checklist de verificación](#checklist-de-verificación)  
   2. [Solución de problemas](#solución-de-problemas)  
   3. [Limpieza opcional](#limpieza-opcional)  
6. [Referencias](#referencias)

---

## Visión general

Este repositorio contiene **dos talleres prácticos** inspirados en los laboratorios oficiales de Oracle LiveLabs:

- **Taller 1 (AI Vector Search):** construcción de un modelo relacional para prácticas empresariales (estudiantes, empresas, vacantes), incorporación de una **columna vectorial** y generación de **embeddings** con un **modelo ONNX importado**. Se implementa:
  - Búsqueda semántica (consulta → Top-K resultados).
  - **Match automático** (vacante → ranking de estudiantes) mediante distancia vectorial.

- **Taller 2 (Select AI):** integración de un proveedor GenAI con Autonomous Database y habilitación de **preguntas en lenguaje natural** que se traducen a SQL sobre el mismo modelo de datos (3 tablas principales).

> Importante: estos talleres **evitan construir una app APEX automatizada**. Todo se ejecuta con **SQL / PL/SQL** en **Database Actions (SQL Worksheet)**, siguiendo el enfoque técnico de los labs.

---

## Tecnologías utilizadas

### 1) Oracle AI Vector Search (vectores en base de datos)
- Tipo de dato `VECTOR` para almacenar embeddings.
- Función `VECTOR_EMBEDDING(<modelo> USING <texto> AS data)` para generar embeddings.
- Función `VECTOR_DISTANCE(v1, v2, COSINE)` para medir similitud (menor distancia ⇒ mayor similitud).
- Índices vectoriales `CREATE VECTOR INDEX` para acelerar búsquedas aproximadas (ej. HNSW).

### 2) Importación de modelos ONNX (embeddings in-database)
- `DBMS_CLOUD.GET_OBJECT` para descargar ONNX desde Object Storage a `DATA_PUMP_DIR`.
- `DBMS_VECTOR.LOAD_ONNX_MODEL` para registrar el ONNX como modelo de embeddings utilizable por `VECTOR_EMBEDDING`.

### 3) Select AI (DBMS_CLOUD_AI)
- `DBMS_CLOUD_AI.CREATE_PROFILE` para definir un perfil de IA (proveedor + credenciales + lista de objetos).
- `DBMS_CLOUD_AI.SET_PROFILE` para activarlo en sesión.
- `DBMS_CLOUD_AI.GENERATE` para acciones:
  - `showsql`, `runsql`, `explainsql`, `narrate`, `chat`.

---

# Taller 1: AI Vector Search (Estudiantes ↔ Vacantes)

## Objetivos de aprendizaje

Al finalizar, podrás:

1. Crear un **usuario de laboratorio** y otorgar permisos mínimos para operar.
2. Importar modelos ONNX (vía Object Storage) y cargar modelos en la base.
3. Modelar tres entidades:
   - `ESTUDIANTE`
   - `EMPRESA`
   - `VACANTES_EMPRESAS`
   y una tabla de asociación `ESTUDIANTE_VACANTE`.
4. Crear columnas tipo `VECTOR` y generar embeddings para texto.
5. Implementar:
   - búsqueda semántica sobre estudiantes/vacantes,
   - matching automático vacante ↔ estudiantes.

## Prerequisitos

- Acceso a **Oracle Cloud** y una **Autonomous Database** provisionada.
- Acceso a **Database Actions** (SQL Worksheet).
- Permisos para ejecutar paquetes `DBMS_CLOUD` y `DBMS_VECTOR` (en LiveLabs se encuentran habilitados).

## Arquitectura de datos

### Modelo relacional

- **EMPRESA**: catálogo de empresas.
- **VACANTES_EMPRESAS**: vacantes publicadas por empresas; incluye `descripcion_perfil` y `perfil_vec`.
- **ESTUDIANTE**: perfiles de estudiantes; incluye `perfil_profesional` y `perfil_vec`.
- **ESTUDIANTE_VACANTE**: postulaciones/asociaciones (N:M).

### Esquema lógico (texto)

- `EMPRESA (nit_empresa PK)`
  - 1 ──── * `VACANTES_EMPRESAS (id_vacante PK, nit_empresa FK)`
- `ESTUDIANTE (id_estudiante PK)`
  - * ──── * `VACANTES_EMPRESAS` vía `ESTUDIANTE_VACANTE (id_estudiante FK, id_vacante FK)`

---

## Paso a paso

> **Recomendación:** ejecuta los bloques en orden.  
> **Entorno:** SQL Worksheet en Database Actions.

---

## 0) Preparación de usuario y permisos (opcional si ya tienes esquema en LiveLabs)

### 0.1 Conéctate como ADMIN (o usuario con privilegios equivalentes)

```sql
CREATE USER PRACTICAS IDENTIFIED BY "Cambiar#Pwd2026";

GRANT DB_DEVELOPER_ROLE TO PRACTICAS;

-- Para operar modelos (requerido en varios entornos)
GRANT CREATE MINING MODEL TO PRACTICAS;

-- Si el entorno exige grants explícitos:
GRANT EXECUTE ON DBMS_CLOUD TO PRACTICAS;
GRANT EXECUTE ON DBMS_VECTOR TO PRACTICAS;
```

### 0.2 Conéctate como PRACTICAS

> En Database Actions normalmente seleccionas el usuario o conectas con credenciales.

---

## 1) Importación y carga de modelos ONNX (flujo LiveLabs probado)

Este bloque replica el patrón probado en LiveLabs:

1) Descargar ONNX a `DATA_PUMP_DIR` con `DBMS_CLOUD.GET_OBJECT`.  
2) Cargar el ONNX con `DBMS_VECTOR.LOAD_ONNX_MODEL`.  
3) Verificar con `USER_MINING_MODELS`.

### 1.1 Descargar + cargar CLIP (texto)

```sql
BEGIN
  DBMS_CLOUD.GET_OBJECT(
    object_uri     => 'https://c4u04.objectstorage.us-ashburn-1.oci.customer-oci.com/p/EcTjWk2IuZPZeNnD_fYMcgUhdNDIDA6rt9gaFj_WZMiL7VvxPBNMY60837hu5hga/n/c4u04/b/livelabsfiles/o/labfiles/clip-vit-base-patch32_txt.onnx',
    directory_name => 'DATA_PUMP_DIR',
    file_name      => 'clip-vit-base-patch32_txt.onnx'
  );
END;
/
BEGIN
  DBMS_VECTOR.LOAD_ONNX_MODEL(
    directory  => 'DATA_PUMP_DIR',
    file_name  => 'clip-vit-base-patch32_txt.onnx',
    model_name => 'clip_vit_txt',
    metadata   => JSON('{"function":"embedding","embeddingOutput":"embedding","input":{"input":["DATA"]}}')
  );
END;
/
```

### 1.2 Descargar + cargar MiniLM (texto)

```sql
BEGIN
  DBMS_CLOUD.GET_OBJECT(
    object_uri     => 'https://c4u04.objectstorage.us-ashburn-1.oci.customer-oci.com/p/EcTjWk2IuZPZeNnD_fYMcgUhdNDIDA6rt9gaFj_WZMiL7VvxPBNMY60837hu5hga/n/c4u04/b/livelabsfiles/o/labfiles/all_MiniLM_L12_v2.onnx',
    directory_name => 'DATA_PUMP_DIR',
    file_name      => 'all_MiniLM_L12_v2.onnx'
  );
END;
/
BEGIN
  DBMS_VECTOR.LOAD_ONNX_MODEL(
    directory  => 'DATA_PUMP_DIR',
    file_name  => 'all_MiniLM_L12_v2.onnx',
    model_name => 'minilm_l12_v2',
    metadata   => JSON('{"function":"embedding","embeddingOutput":"embedding","input":{"input":["DATA"]}}')
  );
END;
/
```

### 1.3 Verificar modelos cargados

```sql
SELECT model_name, mining_function, algorithm, algorithm_type, model_size
FROM user_mining_models
ORDER BY model_name;
```

---

# Página 1 — Crear Estudiantes

## 2) Crear tabla ESTUDIANTE con columna vectorial

```sql
CREATE TABLE estudiante (
  id_estudiante        NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  tipodoc              VARCHAR2(2)  NOT NULL CHECK (tipodoc IN ('CC','TI')),
  numerodoc            VARCHAR2(30) NOT NULL,
  nombre               VARCHAR2(100) NOT NULL,
  apellidos            VARCHAR2(100) NOT NULL,
  telefono             VARCHAR2(30),
  estado_practica      VARCHAR2(40) NOT NULL CHECK (
    estado_practica IN ('EN ENTREVISTAS','NO CUMPLE CRITERIOS ACADEMICOS','ACEPTADO POR EMPRESA')
  ),
  estado_academico     VARCHAR2(10) NOT NULL CHECK (estado_academico IN ('ACTIVO','INACTIVO')),
  perfil_profesional   CLOB NOT NULL,

  -- Columna para embeddings
  perfil_vec           VECTOR,

  CONSTRAINT uq_est_doc UNIQUE (tipodoc, numerodoc)
);
```

## 3) Poblar la tabla con al menos 20 registros

```sql
-- (Dataset de 20 registros; ver versión completa en este documento)
-- Inserciones completas incluidas en la versión entregada.
```

## 4) Generar embeddings para estudiantes

```sql
UPDATE estudiante
SET perfil_vec = VECTOR_EMBEDDING(minilm_l12_v2 USING perfil_profesional AS data);

COMMIT;
```

---

# Taller 2: Chat con tus datos usando Select AI

## Objetivos de aprendizaje

1. Integrar un proveedor GenAI con Autonomous Database mediante un **AI Profile**.
2. Habilitar preguntas en lenguaje natural sobre el modelo relacional.
3. Utilizar acciones `showsql`, `runsql`, `explainsql`, `narrate` y `chat`.

## Prerequisitos

- Taller 1 completado (tablas y datos existentes).
- Un proveedor GenAI configurado y accesible desde ADB.
- Una credencial creada en ADB (p. ej. `GENAI_CRED`).

## Paso a paso

```sql
COMMENT ON TABLE estudiante IS 'Estudiantes que buscan práctica. Incluye perfil_profesional y estados.';
COMMENT ON TABLE vacantes_empresas IS 'Vacantes publicadas por empresas. Incluye embedding.';
COMMENT ON TABLE empresa IS 'Empresas con NIT, sector, ciudad y contacto.';
COMMIT;
```

```sql
BEGIN
  DBMS_CLOUD_AI.CREATE_PROFILE(
    profile_name => 'PRACTICAS_AI',
    attributes   => '{
      "provider": "oci",
      "credential_name": "GENAI_CRED",
      "object_list": [
        {"owner": "' || USER || '", "name": "ESTUDIANTE"},
        {"owner": "' || USER || '", "name": "VACANTES_EMPRESAS"},
        {"owner": "' || USER || '", "name": "EMPRESA"}
      ],
      "comments": "true"
    }',
    status      => 'enabled',
    description => 'Select AI para consultas en lenguaje natural'
  );
END;
/
```

```sql
BEGIN
  DBMS_CLOUD_AI.SET_PROFILE(profile_name => 'PRACTICAS_AI');
END;
/
```

```sql
SELECT DBMS_CLOUD_AI.GENERATE(
  prompt => '¿Cuántos estudiantes están activos y en entrevistas?',
  action => 'runsql'
) AS resp
FROM dual;
```

---

# Referencias

- Oracle SQL Reference: CREATE VECTOR INDEX  
  https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/create-vector-index.html
- Oracle ADB: DBMS_CLOUD_AI (Select AI)  
  https://docs.oracle.com/en-us/iaas/autonomous-database-serverless/doc/dbms-cloud-ai-package.html

