# MVP de Clasificación de Calidad de Tomate con IA (Cámara Fija)

## 1) Descripción del MVP (alcance y fuera de alcance)

### Alcance del MVP
Este MVP está enfocado en **tomate** como único producto inicial para reducir complejidad y acelerar validación en planta.

Incluye:
- Estación de inspección con **cámara fija** (USB o IP), fondo liso, iluminación constante y distancia fija.
- Captura de una imagen por tomate (o pequeño grupo controlado, recomendado 1 unidad por captura para MVP).
- Inferencia de IA para:
  - **Calidad A/B/C**.
  - **Detección de defectos visibles** (MVP: `mancha`, `magulladura`, `podrido/rajadura severa`).
  - **Calibre S/M/L** por tamaño proyectado; opcional conversión px→cm con marcador de referencia (ArUco/tarjeta patrón).
- Persistencia de trazabilidad completa por lote/proveedor/usuario/fecha-hora.
- Visualización de resultados por historial y dashboard básico de KPIs.

### Fuera de alcance (fase posterior)
- Clasificación multiespecie (tomate + chile + cebolla, etc.).
- Integración con PLC/cintas transportadoras en tiempo real industrial.
- MLOps avanzado (entrenamiento continuo automático, drift detection en producción).
- Modelos 3D o multiespectrales (NIR/hyperspectral).

---

## 2) Requerimientos funcionales y no funcionales

### Funcionales
1. Registrar usuarios con roles (`operario`, `supervisor`, `admin`).
2. Crear/seleccionar lote (proveedor/finca, producto, fecha).
3. Capturar imagen desde cámara fija o recibir frame desde stream.
4. Ejecutar inferencia y devolver:
   - Clase de calidad A/B/C
   - Defectos detectados con ubicación (bbox o máscara)
   - Calibre S/M/L
   - Confianzas por salida
5. Mostrar imagen anotada al operario.
6. Guardar inspección y metadatos de trazabilidad.
7. Consultar historial con filtros (fecha, lote, proveedor, calidad, defecto).
8. Generar resumen KPI por periodo/lote/proveedor.

### No funcionales
- **Costo bajo**: hardware commodity (PC + webcam industrial básica + iluminación LED).
- **Latencia**: respuesta por inspección objetivo < 1.5 s en CPU decente; < 500 ms con GPU.
- **Disponibilidad**: operación local aun sin internet.
- **Trazabilidad**: persistencia de evidencia visual (imagen original y anotada).
- **Seguridad**: autenticación, autorización por rol, bitácora de acciones.
- **Escalabilidad**: diseño modular para agregar nuevas verduras/modelos.
- **Mantenibilidad**: API documentada (OpenAPI), versionado de modelo y datos.

---

## 3) Arquitectura propuesta (diagrama textual)

```text
[Cámara Fija USB/IP]
        |
        v
[Servicio de Captura: OpenCV/GStreamer]
        |  (frame + metadata de estación)
        v
[Backend API: FastAPI]
  |        |             |
  |        |             +--> [Almacenamiento de Imágenes: Local/NAS/S3]
  |        +--> [DB PostgreSQL]
  |
  +--> [Servicio de Inferencia IA]
           |- Modelo detección/segmentación defectos (YOLOv8/YOLOv8-seg)
           |- Modelo clasificación calidad (A/B/C) o regla híbrida
           |- Módulo calibre S/M/L (visión + referencia ArUco opcional)

[Frontend Web (Operario/Supervisor)] <--> [Backend API]

[Dashboard BI: Metabase/Power BI/Looker]
      consulta vistas SQL/tabla agregada en PostgreSQL
```

### Recomendación técnica concreta
- **Backend**: FastAPI (rápido, tipado, documentación automática, buen fit con Python/IA).
- **Inferencia**: servicio Python separado (misma máquina en MVP).
- **DB**: PostgreSQL.
- **Imágenes**: carpeta local estructurada por fecha/lote; migrable a NAS/S3.
- **Dashboard**: Metabase self-hosted para bajo costo.

---

## 4) Diseño de base de datos (tablas y campos)

