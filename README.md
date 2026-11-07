# 🛡️ Wazuh SIEM en Docker - Despliegue Autohospedado

[![GitHub](https://img.shields.io/badge/GitHub-wazuh--docker--siem-blue?logo=github)](https://github.com/tu-usuario/wazuh-docker-siem)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)](https://hub.docker.com/r/wazuh/wazuh-manager)
[![License](https://img.shields.io/badge/License-GPL--2.0-orange)](LICENSE)

## 📋 Descripción general

Este repositorio proporciona una guía completa y un `docker-compose.yml` listo para producción para desplegar **Wazuh 4.14.7** —una plataforma SIEM y de detección de intrusiones de código abierto— en un solo nodo usando Docker Compose. Incluye generación automática de certificados TLS, cambio de contraseñas por defecto y configuración endurecida para entornos *homelab* y *self-hosted*.

Wazuh centraliza los registros de tus equipos, detecta intrusiones, vigila cambios en ficheros (FIM) y localiza vulnerabilidades. Este stack despliega los tres componentes principales: **Manager** (recibe datos de agentes y genera alertas), **Indexer** (basado en OpenSearch, almacena y busca eventos) y **Dashboard** (interfaz web en puerto 443).

> 🎥 **Vídeo tutorial**: [Cómo instalar Wazuh en Docker - Canal GENBYTE](https://www.youtube.com/@genbyte)

## ✨ Características principales

- **Despliegue en un solo compose**: Manager, Indexer y Dashboard en un único `docker-compose.yml`
- **Certificados TLS propios**: Contenedor generador emite CA y certificados para cada componente
- **Versiones fijadas**: Imágenes etiquetadas `4.14.7` y repositorio versionado con tag Git `v4.14.7`
- **Contraseñas endurecidas**: Procedimiento documentado para rotar secretos por defecto antes del primer arranque
- **Agente incluido**: Compose adicional para ejecutar el agente Wazuh en contenedor
- **Puertos estándar**: 443 (Dashboard), 1514/1515 (agentes), 514/UDP (syslog), 55000 (API), 9200 (Indexer)

## 📋 Requisitos del sistema

- **RAM**: 6 GB mínimo (recomendado 8 GB+) para que el Indexer funcione con holgura
- **CPU**: 4 núcleos recomendados
- **Disco**: 50 GB libres mínimo, **SSD obligatorio** para rendimiento del Indexer
- **Software**: Docker Engine + plugin Docker Compose + Git instalados
- **Puertos libres en host**:
  - `443/tcp` → Dashboard (HTTPS)
  - `1514/tcp`, `1515/tcp` → Comunicación agentes
  - `514/udp` → Syslog
  - `55000/tcp` → API Wazuh
  - `9200/tcp` → Indexer (OpenSearch) — *opcional, quitar mapeo si no se necesita acceso externo*
- **Kernel**: `vm.max_map_count >= 262144` (ver paso 1 de instalación)

## 🐳 Instalación

### Paso 1: Ajustar `vm.max_map_count` (obligatorio)

El Indexer no arranca con el valor por defecto de la mayoría de sistemas Linux. Aplícalo **ahora** y hazlo **permanente**:

```bash
# Aplicar ahora
sudo sysctl -w vm.max_map_count=262144

# Hacerlo permanente
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Paso 2: Clonar la etiqueta v4.14.7

Usa la etiqueta de la versión, **no** la rama principal (que es desarrollo de la 5.x):

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.7
cd wazuh-docker/single-node/
```

### Paso 3: Generar los certificados TLS

El generador lee `config/certs.yml` y escribe los certificados en `config/wazuh_indexer_ssl_certs/`:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
ls -l config/wazuh_indexer_ssl_certs/
```

**Verificación**: Debes ver los certificados del indexer, manager, dashboard y la CA (`root-ca.pem`, `wazuh.indexer.pem`, `wazuh.manager.pem`, `wazuh.dashboard.pem` + sus claves). Si la carpeta está vacía, **no continúes**: el stack no arrancará sin certificados.

### Paso 4: Cambiar las contraseñas por defecto (CRÍTICO)

Las contraseñas del compose original son públicas (`SecretPassword`, `kibanaserver`, `MyS3cr37P450r.*-`). **Cámbialas antes de arrancar**.

#### 4a. Generar hash bcrypt para la contraseña de admin (Indexer)

```bash
docker run --rm -ti wazuh/wazuh-indexer:4.14.7 \
  bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh \
  -p 'TuContrasenaLargaYUnica'
```

Copia el hash resultante (formato `$2a$12$...`).

#### 4b. Actualizar `config/wazuh_indexer/internal_users.yml`

Cambia el hash en **dos usuarios**, no solo en `admin`:

```yaml
admin:
  hash: "<HASH_GENERADO_CON_TU_INDEXER_PASSWORD>"
  reserved: true
  backend_roles:
    - "admin"
  description: "Admin user"

kibanaserver:
  hash: "<HASH_GENERADO_CON_TU_DASHBOARD_PASSWORD>"
  reserved: true
  description: "Kibana server user"
```

> **Nota**: `kibanaserver` es el usuario con el que el Dashboard se conecta al Indexer. Si no coincide, el Dashboard no puede entrar.

#### 4c. Poner las contraseñas en claro en `docker-compose.yml`

Edita `docker-compose.yml` y sustituye los marcadores `CAMBIA_...` por tus contraseñas reales:

- `INDEXER_PASSWORD` → en servicios `wazuh.manager` y `wazuh.dashboard`
- `API_PASSWORD` → en servicios `wazuh.manager` y `wazuh.dashboard` (usuario `wazuh-wui`)
- `DASHBOARD_PASSWORD` → en servicio `wazuh.dashboard` (usuario `kibanaserver`)

**Comprobación**: El valor de `API_PASSWORD` lo usan manager y dashboard para el usuario `wazuh-wui` (no aparece en `internal_users.yml`). Al arrancar, verifica que el dashboard conecta con la API.

### Paso 5: Levantar el stack

```bash
docker compose up -d
docker compose ps
docker compose logs -f wazuh.indexer
```

El primer arranque tarda **varios minutos** (el Indexer crea sus índices). Ten paciencia antes de dar la instalación por fallida.

> ⚠️ **Si cambiaste contraseñas tras un primer arranque fallido**: Los cambios en `internal_users.yml` pueden no aplicarse si el Indexer ya inicializó su índice de seguridad. La forma segura de empezar de cero:
> ```bash
> docker compose down -v
> docker volume rm wazuh-docker_single-node_wazuh-indexer-data
> docker compose up -d
> ```
> **Perderás los datos indexados**.

### Paso 6: Acceder al Dashboard

Abre `https://IP_DEL_SERVIDOR` en el navegador.

- **Usuario**: `admin`
- **Contraseña**: la que pusiste en `INDEXER_PASSWORD`
- **Certificado**: Firmado por tu CA propia → el navegador avisará (acepta la excepción o importa la CA)

---

## ⚙️ Configuración

1. **Certificados TLS**: Generados automáticamente en `config/wazuh_indexer_ssl_certs/` (Paso 3). No los elimines: son necesarios para reinicios y restauraciones.
2. **Usuarios y contraseñas**: Definidos en `config/wazuh_indexer/internal_users.yml` (hashes bcrypt) y `docker-compose.yml` (en claro para variables de entorno).
3. **Configuración del Manager**: `config/wazuh_cluster/wazuh_manager.conf` → montado como `/wazuh-config-mount/etc/ossec.conf`.
4. **Configuración del Indexer**: `config/wazuh_indexer/wazuh.indexer.yml` → `opensearch.yml`.
5. **Configuración del Dashboard**: `config/wazuh_dashboard/opensearch_dashboards.yml` y `wazuh.yml`.
6. **Límites de sistema**: `ulimits.memlock=-1` y `nofile=65536` en Manager e Indexer (requerido por OpenSearch).
7. **Heap del Indexer**: `OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g` (ajusta según RAM disponible, máx. 50% de RAM del host).
8. **Red Docker**: Los tres servicios comparten red interna; solo los puertos declarados en `ports:` se publican en el host.

---

## 🚀 Primeros pasos

1. Clona el repo en la tag `v4.14.7` y entra en `single-node/`
2. Ejecuta `sysctl -w vm.max_map_count=262144` y hazlo permanente en `/etc/sysctl.conf`
3. Genera certificados con `docker compose -f generate-indexer-certs.yml run --rm generator`
4. Verifica que `config/wazuh_indexer_ssl_certs/` contiene 8+ archivos `.pem`/`.key`
5. Genera hashes bcrypt para `INDEXER_PASSWORD` y `DASHBOARD_PASSWORD`
6. Actualiza `internal_users.yml` con ambos hashes (`admin` y `kibanaserver`)
7. Edita `docker-compose.yml` y reemplaza `CAMBIA_INDEXER`, `CAMBIA_API`, `CAMBIA_DASHBOARD` por tus contraseñas en claro
8. Levanta el stack: `docker compose up -d`
9. Sigue los logs del indexer: `docker compose logs -f wazuh.indexer` (espera a "started")
10. Abre `https://IP_DEL_SERVIDOR`, acepta el certificado y entra con `admin` / tu `INDEXER_PASSWORD`

---

## 💡 Casos de uso

- **SIEM centralizado para homelab**: Recopila logs de servidores, NAS, firewalls y endpoints en un solo panel
- **Detección de intrusos (IDS/HIDS)**: Reglas MITRE ATT&CK, alertas de fuerza bruta, escalada de privilegios, malware
- **File Integrity Monitoring (FIM)**: Vigila cambios en `/etc`, `/bin`, directorios web, bases de datos
- **Escaneo de vulnerabilidades**: Inventario de paquetes y CVE asociados vía Wazuh SCA y Vulnerability Detector
- **Cumplimiento normativo**: Reportes PCI-DSS, GDPR, HIPAA listos para auditoría
- **Integración SOAR**: Envía alertas a TheHive, Shuffle, n8n o webhooks personalizados

---

## 🔒 Acceso remoto seguro

> **No expongas puertos directamente a Internet** sin protección adicional.

Opciones recomendadas:

1. **Tailscale / WireGuard / Tunneler**: Accede al dashboard vía VPN de malla (solo tu tailnet)
2. **Reverse Proxy + Authelia / Authelia + Cloudflare Tunnel**: Termina TLS en el proxy, añade 2FA y WAF
3. **SSH Tunnel**: `ssh -L 443:localhost:443 user@server` para acceso puntual
4. **Restricción por IP en firewall del host**: `ufw allow from 203.0.113.0/24 to any port 443`

**Puertos que NO deben exponerse nunca**: `9200` (Indexer), `55000` (API), `1514/1515` (agentes) — úsalos solo en red interna o VPN.

---

## 🛠️ Gestión y mantenimiento

### Ver estado y logs

```bash
docker compose ps
docker compose logs -f wazuh.manager
docker compose logs -f wazuh.indexer
docker compose logs -f wazuh.dashboard
```

### Consultar contraseñas configuradas

```bash
grep -E 'INDEXER_PASSWORD|API_PASSWORD|DASHBOARD_PASSWORD' docker-compose.yml
```

### Detener el stack

```bash
docker compose down          # Para contenedores, mantiene volúmenes
docker compose down -v       # Para contenedores Y elimina volúmenes de datos (¡pérdida total!)
```

### Copias de seguridad (imprescindibles)

Respalda **siempre** estos dos conjuntos:

1. **Carpeta `config/` completa**: Certificados, `internal_users.yml`, `wazuh_manager.conf`, `opensearch.yml`, `opensearch_dashboards.yml`, y tu `docker-compose.yml` con las contraseñas reales.
   ```bash
   tar -czf wazuh-config-backup-$(date +%F).tar.gz config/
   ```

2. **Volúmenes de datos Docker** (Indexer + Manager + Dashboard):
   ```bash
   # Ejemplo con docker run --volumes-from (adapta a tu driver de almacenamiento)
   docker run --rm -v wazuh-docker_single-node_wazuh-indexer-data:/data -v $(pwd)/backups:/backup alpine tar -czf /backup/indexer-data-$(date +%F).tar.gz -C /data .
   ```

> **Sin los certificados no puedes restaurar la instalación**. Guarda la carpeta `config/` en ubicación segura y fuera del servidor (ej. NAS, S3, USB cifrado).

### Actualización de versión

Wazuh 4.x → 4.x (parche): Cambia la etiqueta en `docker-compose.yml` (ej. `4.14.7` → `4.14.8`), regenera certificados si la CA caducó, y haz `docker compose up -d`.

Wazuh 4.x → 5.x (mayor): **No soportado in-place**. Requiere migración de índices y configuración. Sigue la guía oficial de migración.

---

## 📝 Licencia

Este proyecto de despliegue y documentación se distribuye bajo licencia **GPL-2.0**, igual que Wazuh upstream.

- Código de Wazuh: © Wazuh Inc., GPLv2
- Imágenes Docker: `wazuh/wazuh-manager`, `wazuh/wazuh-indexer`, `wazuh/wazuh-dashboard`, `wazuh/wazuh-certs-generator`
- Este README y compose de ejemplo: GPL-2.0

---

> 📖 **Artículo original y guía extendida**: [¡Monta tu propio SIEM gratis! Wazuh en Docker paso a paso (Tutorial Homelab)](https://genbyte.blogspot.com/2026/11/monta-tu-propio-siem-gratis-wazuh-en.html)  
> 🎬 **Vídeo en YouTube**: [Canal GENBYTE](https://www.youtube.com/@genbyte)  
> ☕ **Apoya el canal**: [Ko-fi](https://ko-fi.com/genbyte) | [Newsletter](https://genbyte.blogspot.com/newsletter) | [Telegram](https://t.me/genbyte) | [Discord](https://discord.gg/genbyte)