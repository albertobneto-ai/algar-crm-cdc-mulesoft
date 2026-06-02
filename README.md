# Algar CRM CDC Sync — MuleSoft 4 Project

## Arquitetura

```
Sales Cloud (Account/Lead/Contact/Opportunity)
    │
    ├── CDC Event (nativo) ──→ MuleSoft listener
    │                              │
    │                              ├── Account → CRM Algar (NATIVE.ACCOUNTS)
    │                              │                → Snowflake (LEGACY_ACCOUNTS)
    │                              │                → Back-update SF (Legacy_CRM_Id__c)
    │                              │
    │                              └── Lead/Contact/Opp → Snowflake direto
    │                                   (CRM_LEADS / CRM_CONTACTS / CRM_OPPORTUNITIES)
    │
    └── _Home streams ──→ Data Cloud ──→ Agentforce

CRM Algar UI → POST /api/crm-algar/account → SF + Snowflake (bidirecional)
```

## Flows

| Flow | Trigger | Origem → Destino |
|------|---------|-----------------|
| account-cdc-sf-to-crm-algar | CDC AccountChangeEvent | SF → CRM Algar → Snowflake → back-update SF |
| lead-cdc-sf-to-snowflake | CDC LeadChangeEvent | SF → Snowflake CRM_LEADS |
| contact-cdc-sf-to-snowflake | CDC ContactChangeEvent | SF → Snowflake CRM_CONTACTS |
| opportunity-cdc-sf-to-snowflake | CDC OpportunityChangeEvent | SF → Snowflake CRM_OPPORTUNITIES |
| crm-algar-create-account-to-sf | HTTP POST webhook | CRM Algar UI → SF + Snowflake |
| ingest-to-data-lake | Sub-flow | CRM Algar nativo → LEGACY_ACCOUNTS |

## Pré-requisitos

1. **Anypoint Studio 7.x** (download: https://www.mulesoft.com/lp/dl/anypoint-mule-studio)
2. **Anypoint Platform account** (trial: https://anypoint.mulesoft.com/login/signup)
3. **Snowflake JDBC Driver** (incluído no pom.xml como dependência Maven)
4. **CDC habilitado na Dev Org** (já configurado para Account, Lead, Contact, Opportunity)

## Setup — Anypoint Studio

### 1. Importar projeto
- File → Import → Maven-based Mule Project
- Selecionar a pasta do projeto
- Aguardar resolução de dependências

### 2. Configurar credenciais
Editar `src/main/resources/application.properties`:
- `sf.password` = sua senha do Salesforce
- `sf.securityToken` = seu security token
- `snowflake.password` = AlgarAlberto2026sf!

### 3. Testar localmente
- Run As → Mule Application
- Verificar logs: "[CDC-Account] Sincronizado" etc.
- Criar um Account no SF e verificar se aparece no Snowflake

### 4. Deploy no CloudHub
- Right-click projeto → Anypoint Platform → Deploy to CloudHub
- Selecionar environment e runtime
- Configurar secure properties para senhas

## Credenciais (referência)

### Salesforce Dev Org
- Username: albertobneto.ce8a76342d1d@agentforce.com
- Instance: orgfarm-6450ce60e0-dev-ed.develop.my.salesforce.com
- Org ID: 00DgK00000PUJwTUAX

### Snowflake
- Account: rajlbkg-yk87012 (AWS sa-east-1)
- Username: ALBERTOBNETO
- Database: ALGAR_CRM_LAKE
- Warehouse: COMPUTE_WH

## Vantagens sobre a camada Heroku atual

| Aspecto | Heroku (atual) | MuleSoft (este projeto) |
|---------|---------------|------------------------|
| Trigger | Apex @future callout | CDC nativo (zero Apex) |
| Conector SF | REST API manual | Connector nativo |
| Conector Snowflake | fetch + SQL strings | JDBC tipado |
| Error handling | try/catch manual | Built-in retry + DLQ |
| Monitoramento | Logs Heroku | Anypoint Monitoring |
| Camadas | 4 (SF→Trigger→Heroku→SF) | 3 (SF→CDC→MuleSoft) |

## Após deploy do MuleSoft

1. Desativar os Apex Triggers na Dev Org (AccountSyncTrigger, LeadSyncTrigger, etc.)
2. Remover Remote Site Setting "Heroku_MCP_Server"
3. Atualizar CRM Algar UI para apontar webhook para o CloudHub URL
4. Manter o Heroku apenas para endpoints de consulta e ferramentas