### `users`
- `id` (PK)
- `username` (unique)
- `password_hash`
- `full_name`
- `role` (`OPERARIO`, `SUPERVISOR`, `ADMIN`)
- `is_active`
- `created_at`

### `suppliers`
- `id` (PK)
- `name`
- `farm_code`
- `location`
- `contact`
- `created_at`

### `products`
- `id` (PK)
- `name` (ej. `tomate`)
- `variety` (opcional)
- `size_rule_json` (rangos S/M/L por cm o px calibrado)
- `created_at`

### `lots`
- `id` (PK)
- `lot_code` (unique)
- `supplier_id` (FK -> suppliers)
- `product_id` (FK -> products)
- `harvest_date` (opcional)
- `received_at`
- `status` (`OPEN`, `CLOSED`)
- `created_by` (FK -> users)

### `inspections`
- `id` (PK)
- `lot_id` (FK -> lots)
- `product_id` (FK -> products)
- `operator_id` (FK -> users)
- `inspection_ts`
- `quality_class` (`A`,`B`,`C`)
- `quality_confidence` (float)
- `size_class` (`S`,`M`,`L`)
- `size_value_px` (float)
- `size_value_cm` (float, nullable)
- `final_decision_reason` (texto/regla aplicada)
- `model_version`
- `station_id` (opcional)

### `inspection_images`
- `id` (PK)
- `inspection_id` (FK -> inspections)
- `original_path`
- `annotated_path`
- `thumbnail_path` (opcional)
- `image_hash` (integridad/deduplicación)
- `created_at`

### `defect_types`
- `id` (PK)
- `code` (`MANCHA`, `MAGULLADURA`, `PODRIDO`)
- `description`
- `severity_default`

### `inspection_defects`
- `id` (PK)
- `inspection_id` (FK -> inspections)
- `defect_type_id` (FK -> defect_types)
- `confidence`
- `bbox_x`, `bbox_y`, `bbox_w`, `bbox_h` (si detección)
- `mask_path` (si segmentación)
- `severity` (leve/media/alta)

### `audit_logs`
- `id` (PK)
- `user_id` (FK -> users)
- `action` (LOGIN, CREATE_LOT, INSPECT, OVERRIDE, etc.)
- `entity_type`
- `entity_id`
- `details_json`
- `created_at`
- `ip_address`

---

## 5) Flujo completo del usuario (paso a paso)

1. **Login** del operario.
2. En “Nueva inspección”, selecciona:
   - Producto = tomate
   - Lote (existente) o crear nuevo
   - Proveedor/finca (si aplica)
3. Coloca tomate en marca física de la estación (posición fija).
4. Presiona **Capturar**.
5. El backend toma frame y envía al servicio de inferencia.
6. La IA retorna calidad, defectos detectados y calibre.
7. UI muestra:
   - Imagen anotada
   - Etiqueta A/B/C + confianza
   - Defectos + confianza
   - Tamaño S/M/L
8. Operario confirma/guarda (o supervisor corrige con motivo).
9. Sistema guarda inspección, imágenes y bitácora.
10. Supervisor revisa historial y dashboard por lote/proveedor.

---

## 6) Modelo de IA recomendado

### ¿Clasificación, detección o segmentación?
- **Clasificación sola** (A/B/C): simple pero no explica “por qué”.
- **Detección**: localiza defectos con cajas; balance ideal costo/beneficio en MVP.
- **Segmentación**: más precisa en área afectada, pero requiere más etiquetado.

### Recomendación MVP
**Enfoque híbrido**:
1. **YOLOv8 (detección)** para defectos (`mancha`, `magulladura`, `podrido`).
2. **Reglas de negocio + señales del detector** para decisión final A/B/C.
3. **Calibre S/M/L** por segmentación simple del contorno del tomate (OpenCV) o usando bbox principal + referencia.

> Si el equipo tiene capacidad de etiquetar máscaras, evolucionar a **YOLOv8-seg** en Fase 2.

### Métricas clave
- Clasificación calidad: `accuracy`, `precision`, `recall`, `F1`, matriz de confusión A/B/C.
- Detección defectos: `mAP@0.5`, `mAP@0.5:0.95`, `precision`, `recall` por clase.
- Operación: latencia media por inferencia, tasa de inspecciones con baja confianza.

