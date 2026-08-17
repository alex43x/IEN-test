# PLAN DE IMPLEMENTACIÓN — Corrección de Hallazgos de Auditoría IEN

## Contexto del proyecto
- Monorepo único (un solo `.git` para `back/` y `frontend/`)
- Branch: `main`, working tree limpio
- Backend: Node.js/Express/MongoDB/Mongoose en `back/`
- Frontend: React/Vite en `frontend/`
- 139 tests backend que pasan; `npm run build` del frontend compila sin errores

## Reglas generales
- No romper los 139 tests de backend ni el build de frontend
- Después de cada fix (o grupo de fixes relacionados), correr tests relevantes
- No refactorizar de más: cada item tiene alcance acotado
- Commits atómicos: un commit por item (o grupo de items triviales del mismo archivo)
- **No se toca frontend** — todos los items son backend-only

---

## Correcciones al reporte original de auditoría

1. **Swagger "30 días": 2 ocurrencias, no 3.** Líneas 120 (register) y 170 (login) de `auth.routes.js`. La respuesta de `/refresh` (línea 213) no tiene campo `description`, solo `type: string`.
2. **`retreatDay` sin días completados devuelve 409**, no 400 ni 404. Código real en `plan.service.js:475-476`: `throw new AppError(409, 'No hay días completados para retroceder')`.
3. **`codigo.controller.js` tiene GET listar (líneas 10-25)**: son 4 usos del patrón `_id`→`id`, no 3 como dijo el plan original.

---

## FASE A — Fixes atómicos, mínimo riesgo (items 11, 12, 13, 14, 15, 16, 17, 21)

### Item 11 — Unificar bcrypt salt rounds a 12
**Archivo:** `back/src/modules/auth/auth.service.js`
- Línea 70: `bcrypt.hash(password, 10)` → `bcrypt.hash(password, 12)`
- Línea 229: `bcrypt.hash(nuevaPassword, 10)` → `bcrypt.hash(nuevaPassword, 12)`
- Línea 255 ya usa 12 — no se toca
- Riesgo verificado: los tests de register y reset-password en `auth.test.js` (líneas 51 y 290) sí ejecutan estas funciones vía HTTP. El cambio solo hace los tests ~2-3s más lentos; 10 y 12 rounds producen hashes bcrypt válidos. No rompe nada.

### Item 12 — Renombrar variable local `enScope` → `estaEnScope`
**Archivo:** `back/src/modules/admin/admin.service.js`
- Línea 331: `const enScope = (creador.tiendas_administradas || [])` → `const estaEnScope = (creador.tiendas_administradas || [])`
- Línea 333: `if (!enScope) {` → `if (!estaEnScope) {`

### Item 13 — jwt.verify con algoritmos explícitos
**Archivo:** `back/src/middlewares/authMiddleware.js`, línea 14
```diff
- req.usuario = jwt.verify(token, process.env.JWT_SECRET);
+ req.usuario = jwt.verify(token, process.env.JWT_SECRET, { algorithms: ['HS256'] });
```

### Item 14 — authLimiter de 10 a 5
**Archivo:** `back/src/modules/auth/auth.routes.js`, línea 13
```diff
-   max: 10,
+   max: 5,
```

### Item 15 — Rate limiter en /logout
**Archivo:** `back/src/modules/auth/auth.routes.js`, línea 252
```diff
- router.post('/logout', logout);
+ router.post('/logout', authLimiter, logout);
```

### Item 16 — enScope() null-safe
**Archivo:** `back/src/utils/scope.js`, líneas 1-4
```diff
 function enScope(id, tiendasPermitidas) {
   if (!tiendasPermitidas) return true;
+  if (id == null) return false;
-  return tiendasPermitidas.some(t => t.toString() === id.toString());
+  return tiendasPermitidas.some(t => t != null && t.toString() === id.toString());
 }
```

### Item 17 — console.error condicional en producción
**Archivo A:** `back/src/middlewares/errorHandler.js`, línea 34
```diff
- console.error(err);
+ if (process.env.NODE_ENV !== 'production') console.error(err);
+ else console.error('[ErrorHandler]', err.message || 'Error desconocido');
```

**Archivo B:** `back/src/server.js`, línea 9
```diff
-   console.error('Unhandled Rejection:', err);
+   if (process.env.NODE_ENV !== 'production') {
+     console.error('Unhandled Rejection:', err);
+   } else {
+     console.error('Unhandled Rejection:', err.message);
+   }
```

