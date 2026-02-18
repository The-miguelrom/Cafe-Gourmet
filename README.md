# MVP IA para clasificación de calidad de tomate en estación fija

## 1) Descripción del MVP
### Alcance
MVP para **tomate** (una sola verdura) en una estación fija de inspección (cámara USB/IP), capaz de:
- Capturar imagen con condiciones controladas (fondo liso, distancia e iluminación fija).
- Clasificar calidad en **A/B/C**.
- Detectar y marcar 3 defectos visibles del MVP:
  - **mancha**,
  - **magulladura/golpe**,
  - **podrido/rajadura**.
- Clasificar calibre **S/M/L** por tamaño aparente y, opcionalmente, tamaño en cm usando marcador de referencia (ArUco/regla).
- Guardar trazabilidad completa por inspección (lote, proveedor, usuario, fecha/hora, resultado, confianza, imágenes).

### Fuera de alcance (MVP)
- Integración con ERP/WMS productivo.
- Detección de defectos internos no visibles (ej. daño interno por presión).
- Inspección multi-objeto por frame en banda transportadora de alta velocidad.
- Autoaprendizaje continuo en línea (se deja para fase 2).

---

## 2) Requerimientos
### Funcionales
1. Login por usuario con rol (operario/supervisor/admin).
2. Crear o seleccionar lote y proveedor/finca.
3. Capturar imagen desde cámara fija o recibir frame desde stream.
4. Ejecutar inferencia y devolver:
   - clase de calidad (A/B/C),
   - defectos detectados con boxes/máscara,
   - confianza,
   - calibre S/M/L.
5. Mostrar resultado y permitir confirmar/guardar (o reintentar captura).
6. Persistir inspección e imágenes (original + anotada).
7. Consultar historial por lote, proveedor, rango de fechas, calidad.
8. Generar reportes agregados (rechazo %, defectos, ranking proveedor).

### No funcionales
- Tiempo de respuesta por inspección: objetivo **< 2 segundos** (CPU) o **< 1 segundo** (GPU).
- Disponibilidad en bodega: operación local offline parcial con sincronización opcional.
- Trazabilidad y auditoría completa de cambios.
- Seguridad por roles + contraseñas hasheadas.
- Escalabilidad para agregar nuevas verduras con reentrenamiento por clase.
- Bajo costo de implementación y mantenimiento.

---

## 3) Arquitectura propuesta (diagrama textual)
```text
[Cámara fija USB/IP]
   -> [Servicio de captura: OpenCV/GStreamer]
      -> [Backend API: FastAPI]
         -> [Servicio de inferencia IA: YOLOv8/YOLOv8-seg + regla de calibre]
            -> [Base de datos PostgreSQL]
            -> [Almacenamiento imágenes: disco local/NAS/S3]
         -> [Dashboard: Metabase o módulo web propio]

Usuarios (Operario/Supervisor/Admin)
   -> UI Web (navegador en PC de estación)
      -> Backend API
```

### Componentes
- **Captura**: proceso local que toma frame con parámetros fijos (resolución, exposición, balance de blancos bloqueado si posible).
- **Backend API**: autenticación, reglas de negocio, persistencia, endpoints.
- **Inferencia**: microservicio Python (PyTorch/Ultralytics) con endpoint interno.
- **DB**: PostgreSQL recomendado por robustez y facilidad de reporteo.
- **Storage**: rutas de imagen en DB, archivos en carpeta estructurada por fecha/lote.
- **BI**: Metabase conectado a PostgreSQL para KPIs sin desarrollar todo desde cero.

---

## 4) Diseño de base de datos (tablas sugeridas)

### `users`
- `id` (PK)
- `name`
- `email` (unique)
- `password_hash`
- `role` (operator|supervisor|admin)
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
- `name` (ej. tomate)
- `variety`
- `active`

### `batches`
- `id` (PK)
- `batch_code` (unique)
- `supplier_id` (FK)
- `product_id` (FK)
- `harvest_date`
- `received_at`
- `created_by` (FK users)

### `inspections`
- `id` (PK)
- `batch_id` (FK)
- `inspected_by` (FK users)
- `inspection_ts`
- `quality_class` (A|B|C)
- `size_class` (S|M|L)
- `size_cm` (nullable)
- `confidence_quality`
- `confidence_size`
- `final_decision` (approved|rework|rejected)
- `notes`

### `defect_types`
- `id` (PK)
- `code` (spot|bruise|rot_crack)
- `name`
- `severity_default`