### Umbral de confianza
- Defecto detectado válido si `conf >= 0.50` (ajustable por clase).
- Si calidad `confidence < 0.60`, enviar a revisión manual.

---

## 7) Plan de dataset y etiquetado (300–800 imágenes)

### Estrategia de captura consistente
- Estación fija con:
  - Fondo mate uniforme (azul/negro según contraste con tomate).
  - 2 luces LED difusas laterales para reducir sombras duras.
  - Distancia cámara-producto fija (soporte rígido).
  - Marca de posicionamiento del fruto.
- Tomar muestras en variación realista:
  - Lotes/proveedores diferentes.
  - Distintos niveles de madurez y daño.
  - Limpio vs leve suciedad (controlado).

### Etiquetas mínimas
- **Defectos (detección)**: `mancha`, `magulladura`, `podrido`.
- **Clase global** por imagen: `A/B/C`.
- **Calibre**: `S/M/L` (derivado de medición o etiqueta manual inicial para validación).

### Herramientas recomendadas
- Inicio rápido: **Roboflow** o **Label Studio**.
- Alternativa local clásica: **LabelImg** (bounding boxes).

### Split y validación
- Split sugerido: 70% train / 15% val / 15% test (estratificado por clase y proveedor).
- Evitar fuga de datos: imágenes del mismo lote idealmente en un solo split cuando sea posible.
- Medir desempeño por proveedor para detectar sesgo.

### Desbalance
- Data augmentation (rotación leve, brillo/contraste moderado, blur suave).
- Sobre-muestreo de clases minoritarias (ej. `podrido`).
- Ajuste de umbrales por clase en validación.

---

## 8) API mínima

### `POST /inspect`
**Entrada (multipart/form-data o JSON + image path):**
- `image` (archivo o frame)
- `lot_code`
- `supplier_id`
- `product_id`
- `operator_id`
- `station_id` (opcional)

**Salida:**
```json
{
  "inspection_id": 10231,
  "quality_class": "B",
  "quality_confidence": 0.87,
  "size_class": "M",
  "size_value_px": 214.2,
  "size_value_cm": 6.8,
  "defects": [
    {"type": "MANCHA", "confidence": 0.79, "bbox": [120, 88, 46, 39]}
  ],
  "annotated_image_path": "images/2026/02/lot-LT001/ann_10231.jpg",
  "rules_applied": ["mancha_leve=>B"]
}
```

### `GET /inspections?filters`
Filtros: `date_from`, `date_to`, `lot_code`, `supplier_id`, `quality_class`, `defect_type`, `page`, `size`.

### `GET /reports`
Retorna agregados:
- rechazo % (C)
- distribución A/B/C
- defectos frecuentes
- ranking proveedor por tasa de rechazo
- tendencia semanal

---

## 9) Pantallas del MVP

1. **Login**
   - Usuario/contraseña
   - Mensajes de error y bloqueo básico por intentos

2. **Nueva inspección**
   - Selección de lote/producto/proveedor
   - Vista previa cámara
   - Botón capturar

3. **Resultado**
   - Imagen anotada
   - Calidad A/B/C con color semáforo
   - Defectos detectados con confianza
   - Calibre S/M/L
   - Botón guardar / repetir

4. **Historial por lote**
   - Tabla paginada
   - Filtros
   - Acceso a imagen original/anotada

5. **Dashboard KPI**
   - `% rechazo`
   - Defectos más frecuentes
   - Ranking de proveedor
   - Tendencia semanal

---

## 10) Reglas de negocio (ejemplos)

1. Si existe defecto `PODRIDO` con `conf >= 0.50` => **C automático**.
2. Si solo `MANCHA` leve y calibre válido => **B**.
3. Si sin defectos detectados y calibre en rango comercial => **A**.
4. Si múltiples defectos medianos (2+) => bajar una categoría (A→B, B→C).
5. Si confianza global < umbral => **Revisión manual** por supervisor.

---

## 11) Seguridad y auditoría

### Roles
- **Operario**: captura/guarda inspecciones.
- **Supervisor**: revisa, corrige clasificación con motivo.
- **Admin**: gestiona usuarios, catálogos, configuración de modelo/umbrales.