### Item 21 — Renombrar moderadorMiddleware.js → roleMiddleware.js
1. Renombrar archivo: `back/src/middlewares/moderadorMiddleware.js` → `back/src/middlewares/roleMiddleware.js`
2. `back/src/modules/admin/admin.routes.js:5`: cambiar import a `'../../middlewares/roleMiddleware'`
3. `back/src/modules/tiendas/tienda.routes.js:5`: cambiar import a `'../../middlewares/roleMiddleware'`
- Solo 2 imports afectados (confirmado con grep)

**Verificación post Fase A:** `npm test` en `back/` — deben seguir pasando 139 tests.

---

## FASE B — Índices MongoDB (items 2, 3, 5)

### Item 2 — Índice `racha_rota_en` en PlanProgreso
**Archivo:** `back/src/models/PlanProgreso.js`
- Agregar después de línea 62 (antes del `module.exports`):
```js
planProgresoSchema.index({ racha_rota_en: 1 });
```
- Evidencia: la única query que escribe este campo es `cronJobs.js:60-61` (`$set: { racha_rota_en: new Date() }`) dentro de un `updateMany` cuyo filtro ya usa índices existentes. No hay queries que filtren/sorteen por este campo hoy → índice simple preventivo, no compuesto.

### Item 3 — Índice `usuario_id` en RefreshToken
**Archivo:** `back/src/models/RefreshToken.js`
- Agregar después de línea 13:
```js
refreshTokenSchema.index({ usuario_id: 1 });
```
- Evidencia: `auth.service.js:233,259` hacen `RefreshToken.updateMany({ usuario_id, revocado: false }, ...)` → collection scan actual sin índice.

### Item 5 — Verificación de aplicación de índices
- El proyecto usa `mongoose.connect(MONGO_URI)` en `server.js:17` sin `autoIndex: false` → Mongoose crea índices automáticamente al conectar.
- Verificación: levantar servidor y revisar que no haya errores de índice. Alternativa: los tests de integración usan `connect()` → los modelos se registran → índices se crean solos.

---

## FASE C — EMAIL_FROM + Swagger (items 4, 6, 8)

### Item 4 — EMAIL_FROM sin fallback silencioso
**Archivo:** `back/src/modules/email/email.service.js`, líneas 26-28
```diff
   try {
+    if (!process.env.EMAIL_FROM) {
+        console.error('[CRITICAL] EMAIL_FROM sin configurar — no se puede enviar correo');
+        await registrarHistorial({ usuario_id, destinatario, tipo_correo, estado: 'fallido', meta: { error: 'EMAIL_FROM no configurado' } });
+        return { success: false, error: 'EMAIL_FROM no configurado' };
+    }
     const resend = new Resend(process.env.RESEND_API_KEY);
-    const from = process.env.EMAIL_FROM || 'onboarding@resend.dev';
+    const from = process.env.EMAIL_FROM;
```

**Verificación adicional hecha:**
- `registrarHistorial` está definida en la línea 4 del mismo archivo → en scope, sin ReferenceError.
- Los 4 jobs (enviarEnLote) en `email.service.js:77` SÍ chequean `resultado.success`.
- Los 3 callers directos (bienvenida en `auth.service.js:86`, recuperación en `:176`, hito en `plan.service.js:409`) NO chequean `success` (fire-and-forget con `.catch`). Esto es un **hallazgo nuevo**, fuera de alcance para esta tarea. El fix mejora la situación (ahora se registra en HistorialCorreo con estado 'fallido') pero esos 3 callers no se enteran del fallo. Se documenta para tarea futura.

### Item 6 — Swagger docs faltantes

#### 6a. `POST /api/plan/testing/retreat`
**Archivo:** `back/src/modules/plan/plan.routes.js` — insertar bloque antes de línea 470:
```js
/**
 * @swagger
 * /api/plan/testing/retreat:
 *   post:
 *     summary: "[DEV] Retroceder al día anterior deshaciendo el último día completado"
 *     tags: [Plan]
 *     security:
 *       - bearerAuth: []
 *     responses:
 *       200:
 *         description: Día retrocedido
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 dia_retrocedido:
 *                   type: number
 *                 dia_actual:
 *                   type: number
 *                 racha_dias:
 *                   type: number
 *                 racha_maxima:
 *                   type: number
 *                 estado:
 *                   type: string
 *       404:
 *         description: No hay plan activo
 *       409:
 *         description: No hay días completados para retroceder
 */
```
- Verificado: 404 real en `plan.service.js:469` (`if (!plan) throw new AppError(404, ...)`) y 409 real en `:475-476`.

