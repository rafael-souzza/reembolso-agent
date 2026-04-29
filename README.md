# 💸 Automação de Reembolsos com IA

> Pipeline end-to-end de processamento de reembolsos com OCR duplo, detecção de fraude via LLM, aprovação automática por regras ANS e pagamento via Stripe — tudo orquestrado em n8n.

---

## 🎯 O Problema que Resolve

Processos de reembolso manuais são lentos, sujeitos a erros e difíceis de auditar. Este agente automatiza todo o ciclo — da nota fiscal ao pagamento — com inteligência artificial para leitura de documentos, validação de regras e detecção de fraude, reduzindo intervenção humana para casos realmente excepcionais.

---

## ⚙️ Como Funciona

### Pipeline completo (27 nós)

```
📥 Webhook — recebe upload do documento
    ↓
🔧 Normalização do payload + geração de request_id único
    ↓
✅ Validação do arquivo (formato, tamanho)
    ↓
☁️ Upload para S3
💾 Registro inicial no Postgres
    ↓
👁️ OCR — Google Vision (primário)
    ↓ (se confiança baixa)
🔄 OCR Fallback — AWS Textract
    ↓
🤖 LLM Parsing — Claude extrai dados estruturados do texto
    ↓
📋 Validação de schema dos dados extraídos
    ↓
📜 Busca das regras do plano no Postgres
⚖️ Aplicação das regras ANS
    ↓
🕵️ Detecção de Fraude — análise via LLM
    ↓
🚦 Decisão: Aprovação automática ou revisão manual?
    ↓
    ├── ✅ Auto-aprovado → Pagamento via Stripe
    │                    → Notificação por email (SendGrid)
    │                    → Atualização de status no Postgres
    │
    └── 👤 Revisão manual → Alerta no Slack para analista
                          → Dead Letter Queue para reprocessamento
```

---

## 🧠 Inteligência Artificial

Duas chamadas de LLM distintas no pipeline:

**1. LLM Parsing (Claude)**
- Recebe o texto bruto extraído pelo OCR
- Extrai campos estruturados: valor, data, fornecedor, categoria, CNPJ, etc.
- Retorna JSON validado contra schema

**2. Detecção de Fraude (Claude)**
- Analisa os dados extraídos em busca de anomalias
- Cruza com histórico do beneficiário no banco de dados
- Sinaliza inconsistências: valor fora do padrão, fornecedor suspeito, duplicidade

---

## 🛠️ Stack Técnica

| Camada | Tecnologia |
|---|---|
| Orquestração | n8n |
| OCR Primário | Google Cloud Vision API |
| OCR Fallback | AWS Textract |
| LLM / IA | Anthropic Claude (parsing + fraude) |
| Armazenamento | AWS S3 (documentos) + PostgreSQL (dados) |
| Pagamentos | Stripe API |
| Notificações | SendGrid (email) + Slack (alertas internos) |
| Resiliência | Dead Letter Queue + Error Trigger |

---

## 🔒 Resiliência e Confiabilidade

- **OCR duplo com fallback:** se o Google Vision retorna baixa confiança, o Textract é acionado automaticamente
- **Dead Letter Queue:** requisições com erro são salvas no Postgres para reprocessamento sem perda de dados
- **Error Trigger:** qualquer falha no pipeline dispara alerta automático no Slack
- **request_id único:** rastreabilidade completa de cada reembolso do início ao fim

---

## 📦 Estrutura do Repositório

```
📁 reembolso-agent/
├── reembolso_n8n_workflow.json   # Workflow completo — importar no n8n
└── README.md
```

---

## 🚀 Como Usar

### Pré-requisitos

- n8n instalado (self-hosted ou cloud)
- Conta Google Cloud com Vision API habilitada
- Conta AWS com Textract habilitado e bucket S3 criado
- Banco PostgreSQL provisionado (Supabase funciona perfeitamente)
- Conta Stripe para pagamentos
- SendGrid para envio de emails
- Bot do Slack configurado para alertas

### Instalação

1. Importe `reembolso_n8n_workflow.json` no n8n via **Workflows → Import from file**
2. Configure as credenciais no painel do n8n:
   - Google Cloud (Vision API)
   - AWS (Textract + S3)
   - PostgreSQL
   - Stripe
   - SendGrid
   - Slack
3. Execute o schema SQL para criar as tabelas necessárias (ver abaixo)
4. Ative o workflow e envie uma requisição de teste via `POST /webhook/upload`

### Schema do Banco de Dados

```sql
-- Tabela principal de reembolsos
CREATE TABLE reimbursements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id VARCHAR(50) UNIQUE NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    document_url TEXT,
    extracted_data JSONB,
    fraud_analysis JSONB,
    amount DECIMAL(10,2),
    plan_rules JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Tabela de regras por plano
CREATE TABLE plan_rules (
    id SERIAL PRIMARY KEY,
    plan_id VARCHAR(50) NOT NULL,
    category VARCHAR(100),
    max_amount DECIMAL(10,2),
    requires_receipt BOOLEAN DEFAULT true,
    auto_approve_limit DECIMAL(10,2),
    active BOOLEAN DEFAULT true
);

-- Dead Letter Queue
CREATE TABLE dead_letter_queue (
    id SERIAL PRIMARY KEY,
    request_id VARCHAR(50),
    error_message TEXT,
    payload JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 👤 Autor

**Rafael Souzza** — AI/ML Engineer & Automation Consultant  
📍 São Paulo, SP  
🔗 [LinkedIn](https://linkedin.com/in/seu-perfil) | [GitHub](https://github.com/seu-usuario)
