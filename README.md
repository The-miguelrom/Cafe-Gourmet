# MVP IA para Clasificación de Calidad de Tomate en Estación Fija

## 1) Descripción del MVP

### Alcance
MVP orientado a **1 verdura: tomate** (escalable luego a chile pimiento y otras). El sistema usa una **cámara fija** en estación de inspección para:
- Clasificar calidad en **A/B/C**.
- Detectar y marcar defectos visibles (MVP: **manchas**, **magulladuras/golpes**, **podrido/rajadura crítica**).
- Clasificar calibre **S/M/L**.
- Guardar trazabilidad completa por lote (metadatos + imagen original + imagen anotada + resultado).

### Fuera de alcance (MVP)
- Robótica de banda transportadora o rechazo automático físico.
- Integración ERP compleja bidireccional.
- Multi-cámara 360° o inspección interna del fruto.
- MLOps avanzado (A/B testing de modelos en producción, drift detection fully automated).

---

## 2) Requerimientos

### Funcionales
1. Registrar usuarios con rol (operario/supervisor/admin).
2. Crear/seleccionar lote activo con proveedor/finca y producto.
3. Capturar imagen desde cámara fija o cargar imagen desde estación.
4. Ejecutar inferencia IA y devolver:
   - categoría calidad A/B/C,
   - defectos detectados con ubicación,
   - calibre S/M/L,
   - confianza.
5. Mostrar resultado visual (imagen anotada).
6. Permitir corrección manual por supervisor (si aplica).
7. Guardar trazabilidad de cada inspección.
8. Consultar historial con filtros (fecha, lote, proveedor, clase, defecto).
9. Mostrar reportes KPI básicos.

### No funcionales
- **Costo bajo**: hardware commodity (PC + webcam/IP cam + iluminación LED).
- **Disponibilidad**: operación offline local en bodega.
- **Latencia**: respuesta de inferencia objetivo < 1.5 s por fruto (CPU optimizado).
- **Usabilidad**: flujo de operario en ≤ 3 clics después de seleccionar lote.
- **Seguridad**: autenticación, autorización por rol, auditoría.
- **Escalabilidad funcional**: agregar nuevas verduras con retraining incremental.

---

## 3) Arquitectura propuesta (diagrama textual)

```text
[Cámara fija USB/IP]
      |
      v
[Servicio de Captura: OpenCV/GStreamer]
      |
      v
[Backend API: FastAPI]
   |           \
   |            \-> [Servicio de Inferencia IA: YOLOv8/YOLOv8-seg + clasificador calibre]
   |
   +--> [PostgreSQL]
   +--> [Almacenamiento imágenes: disco local/NAS/S3]
   +--> [Dashboard BI: Metabase o módulo web KPI]
```

### Componentes sugeridos
- **Captura**: Python + OpenCV (simple y económico), GStreamer si cámara IP RTSP.
- **Backend**: FastAPI (rápido, tipado, docs OpenAPI).
- **Inferencia**:
  - detección defectos: YOLOv8 (detect) o YOLOv8-seg.
  - calibre: regla basada en bounding box + escala ArUco (px->cm).
  - calidad final: reglas de negocio sobre defectos + tamaño + confianza.
- **DB**: PostgreSQL.
- **Dashboard**: Metabase conectado a PostgreSQL.
- **Storage**: local/NAS en MVP; S3 compatible si crece.

---

## 4) Diseño de base de datos (tablas y campos)

### `users`
- `id` (PK)
- `username` (unique)
- `password_hash`
- `role` (operator, supervisor, admin)
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
- `variety` (ej. saladette)
- `caliber_schema` (JSON: rangos S/M/L en cm)
- `created_at`

### `lots`
- `id` (PK)
- `lot_code` (unique)
- `supplier_id` (FK)
- `product_id` (FK)
- `harvest_date`
- `received_at`
- `status` (open/closed)
- `created_by` (FK users)

### `inspections`
- `id` (PK)
- `lot_id` (FK)
- `inspected_at`
- `operator_id` (FK users)
- `quality_class` (A/B/C)
- `caliber` (S/M/L)
- `overall_confidence` (0..1)
- `manual_override` (bool)
- `override_reason` (nullable)
- `notes`

### `defect_types`
- `id` (PK)
- `code` (spot, bruise, rot_crack)
- `name`
- `severity_rule` (JSON)

### `inspection_defects`
- `id` (PK)
- `inspection_id` (FK)
- `defect_type_id` (FK)
- `confidence`
- `bbox_x`, `bbox_y`, `bbox_w`, `bbox_h` (nullable si segmentación)
- `mask_path` (nullable)
- `severity` (low/medium/high)

