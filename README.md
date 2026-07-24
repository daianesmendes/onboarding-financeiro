# Onboarding Financeiro Automatizado - Pós-Assinatura do Cliente (n8n)

Automação que assume o cliente **no momento em que ele assina o contrato** e faz todo o
cadastro financeiro sozinha: valida o contrato, enriquece os dados na Receita, confirma
com um humano e cadastra cliente + assinatura no ERP de cobrança — anexando tudo no CRM.

Construído em **n8n**, integra 6 sistemas externos numa única esteira, com **validação
humana** no meio e **geração de documentos** sob demanda.

> ⚠️ **Fluxos sanitizados para portfólio.** Credenciais, IDs de credencial, domínios,
> e-mails, IPs e dados pessoais foram substituídos por placeholders. Nenhum segredo real
> está presente. Ajuste as configurações e crie suas próprias credenciais antes de usar.

## Arquitetura

```
D4Sign (assinatura)                        ┌─ Scheduler (a cada 5 min) ─┐
        │  webhook                         │ lista documentos e registra │
        ▼                                  │ o webhook de assinatura     │
┌───────────────────────────────┐         └─────────────────────────────┘
│ 1. Valida contrato             │  baixa PDF → regex (CNPJ, e-mails, telefones, razão)
│ 2. Enriquece                   │  cnpja (sócios/QSA/endereço) + SQL Server (ERP/pendência)
│ 3. Encontra a proposta         │  casa por e-mail → telefone → domínio → razão social
│ 4. APROVAÇÃO HUMANA            │  e-mail com validações + lista de propostas (sendAndWait)
└───────────────┬───────────────┘
                ▼  (analista responde o ID da proposta)
┌───────────────────────────────┐
│ 5. Cadastra no ERP de cobrança │  cliente + assinatura (plano base + extras + adesão)
│ 6. Gera PDF do QSA             │  encoder de PDF escrito à mão em JS puro
│ 7. Anexa no CRM (Pipedrive)    │  contrato + QSA + NF/Serasa/Boleto
│ 8. E-mails de confirmação      │
└───────────────────────────────┘
```

## Workflows

| Arquivo | Papel |
|---|---|
| `financeiro-01-scheduler-validador.json` | A cada 5 min, lista documentos no D4Sign, filtra os que aguardam assinatura e registra o webhook de assinatura em cada um (dedup via `staticData`). |
| `financeiro-02-novas-contratacoes.json`  | Esteira principal: da assinatura até o cadastro no ERP, com aprovação humana, geração do QSA e anexo no CRM. |

## Destaques técnicos

- **Gerador de PDF em JavaScript puro** (nó de código): monta o documento do QSA
  do zero — objetos, streams, tabela `xref`, larguras de glifos da Helvetica e quebra
  de linha por largura real de texto. Sem bibliotecas externas.
- **Parsing de contrato por regex**: extrai CNPJ, razão social, e-mails e telefones de
  blocos específicos do PDF (responsável, boletos, implantação), com deduplicação e
  filtro de e-mails internos/gratuitos.
- **Casamento de proposta em cascata**: encontra a proposta certa do cliente por
  prioridade — e-mail → telefone (últimos 8 dígitos) → domínio → tokens da razão social.
- **Human-in-the-loop** com `sendAndWait`: o financeiro recebe as validações e responde
  com o ID da proposta; um segundo e-mail coleta anexos (NF, Serasa, Boleto) por formulário.
- **Enriquecimento de dados**: consulta a **cnpja** (situação na Receita, quadro
  societário, endereço) e cruza com o endereço declarado na proposta.
- **De-para de produtos e regras fiscais**: mapeia planos/extras para os IDs do ERP,
  trata Simples Nacional (retenção de impostos) e recorrente vs. à vista.

## Pré-requisitos

- **n8n** (self-hosted)
- **D4Sign** — assinatura eletrônica (webhook + API)
- **cnpja** — consulta de CNPJ (API pública)
- **Microsoft SQL Server** — base de propostas/ERP
- **Superlógica** — ERP financeiro / cobrança (API)
- **Pipedrive** — CRM
- **Gmail** — envio e aprovação humana (`sendAndWait`)

## Configuração

1. **Importe** os dois arquivos de `workflows/` no seu n8n.
2. **Crie as credenciais** (os IDs foram removidos na sanitização — o n8n vai pedir para vincular):
   - D4Sign (HTTP Header Auth), Superlógica (HTTP Header Auth), Microsoft SQL Server, Gmail (OAuth2), Pipedrive.
3. **Substitua os placeholders** pelos seus valores:

   | Placeholder | Troque por |
   |---|---|
   | `n8n.example.com` | domínio do seu n8n (usado no webhook do D4Sign) |
   | `example.com` (e-mails `financeiro@`, filtro de e-mail interno) | seu domínio corporativo |
   | `suaempresa.superlogica.net` / `suaempresa.pipedrive.com` | suas contas |
   | `CRED_*` | vínculo com as credenciais criadas no passo 2 |

4. **Ajuste as constantes de negócio** ao seu ambiente (nas seções de "de-para" do nó
   `Monta dados + de-para` e `Monta body assinatura`): IDs de plano, IDs de produto e
   nomes dos planos.
5. **Tabelas / views esperadas** no SQL Server: `gpc_proposta_comercial`,
   `gpc_proposta_comercial_plano_extra`, `plano`, `empresa`, `view_empresa_bill`.

Feito isso, ative os workflows e assine um contrato de teste no D4Sign: o e-mail de
validação deve chegar ao financeiro antes de qualquer cadastro no ERP.

## Stack

`n8n` · `D4Sign` · `cnpja` · `SQL Server` · `Superlógica` · `Pipedrive` · `Gmail`