#### 6b. `PATCH /api/admin/sucursales/:id/reactivar`
**Archivo:** `back/src/modules/tiendas/tienda.routes.js` — insertar antes de línea 132:
```js
/**
 * @swagger
 * /api/admin/sucursales/{id}/reactivar:
 *   patch:
 *     summary: "[ADMIN] Reactivar sucursal desactivada"
 *     tags: [Admin - Tiendas]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *     responses:
 *       200:
 *         description: Sucursal reactivada
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 mensaje:
 *                   type: string
 *                 tienda:
 *                   type: object
 *       403:
 *         description: Fuera de scope o sin permisos
 *       404:
 *         description: Sucursal no encontrada
 */
```

### Item 8 — Swagger refresh_token: "30 días" → "2 días"
**Archivo:** `back/src/modules/auth/auth.routes.js`
- Línea 120: `Token de refresco (30 días de expiración` → `Token de refresco (2 días de expiración`
- Línea 170: `Token de refresco (30 días de expiración` → `Token de refresco (2 días de expiración`
- Solo 2 ocurrencias confirmadas (el refresh Swagger en línea 213 no tiene campo `description`)

---

## FASE D — Unificación de respuesta JSON + campos_respuesta (items 9, 10)

### Item 9 — Unificar `_id` → `id` + extraer util `toResponse`

#### 9a. Crear util compartido
**Nuevo archivo:** `back/src/utils/toResponse.js`
```js
function toResponse(doc) {
  const obj = typeof doc.toObject === 'function' ? doc.toObject() : doc;
  const { _id, ...rest } = obj;
  return { id: _id, ...rest };
}

module.exports = { toResponse };
```

#### 9b. Reemplazar en `producto.controller.js`
Agregar import: `const { toResponse } = require('../../utils/toResponse');`

| Línea | Actual | Nuevo |
|-------|--------|-------|
| 16 | `res.json(productos);` | `res.json(productos.map(toResponse));` |
| 31-32 | `const { _id, ...prest } = producto.toObject(); res.status(201).json({ id: _id, ...prest });` | `res.status(201).json(toResponse(producto));` |
| 56-57 | `const { _id: pid, ...prest2 } = producto.toObject(); res.json({ id: pid, ...prest2 });` | `res.json(toResponse(producto));` |

#### 9c. Reemplazar en `tienda.controller.js`
Agregar import: `const { toResponse } = require('../../utils/toResponse');`

| Línea | Actual | Nuevo |
|-------|--------|-------|
| 20 | `res.json(tiendas);` (usa `.lean()` → plain objects) | `res.json(tiendas.map(toResponse));` |
| 35-36 | `const { _id, ...rest } = tienda.toObject(); res.status(201).json({ id: _id, ...rest });` | `res.status(201).json(toResponse(tienda));` |
| 57-58 | `const { _id: tid, ...trest } = tienda.toObject(); res.json({ id: tid, ...trest });` | `res.json(toResponse(tienda));` |
| 70-71 | `const { _id: eid, ...erest } = tienda.toObject(); res.json({ mensaje: 'Sucursal desactivada', tienda: { id: eid, ...erest } });` | `res.json({ mensaje: 'Sucursal desactivada', tienda: toResponse(tienda) });` |
| 83-84 | `const { _id: rid, ...rrest } = tienda.toObject(); res.json({ mensaje: 'Sucursal reactivada', tienda: { id: rid, ...rrest } });` | `res.json({ mensaje: 'Sucursal reactivada', tienda: toResponse(tienda) });` |

#### 9d. Reemplazar en `codigo.controller.js`
Agregar import: `const { toResponse } = require('../../utils/toResponse');`

| Línea | Actual | Nuevo |
|-------|--------|-------|
| 19-23 | `codigos.map(c => { const obj = c.toObject(); const { _id, ...rest } = obj; return { id: _id, ...rest }; })` | `codigos.map(toResponse)` |
| 47-48 | `const { _id: cid, ...crest } = doc.toObject(); res.status(201).json({ id: cid, ...crest });` | `res.status(201).json(toResponse(doc));` |
| 62-63 | `const { _id: aid, ...arest } = doc.toObject(); res.json({ mensaje: 'Código activado', codigo: { id: aid, ...arest } });` | `res.json({ mensaje: 'Código activado', codigo: toResponse(doc) });` |
| 77-78 | `const { _id: did, ...drest } = doc.toObject(); res.json({ mensaje: 'Código desactivado', codigo: { id: did, ...drest } });` | `res.json({ mensaje: 'Código desactivado', codigo: toResponse(doc) });` |