### `inspection_images`
- `id` (PK)
- `inspection_id` (FK)
- `original_path`
- `annotated_path`
- `thumbnail_path`
- `checksum`
- `captured_device`
- `created_at`

### `audit_logs`
- `id` (PK)
- `user_id` (FK)
- `action` (login, create_lot, inspect, override, export_report)
- `entity`
- `entity_id`
- `payload` (JSON)
- `created_at`

---

## 5) Flujo completo de usuario

1. Operario inicia sesión.
2. Selecciona producto (tomate) y lote activo.
3. Coloca tomate en marca de posición de la estación.
4. Presiona “Capturar”.
5. Sistema toma imagen desde cámara fija.
6. Backend envía imagen a inferencia.
7. IA devuelve defectos + calibre + clase sugerida A/B/C.
8. UI muestra resultado con imagen anotada y confianza.
9. Operario guarda; supervisor puede corregir en casos límite.
10. Registro queda trazable y disponible en historial/reportes.

---

## 6) Modelo de IA recomendado

### Clasificación vs detección vs segmentación
- **Clasificación pura** (A/B/C): simple, pero no explica dónde está el defecto.
- **Detección**: identifica defectos y su ubicación (boxes), mejor explicabilidad para auditoría.
- **Segmentación**: más precisa para área afectada, pero más costosa de etiquetar.

### Recomendación MVP
1. **YOLOv8 detect** para defectos (más rápido de implementar con dataset pequeño).
2. Regla geométrica para calibre S/M/L con marcador ArUco para escala.
3. Motor de reglas para calidad final A/B/C usando defectos + tamaño.

> Si el equipo logra buen etiquetado temprano, migrar a **YOLOv8-seg** en iteración 2 para cuantificar porcentaje de área dañada.

### Métricas objetivo MVP
- Clasificación A/B/C: Accuracy y matriz de confusión.
- Defectos: precision, recall, mAP@50 y mAP@50-95.
- Calibre: exactitud de clase S/M/L y error medio de diámetro estimado (cm).
- Operación: latencia promedio por inspección.

### Umbral de confianza
- Sugerido inicial: `0.45–0.55` para detección.
- Política: si confianza global < umbral mínimo (ej. 0.6), marcar como “revisión supervisor”.

---

## 7) Plan de dataset y etiquetado (300–800 imágenes)

### Captura consistente
- Cámara fija a distancia constante (ej. 40–60 cm).
- Fondo mate uniforme (negro o blanco según contraste del tomate).
- Luz LED difusa constante (evitar sombras duras).
- Marca física de posición + guía de orientación.
- Incluir **marcador ArUco** o tarjeta patrón para px->cm.

### Distribución sugerida de dataset
- 50% muestras buenas (A), 30% medias (B), 20% rechazo (C).
- Defectos mínimos por clase: manchas, magulladuras, podrido/rajadura.
- Variar lotes/proveedores para robustez.

### Etiquetado
- Herramientas: LabelImg (detección), Label Studio o Roboflow.
- Etiquetas:
  - `spot`
  - `bruise`
  - `rot_crack`
  - `tomato` (opcional si se detecta objeto principal)
- Split sugerido: 70/15/15 (train/val/test) por lote para evitar fuga de datos.

### Desbalance y validación
- Augmentations moderadas: brillo/contraste, rotación leve, blur suave.
- Oversampling en defectos raros.
- Validación cruzada por lote/proveedor para medir generalización real.

---

## 8) API mínima

### `POST /inspect`
Entrada:
- imagen (multipart o path capturado)
- `lot_code`, `supplier_id`, `product_id`, `operator_id`, `timestamp`

Salida JSON (ejemplo):
```json
{
  "inspection_id": 1021,
  "quality_class": "B",
  "caliber": "M",
  "overall_confidence": 0.87,
  "defects": [
    {"type": "spot", "confidence": 0.81, "bbox": [120, 80, 40, 35]},
    {"type": "bruise", "confidence": 0.74, "bbox": [200, 140, 60, 50]}
  ],
  "original_image_path": "images/2026/lot123/orig_1021.jpg",
  "annotated_image_path": "images/2026/lot123/ann_1021.jpg"
}
```

### `GET /inspections?filters`
- filtros: fecha desde/hasta, lote, proveedor, calidad, defecto, operador.

### `GET /reports`
- resumen por fecha/proveedor/lote:
  - total inspecciones
  - %A, %B, %C
  - defectos más frecuentes
  - tendencia semanal rechazo

---

## 9) Pantallas MVP

