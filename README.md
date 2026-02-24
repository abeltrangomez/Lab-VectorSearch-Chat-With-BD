# Guía: Co-formación con Oracle Autonomous Database  IA
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


GRANT EXECUTE ON DBMS_VECTOR TO COFORMACION;

GRANT EXECUTE ON C##CLOUD$SERVICE.DBMS_CLOUD_AI TO COFORMACION;


```

### D) Validación rápida
Conectado como `COFORMACION`:

```sql
SELECT USER FROM dual;
SELECT directory_name FROM all_directories WHERE directory_name='DATA_PUMP_DIR';
```



## 3.3 Modelo ONNX + embeddings

### A) Importar y cargar modelo ONNX (texto)
Conectado como `COFORMACION`:

```sql
begin
  dbms_cloud.get_object(
    object_uri=>'https://c4u04.objectstorage.us-ashburn-1.oci.customer-oci.com/p/EcTjWk2IuZPZeNnD_fYMcgUhdNDIDA6rt9gaFj_WZMiL7VvxPBNMY60837hu5hga/n/c4u04/b/livelabsfiles/o/labfiles/clip-vit-base-patch32_txt.onnx',
    directory_name=>'DATA_PUMP_DIR',
    file_name=>'clip-vit-base-patch32_txt.onnx'
  );
end;
/
begin
  dbms_vector.load_onnx_model(directory=>'DATA_PUMP_DIR', 
    file_name=>'clip-vit-base-patch32_txt.onnx', model_name=>'clip_vit_txt',
    metadata=>JSON('{"function" : "embedding", "embeddingOutput" : "embedding" , "input": {"input": ["DATA"]}}'));
end;
/
begin
  dbms_cloud.get_object(
    object_uri=>'https://c4u04.objectstorage.us-ashburn-1.oci.customer-oci.com/p/EcTjWk2IuZPZeNnD_fYMcgUhdNDIDA6rt9gaFj_WZMiL7VvxPBNMY60837hu5hga/n/c4u04/b/livelabsfiles/o/labfiles/all_MiniLM_L12_v2.onnx',
    directory_name=>'DATA_PUMP_DIR',
    file_name=>'all_MiniLM_L12_v2.onnx'
  );
end;
/

BEGIN
  DBMS_VECTOR.LOAD_ONNX_MODEL('DATA_PUMP_DIR','all_MiniLM_L12_v2.onnx','minilm_l12_v2',
  JSON('{"function" : "embedding", "embeddingOutput" : "embedding", "input": {"input": ["DATA"]}}'));
END;



```

Verificación:

```sql
SELECT model_name, mining_function, algorithm, algorithm_type, model_size
FROM user_mining_models
ORDER BY model_name;
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

## 3.4 Búsqueda semántica y matching automático

### A) Búsqueda semántica: texto libre → Top estudiantes
```sql
VAR q VARCHAR2(4000);
EXEC :q := 'Devops';

SELECT
  e.nombre || ' ' || e.apellidos AS name,
  VECTOR_DISTANCE(
    e.perfil_vec,
    VECTOR_EMBEDDING(minilm_l12_v2 USING :q AS data),
    COSINE
  ) AS distance,
  e.perfil_profesional AS description
FROM estudiante e
WHERE e.estado_academico = 'ACTIVO'
  AND e.perfil_vec IS NOT NULL
ORDER BY 2
FETCH EXACT FIRST 5 ROWS ONLY;
```

