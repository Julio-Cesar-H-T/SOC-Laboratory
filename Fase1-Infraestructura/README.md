# SOC Home Lab - Fase 1: Infraestructura y Pipeline de Telemetría

**Autor:** Julio Cesar Hernandez Tibrey / Vejetto89  
**Institución:** ITLA (Instituto Tecnológico de Las Américas)  
**Fecha:** Octubre 2026  
**Estado:** Completado y validado

---

## Índice

1. [Objetivo](#1-objetivo)
2. [Arquitectura](#2-arquitectura)
3. [Especificaciones del host](#3-especificaciones-del-host)
4. [Red interna](#4-red-interna)
5. [VM1 - Rocky Linux 8 (Wazuh)](#5-vm1---rocky-linux-8-wazuh)
6. [VM2 - Ubuntu Server 22.04 (Elastic Stack)](#6-vm2---ubuntu-server-2204-elastic-stack)
7. [VM3 - Windows 10 Pro (Endpoint)](#7-vm3---windows-10-pro-endpoint)
   - [7.1 Sysmon](#71-sysmon)
   - [7.2 Wazuh Agent](#72-wazuh-agent)
   - [7.3 Winlogbeat (ECS & TLS)](#73-winlogbeat-ecs--tls)
8. [Validación del pipeline y Baseline](#8-validación-del-pipeline-y-baseline)
9. [Problemas encontrados y soluciones](#9-problemas-encontrados-y-soluciones)
10. [Queries KQL validadas (Estándar ECS)](#10-queries-kql-validadas-estándar-ecs)

---

## 1. Objetivo

Montar un entorno SOC funcional en casa para:

- Familiarizarse con SIEMs reales (Wazuh y Elastic Stack).
- Simular ciclos de ataque completos con Atomic Red Team.
- Detectar, analizar y documentar TTPs del framework MITRE ATT&CK.
- Construir un portfolio técnico documentado en GitHub.

---

## 2. Arquitectura

```text
Fedora 43 (Host - HP Victus 16)
│
├── VM1: Rocky Linux 8          →  Wazuh Server + Indexer + Dashboard
│         192.168.200.96               (OpenSearch + Wazuh Dashboard)
│
├── VM2: Ubuntu Server 22.04    →  Elastic Stack
│         192.168.200.87               (Elasticsearch 8.19 + Kibana)
│
├── VM3: Windows 10 Pro         →  Endpoint / Victima
│         192.168.200.154              (Sysmon + Wazuh Agent + Winlogbeat)
│
└── VM4: Kali Linux             →  Atacante (bajo demanda)
          192.168.200.82
```

### Flujo de telemetría

```text
Windows 10 Endpoint
│
├── Sysmon (kernel-level events)
│     │
│     ├──→ Wazuh Agent ──→ Wazuh Manager (192.168.200.96)
│     │                        └── OpenSearch + Wazuh Dashboard
│     │
│     └──→ Winlogbeat ──→ Elasticsearch (192.168.200.87:9200)
│                              └── Kibana (192.168.200.87:5601)
│
└── Windows Event Log (Security, System, Application, PowerShell)
      ├──→ Wazuh Agent
      └──→ Winlogbeat
```

El mismo endpoint envía telemetría **simultáneamente** a ambos SIEMs, permitiendo comparar detecciones y practicar tanto con Wazuh (reglas XML) como con Elastic Security (KQL y EQL).

---

## 3. Especificaciones del host

| Componente | Especificación |
|---|---|
| Modelo | HP Victus 16 |
| CPU | AMD Ryzen 7 7840HS (8 cores / 16 threads) |
| GPU | NVIDIA RTX 4060 Laptop |
| RAM | 16 GB DDR5 5600 MT/s |
| Almacenamiento | 1 TB SSD NVMe M.2 |
| OS Host | Dual boot: Windows 11 (500 GB) / Fedora 43 (500 GB) |
| Hipervisor | KVM / QEMU (`libvirt` + `virt-manager`) |

### Distribución de RAM

| VM | RAM | vCPUs | Cuándo encenderla |
|---|---|---|---|
| Rocky Linux 8 (Wazuh) | 3 GB | 2 | Sesiones Wazuh |
| Ubuntu Server (Elastic) | 3 GB | 2 | Sesiones Kibana/KQL |
| Windows 10 (Endpoint) | 4 GB | 2 | Siempre en sesiones SOC |
| Kali Linux (Atacante) | 2 GB | 2 | Solo durante ataques |

---

## 4. Red interna

Red NAT interna creada con libvirt: `soc-lab` (`192.168.200.0/24`)

```bash
# Crear la red interna en Fedora host
cat > /tmp/soc-lab-net.xml << 'EOF'
<network>
  <name>soc-lab</name>
  <forward mode='nat'/>
  <bridge name='virbr-soc' stp='on' delay='0'/>
  <ip address='192.168.200.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.200.50' end='192.168.200.200'/>
    </dhcp>
  </ip>
</network>
EOF

virsh net-define /tmp/soc-lab-net.xml
virsh net-start soc-lab
virsh net-autostart soc-lab
```

> **Nota:** La red original planificada era `192.168.100.0/24`, pero entró en conflicto con la red LAN existente del host. Se migró a `192.168.200.0/24` sin otros cambios.

---

## 5. VM1 - Rocky Linux 8 (Wazuh)

### Creación de la VM

```bash
virt-install \
  --name wazuh-server \
  --ram 3072 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/wazuh-server.qcow2,size=50 \
  --cdrom ~/isos/Rocky-8-latest-x86_64-minimal.iso \
  --network network=soc-lab \
  --os-variant rhel8.0 \
  --graphics spice \
  --channel unix,target_type=virtio,name=org.qemu.guest_agent.0
```

### Instalación de Wazuh All-in-One

```bash
# Como root en Rocky Linux
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml

# Ajustar IPs en config.yml
sed -i 's/<indexer-node-ip>/192.168.200.96/g' config.yml
sed -i 's/<wazuh-manager-ip>/192.168.200.96/g' config.yml
sed -i 's/<dashboard-node-ip>/192.168.200.96/g' config.yml

bash wazuh-install.sh -a
# Guardar las credenciales que muestra al final
```

### Optimización de RAM - limitar Heap de OpenSearch

Sin este ajuste, OpenSearch puede consumir toda la RAM de la VM y ser matado por el OOM Killer.

```bash
sed -i 's/^-Xms.*/-Xms1g/' /etc/opensearch/jvm.options
sed -i 's/^-Xmx.*/-Xmx1g/' /etc/opensearch/jvm.options

systemctl restart opensearch wazuh-manager wazuh-dashboard
```

### Acceso

- **URL:** `https://192.168.200.96`
- **Usuario:** `admin`
- **Password:** [generada durante instalación]

---

## 6. VM2 - Ubuntu Server 22.04 (Elastic Stack)

### Creación de la VM

```bash
virt-install \
  --name elastic-server \
  --ram 3072 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/elastic-server.qcow2,size=40 \
  --cdrom ~/isos/ubuntu-22.04.4-live-server-amd64.iso \
  --network network=soc-lab \
  --os-variant ubuntu22.04 \
  --graphics spice \
  --channel unix,target_type=virtio,name=org.qemu.guest_agent.0
```

### Instalación de Elasticsearch y Kibana

```bash
# Importar clave GPG y repositorio
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | tee /etc/apt/sources.list.d/elastic-8.x.list

apt update && apt install -y elasticsearch kibana

# Limitar Heap de Elasticsearch (crítico con 3 GB de RAM)
echo "-Xms1g" > /etc/elasticsearch/jvm.options.d/heap.options
echo "-Xmx1g" >> /etc/elasticsearch/jvm.options.d/heap.options

systemctl enable --now elasticsearch kibana
```

### Configurar Kibana para acceso desde el host

```bash
sed -i 's/#server.host: "localhost"/server.host: "0.0.0.0"/' /etc/kibana/kibana.yml
systemctl restart kibana
```

### Versiones en producción (Actualizado)

| Componente | Versión |
|---|---|
| Elasticsearch | 8.19.23 |
| Kibana | 8.19.x |
| Winlogbeat | 8.19.0 |

### Acceso

- **URL:** `http://192.168.200.87:5601`

---

## 7. VM3 - Windows 10 Pro (Endpoint)

- **Hostname:** `DESKTOP-H1LLRFP`  
- **IP:** `192.168.200.154`  
- **OS:** Windows 10 Pro Build 19045.2965

Esta VM es el corazón del laboratorio. Envía telemetría simultáneamente a Wazuh y a Elastic Stack, permitiendo comparar la misma detección en ambas interfaces.

---

### 7.1 Sysmon

Sysmon (System Monitor) añade visibilidad de nivel EDR sobre los logs nativos de Windows: relaciones proceso padre/hijo con hashes, conexiones de red por proceso, acceso a LSASS, modificaciones de registro, DNS queries, y más.

#### Instalación

```powershell
# Descargar Sysmon desde Sysinternals
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive Sysmon.zip -DestinationPath .\Sysmon

# Descargar configuración de SwiftOnSecurity (estándar de la industria)
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmonconfig.xml"

# Instalar con la configuración
cd .\Sysmon
.\Sysmon64.exe -i ..\sysmonconfig.xml -accepteula
```

#### Event IDs clave para SOC

| Event ID | Descripción | Detecciones clave |
|---|---|---|
| 1 | Process Creation | Hashes, línea de comandos, proceso padre |
| 3 | Network Connection | Conexiones por proceso, IP destino, puerto |
| 7 | Image Loaded | DLLs cargadas - detecta DLL hijacking |
| 8 | CreateRemoteThread | Inyección de código entre procesos |
| 10 | Process Access | Acceso a LSASS - credential dumping |
| 11 | FileCreate | Creación de archivos con hashes |
| 13 | Registry Value Set | Modificaciones de registro (persistencia) |
| 22 | DNS Query | Resoluciones DNS por proceso |
| 23 | FileDelete | Eliminación de archivos (anti-forense) |

---

### 7.2 Wazuh Agent

#### Instalación

```powershell
# En PowerShell como Administrador
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi" -OutFile wazuh-agent.msi

msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.200.96" WAZUH_AGENT_NAME="Windows-Endpoint"

NET START WazuhSvc
```

#### Configuración para ingestar Sysmon

Editar `C:\Program Files (x86)\ossec-agent\ossec.conf` y añadir dentro de `<ossec_config>`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>

<localfile>
  <location>Microsoft-Windows-PowerShell/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

```powershell
Restart-Service WazuhSvc
```

---

### 7.3 Winlogbeat (ECS & TLS)

Winlogbeat envía eventos a Elasticsearch bajo el estándar **ECS (Elastic Common Schema)**. Para garantizar la seguridad y el correcto procesamiento de los datos, se implementó la validación TLS mutua y el pipeline de ingesta de Elastic.

#### 1. Instalación (v8.19.0)

```powershell
# Descargar Winlogbeat 8.19.0
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-8.19.0-windows-x86_64.zip" -OutFile "$env:TEMP\wb819.zip"
Expand-Archive "$env:TEMP\wb819.zip" -DestinationPath "C:\Program Files\Winlogbeat"
cd "C:\Program Files\Winlogbeat\winlogbeat-8.19.0-windows-x86_64"
```

#### 2. Configurar TLS (Certificado CA)

Para que Winlogbeat confíe en la CA autofirmada de Elasticsearch, se descarga el certificado `http_ca.crt` desde la VM de Ubuntu y se guarda en la carpeta de instalación de Winlogbeat.

```powershell
# (En la VM de Ubuntu: sudo cp /etc/elasticsearch/certs/http_ca.crt /tmp/ && cd /tmp && sudo python3 -m http.server 8080)
Invoke-WebRequest -Uri "http://192.168.200.87:8080/http_ca.crt" -OutFile "C:\Program Files\Winlogbeat\winlogbeat-8.19.0-windows-x86_64\http_ca.crt"
```

#### 3. Configuración (`winlogbeat.yml`)

```yaml
winlogbeat.event_logs:
  - name: Application
    ignore_older: 72h
  - name: System
    ignore_older: 72h
  - name: Security
    ignore_older: 72h
  - name: Microsoft-Windows-Sysmon/Operational
    ignore_older: 72h
  - name: Microsoft-Windows-PowerShell/Operational
    event_id: 4103, 4104, 4105, 4106
    ignore_older: 72h

setup.kibana:
  host: "http://192.168.200.87:5601"

output.elasticsearch:
  hosts: ["https://192.168.200.87:9200"]
  username: "elastic"
  password: "{ES_PWD}"
  pipeline: "winlogbeat-%{[agent.version]}-routing"
  ssl:
    certificate_authorities: ["C:\\Program Files\\Winlogbeat\\winlogbeat-8.19.0-windows-x86_64\\http_ca.crt"]

processors:
  - add_host_metadata: ~
```

#### 4. Inicialización del Pipeline y Arranque

Es **crítico** ejecutar `setup` primero para que Elasticsearch cree el pipeline de ingesta (que traduce los logs crudos a ECS).

```powershell
# Instalar servicio
.\install-service-winlogbeat.ps1

# 1. Crear pipelines en Elasticsearch
.\winlogbeat.exe setup -e

# 2. Arrancar el servicio
Start-Service winlogbeat
Get-Service winlogbeat
```

---

## 8. Validación del pipeline y Baseline

### Confirmación en logs de Winlogbeat
Los logs ahora se ubican en `C:\ProgramData\winlogbeat\logs\`. Una conexión exitosa y segura se ve así:
```json
{"log.level":"info","message":"Connection to backoff(elasticsearch(https://192.168.200.87:9200)) established"}
```
*(Nota: El mensaje `SSL/TLS verifications disabled` ya no debe aparecer).*

### Captura de Baseline (Línea Base)
Para entender el comportamiento normal del endpoint antes de atacar, se ejecutó una agregación en Kibana (Dev Tools) para identificar los procesos más comunes y sus padres:

```json
GET winlogbeat-*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "event.code": "1" } },
        { "range": { "@timestamp": { "gte": "now-48h", "lte": "now" } } }
      ]
    }
  },
  "aggs": {
    "por_proceso": {
      "terms": { "field": "process.name", "size": 60 },
      "aggs": {
        "por_padre":  { "terms": { "field": "process.parent.name", "size": 8 } },
        "por_usuario": { "terms": { "field": "user.name", "size": 5 } },
        "por_hora":   { "date_histogram": { "field": "@timestamp", "fixed_interval": "1h" } }
      }
    },
    "top_padres": { "terms": { "field": "process.parent.name", "size": 40 } }
  }
}
```
*Hallazgos:* Procesos como `svchost.exe` y `explorer.exe` dominan la telemetría. Se estableció que `powershell.exe` es lanzado normalmente por `CompatTelRunner.exe` (telemetría de Windows).

---

## 9. Problemas encontrados y soluciones

### Problema 1 - Conflicto de red `192.168.100.0/24`
**Solución:** Migrar toda la infraestructura a `192.168.200.0/24`.

### Problema 2 - Winlogbeat enviando a `http://` en lugar de `https://`
**Solución:** Especificar explícitamente `https://` en el host. Adicionalmente, se reemplazó `ssl.verification_mode: "none"` por la configuración correcta `certificate_authorities` apuntando al `http_ca.crt` de Elastic.

### Problema 3 - Ruta de logs de Winlogbeat no encontrada
**Solución:** Los logs están en `C:\ProgramData\winlogbeat\logs\`, no en la carpeta de instalación. Se diagnostica con `sc.exe qc winlogbeat`.

### Problema 4 - Pipeline `winlogbeat-routing` no existe
**Causa:** Se descomentó la línea `pipeline:` sin ejecutar primero `winlogbeat setup`.
**Solución:** Ejecutar `.\winlogbeat.exe setup -e` antes de arrancar el servicio. Esto crea el pipeline `winlogbeat-8.19.0-routing` en Elasticsearch y permite el mapeo ECS.

### Problema 5 - Errores de sintaxis XML en `ossec.conf`
**Solución:** Editar con un editor que valide XML en tiempo real (VS Code) y verificar cierres.

### Problema 6 - Mismatch de versiones (8.10 vs 8.19)
**Solución:** Actualizar Winlogbeat a la versión 8.19.0 para emparejar con Elasticsearch 8.19.23 y evitar problemas de compatibilidad de mapeos y pipelines.

---

## 10. Queries KQL validadas (Estándar ECS)

Gracias al pipeline de Winlogbeat, ahora todas las búsquedas se realizan sobre el esquema ECS (campos como `process.name` en lugar de `winlog.event_data.Image`). Esto hace las queries portables y profesionales.

```kql
# EID 10 - ProcessAccess sobre LSASS
# OJO: TargetImage no tiene mapeo ECS en Winlogbeat -> se usa el campo crudo del proveedor
event.code: "10" and winlog.event_data.TargetImage: *lsass*

# Ver todos los eventos de creación de procesos (Sysmon EID 1)
event.code: "1"

# Filtrar por nombre de proceso
process.name: "powershell.exe"

# Ver la línea de comandos completa (ej. detectar Base64)
process.command_line: *EncodedCommand*

# Relaciones padre/hijo (ej. cmd lanzado por explorer)
process.name: "cmd.exe" and process.parent.name: "explorer.exe"

# Ejecución de herramientas de descubrimiento (netsh)
process.name: "netsh.exe"

# Acceso a LSASS (Sysmon EID 10 - Credential Dumping)

event.code: "10" and winlog.event_data.TargetImage: *lsass*

# PowerShell Script Block Logging (Event ID 4104)
event.code: "4104"
```
