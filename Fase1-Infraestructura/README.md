# SOC Home Lab - Fase 1: Infraestructura y Pipeline de Telemetria

**Autor:** Julio Cesar Hernandez Tibrey / Vejetto89  
**Institucion:** ITLA (Instituto Tecnologico de Las Americas)  
**Fecha:** Octubre 2026  
**Estado:** Completado y validado

---

## Indice

1. [Objetivo](#1-objetivo)
2. [Arquitectura](#2-arquitectura)
3. [Especificaciones del host](#3-especificaciones-del-host)
4. [Red interna](#4-red-interna)
5. [VM1 - Rocky Linux 8 (Wazuh)](#5-vm1---rocky-linux-8-wazuh)
6. [VM2 - Ubuntu Server 22.04 (Elastic Stack)](#6-vm2---ubuntu-server-2204-elastic-stack)
7. [VM3 - Windows 10 Pro (Endpoint)](#7-vm3---windows-10-pro-endpoint)
   - [7.1 Sysmon](#71-sysmon)
   - [7.2 Wazuh Agent](#72-wazuh-agent)
   - [7.3 Winlogbeat](#73-winlogbeat)
8. [Validacion del pipeline](#8-validacion-del-pipeline)
9. [Problemas encontrados y soluciones](#9-problemas-encontrados-y-soluciones)
10. [Queries KQL validadas](#10-queries-kql-validadas)

---

## 1. Objetivo

Montar un entorno SOC funcional en casa para:

- Familiarizarse con SIEMs reales (Wazuh y Elastic Stack)
- Simular ciclos de ataque completos con Atomic Red Team
- Detectar, analizar y documentar TTPs del framework MITRE ATT&CK
- Construir un portfolio tecnico documentado en GitHub

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

### Flujo de telemetria

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

El mismo endpoint envia telemetria **simultaneamente** a ambos SIEMs, permitiendo comparar detecciones y practicar tanto con Wazuh (reglas XML) como con Elastic Security (KQL).

---

## 3. Especificaciones del host

| Componente | Especificacion |
|---|---|
| Modelo | HP Victus 16 |
| CPU | AMD Ryzen 7 7840HS (8 cores / 16 threads) |
| GPU | NVIDIA RTX 4060 Laptop |
| RAM | 16 GB DDR5 5600 MT/s |
| Almacenamiento | 1 TB SSD NVMe M.2 |
| OS Host | Dual boot: Windows 11 (500 GB) / Fedora 43 (500 GB) |
| Hipervisor | KVM / QEMU (`libvirt` + `virt-manager`) |

### Distribucion de RAM

| VM | RAM | vCPUs | Cuando encenderla |
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

> **Nota:** La red original planificada era `192.168.100.0/24`, pero entro en conflicto con la red LAN existente del host. Se migro a `192.168.200.0/24` sin otros cambios.

---

## 5. VM1 - Rocky Linux 8 (Wazuh)

### Creacion de la VM

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

### Instalacion de Wazuh All-in-One

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

### Optimizacion de RAM - limitar Heap de OpenSearch

Sin este ajuste, OpenSearch puede consumir toda la RAM de la VM y ser matado por el OOM Killer.

```bash
sed -i 's/^-Xms.*/-Xms1g/' /etc/opensearch/jvm.options
sed -i 's/^-Xmx.*/-Xmx1g/' /etc/opensearch/jvm.options

systemctl restart opensearch wazuh-manager wazuh-dashboard
```

### Acceso

- **URL:** `https://192.168.200.96`
- **Usuario:** `admin`
- **Password:** [generada durante instalacion]

---

## 6. VM2 - Ubuntu Server 22.04 (Elastic Stack)

### Creacion de la VM

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

### Instalacion de Elasticsearch y Kibana

```bash
# Importar clave GPG y repositorio
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | tee /etc/apt/sources.list.d/elastic-8.x.list

apt update && apt install -y elasticsearch kibana

# Limitar Heap de Elasticsearch (critico con 3 GB de RAM)
echo "-Xms1g" > /etc/elasticsearch/jvm.options.d/heap.options
echo "-Xmx1g" >> /etc/elasticsearch/jvm.options.d/heap.options

systemctl enable --now elasticsearch kibana
```

### Configurar Kibana para acceso desde el host

```bash
sed -i 's/#server.host: "localhost"/server.host: "0.0.0.0"/' /etc/kibana/kibana.yml
systemctl restart kibana
```

### Version en produccion

| Componente | Version |
|---|---|
| Elasticsearch | 8.19.23 |
| Kibana | 8.x |
| Winlogbeat | 8.10.0 |

### Acceso

- **URL:** `http://192.168.200.87:5601`

---

## 7. VM3 - Windows 10 Pro (Endpoint)

- **Hostname:** `DESKTOP-H1LLRFP`  
- **IP:** `192.168.200.154`  
- **OS:** Windows 10 Pro Build 19045.2965

Esta VM es el corazon del laboratorio. Envia telemetria simultaneamente a Wazuh y a Elastic Stack, permitiendo comparar la misma deteccion en ambas interfaces.

---

### 7.1 Sysmon

Sysmon (System Monitor) es el componente mas critico del endpoint. Anade visibilidad de nivel EDR sobre los logs nativos de Windows: relaciones proceso padre/hijo con hashes, conexiones de red por proceso, acceso a LSASS, modificaciones de registro, DNS queries, y mas.

#### Instalacion

```powershell
# Descargar Sysmon desde Sysinternals
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive Sysmon.zip -DestinationPath .\Sysmon

# Descargar configuracion de SwiftOnSecurity (estandar de la industria)
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmonconfig.xml"

# Instalar con la configuracion
cd .\Sysmon
.\Sysmon64.exe -i ..\sysmonconfig.xml -accepteula
```

#### Event IDs clave para SOC

| Event ID | Descripcion | Detecciones clave |
|---|---|---|
| 1 | Process Creation | Hashes, linea de comandos completa, proceso padre |
| 3 | Network Connection | Conexiones por proceso, IP destino, puerto |
| 7 | Image Loaded | DLLs cargadas - detecta DLL hijacking |
| 8 | CreateRemoteThread | Inyeccion de codigo entre procesos |
| 10 | Process Access | Acceso a LSASS - credential dumping |
| 11 | FileCreate | Creacion de archivos con hashes |
| 13 | Registry Value Set | Modificaciones de registro (persistencia) |
| 22 | DNS Query | Resoluciones DNS por proceso |
| 23 | FileDelete | Eliminacion de archivos (anti-forense) |

---

### 7.2 Wazuh Agent

#### Instalacion

```powershell
# En PowerShell como Administrador
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi" -OutFile wazuh-agent.msi

msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.200.96" WAZUH_AGENT_NAME="Windows-Endpoint"

NET START WazuhSvc
```

#### Configuracion para ingestar Sysmon

Editar `C:\Program Files (x86)\ossec-agent\ossec.conf` y anadir dentro de `<ossec_config>`:

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

> **Problema encontrado:** Errores de sintaxis XML en `ossec.conf` impedian que el agente iniciara. Causa: etiquetas mal cerradas al editar manualmente. Solucion: validar el XML antes de reiniciar con un editor que resalte errores de sintaxis (VS Code, Notepad++).

---

### 7.3 Winlogbeat

Winlogbeat es el agente oficial de Elastic para logs de Windows. Envia eventos directamente a Elasticsearch bajo el estandar ECS (Elastic Common Schema), lo que permite usar KQL en Kibana.

#### Instalacion

```powershell
# Descargar Winlogbeat 8.10.0
Invoke-WebRequest -Uri "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-8.10.0-windows-x86_64.zip" -OutFile winlogbeat.zip
Expand-Archive winlogbeat.zip -DestinationPath "C:\Program Files\Winlogbeat"

cd "C:\Program Files\Winlogbeat\winlogbeat-8.10.0-windows-x86_64"
```

#### Configuracion (`winlogbeat.yml`)

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
  password: "TU_PASSWORD_AQUI"
  ssl.verification_mode: "none"

processors:
  - add_host_metadata: ~
```

#### Instalacion del servicio y arranque

```powershell
# Instalar servicio
.\install-service-winlogbeat.ps1

# Validar configuracion ANTES de arrancar
.\winlogbeat.exe test config -e

# Validar conectividad con Elasticsearch
.\winlogbeat.exe test output -e

# Arrancar
Start-Service winlogbeat
Get-Service winlogbeat
```

#### Ruta de logs del servicio

```text
C:\ProgramData\winlogbeat\logs\winlogbeat-YYYYMMDD.ndjson
```

> **Nota:** La carpeta de logs NO es `C:\Program Files\Winlogbeat\...\logs\` sino `C:\ProgramData\winlogbeat\logs\`. El servicio registra `--path.logs` en su configuracion, visible con `sc.exe qc winlogbeat`.

---

## 8. Validacion del pipeline

### Confirmacion en logs de Winlogbeat

Senales de funcionamiento correcto en `C:\ProgramData\winlogbeat\logs\winlogbeat-*.ndjson`:

```json
{"log.level":"info","message":"Connection to backoff(elasticsearch(https://192.168.200.87:9200)) established"}
{"libbeat":{"output":{"events":{"acked":2481,"active":0,"batches":51,"total":2481}}}}
```

### Comandos de diagnostico

```powershell
# Ver estado del servicio
Get-Service winlogbeat

# Leer log en tiempo real
Get-Content "C:\ProgramData\winlogbeat\logs\winlogbeat-20261021-3.ndjson" -Wait

# Filtrar solo errores y warnings de todos los logs
Get-Content "C:\ProgramData\winlogbeat\logs\*.ndjson" |
  Where-Object { $_ -match '"log.level":"error"' -or $_ -match '"log.level":"warn"' } |
  Select-Object -Last 20

# Ver ruta exacta registrada por el servicio
sc.exe qc winlogbeat
```

### Verificacion en Kibana

1. Ir a `http://192.168.200.87:5601`
2. **Management -> Stack Management -> Kibana -> Data Views -> Create data view**
3. Index pattern: `winlogbeat-*` / Timestamp: `@timestamp`
4. En **Discover**, filtrar con KQL:

```kql
winlog.channel : "Microsoft-Windows-Sysmon/Operational"
```

---

## 9. Problemas encontrados y soluciones

### Problema 1 - Conflicto de red `192.168.100.0/24`

**Sintoma:** La red NAT `192.168.100.0/24` planificada originalmente entraba en conflicto con la red LAN del router del host.

**Solucion:** Migrar toda la infraestructura a `192.168.200.0/24`. No requirio cambios de arquitectura, solo actualizar las IPs en todos los archivos de configuracion.

---

### Problema 2 - Winlogbeat enviando a `http://` en lugar de `https://`

**Sintoma:** El log mostraba repetidamente: `Failed to connect to backoff(elasticsearch(http://192.168.200.87:9200)): EOF`

**Causa:** Elasticsearch 8.x habilita HTTPS por defecto. El `winlogbeat.yml` original tenia `protocol: "http"` o carecia de la linea `protocol: "https"`.

**Solucion:** Especificar explicitamente el protocolo en el host:

```yaml
output.elasticsearch:
  hosts: ["https://192.168.200.87:9200"]
  ssl.verification_mode: "none"
```

---

### Problema 3 - Ruta de logs de Winlogbeat no encontrada

**Sintoma:** `Cannot find path 'C:\Program Files\Winlogbeat\...\logs\winlogbeat'`

**Causa:** Al instalar el servicio con `install-service-winlogbeat.ps1`, Windows registra `--path.logs "C:\ProgramData\winlogbeat\logs"` (una ruta diferente a la carpeta de instalacion).

**Diagnostico:**

```powershell
sc.exe qc winlogbeat
```

**Solucion:** Los logs estan siempre en `C:\ProgramData\winlogbeat\logs\`, no en la carpeta de instalacion.

---

### Problema 4 - Pipeline `winlogbeat-routing` no existe

**Sintoma:** Winlogbeat no enviaba eventos a pesar de que el servicio reportaba `Running`.

**Causa:** El `winlogbeat.yml` original incluia `pipeline: "winlogbeat-%{[agent.version]}-routing"`. Este pipeline de ingest se crea automaticamente cuando se corre `winlogbeat setup`, comando que no se ejecuto.

**Solucion:** Comentar o eliminar la linea `pipeline:` del `winlogbeat.yml` para que los eventos se indexen directamente.

---

### Problema 5 - Errores de sintaxis XML en `ossec.conf` de Wazuh Agent

**Sintoma:** El servicio `WazuhSvc` no iniciaba despues de editar `ossec.conf`.

**Causa:** Edicion manual del XML con etiquetas mal cerradas o caracteres invalidos.

**Solucion:** Editar con un editor que valide XML en tiempo real (VS Code con extension XML, o Notepad++). Verificar que cada `<localfile>` tenga su cierre `</localfile>` correcto.

---

## 10. Queries KQL validadas

Queries probadas en Kibana (Data View: `winlogbeat-*`) que confirmaron recepcion de telemetria real:

```kql
# Ver todos los eventos de Sysmon
winlog.channel : "Microsoft-Windows-Sysmon/Operational"

# Process Creation (Event ID 1) - base de la mayoria de detecciones
winlog.event_id : "1"

# cmd.exe spawneado por cmd.exe (shell chaining sospechoso)
winlog.event_id: "1" and winlog.event_data.Image: *cmd.exe and winlog.event_data.ParentImage: *cmd.exe

# Equivalente en formato ECS estandar
event.code: "1" and process.name: "cmd.exe" and process.parent.name: "cmd.exe"

# Actividad de plink.exe (usado por Atomic Red Team T1021.004)
winlog.event_data.Image: *plink.exe

# winPEAS execution (T1082 / T1069)
winlog.event_data.Image: *winpeas*

# Procesos con integridad High (elevados)
winlog.event_data.IntegrityLevel: "High"

# PowerShell Script Block Logging (Event ID 4104)
winlog.event_id: "4104"

# Acceso a LSASS (credential dumping - Event ID 10)
winlog.event_id: "10" and winlog.event_data.TargetImage: *lsass*
```
