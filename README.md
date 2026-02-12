# VotaCol – Flujo del sistema (versión simple para GitHub)

Este README describe el flujo general de **VotaCol**, separando claramente:
1) **Validación de identidad (jurado + sistema)**  
2) **Votación (cabina/pantalla asignada)**  
3) **Auditoría + comprobante**  
4) **Conteo automatizado de votos por candidato**

> Nota: El diseño busca **mantener el secreto del voto** (no se debe poder relacionar “persona → candidato”).

---

## 1. Objetivo del flujo

- Verificar que la persona **puede votar** (simulando Registraduría).
- Permitir el voto en una cabina/pantalla asignada.
- Guardar evidencia/auditoría del proceso **sin revelar por quién votó**.
- Generar comprobante (impreso o por correo) que confirme que el voto quedó registrado.
- Generar el **conteo automático por candidato** en tiempo real (o al cierre).

---

## 2. Stack propuesto (simple y justificable)

> El stack puede cambiar, pero esta combinación es fácil para un proyecto académico y muy común en apps web.

### Opción A 
- **Backend:** FastApi o Django
- **Base de datos:** PostgreSQL
- **Frontend:** React 
- **Almacenamiento de archivos:** carpeta local
- **Autenticación:** JWT (solo para jurados/admin)

**¿Por qué?**
- Mucha documentación y ejemplos.
- Rápido de desarrollar.
- PostgreSQL es fuerte para consultas y reportes (conteos).
- React ayuda a hacer la pantalla de votación más ordenada.

## 3. Bases de datos a utilizar

1. **DB principal (VotaCol)**
   - Sesiones
   - Votos (sin identidad)
   - Auditoría
   - Rutas/hash de archivos (fotos/videos)

2. **DB simulada (Registraduría)**
   - Cédulas habilitadas/no habilitadas
   - Estado `ya_voto` para evitar doble voto

---

## 4. Flujo completo paso a paso

### Fase A — Validación en mesa (jurado)
1. La persona se acerca al jurado y entrega la cédula.
2. El jurado revisa visualmente:
   - que la cédula sea original
   - que la cara coincida
3. El sistema toma:
   - foto de la cédula
   - foto de la persona
4. El sistema valida contra la **Registraduría simulada**:
   - existe / no existe
   - habilitado / no habilitado
   - ya votó / no ha votado
5. Si **NO pasa** la validación:
   - se registra un evento de auditoría (“rechazado”)
6. Si **SÍ pasa** la validación:
   - se crea una **sesión de votación** (ID de sesión)
   - se asigna una cabina/pantalla disponible
   - se marca en registraduría simulada: `ya_voto = true` (o “en_proceso”)

Resultado: la persona queda autorizada y con una sesión asignada.

---

### Fase B — Votación en cabina/pantalla
1. La persona llega a la cabina asignada.
2. Aparece una pantalla inicial con instrucciones + conteo regresivo:
   - `5...4...3...2...1`
3. Se muestran los candidatos.
4. La persona selecciona su candidato y confirma.
5. El sistema registra el voto usando el **ID de sesión** (NO la cédula).
6. La sesión se cierra y la cabina queda lista para el siguiente usuario.

Resultado: voto guardado sin asociarlo a identidad.

---

### Fase C — Auditoría + comprobante
1. El sistema guarda auditoría del proceso como eventos:
   - “sesión creada”
   - “cabina asignada”
   - “pantalla de candidatos cargada”
   - “voto confirmado”
   - “sesión cerrada”
2. Para evitar manipulación de auditorías, cada evento guarda:
   - `hash_actual = SHA256(datos_evento + hash_evento_anterior)`
3. Se genera comprobante:
   - impreso o enviado por correo
   - incluye: `ID_sesion`, fecha/hora, y/o un QR para verificación
   - **NO incluye el candidato**

Resultado: trazabilidad sin revelar por quién votó.

---

## 5. Conteo automatizado de votos por candidato

El conteo se hace automáticamente con consultas SQL sobre la tabla `votes`.

