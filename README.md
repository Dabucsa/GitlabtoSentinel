Copilot said: # 🚀 GUÍA COMPLETA: GitLab Audit Events
🚀 GUÍA COMPLETA: GitLab Audit Events → Azure Sentinel (Desde Cero)
Autor: Danilo Bustamante (Dabucsa)
Fecha: 2025-10-30
Versión: Final - Probada y Funcional

📋 TABLA DE CONTENIDOS
Resumen Ejecutivo
Arquitectura del Sistema
Prerequisitos
Fase 1: Crear Infraestructura Base
Fase 2: Configurar Key Vault con Private Endpoint
Fase 3: Configurar Log Analytics y DCR
Fase 4: Crear y Configurar Logic App
Fase 5: Configurar Permisos RBAC
Fase 6: Desplegar Workflow
Fase 7: Probar el Sistema
Mantenimiento y Troubleshooting
🎯 RESUMEN EJECUTIVO
¿Qué vamos a construir?
Un sistema automatizado que:

✅ Extrae audit events de GitLab cada 10 minutos
✅ Usa ingesta incremental (sin duplicados)
✅ Almacena checkpoint en Azure Key Vault
✅ Transforma datos con DCR (Data Collection Rule)
✅ Envía eventos a Azure Sentinel vía Log Analytics
✅ Funciona con Key Vault en Private Endpoint
✅ Usa Managed Identity para autenticación RBAC
Componentes principales:
Componente	Propósito
Azure Key Vault	Almacena token de GitLab y checkpoint (último timestamp procesado)
Logic App Standard	Ejecuta la extracción cada 10 minutos con paginación
Data Collection Endpoint (DCE)	Recibe los eventos de la Logic App
Data Collection Rule (DCR)	Transforma datos (corrige TimeGenerated)
Log Analytics Workspace	Almacena eventos en tabla gitlabaudit_CL
Virtual Network (VNET)	Conecta Logic App con Key Vault privado
🏗️ ARQUITECTURA DEL SISTEMA
Code
┌─────────────────────────────────────────────────────────────────┐
│                       GITLAB.COM                                 │
│                  API: /groups/{id}/audit_events                  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ HTTPS (cada 10 min)
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                    AZURE VNET (10.0.0.0/16)                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────────────────────────┐                     │
│  │  Logic App (Standard)                  │                     │
│  │  - Managed Identity habilitada         │                     │
│  │  - VNET Integration                    │                     │
│  │  - Ejecuta cada 10 minutos             │                     │
│  └────────┬───────────────────────────────┘                     │
│           │                     │                                │
│           │                     │                                │
│           │ (Lee/Escribe)       │ (Envía eventos)               │
│           │                     │                                │
│  ┌────────▼─────────────┐      │                                │
│  │  Key Vault           │      │                                │
│  │  - Private Endpoint  │      │                                │
│  │  - RBAC habilitado   │      │                                │
│  │  Secretos:           │      │                                │
│  │  • Token GitLab      │      │                                │
│  │  • Checkpoint        │      │                                │
│  └──────────────────────┘      │                                │
│                                 │                                │
└─────────────────────────────────┼────────────────────────────────┘
                                  │
                                  │ HTTPS POST
                                  │
                    ┌─────────────▼──────────────┐
                    │  Data Collection Endpoint  │
                    │  (DCE)                     │
                    └─────────────┬──────────────┘
                                  │
                                  │
                    ┌─────────────▼──────────────┐
                    │  Data Collection Rule      │
                    │  (DCR)                     │
                    │  Transformación KQL:       │
                    │  TimeGenerated = EventTime │
                    └─────────────┬──────────────┘
                                  │
                                  │
                    ┌─────────────▼──────────────┐
                    │  Log Analytics Workspace   │
                    │  Tabla: gitlabaudit_CL     │
                    └─────────────┬──────────────┘
                                  │
                                  │
                    ┌─────────────▼──────────────┐
                    │  Microsoft Sentinel        │
                    │  (Alertas y Análisis)      │
                    └────────────────────────────┘