**CHECKPOINT CRÍTICO:** Inmediatamente después de esta fase, correr `npm run build` en `frontend/`. Este item cambia el shape de respuesta JSON que consume el frontend. Si hay tipos rotos, aislarlo aquí antes de seguir.

### Item 10 — Extraer `campos_respuesta` a función compartida

#### 10a. Crear util compartido
**Nuevo archivo:** `back/src/utils/camposRespuesta.js`
```js
function mapearCamposRespuesta(pasos) {
  return Array.isArray(pasos)
    ? pasos
        .filter(p => p.respuesta_tipo !== 'accion' || p.texto)
        .map((p, i) => ({
          id: p.id || `paso_${i + 1}`,
          etiqueta: (typeof p.texto === 'string' ? p.texto : `Paso ${i + 1}`).substring(0, 300),
          tipo: p.respuesta_tipo === 'escala' ? 'escala'
            : p.respuesta_tipo === 'accion' ? 'accion'
            : 'texto',
          min: p.min,
          max: p.max
        }))
    : [];
}

module.exports = { mapearCamposRespuesta };
```

#### 10b. Reemplazar en `plan.service.js`
- Agregar import: `const { mapearCamposRespuesta } = require('../../utils/camposRespuesta');`
- Líneas 43-55: reemplazar todo el bloque `campos_respuesta` por `const campos_respuesta = mapearCamposRespuesta(pasos);`

#### 10c. Reemplazar en `admin.service.js`
- Agregar import: `const { mapearCamposRespuesta } = require('../../utils/camposRespuesta');`
- Líneas 140-152: reemplazar todo el bloque `campos_respuesta` por `const campos_respuesta = mapearCamposRespuesta(pasos);`
- Esto unifica `substring(0, 80)` a `substring(0, 300)` — el commit `aaa5e59` lo actualizó a 300 en plan.service.js pero no actualizó el código duplicado en admin.

---

## FASE E — Lógica de negocio (items 1, 20)

### Item 1 — Validación de Tienda en Producto.crear/actualizar + tests

**Archivo:** `back/src/modules/productos/producto.controller.js`

**Agregar import** al inicio:
```js
const Tienda = require('../../models/Tienda');
```

**En `crear` — insertar entre línea 28 y 30 actual:**
```js
  const tiendaExiste = await Tienda.findById(tienda_id).select('_id').lean();
  if (!tiendaExiste) throw new AppError(400, 'La tienda indicada no existe');
```

**En `actualizar` — línea 53 actual pasa a:**
```js
  if (tienda_id !== undefined) {
      const tiendaExiste = await Tienda.findById(tienda_id).select('_id').lean();
      if (!tiendaExiste) throw new AppError(400, 'La tienda indicada no existe');
      producto.tienda_id = tienda_id;
  }
```
- Alcance acotado: solo valida que la tienda exista. No se agrega chequeo de `activo` sin confirmación de producto.

**Tests en** `back/tests/productos.test.js` — agregar en bloque `describe('Productos - admin_general', ...)` (antes de línea 76):
```js
  test('POST /api/admin/productos - tienda_id inexistente → 400', async () => {
      const fakeId = '507f1f77bcf86cd799439011';
      const res = await request(app)
        .post('/api/admin/productos')
        .set('Authorization', `Bearer ${token}`)
        .send({ nombre: 'Test', tienda_id: fakeId });
      expect(res.status).toBe(400);
      expect(res.body.error).toBe('La tienda indicada no existe');
  });

  test('PUT /api/admin/productos/:id - cambiar a tienda_id inexistente → 400', async () => {
      const fakeId = '507f1f77bcf86cd799439011';
      const res = await request(app)
        .put(`/api/admin/productos/${data.producto1Id}`)
        .set('Authorization', `Bearer ${token}`)
        .send({ tienda_id: fakeId });
      expect(res.status).toBe(400);
      expect(res.body.error).toBe('La tienda indicada no existe');
  });
```

### Item 20 — Test del hito 28

**Archivo:** `back/tests/hitos.test.js`

