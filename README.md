# Desafio Técnico - Head de Dados (Solved - Davi Hora)

## Visão Geral

Este desafio tem como objetivo avaliar minhas habilidades em engenharia de dados, integração de sistemas e arquitetura de soluções. Você deverá construir um pipeline de dados completo que extrai informações de um servidor SFTP, transforma e consolida os dados, armazena em um Data Lake (S3) e sincroniza com um banco de dados PostgreSQL.

## Contexto do Negócio

Uma rede de farmácias precisa consolidar dados de **Associados** e **Terceiros** em uma base única para análises e operações. Os dados chegam em arquivos CSV através de um servidor SFTP e precisam ser processados, transformados e disponibilizados em um banco de dados relacional.

## Objetivos

### 1. Extração (SFTP → S3)

🔄 Estratégia de Processamento Incremental

A extração do SFTP foi implementada de forma incremental e idempotente.

Estratégia adotada:

Infraestrutura AWS -> AWS LAMBDA -> PYTHON FUNCTION -> S3

Leitura dos arquivos SFTP, consolidado no S3

Comparação com o estado atual do banco

Classificação dos registros em:

Novos

Alterados

Removidos (soft delete)

Regras aplicadas:
🆕 Novo registro

INSERT

created_at = NOW()

enabled = true

🔁 Registro alterado

UPDATE

last_modified = NOW()

❌ Registro removido

UPDATE enabled = false

last_modified = NOW()

Nunca é realizado DELETE físico




### 2. Mapeamento de Dados

O arquivo `pharmacy.csv` deve conter as seguintes colunas extraídas/derivadas dos arquivos fonte:

| Coluna Destino | Origem | Descrição |
|----------------|--------|-----------|
| `code_pharmacy` | ID (todos os arquivos) | Código único da farmácia |
| `nit` | NIT (Maestro.csv) | Número de Identificação Tributária |
| `trade_name` | NOME (Associados/Terceros) | Nome comercial |
| `corporate_name` | NOME_FANTASIA (Maestro.csv) | Razão social |
| `lat` | LATITUDE (Associados/Terceros) | Latitude |
| `lon` | LONGITUDE (Associados/Terceros) | Longitude |
| `category` | Nome do arquivo de origem | `ASSOCIADO` ou `TERCERO` |
| `enabled` | OBSERVACAO (Associados/Terceros) | Status ativo/inativo |

**Regra para `enabled`:**
- Se OBSERVACAO contém "Ativo", "Em dia", "Cadastro ativo", "Verificado" ou "Contrato vigente" → `true`
- Caso contrário → `false`

### 3. Carga no PostgreSQL

Criar um serviço que:

- Leia o arquivo `pharmacy.csv` do S3
- Insira os dados na tabela `pharmacy` com o seguinte schema:

```sql
CREATE TABLE pharmacy (
    id SERIAL PRIMARY KEY,
    code_pharmacy VARCHAR(20) UNIQUE NOT NULL,
    trade_name VARCHAR(255) NOT NULL,
    category VARCHAR(20) NOT NULL,
    corporate_name VARCHAR(255),
    address VARCHAR(500) DEFAULT 'não informado',
    nit VARCHAR(20),
    lat DECIMAL(10, 6),
    lon DECIMAL(10, 6),
    enabled BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_modified TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_pharmacy_code ON pharmacy(code_pharmacy);
CREATE INDEX idx_pharmacy_category ON pharmacy(category);
CREATE INDEX idx_pharmacy_enabled ON pharmacy(enabled);
```

### 4. Sincronização Incremental

Implementar uma rotina que:

- Execute a cada **15 minutos**
- Verifique se houve alterações no arquivo `pharmacy.csv` no S3
- Realize **atualização incremental** no banco de dados:
  - Novos registros → INSERT com `created_at` = timestamp atual e `enabled = true`
  - Registros alterados → UPDATE com `last_modified` = timestamp atual
  - Registros removidos → UPDATE `enabled = false` (exclusão lógica / soft delete)

**Importante sobre Soft Delete:**
- A coluna `enabled` (BOOLEAN) controla a exclusão lógica
- `enabled = true` → Registro ativo
- `enabled = false` → Registro excluído logicamente
- Registros que existiam no banco mas não estão mais no arquivo fonte devem ter `enabled` alterado para `false`
- NUNCA realizar DELETE físico dos registros

## Requisitos Técnicos

### Tecnologias 

Escolha uma das seguintes stacks:

- **Python** (recomendado: pandas, boto3, paramiko, psycopg2, SQLAlchemy)
- **Databricks** (PySpark, Delta Lake)

### Requisitos Obrigatórios

1. **Código limpo e bem documentado**
2. **Tratamento de erros** 
6. **Docker** para containerização da aplicação
7. **README** com instruções de execução

### Requisitos Desejáveis

- Documentação de arquitetura (diagrama)



## Credenciais

As credenciais de acesso serão fornecidas separadamente:

- **SFTP**: host, porta, usuário e senha
- **S3**: utilizar uma conta pessoal
- **PostgreSQL**: local

## Critérios de Avaliação

| Critério | Peso |
|----------|------|
| Funcionalidade completa | 45% |
| Qualidade do código | 25% |
| Arquitetura e design | 20% |
| Documentação | 10% |

## Prazo

- **Entrega**: 7 dias corridos a partir do recebimento das credenciais
- **Apresentação**: Agendar call de 30-45 min para apresentação da solução

## Entrega

1. Repositório Git (GitHub)
2. Acesso de leitura para o avaliador
3. Branch `main` com a solução final
4. README com instruções claras de execução

## Dúvidas

Em caso de dúvidas sobre o desafio, entre em contato através do e-mail fornecido pelo recrutador.

---

**Boa sorte! 🚀**