✅ PREREQUISITOS
Antes de comenzar, necesitas:

En Azure:
✅ Suscripción de Azure activa
✅ Permisos de Contributor o Owner en la suscripción
✅ Acceso al Azure Portal (https://portal.azure.com)
En GitLab:
✅ Cuenta en GitLab.com (o instancia self-hosted)
✅ Permisos de Owner en el grupo de GitLab
✅ GitLab Premium/Ultimate (para acceso a Audit Events API)
Información que debes tener a mano:
✅ GitLab Group ID (ejemplo: 116948591)
Lo encuentras en la URL: https://gitlab.com/groups/116948591
✅ GitLab Personal Access Token (PAT) con scope api
Créalo en: GitLab → Settings → Access Tokens
🏗️ FASE 1: CREAR INFRAESTRUCTURA BASE
Paso 1.1: Crear Resource Group
Abre Azure Portal: https://portal.azure.com
En el buscador superior, escribe: Resource groups
Haz clic en "Resource groups"
Haz clic en "+ Create"
Completa el formulario:
Subscription: Selecciona tu suscripción
Resource group: gitlab-audit-rg
Region: East US
Haz clic en "Review + create"
Haz clic en "Create"
✅ Verificación: Verás "Your deployment is complete"

Paso 1.2: Crear Virtual Network (VNET)
En el buscador, escribe: Virtual networks
Haz clic en "+ Create"
En la pestaña Basics:
Subscription: Tu suscripción
Resource group: gitlab-audit-rg
Name: gitlab-audit-vnet
Region: East US
Haz clic en "Next: IP Addresses"
En IPv4 address space:
Mantén: 10.0.0.0/16
Haz clic en "+ Add subnet"
Subnet name: keyvault-pe-subnet
Subnet address range: 10.0.2.0/24
Haz clic en "Add"
Haz clic en "+ Add subnet" nuevamente
Subnet name: logicapp-subnet
Subnet address range: 10.0.1.0/24
Delegation: Selecciona Microsoft.Web/serverFarms
Haz clic en "Add"
Haz clic en "Review + create"
Haz clic en "Create"
✅ Verificación: VNET creada con 2 subnets

🔐 FASE 2: CONFIGURAR KEY VAULT CON PRIVATE ENDPOINT
Paso 2.1: Crear Key Vault
En el buscador, escribe: Key vaults
Haz clic en "+ Create"
En la pestaña Basics:
Subscription: Tu suscripción
Resource group: gitlab-audit-rg
Key vault name: gitlab-audit-kv-XXXXX
⚠️ Reemplaza XXXXX con 5 números únicos (ejemplo: gitlab-audit-kv-98765)
Debe ser único globalmente
Region: East US
Pricing tier: Standard
Haz clic en "Next: Access configuration"
En Permission model:
Selecciona "Azure role-based access control" ⚠️ (IMPORTANTE)
Haz clic en "Next: Networking"
En Network access:
Selecciona "Disable public access" (máxima seguridad)
Haz clic en "Review + create"
Haz clic en "Create"
✅ Verificación: Key Vault creado con RBAC y sin acceso público

Paso 2.2: Crear Private Endpoint para Key Vault
Abre el Key Vault recién creado (gitlab-audit-kv-XXXXX)
En el menú lateral, ve a "Networking"
Haz clic en la pestaña "Private endpoint connections"
Haz clic en "+ Private endpoint"
En la pestaña Basics:
Subscription: Tu suscripción
Resource group: gitlab-audit-rg
Name: keyvault-pe
Region: East US
Haz clic en "Next: Resource"
En Target sub-resource:
Selecciona vault
Haz clic en "Next: Virtual Network"
Configura:
Virtual network: gitlab-audit-vnet
Subnet: keyvault-pe-subnet
En Private DNS integration:
Integrate with private DNS zone: Yes
Se creará automáticamente: privatelink.vaultcore.azure.net
Haz clic en "Next: Tags" → "Next: Review + create"
Haz clic en "Create"
✅ Verificación: Private Endpoint creado y Private DNS Zone vinculada

Paso 2.3: Crear Secretos en Key Vault
En el Key Vault, ve a "Secrets" (menú lateral)
Haz clic en "+ Generate/Import"
Secreto 1: Token de GitLab 3. Completa:

Name: gitlabaudit-token
Value: Pega tu GitLab Personal Access Token (PAT)
Ejemplo: glpat-xxxxxxxxxxxxxxxxxxxx
Haz clic en "Create"
Secreto 2: Checkpoint inicial 5. Haz clic en "+ Generate/Import" nuevamente 6. Completa:

Name: gitlabaudit-lasttime
Value: 2020-01-01T00:00:00Z
Haz clic en "Create"
✅ Verificación: 2 secretos creados en Key Vault

📊 FASE 3: CONFIGURAR LOG ANALYTICS Y DCR
Paso 3.1: Crear Log Analytics Workspace
En el buscador, escribe: Log Analytics workspaces
Haz clic en "+ Create"
Completa:
Subscription: Tu suscripción
Resource group: gitlab-audit-rg
Name: gitlab-audit-workspace
Region: East US
Haz clic en "Review + Create"
Haz clic en "Create"
✅ Verificación: Workspace creado

Paso 3.2: Crear Data Collection Endpoint (DCE)
En el buscador, escribe: Monitor
Abre "Monitor"
En el menú lateral, ve a "Settings" → "Data Collection Endpoints"
Haz clic en "+ Create"
Completa:
Endpoint name: gitlab-audit-dce
Subscription: Tu suscripción
Resource group: gitlab-audit-rg
Region: East US
En Network isolation:
Deja "All networks" (puedes cambiarlo después si necesitas Private Endpoint)
Haz clic en "Review + create"
Haz clic en "Create"
📝 GUARDAR ENDPOINT URL: 9. Abre el DCE recién creado 10. En "Overview", copia el "Logs ingestion" endpoint 11. Ejemplo: https://gitlab-audit-dce-abc1.eastus-1.ingest.monitor.azure.com 12. Guárdalo en un archivo de texto (lo necesitarás después)

✅ Verificación: DCE creado y endpoint URL guardado

Paso 3.3: Crear Data Collection Rule (DCR) con Transformación
En Monitor, ve a "Settings" → "Data Collection Rules"
Haz clic en "+ Create"
En la pestaña Basics:
Rule name: gitlab-audit-dcr
Subscription: Tu suscripción
Resource group: gitlab-audit-rg
Region: East US
Platform Type: Custom
Haz clic en "Next: Resources"
No agregues recursos todavía (la Logic App enviará directamente)
Haz clic en "Next: Collect and deliver"
Haz clic en "+ Add data source"
En Data source type:
Selecciona Custom Text Logs
En File pattern:
Déjalo vacío (no aplica para Logic App)
Haz clic en "Next"
⚠️ TRANSFORMACIÓN CRÍTICA: 11. En Transformation editor, pega este código KQL:

kql
source
| extend TimeGenerated = todatetime(EventTime)
Haz clic en "Next"
En Destination:
Destination type: Azure Monitor Logs
Subscription: Tu suscripción
Account or namespace: gitlab-audit-workspace
Table: Escribe gitlabaudit_CL (se creará automáticamente)
Haz clic en "Add data source"
Haz clic en "Next: Review + create"
Haz clic en "Create"
📝 GUARDAR DCR IMMUTABLE ID: 17. Abre el DCR recién creado 18. En "Overview", copia el "Immutable ID" 19. Ejemplo: dcr-831bcb9021c04abe9389cbf646e25357 20. Guárdalo junto con el endpoint URL

✅ Verificación: DCR creado con transformación KQL

💻 FASE 4: CREAR Y CONFIGURAR LOGIC APP
Paso 4.1: Crear Storage Account (requerido para Logic App Standard)
En el buscador, escribe: Storage accounts
Haz clic en "+ Create"
Completa:
Subscription: Tu suscripción
Resource group: gitlab-audit-rg
Storage account name: gitlabauditst + 5 dígitos únicos
Ejemplo: gitlabauditst98765
Solo letras minúsculas y números
Region: East US
Performance: Standard
Redundancy: Locally-redundant storage (LRS)
Haz clic en "Review + create"
Haz clic en "Create"
✅ Verificación: Storage Account creado

Paso 4.2: Crear Logic App (Standard)
En el buscador, escribe: Logic apps
Haz clic en "+ Add"
En la pestaña Basics:
Subscription: Tu suscripción
Resource group: gitlab-audit-rg
Type: Standard
Logic App name: gitlab-audit-logicapp
Publish: Workflow
Region: East US
Plan type: Workflow Standard
Zone redundancy: Disabled
Haz clic en "Next: Hosting"
En Hosting:
Storage type: Azure Storage (default)
Storage account: Selecciona gitlabauditst98765 (el que creaste)
Haz clic en "Review + create"
Haz clic en "Create"
✅ Verificación: Logic App creada

Paso 4.3: Habilitar Managed Identity
Abre la Logic App recién creada (gitlab-audit-logicapp)
En el menú lateral, ve a "Identity"
En la pestaña "System assigned":
Cambia Status a "On"
Haz clic en "Save"
Confirma haciendo clic en "Yes"
📝 GUARDAR OBJECT ID: 6. Copia el "Object (principal) ID" que aparece 7. Ejemplo: 12345678-1234-1234-1234-123456789abc 8. Guárdalo (lo necesitarás para permisos RBAC)

✅ Verificación: Managed Identity habilitada

Paso 4.4: Configurar VNET Integration (CRÍTICO)
En la Logic App, ve a "Networking" (menú lateral)
En la sección "Outbound Traffic", haz clic en "VNET integration"
Haz clic en "+ Add VNet"
Completa:
Virtual Network: gitlab-audit-vnet
Subnet: logicapp-subnet
Haz clic en "OK" o "Connect"
Espera 1-2 minutos hasta que el estado sea "Connected"
✅ Verificación: Status = "Connected" en VNET Integration

🔑 FASE 5: CONFIGURAR PERMISOS RBAC
Paso 5.1: Asignar rol en Key Vault
Ve a tu Key Vault (gitlab-audit-kv-XXXXX)
En el menú lateral, ve a "Access control (IAM)"
Haz clic en "+ Add" → "Add role assignment"
En la pestaña "Role":
En el buscador, escribe: Key Vault Secrets Officer
Selecciona "Key Vault Secrets Officer"
Haz clic en "Next"
En la pestaña "Members":
Assign access to: Selecciona "Managed identity"
Haz clic en "+ Select members"
En el panel lateral:
Subscription: Tu suscripción
Managed identity: Selecciona "Logic app (Standard)" del dropdown
Busca: gitlab-audit-logicapp
Selecciónala (aparecerá con un check)
Haz clic en "Select"
Haz clic en "Next"
Haz clic en "Review + assign"
Haz clic en "Review + assign" nuevamente
✅ Verificación: Logic App tiene rol "Key Vault Secrets Officer"

Paso 5.2: Asignar rol en DCR
En Monitor, ve a "Data Collection Rules"
Abre gitlab-audit-dcr
En el menú lateral, ve a "Access control (IAM)"
Haz clic en "+ Add" → "Add role assignment"
En la pestaña "Role":
Busca: Monitoring Metrics Publisher
Selecciona "Monitoring Metrics Publisher"
Haz clic en "Next"
En la pestaña "Members":
Assign access to: Managed identity
Haz clic en "+ Select members"
En el panel lateral:
Managed identity: Logic app (Standard)
Selecciona gitlab-audit-logicapp
Haz clic en "Select"
Haz clic en "Review + assign" → "Review + assign"
✅ Verificación: Logic App tiene rol "Monitoring Metrics Publisher"

🚀 FASE 6: DESPLEGAR WORKFLOW
Paso 6.1: Crear Workflow en Logic App
Ve a la Logic App (gitlab-audit-logicapp)
En el menú lateral, ve a "Workflows"
Haz clic en "+ Add"
Completa:
Workflow name: gitlab-audit-ingestion
State type: Stateful
Haz clic en "Create"
✅ Verificación: Workflow creado

Paso 6.2: Pegar el código del workflow
Abre el workflow recién creado (gitlab-audit-ingestion)
En la barra superior, haz clic en "Code view" (icono </>)
Borra todo el contenido que aparece por defecto
Pega este código completo:

Paso 6.3: Configurar parámetros del workflow
En el workflow, haz clic en "Designer" (cambia de Code view a Designer)
En la parte superior, haz clic en "Parameters"
Completa cada parámetro con TUS valores:
Parámetro	Valor a ingresar	Ejemplo
keyVaultName	Nombre de tu Key Vault (sin https://)	gitlab-audit-kv-98765
tokenSecretName	Nombre del secreto del token	gitlabaudit-token
checkpointSecretName	Nombre del secreto del checkpoint	gitlabaudit-lasttime
gitlabGroupId	ID de tu grupo de GitLab	116948591
dcrEndpoint	Endpoint del DCE que guardaste	https://gitlab-audit-dce-abc1.eastus-1.ingest.monitor.azure.com
dcrImmutableId	Immutable ID del DCR que guardaste	dcr-831bcb9021c04abe9389cbf646e25357
streamName	Nombre del stream	Custom-gitlabaudit_CL
Haz clic en "Save"
✅ Verificación: Todos los parámetros configurados

✅ FASE 7: PROBAR EL SISTEMA
Paso 7.1: Ejecutar workflow manualmente
En el workflow (gitlab-audit-ingestion), ve a "Overview"
Haz clic en "Run Trigger"
Haz clic en "Run"
Espera 1-3 minutos (depende de cuántos eventos tenga GitLab)
✅ Verificación: Ejecución completa sin errores

Paso 7.2: Revisar historial de ejecución
En el workflow, ve a "Run history"

Haz clic en la ejecución más reciente (debería estar en verde ✅)

Revisa cada acción:

✅ Get_GitLab_Token → Verde
✅ Get_LastTime → Verde
✅ Until_Paginate → Verde (expándelo para ver detalles)
✅ GitLab_Request → Verde (dentro del loop)
✅ POST_to_DCR → Verde
✅ Update_LastTime → Verde
Haz clic en GitLab_Request → Outputs

Verifica que el body contenga eventos de GitLab (JSON array)

✅ Verificación: Todas las acciones en verde

Paso 7.3: Verificar datos en Log Analytics
Ve a tu Log Analytics Workspace (gitlab-audit-workspace)
En el menú lateral, ve a "Logs"
Cierra el panel de queries de ejemplo (si aparece)
Pega esta query:
kql
gitlabaudit_CL
| where TimeGenerated >= ago(1h)
| take 10
Haz clic en "Run"
Resultado esperado:

Deberías ver 10 filas con eventos de GitLab
Columnas: TimeGenerated, EventTime, Action, AuthorName, etc.
✅ Verificación: Eventos de GitLab visibles en Log Analytics

Paso 7.4: Verificar transformación de TimeGenerated
En Log Analytics, ejecuta esta query:
kql
gitlabaudit_CL
| where TimeGenerated >= ago(1h)
| project TimeGenerated, EventTime
| take 5
Resultado esperado:

TimeGenerated y EventTime deben tener el mismo valor
Ejemplo:
Code
TimeGenerated: 2025-10-14T16:09:12Z
EventTime: 2025-10-14T16:09:12Z
✅ Verificación: TimeGenerated corregido por DCR

Paso 7.5: Verificar checkpoint actualizado
Ve a tu Key Vault (gitlab-audit-kv-XXXXX)
Ve a "Secrets"
Haz clic en gitlabaudit-lasttime
Haz clic en la versión actual (Current Version)
Revisa el "Secret value"
Resultado esperado:

Ya NO debería ser 2020-01-01T00:00:00Z
Debería ser una fecha reciente (el evento más nuevo + 1 segundo)
Ejemplo: 2025-10-14T16:09:13Z
✅ Verificación: Checkpoint actualizado correctamente

Paso 7.6: Verificar que NO hay duplicados
En Log Analytics, ejecuta esta query:
kql
gitlabaudit_CL
| where TimeGenerated >= ago(24h)
| summarize Count = count() by EventId
| where Count > 1
Resultado esperado:

0 filas (sin duplicados)
✅ Verificación: Sistema funcionando sin duplicados

Paso 7.7: Verificar trigger automático
En el workflow, ve a "Designer"
Haz clic en el trigger "Recurrence"
Verifica la configuración:
Interval: 10
Frequency: Minute
Status: Enabled (habilitado por defecto)
El workflow se ejecutará automáticamente cada 10 minutos. ⏰

✅ Verificación: Trigger configurado y activo

🔧 MANTENIMIENTO Y TROUBLESHOOTING
Queries útiles para monitoreo:
1. Resumen diario de eventos:
kql
gitlabaudit_CL
| where TimeGenerated >= startofday(now())
| summarize 
    TotalEvents = count(),
    UniqueEvents = dcount(EventId),
    Actions = dcount(Action),
    Users = dcount(AuthorName)
| extend Health = iff(TotalEvents == UniqueEvents, "✅ Saludable", "⚠️ Duplicados")
2. Eventos críticos de seguridad (últimas 24h):
kql
gitlabaudit_CL
| where TimeGenerated >= ago(24h)
| where Action in (
    "group_destroy_marked",
    "user_removed_from_group",
    "permission_changed",
    "token_created",
    "webhook_created"
)
| project TimeGenerated, Action, AuthorName, EntityPath, IpAddress
| order by TimeGenerated desc
3. Top usuarios más activos:
kql
gitlabaudit_CL
| where TimeGenerated >= ago(7d)
| summarize 
    Actions = count(),
    LastActivity = max(TimeGenerated)
    by AuthorName
| top 10 by Actions desc
4. Timeline de actividad:
kql
gitlabaudit_CL
| where TimeGenerated >= ago(7d)
| summarize count() by bin(TimeGenerated, 1h), Action
| render timechart
Problemas comunes y soluciones:
Error: Logic App no puede acceder a Key Vault
Code
Error: Forbidden (403)
Message: The user, group or application does not have secrets get permission
Solución:

Ve a Key Vault → Access control (IAM)
Verifica que Logic App tenga rol "Key Vault Secrets Officer"
Espera 5-10 minutos (propagación RBAC)
Ejecuta nuevamente
Error: No llegan eventos a Log Analytics
Verificar:

Logic App se ejecutó sin errores (Run history → Verde)
GitLab_Request devolvió eventos (ver Outputs)
POST_to_DCR fue exitoso (status 204 o 200)
Espera 5-10 minutos (ingesta de Log Analytics)
Query para verificar:

kql
gitlabaudit_CL
| where TimeGenerated >= ago(30m)
| count
Error: Duplicados detectados
Query para identificar:

kql
gitlabaudit_CL
| summarize Count = count() by EventId
| where Count > 1
| top 10 by Count desc
Solución:

Verifica que Compute_MaxTime use first() (no last())
Resetea el checkpoint si es necesario:
Code
Key Vault → gitlabaudit-lasttime → Set value → Última fecha conocida
Resetear checkpoint manualmente
Si necesitas recolectar eventos desde una fecha específica:

Ve a Key Vault → Secrets → gitlabaudit-lasttime
Haz clic en "+ New Version"
En Value, ingresa la fecha desde donde quieres empezar:
Ejemplo: 2025-10-01T00:00:00Z
Haz clic en "Create"
Ejecuta la Logic App manualmente
Verificar salud del sistema:
kql
let LastRun = ago(15m);
gitlabaudit_CL
| summarize 
    LastIngestion = max(TimeGenerated),
    EventsLast15min = countif(TimeGenerated >= LastRun)
| extend 
    Status = iff(LastIngestion >= LastRun, "✅ Operativo", "⚠️ Sin ingesta reciente"),
    MinutesSinceLastEvent = datetime_diff('minute', now(), LastIngestion)
📊 CHECKLIST FINAL DE VERIFICACIÓN
Code
✅ Resource Group creado: gitlab-audit-rg
✅ VNET creada con 2 subnets:
   ✅ keyvault-pe-subnet (10.0.2.0/24)
   ✅ logicapp-subnet (10.0.1.0/24) con delegación Microsoft.Web/serverFarms
✅ Key Vault creado con:
   ✅ RBAC habilitado
   ✅ Private Endpoint configurado
   ✅ Private DNS Zone vinculada
   ✅ 2 secretos creados (token + checkpoint)
✅ Log Analytics Workspace creado
✅ Data Collection Endpoint (DCE) creado
✅ Data Collection Rule (DCR) creado con:
   ✅ Transformación KQL: source | extend TimeGenerated = todatetime(EventTime)
✅ Storage Account creado para Logic App
✅ Logic App (Standard) creada con:
   ✅ Managed Identity habilitada
   ✅ VNET Integration configurada
   ✅ Workflow desplegado
   ✅ Parámetros configurados
✅ Permisos RBAC configurados:
   ✅ Logic App → Key Vault: Key Vault Secrets Officer
   ✅ Logic App → DCR: Monitoring Metrics Publisher
✅ Sistema probado:
   ✅ Ejecución manual exitosa
   ✅ Eventos llegando a Log Analytics
   ✅ TimeGenerated = EventTime (transformación OK)
   ✅ Checkpoint actualizándose
   ✅ Sin duplicados
   ✅ Trigger automático activo (cada 10 min)
🎯 PRÓXIMOS PASOS OPCIONALES
1. Agregar soporte para subgrupos de GitLab
Si necesitas capturar eventos de subgrupos, pregúntame y te doy la versión extendida del código con ForEach dinámico.

2. Configurar alertas en Sentinel
Ejemplo de alerta para usuarios externos agregados:

kql
gitlabaudit_CL
| where TimeGenerated >= ago(5m)
| where Action == "user_added_to_group"
| where IpAddress !startswith "TU_RANGO_IP_CORPORATIVA"
3. Crear Workbook en Sentinel
Panel visual con:

Timeline de actividad
Top usuarios
Acciones críticas
Métricas de ingesta
📝 VALORES PARA REUTILIZAR EN OTRO ENTORNO
Cuando repliques esto en otro entorno, solo cambia estos parámetros:

Parámetro	Dónde cambiarlo
keyVaultName	Workflow → Parameters
tokenSecretName	Workflow → Parameters
checkpointSecretName	Workflow → Parameters
gitlabGroupId	Workflow → Parameters
dcrEndpoint	Workflow → Parameters
dcrImmutableId	Workflow → Parameters
El código NO cambia, solo los parámetros. ✅

✅ RESUMEN FINAL
Has creado un sistema de clase empresarial que:

✅ Extrae audit events de GitLab automáticamente cada 10 minutos
✅ Usa ingesta incremental sin duplicados
✅ Funciona con Key Vault en Private Endpoint
✅ Usa Managed Identity y RBAC (sin credenciales hardcodeadas)
✅ Transforma datos para corregir TimeGenerated
✅ Es reutilizable y parametrizable
✅ Incluye manejo de errores y retry automático
✅ Está listo para producción