Línea 4 — corregir comentario:
```diff
- // --- hitos que SÍ existen en HITOS_RACHA = [7, 14, 21] ---
+ // --- hitos que SÍ existen en HITOS_RACHA = [7, 14, 21, 28] ---
```

Agregar después de línea 16 (test de hito 21):
```js
  test('racha_dias = 28 que no está en hitos_alcanzados devuelve 28', () => {
      expect(detectarHito(28, [7, 14, 21])).toBe(28);
  });
```

Agregar después de línea 30 (test de deduplicación de hito 21):
```js
  test('racha_dias = 28 ya alcanzado devuelve null', () => {
      expect(detectarHito(28, [7, 14, 21, 28])).toBeNull();
  });
```

---

## FASE F — Tests nuevos + seed (items 7, 18)

### Item 7 — Tests para 3 endpoints sin cobertura

#### 7a. `POST /api/plan/testing/retreat`
**Archivo:** `back/tests/plan.test.js` — agregar en bloque `describe('Plan - testing endpoints', ...)` (después de línea 194):

```js
  test('POST /api/plan/testing/retreat - success', async () => {
      const respuestas = Array.from({ length: data.totalPreguntas }, (_, i) => ({
          numero: i + 1, score: 3
      }));
      await request(app)
        .post('/api/plan/setup-test')
        .set('Authorization', `Bearer ${token}`)
        .send({ respuestas });

      await request(app)
        .post('/api/plan/testing/advance')
        .set('Authorization', `Bearer ${token}`);

      const res = await request(app)
        .post('/api/plan/testing/retreat')
        .set('Authorization', `Bearer ${token}`);
      expect(res.status).toBe(200);
      expect(res.body.dia_retrocedido).toBeDefined();
      expect(res.body.dia_actual).toBeDefined();
  });

  test('POST /api/plan/testing/retreat - 409 sin días completados', async () => {
      const respuestas = Array.from({ length: data.totalPreguntas }, (_, i) => ({
          numero: i + 1, score: 3
      }));
      await request(app)
        .post('/api/plan/setup-test')
        .set('Authorization', `Bearer ${token}`)
        .send({ respuestas });

      const res = await request(app)
        .post('/api/plan/testing/retreat')
        .set('Authorization', `Bearer ${token}`);
      expect(res.status).toBe(409);
  });
```
- Verificado: `retreatDay` (`plan.service.js:475-476`) lanza `AppError(409, 'No hay días completados para retroceder')` — test usa exactamente 409.

#### 7b. `POST /api/auth/change-password`
**Archivo:** `back/tests/auth.test.js` — agregar nuevo bloque `describe` antes del cierre del archivo:

```js
describe('Auth - change-password', () => {
    let data, token;
    beforeEach(async () => {
        data = await seed();
        token = generateToken(data.usuario);
    });

    test('POST /api/auth/change-password - success', async () => {
        const res = await request(app)
          .post('/api/auth/change-password')
          .set('Authorization', `Bearer ${token}`)
          .send({ current_password: 'user123', nueva_password: 'newpass456' });
        expect(res.status).toBe(200);
        expect(res.body.mensaje).toMatch(/actualizada/i);

        const loginRes = await request(app)
          .post('/api/auth/login')
          .send({ email: data.usuario.email, password: 'newpass456' });
        expect(loginRes.status).toBe(200);
    });

    test('POST /api/auth/change-password - contraseña actual incorrecta', async () => {
        const res = await request(app)
          .post('/api/auth/change-password')
          .set('Authorization', `Bearer ${token}`)
          .send({ current_password: 'wrongpass', nueva_password: 'newpass456' });
        expect(res.status).toBe(401);
    });

    test('POST /api/auth/change-password - campos faltantes', async () => {
        const res = await request(app)
          .post('/api/auth/change-password')
          .set('Authorization', `Bearer ${token}`)
          .send({ current_password: 'user123' });
        expect(res.status).toBe(400);
    });
});
```
- Verificado: `auth.service.js:252` lanza `AppError(401, 'La contraseña actual es incorrecta')`, controller línea 132-133 devuelve 400 para campos faltantes.

#### 7c. `PATCH /api/admin/sucursales/:id/reactivar`
**Archivo:** `back/tests/tiendas.test.js`