### `inspection_defects`
- `id` (PK)
- `inspection_id` (FK)
- `defect_type_id` (FK)
- `confidence`
- `severity` (low|medium|high)
- `bbox_x`, `bbox_y`, `bbox_w`, `bbox_h` (nullable si segmentación)
- `mask_path` (nullable)

### `inspection_images`
- `id` (PK)
- `inspection_id` (FK)
- `image_type` (original|annotated)
- `storage_path`
- `checksum`
- `width`, `height`
- `created_at`

### `audit_logs`
- `id` (PK)
- `user_id` (FK)
- `action` (login/create_inspection/update_rule/...)
- `entity`
- `entity_id`
- `payload_json`
- `created_at`

---

## 5) Flujo completo del usuario
1. Operario inicia sesión.
2. Selecciona producto (tomate), proveedor y lote existente o crea nuevo lote.
3. Coloca tomate en marca física de la estación.
4. Presiona “Capturar”.
5. Sistema toma imagen y ejecuta inferencia.
6. UI muestra:
   - calidad A/B/C,
   - calibre S/M/L,
   - defectos marcados,
   - confianza.
7. Operario confirma guardar o repite captura si hubo problema.
8. Sistema guarda inspección + imágenes + bitácora.
9. Supervisor consulta historial y dashboard por lote/proveedor/fecha.

---

## 6) Modelo de IA recomendado

### Recomendación técnica
Para este MVP conviene **YOLOv8-seg** (o YOLOv8 detect si se quiere simplificar):
- Permite detectar defectos localizados y segmentarlos (mejor explicación visual para defensa académica).
- Puede coexistir con clasificación final por reglas de negocio.

### Alternativas
- **Solo clasificación (EfficientNet/MobileNet)**: más simple, pero no explica dónde está el defecto.
- **YOLOv8 detect**: buen balance velocidad/simplicidad con cajas.
- **YOLOv8-seg**: mejor precisión espacial para defectos irregulares (manchas/podrido).

### Estrategia de salida
- Modelo devuelve defectos + confidencias.
- Regla de negocio compone la clase final A/B/C.
- Calibre S/M/L desde área segmentada + referencia física (ArUco).

### Métricas
- Clasificación calidad: accuracy, precision, recall, F1, matriz de confusión.
- Detección/segmentación: mAP@50 y mAP@50:95, precision/recall por defecto.
- Calibre: exactitud por clase S/M/L y error medio en cm (si aplica).
- Definir umbral inicial de confianza (ej. 0.50) y ajustar por validación.

---

## 7) Plan de dataset y etiquetado (300–800 imágenes)

### Captura consistente
- Cámara fija sobre trípode/soporte rígido.
- Distancia fija (ej. 40–60 cm) y misma resolución.
- Caja de luz o iluminación LED constante (CRI alto).
- Fondo mate uniforme (negro/azul) para separar objeto.
- Marca de posición del tomate y marcador ArUco visible.

### Etiquetas del MVP
- `tomato` (objeto principal, opcional para segmentación global).
- Defectos: `spot`, `bruise`, `rot_crack`.
- Label auxiliar por imagen: calidad A/B/C y calibre S/M/L.

### Herramientas
- LabelImg (boxes),
- Roboflow o Label Studio (workflow colaborativo + export YOLO).

### Estrategia de partición
- Train/Val/Test: 70/20/10, separado por lote para evitar fuga de datos.

### Manejo de desbalance
- Aumentar muestras minoritarias (especialmente clase C).
- Data augmentation controlada: brillo, rotación leve, blur leve, ruido.
- Focal loss o class weights en entrenamiento.

### Validación
- Validación cruzada por lote/proveedor (si dataset pequeño).
- Revisión manual de falsos positivos/negativos cada iteración.

---

## 8) API mínima

### `POST /inspect`
Request:
- imagen (multipart o `image_url/path`)
- `batch_code`, `product_id`, `supplier_id`, `user_id`, `timestamp`

Response JSON:
- `inspection_id`
- `quality_class` (A/B/C)
- `size_class` (S/M/L)
- `size_cm` (opcional)
- `defects`: lista `{type, confidence, bbox/mask}`
- `confidence_overall`
- `image_original_path`
- `image_annotated_path`

### `GET /inspections?filters`
Filtros por fecha, proveedor, lote, calidad, usuario.