### Controles
- JWT con expiración corta + refresh token.
- Hash de contraseñas con bcrypt/argon2.
- Autorización por endpoint/rol.
- Bitácora obligatoria de eventos críticos (login, override, eliminación lógica).
- Versionado de modelo (`model_version`) en cada inspección para trazabilidad técnica.

---

## 12) Plan de implementación (6–10 semanas)

### Semana 1
- Levantamiento detallado de criterios de calidad (con experto agrónomo/supervisor).
- Diseño de estación física y protocolo de captura.

### Semana 2
- Montaje de estación (cámara + iluminación + fondo + marca).
- Prototipo de captura con OpenCV.

### Semana 3
- Recolección inicial de imágenes (300–800).
- Definición de guía de etiquetado.

### Semana 4
- Etiquetado y QA de etiquetas.
- Entrenamiento base YOLOv8 + baseline reglas A/B/C.

### Semana 5
- Construcción Backend API + esquema DB + almacenamiento de imágenes.

### Semana 6
- UI web MVP (login, nueva inspección, resultado, historial).
- Integración end-to-end.

### Semana 7
- Validación en piso (pruebas piloto reales).
- Ajuste de umbrales y reglas.

### Semana 8
- Dashboard KPI + reportes.
- Hardening básico de seguridad y auditoría.

### Semanas 9–10 (opcional)
- Mejoras de precisión, documentación final, demo técnica y defensa académica.

---

## 13) Riesgos y mitigación

1. **Cambios de iluminación**
   - Mitigación: caja de luz/iluminación fija, balance de blancos fijo, checklist diario.

2. **Variación natural del tomate (variedad/madurez)**
   - Mitigación: dataset diverso por proveedor/temporada; reevaluación periódica.

3. **Fondo sucio o reflejos**
   - Mitigación: protocolo de limpieza por turno; material mate antirreflejo.

4. **Ángulos/posición inconsistentes**
   - Mitigación: guía física de posicionamiento; distancia fija de cámara.

5. **Desbalance de clases (pocos podridos)**
   - Mitigación: muestreo dirigido, augmentación, revisión de métricas por clase.

6. **Sesgo por proveedor**
   - Mitigación: partición y reporte por proveedor; ampliar data de minoritarios.

---

## 14) Stack final + despliegue

### Stack recomendado MVP
- **Captura**: Python + OpenCV (GStreamer opcional para IP cam).
- **Inferencia**: Ultralytics YOLOv8 en Python.
- **Backend**: FastAPI + Uvicorn + SQLAlchemy.
- **DB**: PostgreSQL.
- **Frontend**: React (o plantilla simple server-rendered para acelerar).
- **BI**: Metabase.
- **Storage**: disco local estructurado; luego NAS/S3.

### Opción A: Local en PC de bodega (recomendada para MVP)
- Todo en una sola máquina (captura + API + inferencia + DB).
- Pros: bajo costo, baja latencia, no depende de internet.
- Contras: menor elasticidad, respaldo manual requerido.

### Opción B: Híbrida/Nube
- Captura e inferencia local, sincronización de resultados a nube.
- Dashboard y respaldo centralizados.
- Pros: multi-sede y analítica consolidada.
- Contras: mayor complejidad y costo operativo.

### Hardware mínimo sugerido
- **CPU only (MVP básico):**
  - Intel i5/i7 o Ryzen 5/7 reciente
  - 16 GB RAM
  - SSD 512 GB
  - Webcam 1080p + luces LED
- **Con GPU (recomendado si mayor volumen):**
  - NVIDIA RTX 3060/4060 (8 GB+) para inferencia más rápida y reentrenos.

---

## Enfoque de defensa académica (proyecto de graduación)

Para hacerlo defendible en Ingeniería en Sistemas:
1. Definir hipótesis medible: “La IA reduce variabilidad de inspección manual y mejora trazabilidad”.
2. Comparar contra baseline humano (tiempo, consistencia, tasa de error).
3. Presentar métricas técnicas + métricas de negocio (rechazo, reproceso, tiempos).
4. Documentar arquitectura, seguridad, auditoría y costos.
5. Mostrar escalabilidad del diseño a múltiples productos sin rediseño total.