1. **Login** (usuario/clave).
2. **Nueva inspección** (selección lote + captura).
3. **Resultado** (imagen marcada, defectos, calidad, calibre, confianza, guardar/corregir).
4. **Historial por lote** (tabla + filtros + detalle).
5. **Dashboard KPI**:
   - tasa de rechazo (%)
   - defectos frecuentes
   - ranking por proveedor/finca
   - tendencia semanal/mensual

---

## 10) Reglas de negocio (ejemplo inicial)

1. Si `rot_crack` con severidad media/alta => **C automático**.
2. Si solo `spot` leve y calibre en rango comercial => **B**.
3. Si sin defectos relevantes y calibre válido => **A**.
4. Si múltiples defectos moderados (2+) => bajar una categoría.
5. Si confianza global baja => estado “pendiente revisión”.

> Reglas deben versionarse (`ruleset_version`) para trazabilidad histórica.

---

## 11) Seguridad y auditoría

### Roles
- **Operario**: capturar y registrar inspecciones.
- **Supervisor**: aprobar/corregir clasificaciones, validar casos de baja confianza.
- **Admin**: gestión usuarios, catálogos, configuración cámara/modelo.

### Controles
- JWT o sesión segura + hash de contraseñas (bcrypt/argon2).
- Bitácora de acciones en `audit_logs`.
- Firma de tiempo y checksum de imagen para integridad.
- Backups diarios de DB + almacenamiento de imágenes.

---

## 12) Plan de implementación (8 semanas)

### Semana 1
- Diseño detallado, criterios de calidad, definición de defectos y reglas.
- Montaje físico estación (cámara + luz + fondo + marca).

### Semana 2
- Prototipo captura OpenCV + almacenamiento imagen.
- Diseño BD y API base.

### Semana 3
- Recolección dataset piloto (150–250 imágenes).
- Guía de etiquetado y control de calidad de etiquetas.

### Semana 4
- Etiquetado + entrenamiento baseline YOLOv8 detect.
- Métricas iniciales y ajustes de umbral.

### Semana 5
- Integración end-to-end (captura -> inferencia -> guardar).
- Implementación reglas A/B/C + calibre S/M/L con ArUco.

### Semana 6
- UI web MVP (login, inspección, resultado, historial).
- Auditoría y roles.

### Semana 7
- Dashboard KPI + reportes.
- Pruebas con datos reales de bodega.

### Semana 8
- Hardening, documentación técnica y manual de operación.
- Presentación de resultados (métricas + limitaciones + roadmap).

---

## 13) Riesgos y mitigación

- **Variación de iluminación** -> caja de luz/hood + exposición fija cámara.
- **Suciedad o ruido en fondo** -> protocolo de limpieza por turno.
- **Variación de ángulos** -> guía física de colocación del fruto.
- **Dataset pequeño y sesgo por proveedor** -> muestreo balanceado multi-lote.
- **Sobreajuste** -> validación por lote, augmentations moderadas.
- **Cambios estacionales del producto** -> retraining mensual/trimestral.

---

## 14) Stack final + despliegue

### Opción A: Local en PC de bodega (recomendada MVP)
- **Backend + inferencia**: Python FastAPI + Ultralytics YOLOv8.
- **DB**: PostgreSQL local.
- **Storage**: disco local/NAS.
- **Dashboard**: Metabase local.
- **Ventaja**: funciona sin internet, bajo costo, rápida adopción.

### Opción B: Híbrido/Nube
- Captura local + API/inferencia en nube.
- Requiere conectividad estable y control de latencia.
- Útil para múltiples plantas y analítica centralizada.

### Requisitos mínimos hardware
- **CPU only (MVP básico)**:
  - Intel i5/Ryzen 5 (6 núcleos), 16 GB RAM, SSD 512 GB.
  - ~1–2 s por inferencia (dependiendo modelo/resolución).
- **Con GPU (recomendado si volumen alto)**:
  - NVIDIA RTX 3060/4060 (8 GB VRAM) o superior.
  - inferencia < 300 ms aprox en lotes optimizados.
- Cámara: USB 1080p o IP cam 2MP con foco fijo.
- Iluminación: panel LED difuso 5500K constante.

---

## Propuesta técnica defendible para proyecto de graduación

Este MVP es defendible académicamente porque combina:
1. Ingeniería de software (arquitectura por servicios, API, DB, seguridad).
2. Visión por computador aplicada a industria alimentaria.
3. Evaluación cuantitativa (métricas IA + KPI operativos).
4. Impacto de negocio (reducción de subjetividad y mejor trazabilidad).

**Evolución posterior**: multi-producto, segmentación precisa de área dañada, integración con ERP y tableros corporativos.
