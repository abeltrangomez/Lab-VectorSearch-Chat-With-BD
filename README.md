# Guía Académica y Técnica: Co-formación con Oracle Autonomous Database  
**Tema:** búsqueda semántica (vectores) y consulta en lenguaje natural sobre datos de co-formación (estudiantes–empresas–vacantes)  
**Entorno:** Oracle Autonomous Database (ADB) + Database Actions (SQL Worksheet)

---

## Contenido
1. [Introducción](#introducción)  
2. [Tecnologías y conceptos](#tecnologías-y-conceptos)  
3. [Taller 1 — Búsqueda semántica y matching Estudiante↔Vacante](#taller-1--búsqueda-semántica-y-matching-estudiantevacante)  
   - 3.1 [Objetivos](#31-objetivos)  
   - 3.2 [Preparación: usuario y permisos (COFORMACION)](#32-preparación-usuario-y-permisos-coformacion)  
   - 3.3 [Bootstrap: tablas, columnas VECTOR y datos](#33-bootstrap-tablas-columnas-vector-y-datos)  
   - 3.4 [Modelo ONNX + embeddings](#34-modelo-onnx--embeddings)  
   - 3.5 [Búsqueda semántica y matching automático](#35-búsqueda-semántica-y-matching-automático)  
4. [Taller 2 — Preguntas en lenguaje natural con Select AI](#taller-2--preguntas-en-lenguaje-natural-con-select-ai)  
   - 4.1 [Objetivos](#41-objetivos)  
   - 4.2 [Integración del proveedor GenAI](#42-integración-del-proveedor-genai)  
   - 4.3 [Consultas en lenguaje natural](#43-consultas-en-lenguaje-natural)  
5. [Anexos](#anexos)  
   - A. [Checklist de verificación](#a-checklist-de-verificación)  
   - B. [Solución de problemas](#b-solución-de-problemas)

---

## Introducción

En un proceso de **co-formación** (prácticas, pasantías o vinculación universidad–empresa), es común gestionar:
- **Estudiantes** con un perfil profesional (skills, intereses, experiencia).
- **Empresas** que publican **vacantes** con un perfil requerido.
- Un proceso de **matching** para identificar qué estudiantes ajustan mejor a cada vacante.

Esta guía implementa un prototipo reproducible sobre **Oracle Autonomous Database** que habilita:
1. **Búsqueda semántica** sobre texto (perfil del estudiante y perfil requerido de la vacante) mediante **vectores**.
2. **Ranking automático** (match) Estudiante↔Vacante usando distancia de vectores.
3. Consultas en **lenguaje natural** sobre el mismo modelo relacional usando **Select AI**.

---

## Tecnologías y conceptos

### 1) Modelo relacional (3 entidades + asociación)
- `ESTUDIANTE`: perfil profesional del estudiante.
- `EMPRESA`: información de empresa.
- `VACANTES_EMPRESAS`: vacantes por empresa.
- `ESTUDIANTE_VACANTE`: tabla de asociación (postulaciones / asignaciones).

### 2) Vectores y embeddings
- Un **embedding** transforma texto en un vector numérico de alta dimensión.
- Se almacena en columnas tipo `VECTOR`.
- La similitud se calcula con métricas como **COSINE**.

### 3) Modelos ONNX en base de datos
- ONNX es un formato estándar para modelos.
- El modelo se carga en la base y se usa para generar embeddings desde SQL/PLSQL.

### 4) Select AI
- Permite que el usuario haga preguntas en lenguaje natural y la base genere/ejecute SQL.

---

# Parte 1 — Búsqueda semántica y matching Estudiante↔Vacante

## 3.1 Objetivos
Al finalizar, el estudiante podrá:
- Crear un **esquema de trabajo** (usuario) con privilegios mínimos para ejecutar tareas de co-formación.
- Construir el **modelo de datos** y poblarlo con datos de prueba.
- Importar un **modelo ONNX** y generar embeddings en columnas `VECTOR`.
- Ejecutar **búsqueda semántica** y **matching automático**.

---

## 3.2 Preparación: usuario y permisos (COFORMACION)

### Debe ya tener creada la ADB version 26.ai en la region correcta

### A) Crear usuario desde Database Actions (UI)
En **Database Actions → Administration → Database Users → Create User**:

Configurar:
- **User Name:** `COFORMACION`
- **Password:** `Welcome_12345`
- **Quota on tablespace DATA:** `UNLIMITED`
- **Password Expired:** OFF  
- **Account is Locked:** OFF  
- **Graph:** OFF  
- **OML:** OFF  
- **REST, GraphQL, MongoDB, and Web access:** ON  
- Click **Create User**

### B) Conceder roles y habilitar REST (SQL)
En **SQL Worksheet**, conectado como `ADMIN`, ejecutar:

```sql
GRANT CONNECT TO COFORMACION;
GRANT RESOURCE TO COFORMACION;

BEGIN
  ORDS_ADMIN.ENABLE_SCHEMA(
    p_enabled             => TRUE,
    p_schema              => 'COFORMACION',
    p_url_mapping_type    => 'BASE_PATH',
    p_url_mapping_pattern => 'coformacion',
    p_auto_rest_auth      => TRUE
  );
  COMMIT;
END;
/
```

### C) Permisos operativos (descargas ONNX, directorio, vector)
Ejecutar como `ADMIN`:

```sql
GRANT ORDS_RUNTIME_ROLE TO COFORMACION;

GRANT EXECUTE ON DBMS_CLOUD TO COFORMACION;
GRANT EXECUTE ON DBMS_NETWORK_ACL_ADMIN TO COFORMACION;

GRANT READ, WRITE ON DIRECTORY DATA_PUMP_DIR TO COFORMACION;

GRANT CREATE MINING MODEL TO COFORMACION;

GRANT SELECT ON SYS.V_$VECTOR_MEMORY_POOL TO COFORMACION;
GRANT SELECT ON SYS.V_$VECTOR_INDEX TO COFORMACION;

-- Si al cargar modelos ONNX aparece "insufficient privileges":
GRANT EXECUTE ON DBMS_VECTOR TO COFORMACION;
```

### D) Validación rápida
Conectado como `COFORMACION`:

```sql
SELECT USER FROM dual;
SELECT directory_name FROM all_directories WHERE directory_name='DATA_PUMP_DIR';
```

---

## 3.3 Bootstrap: tablas, columnas VECTOR y datos

### Ejecutar bootstrap (un solo script)
1. Conéctate como `COFORMACION`.
2. En SQL Worksheet, ejecuta el archivo [`bootstrap_practicas_vector_schema.sql`](bootstrap_practicas_vector_schema.sql).

Resultados esperados:
- 5 empresas
- 10 vacantes
- 20 estudiantes
- 10 asociaciones

---

## 3.4 Modelo ONNX + embeddings

### A) Importar y cargar modelo ONNX (texto)
Conectado como `COFORMACION`:

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

Verificación:

```sql
SELECT model_name, mining_function, algorithm, algorithm_type, model_size
FROM user_mining_models
ORDER BY model_name;
```

### B) Generar embeddings para estudiantes y vacantes
```sql
UPDATE estudiante
SET perfil_vec = VECTOR_EMBEDDING(minilm_l12_v2 USING perfil_profesional AS data);

UPDATE vacantes_empresas
SET perfil_vec = VECTOR_EMBEDDING(minilm_l12_v2 USING descripcion_perfil AS data);

COMMIT;
```

---

## 3.5 Búsqueda semántica y matching automático

### A) Búsqueda semántica: texto libre → Top estudiantes
```sql
VAR q CLOB;
EXEC :q := 'DevOps junior con Docker, CI/CD, Kubernetes básico y monitoreo';

WITH query_vec AS (
  SELECT VECTOR_EMBEDDING(minilm_l12_v2 USING :q AS data) AS v
  FROM dual
)
SELECT
  e.id_estudiante, e.nombre, e.apellidos,
  VECTOR_DISTANCE(e.perfil_vec, q.v, COSINE) AS distancia
FROM estudiante e
CROSS JOIN query_vec q
WHERE e.estado_academico = 'ACTIVO'
ORDER BY distancia
FETCH FIRST 5 ROWS ONLY;
```

### B) Matching automático: una vacante → ranking de estudiantes
```sql
VAR vacante_id NUMBER;
EXEC :vacante_id := 7;

WITH vac AS (
  SELECT perfil_vec FROM vacantes_empresas WHERE id_vacante = :vacante_id
)
SELECT
  e.id_estudiante, e.nombre, e.apellidos,
  VECTOR_DISTANCE(e.perfil_vec, v.perfil_vec, COSINE) AS distancia
FROM estudiante e
CROSS JOIN vac v
WHERE e.estado_academico = 'ACTIVO'
  AND e.estado_practica <> 'NO CUMPLE CRITERIOS ACADEMICOS'
ORDER BY distancia
FETCH FIRST 10 ROWS ONLY;
```

### C) (Opcional) Índices vectoriales
```sql
CREATE VECTOR INDEX idx_est_perfil_vec
ON estudiante (perfil_vec)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE;

CREATE VECTOR INDEX idx_vac_perfil_vec
ON vacantes_empresas (perfil_vec)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE;
```

---

# Parte 2 — Preguntas en lenguaje natural con Select AI

## 4.1 Objetivos
Al finalizar, el estudiante podrá:
- Crear un perfil de IA para consultar el esquema de co-formación en lenguaje natural.
- Generar SQL (`showsql`) y ejecutar SQL (`runsql`) desde prompts.
- Obtener explicaciones (`explainsql`) y narrativas (`narrate`).

## 4.2 Integración del proveedor GenAI

### A) Añadir metadatos (comentarios)
```sql
COMMENT ON TABLE estudiante IS 'Estudiantes del programa de co-formación. Incluye perfil profesional y estados.';
COMMENT ON TABLE vacantes_empresas IS 'Vacantes por empresa. Incluye perfil requerido y embeddings.';
COMMENT ON TABLE empresa IS 'Empresas con NIT, sector, ciudad y contacto.';
COMMENT ON COLUMN estudiante.estado_practica IS 'EN ENTREVISTAS / NO CUMPLE CRITERIOS ACADEMICOS / ACEPTADO POR EMPRESA';
COMMIT;
```

### B) Crear y activar perfil Select AI
```sql
BEGIN
  DBMS_CLOUD_AI.CREATE_PROFILE(
    profile_name => 'COFORMACION_AI',
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
    description => 'Perfil para consultas en lenguaje natural sobre co-formación'
  );
END;
/
BEGIN
  DBMS_CLOUD_AI.SET_PROFILE(profile_name => 'COFORMACION_AI');
END;
/
```

---

## 4.3 Consultas en lenguaje natural

```sql
SELECT DBMS_CLOUD_AI.GENERATE(
  prompt => '¿Cuántos estudiantes están ACTIVO y en EN ENTREVISTAS?',
  action => 'showsql'
) AS resp
FROM dual;
```

```sql
SELECT DBMS_CLOUD_AI.GENERATE(
  prompt => 'Lista las 5 vacantes más próximas a cerrar con nombre de empresa y ciudad.',
  action => 'runsql'
) AS resp
FROM dual;
```

---

# Anexos

## A) Checklist de verificación

```sql
SELECT COUNT(*) AS empresas FROM empresa;
SELECT COUNT(*) AS vacantes FROM vacantes_empresas;
SELECT COUNT(*) AS estudiantes FROM estudiante;
SELECT COUNT(*) AS asociaciones FROM estudiante_vacante;
```

```sql
SELECT model_name FROM user_mining_models ORDER BY 1;
```

```sql
SELECT COUNT(*) total, SUM(CASE WHEN perfil_vec IS NOT NULL THEN 1 ELSE 0 END) con_vector
FROM estudiante;

SELECT COUNT(*) total, SUM(CASE WHEN perfil_vec IS NOT NULL THEN 1 ELSE 0 END) con_vector
FROM vacantes_empresas;
```

## B) Solución de problemas

- **Permisos ONNX (GET_OBJECT):** verifica `DBMS_CLOUD` y `DATA_PUMP_DIR`.
- **Permisos LOAD_ONNX_MODEL:** concede `EXECUTE ON DBMS_VECTOR`.
- **Modelo no visible:** revisar `USER_MINING_MODELS`.