### `GET /reports`
Agregados:
- rechazo %,
- distribución A/B/C,
- defectos más frecuentes,
- ranking de proveedores,
- tendencia semanal.

---

## 9) Pantallas del MVP
1. **Login**: email/contraseña + rol.
2. **Nueva inspección**: selección de lote/proveedor/producto y vista previa de cámara.
3. **Resultado**: imagen anotada, clase A/B/C, S/M/L, confianza, botón guardar/reintentar.
4. **Historial por lote**: tabla con filtros + detalle de inspección.
5. **Dashboard KPI**:
   - % rechazo,
   - defectos frecuentes,
   - ranking proveedor,
   - tendencia semanal.

---

## 10) Reglas de negocio (ejemplo inicial)
- Si `rot_crack` con confianza >= umbral alto (ej. 0.70) => **C automático**.
- Si hay `spot` o `bruise` leve y calibre válido => **B**.
- Si sin defectos relevantes + calibre en rango objetivo => **A**.
- Si confianza global < umbral mínimo => estado **revisión manual**.

---

## 11) Seguridad y auditoría
- Autenticación con JWT (expiración corta + refresh).
- Autorización por roles:
  - Operario: capturar y consultar lo propio.
  - Supervisor: validar, corregir clasificación, ver reportes.
  - Admin: usuarios, reglas, parámetros del sistema.
- Bitácora de acciones críticas en `audit_logs`.
- Hash de contraseña con Argon2/Bcrypt.
- Backups diarios de DB + almacenamiento de imágenes.

---

## 12) Plan de implementación (8 semanas)
- **Semana 1**: diseño detallado, instalación estación, protocolo de captura.
- **Semana 2**: recolección inicial de dataset (300–500), definición de etiquetas.
- **Semana 3**: etiquetado y QA de etiquetas.
- **Semana 4**: entrenamiento baseline (YOLOv8 detect/seg) + métricas iniciales.
- **Semana 5**: backend API + esquema DB + persistencia imágenes.
- **Semana 6**: frontend MVP (login, nueva inspección, resultado, historial).
- **Semana 7**: integración extremo a extremo + dashboard KPI.
- **Semana 8**: pruebas piloto en bodega, ajustes de umbral y documentación.

---

## 13) Riesgos y mitigación
- **Variación de luz**: caja de luz, bloqueo de exposición, calibración diaria.
- **Suciedad/fondo variable**: protocolo de limpieza por turno.
- **Ángulos no consistentes**: guía física de colocación + soporte fijo.
- **Dataset pequeño/sesgado**: muestreo por proveedor/lote/estado madurez.
- **Concept drift estacional**: plan de reetiquetado y reentrenamiento mensual.
- **Sobreajuste**: separar test por lote, augmentación prudente.

---

## 14) Stack final + despliegue

### Stack recomendado
- **Captura**: Python + OpenCV/GStreamer.
- **IA**: Ultralytics YOLOv8 (detect/seg) + PyTorch.
- **Backend**: FastAPI.
- **Frontend**: React (o plantilla simple server-side).
- **DB**: PostgreSQL.
- **BI**: Metabase.
- **Storage**: local NAS (MVP) con opción S3.

### Opción A: local en bodega (recomendada MVP)
- Todo en una PC industrial/desktop local.
- Baja latencia y dependencia mínima de internet.
- Sincronización opcional nocturna a nube.

### Opción B: nube híbrida
- Captura e inferencia local; métricas y reportes replicados en nube.
- Útil para gestión corporativa multi-planta.

### Hardware mínimo sugerido
- **CPU only**: Intel i5/i7 gen reciente, 16 GB RAM, SSD 512 GB (menor throughput).
- **Con GPU** (preferido): NVIDIA RTX 3050/3060 o equivalente, 16–32 GB RAM.
- Cámara USB 1080p con lente fijo; iluminación LED estable.

---

## Cierre técnico (defendible en tesis)
Este MVP es implementable, de bajo costo y defendible como proyecto de Ingeniería en Sistemas porque integra:
1. Visión por computador aplicada a problema real de cadena agroindustrial.
2. Arquitectura de software completa (captura, inferencia, API, BD, BI).
3. Trazabilidad y gobernanza de datos para decisiones operativas.
4. Métricas objetivas para validar desempeño y plan de mejora continua.

La ruta recomendada es iniciar con **tomate**, estabilizar operación y luego escalar a nuevas verduras reutilizando la misma arquitectura y flujo de datos.