Agregar en bloque `Sucursales - admin_general` (antes de línea 68):
```js
  test('PATCH /api/admin/sucursales/:id/reactivar - success', async () => {
      await request(app)
        .delete(`/api/admin/sucursales/${data.tienda1Id}`)
        .set('Authorization', `Bearer ${token}`);

      const res = await request(app)
        .patch(`/api/admin/sucursales/${data.tienda1Id}/reactivar`)
        .set('Authorization', `Bearer ${token}`);
      expect(res.status).toBe(200);
      expect(res.body.mensaje).toMatch(/reactivada/i);
  });
```

Agregar en bloque `Sucursales - admin_negocio (scoped)` (antes de línea 92):
```js
  test('PATCH /api/admin/sucursales/:id/reactivar - forbidden fuera de scope', async () => {
      const res = await request(app)
        .patch(`/api/admin/sucursales/${data.tienda2Id}/reactivar`)
        .set('Authorization', `Bearer ${token}`);
      expect(res.status).toBe(403);
  });
```
- Verificado: `tienda.controller.js:84` devuelve `{ mensaje: 'Sucursal reactivada', tienda: { ... } }`. Scope check en línea 78 protege contra admin_negocio fuera de su tienda.

### Item 18 — Corregir seed data inconsistente

**Archivo:** `back/src/seed.js:1306-1321` — tabla `planesConfig`

**Hitos:** los hitos reales del sistema son `[7, 14, 21, 28]` (`plan.service.js:32`). Corrección basada en `racha_max` alcanzado:

```
idx=0:  hitos: []     → []              (racha_max=4, sin hitos)
idx=1:  hitos: [5]    → []              (racha_max=5, sin hitos)
idx=2:  hitos: [5,10] → [7]             (racha_max=8, alcanzó 7)
idx=3:  hitos: [5,10] → [7]             (racha_max=10, alcanzó 7)
idx=4:  hitos: [5,10,15] → [7,14]       (racha_max=12, alcanzó 7 y 14)
idx=5:  hitos: [5,10,15] → [7,14]       (racha_max=14, alcanzó 7 y 14)
idx=6:  hitos: [5]    → [7]             (racha_max=8, alcanzó 7 antes de abandonar)
idx=7:  hitos: [5,10,15,20,25] → [7,14] (racha_max=18, alcanzó 7 y 14, no 21)
idx=8:  hitos: [5,10,15,20,25] → [7,14] (racha_max=20, alcanzó 7 y 14, no 21)
idx=9:  hitos: [5,10,15,20,25] → [7,14,21] (racha_max=22, alcanzó 7,14,21)
idx=10: hitos: [5,10,15,20,25,30] → [7,14,21,28] (completado, racha_max=24)
idx=11: hitos: [5,10,15,20,25,30] → [7,14,21,28] (completado, racha_max=26)
idx=12: hitos: []     → []              (racha_max=2)
idx=13: hitos: [5]    → []              (abandonado, racha_max=5 < 7)
idx=14: hitos: []     → []              (racha_max=3)
```

**dia_actual para completados:**
```
idx=10: dia_actual: 30 → 31
idx=11: dia_actual: 30 → 31
```
- En runtime real, `marcarDiaCompletado` incrementa `dia_actual` de 30 a 31 antes de setear `estado: 'completado'`.

---

## VERIFICACIÓN FINAL

Al terminar todas las fases:
- [ ] `npm test` en `back/` → deben pasar 139 tests originales + 9 nuevos = **~148 tests**
- [ ] `npm run build` en `frontend/` → sin errores (también se corrió después de Fase D como checkpoint intermedio)
- [ ] Cada fix tiene su commit con mensaje descriptivo

---

## HALLAZGOS FUERA DE ALCANCE (documentados, no implementados)

1. **3 callers directos de `enviarCorreo` no chequean `success`**: `auth.service.js:86` (bienvenida), `:176` (recuperación), `plan.service.js:409` (hito). Son fire-and-forget con `.catch()` que no captura `{ success: false }`. El fix del Item 4 mejora la situación (registra en HistorialCorreo) pero esos callers no se enteran del fallo.

2. **Item 19 original (sendReminders en runDaily)**: excluido deliberadamente. `recordatorio_diario` será rediseñado como parte de funcionalidad nueva del cliente.

3. **4 templates de email con fallback hardcodeado a `https://ien.app`**: `rachaRota.js`, `recordatorioDiario.js`, `recuperacionInactividad.js`, `urgenciaActivacion.js` usan `'https://ien.app'` como tercer fallback de URL. Depende de decisión de config de entornos.

4. **Ítems informativos I1-I8 del reporte original**: observaciones no bugs.