### B) Matching automático: una vacante → ranking de estudiantes
```sql


VAR vacante_id NUMBER;
EXEC :vacante_id := 7;

SELECT
  v.id_vacante,
  e.nombre_empresa,
  e.ciudad,
  v.fecha_apertura_vacante,
  v.fecha_cierre_vacante,
  v.descripcion_perfil
FROM vacantes_empresas v
JOIN empresa e
  ON e.nit_empresa = v.nit_empresa
WHERE v.id_vacante = :vacante_id;

-- Detalle / Ranking de candidatos (NO repite la descripción)
VAR vacante_id NUMBER;
EXEC :vacante_id := 7;

WITH vac AS (
  SELECT perfil_vec
  FROM vacantes_empresas
  WHERE id_vacante = :vacante_id
)
SELECT
  e.id_estudiante,
  e.nombre,
  e.apellidos,
  e.perfil_profesional AS perfil_estudiante,
  VECTOR_DISTANCE(e.perfil_vec, (SELECT perfil_vec FROM vac), COSINE) AS distancia
FROM estudiante e
WHERE e.estado_academico = 'ACTIVO'
  AND e.estado_practica <> 'NO CUMPLE CRITERIOS ACADEMICOS'
  AND e.perfil_vec IS NOT NULL
  AND (SELECT perfil_vec FROM vac) IS NOT NULL
ORDER BY 5
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
  DBMS_CLOUD_AI.DROP_PROFILE(profile_name => 'genai', force => TRUE);

  DBMS_CLOUD_AI.CREATE_PROFILE(
    profile_name => 'genai',
    attributes   => '{
      "provider": "oci",
      "credential_name": "OCI$RESOURCE_PRINCIPAL",
      "region": "sa-saopaulo-1",
      "comments": true,
      "object_list": [
        {"owner": "COFORMACION", "name": "EMPRESA"},
        {"owner": "COFORMACION", "name": "VACANTES_EMPRESAS"},
        {"owner": "COFORMACION", "name": "ESTUDIANTE"},
        {"owner": "COFORMACION", "name": "ESTUDIANTE_VACANTE"}
      ]
    }'
  );
END;
/

BEGIN
  DBMS_CLOUD_AI.SET_PROFILE('genai');
END;
/

BEGIN
  DBMS_CLOUD_ADMIN.ENABLE_RESOURCE_PRINCIPAL(
    username => 'COFORMACION'
  );
END;
/
```



## 4.3 Consultas en lenguaje natural

```sql

-- 1) SHOWSQL: Solo genera el SQL (no ejecuta)
SELECT DBMS_CLOUD_AI.GENERATE(
         prompt       => 'Dame el tipo de documento, el número de documento y el nombre de los estudiantes que están en ''EN ENTREVISTAS''.',
         profile_name => 'genai',
         action       => 'showsql'
       ) AS resp
FROM (SELECT 1 AS x);

-- 2) RUNSQL: Genera y ejecuta 
SELECT DBMS_CLOUD_AI.GENERATE(
         prompt       => 'Dame el tipo de documento, el número de documento y el nombre de los estudiantes que están en ''EN ENTREVISTAS''.',
         profile_name => 'genai',
         action       => 'runsql'
       ) AS resp
FROM (SELECT 1 AS x);

-- 3) NARRATE: Ejecuta y responde en español
SELECT DBMS_CLOUD_AI.GENERATE(
         prompt       => 'En español y de forma breve: lista el tipo de documento, número de documento y nombre de los estudiantes que están en ''EN ENTREVISTAS''.',
         profile_name => 'genai',
         action       => 'narrate'
       ) AS resp
FROM (SELECT 1 AS x);

-- 4) CHAT: Pregunta general (no orientada a SQL)
SELECT DBMS_CLOUD_AI.GENERATE(
         prompt       => 'Explica en 6 a 8 líneas qué es Oracle Autonomous Database y cuáles son tres beneficios principales.',
         profile_name => 'genai',
         action       => 'chat'
       ) AS resp
FROM (SELECT 1 AS x);

```

```sql
SELECT DBMS_CLOUD_AI.GENERATE(
  prompt       => 'Lista las 5 vacantes más próximas a cerrar con nombre de empresa y ciudad.',
  profile_name => 'genai',
  action       => 'narrate'
) AS resp
FROM (SELECT 1);
```

---

## 5. Creacion de Stack para ejecucion archivos de IaC main.tf

<img width="621" height="532" alt="image" src="https://github.com/user-attachments/assets/d09b15ed-3fc7-4be9-ad28-febe097de1e2" />


<img width="1567" height="673" alt="image" src="https://github.com/user-attachments/assets/0e93e04c-d6ff-43d9-aa29-baeed4e40dfe" />
<img width="480" height="268" alt="image" src="https://github.com/user-attachments/assets/ff0e679a-85d2-4672-b216-09ec247bce77" />
<img width="1133" height="576" alt="image" src="https://github.com/user-attachments/assets/db947880-3e35-451e-949d-8d3a01952198" />
<img width="1052" height="261" alt="image" src="https://github.com/user-attachments/assets/733cb734-67c0-40f1-b52a-07196abb271c" />
<img width="970" height="241" alt="image" src="https://github.com/user-attachments/assets/6f6829e0-cdd6-4db5-9d65-290df73e3752" />
<img width="1225" height="593" alt="image" src="https://github.com/user-attachments/assets/51dd937c-4ccf-4bdd-9cb7-55d7d87dfc2c" />







