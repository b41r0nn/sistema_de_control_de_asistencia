# Integraciones con dispositivos biométricos

**Última revisión:** 2026-11-10

Este documento separa lo que **está implementado y funcionando** de lo que **podría funcionar pero no se ha verificado**. La separación no es formal: nada entra en la primera lista sin haberse probado contra un aparato real.

---

## 1. Implementada

### Hikvision — ISAPI sobre HTTP

| | |
| --- | --- |
| **Estado** | ✅ **Verificada en producción** |
| **Protocolo** | ISAPI, API HTTP propia de Hikvision |
| **Sentido** | El servidor consulta al aparato (*pull*) |
| **Código** | `extractor_hikvision.py`, `backend/sync_empleados.py`, `backend/device_config.py` |
| **Probada contra** | El biométrico de REDIHOS, en explotación diaria |

Es la única integración que el producto declara soportar hoy. Extrae marcaciones por rango de fechas, sincroniza empleados y lleva meses en uso real.

**Limitación conocida:** la dirección y las credenciales están hoy fijadas como constantes de módulo en `backend/device_config.py`, leídas del entorno al importar. Un solo aparato, fijado al arrancar el proceso. Resolverlo es el objeto del plan de implementación.

---

## 2. Posibles, no verificadas

Ninguna de estas está implementada. Todas comparten la misma duda de fondo: **no hay aparato contra el que probarlas**. Se enumeran con la duda concreta de cada una, para que nadie las dé por soportadas.

### ZKTeco — empuje (ADMS / Push SDK)

**Por qué es la candidata más fuerte.** El aparato no espera a que lo consulten: hace `POST` a `/iclock/cdata` contra una URL que se le configura, y consulta `/iclock/getrequest` para recibir órdenes. Como el equipo inicia la conexión, **no hace falta IP fija ni abrir puertos** en la red del cliente. Es además la marca más extendida en Latinoamérica.

**Dudas:**
- No se encontró especificación oficial completa del formato. Lo que hay son implementaciones de terceros coincidentes entre sí, y un PDF introductorio de ZKTeco de 2012 que no detalla el formato de los campos.
- **Variantes de ruta según firmware:** hay modelos que piden `/iclock/cdata.aspx` en lugar de `/iclock/cdata`. Cuántas variantes existen, se desconoce.
- Muchos firmwares solo hablan **HTTP sin cifrar** hacia el servidor.
- Sin aparato, no se puede confirmar el formato exacto de `ATTLOG` ni las respuestas que el equipo espera.

### ZKTeco — consulta (puerto 4370)

**Por qué podría funcionar.** Existe `pyzk`, librería de Python que implementa el protocolo propio sobre TCP/4370 y expone `get_attendance()`.

**Dudas:**
- `pyzk` **no es oficial**; es un proyecto de la comunidad construido por ingeniería inversa.
- Su propia documentación advierte que la compatibilidad varía: algunos equipos usan protocolos distintos.
- Hay casos documentados de modelos que responden al ping y al puerto, pero agotan el tiempo al pedir datos.
- Sin aparato, no se puede saber qué modelos y firmwares responden.

### ESSL

**Por qué podría funcionar.** Son equipos de base ZK, así que en principio responden al mismo protocolo del puerto 4370.

**Dudas:**
- No existe protocolo publicado propio.
- Es revelador que varios integradores no hablen con el aparato: leen directamente la **base de datos SQL Server del software de ESSL**. Eso sugiere que la vía por protocolo no les resultó fiable.
- Cuánto se parece cada modelo a un ZKTeco estándar, se desconoce.

### Dahua

**Por qué podría funcionar.** Los equipos de control de acceso y asistencia exponen una API CGI sobre HTTP, y las fichas de producto la mencionan entre las opciones de interoperabilidad.

**Dudas:**
- **La documentación pública es pobre.** Haría falta el PDF oficial `DAHUA_HTTP_API` correspondiente a la versión de firmware concreta.
- En foros hay consultas sin respuesta sobre qué tabla de `recordFinder` contiene la correspondencia entre tarjeta y nombre, y llamadas que devuelven `400 Bad Request` según firmware.
- Hay informes de suscripciones a eventos que tardan un minuto o agotan el tiempo.
- Es la que más trabajo de investigación exige, aunque **no hay motivo para creer que sea inviable**.

### Suprema

**Por qué podría funcionar.** Tiene API REST documentada de verdad: `POST /api/events/search` para consultar eventos, con paginación, y WebSocket para tiempo real.

**La duda no es técnica, es de producto.** Esa API **la expone el servidor BioStar 2, no el aparato**. Para usarla, el cliente tiene que tener BioStar 2 instalado y licenciado. Si ya lo tiene, la integración es casi gratis. Si no, vender este producto implicaría venderle antes el de otro fabricante.

### Anviz

**Por qué podría funcionar.** Ofrece una Local Web API que envuelve su SDK de dispositivo en un servicio web.

**La duda es comercial y no la controlamos.** La descarga del SDK **está detrás de un registro de partner**. Si el trámite no prospera, la marca queda fuera, y eso no se resuelve programando.

---

## 3. Criterio para pasar de la lista 2 a la lista 1

Una integración solo se mueve a «implementada» cuando **extrae marcaciones reales de un aparato real de forma repetible**. Una marca se abandona por tres causas, y ninguna es que el protocolo esté mal documentado:

1. **Puerta comercial que no controlamos** — caso de Anviz.
2. **Obliga al cliente a comprar otro software** — caso de Suprema sin BioStar 2.
3. **Prueba fallida con plazo** — dos días de trabajo por marca; si no se consigue leer marcaciones de forma repetible, se documenta lo aprendido y se abandona.

Mientras tanto, cada modelo llevará un estado visible —verificado, en pruebas, sin verificar— para que **ningún cliente compre por una compatibilidad declarada que no se ha probado**.

---

## Fuentes

- ZKTeco ADMS/Push: [implementación de referencia](https://packagist.org/packages/dewanlab/laravel-zkteco-adms-core) · [sentido del tráfico](https://get.clockit.io/en/zkteco/zkteco-iclock-680) · [variante de ruta por firmware](https://kiwix.lema.org/stackoverflow.com_en_all_2023-11/questions/49049235/nodejs-express-receive-data-from-zkteco-device)
- ZKTeco por consulta: [pyzk](https://pyzk.readthedocs.io/)
- Suprema: [API de búsqueda de eventos](https://support.supremainc.com/en/support/solutions/articles/24000054050--biostar-2-api-how-to-retrieve-the-event-logs-through-biostar-2-new-local-api)
- Anviz: [Local Web API tras registro de partner](https://community.anviz.com/t/anviz-local-web-api/2761)
- ESSL: [integración de terceros por base de datos](https://apps.odoo.com/apps/modules/19.0/aspl_biometric_attendance)
