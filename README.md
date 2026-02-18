# Desafio Técnico - Head de Dados (Solved - Davi Hora)

## Visão Geral

Este desafio tem como objetivo avaliar minhas habilidades em engenharia de dados, integração de sistemas e arquitetura de soluções. Foi construido um pipeline de dados completo que extrai informações de um servidor SFTP, transforma e consolida os dados, armazena em um Data Lake (S3) e sincroniza com um banco de dados PostgreSQL.

## Stack Tecnológica

### ☁️ Cloud & Infraestrutura
- AWS Lambda
- AWS S3
- AWS CloudWatch
- AWS EC2 (Windows - PostgreSQL)
- AWS EC2 (Linux - Lambda Layers)

### 🗄 Banco de Dados
- PostgreSQL

### 🐳 Containerização
- Docker (Lambda Layer)

## Contexto do Negócio

Uma rede de farmácias precisa consolidar dados de **Associados** e **Terceiros** em uma base única para análises e operações. Os dados chegam em arquivos CSV através de um servidor SFTP e precisam ser processados, transformados e disponibilizados em um banco de dados relacional.



### 1. Extração (SFTP → S3) 

#### Estratégia adotada:

🔄 Estratégia de Processamento Incremental 

A extração do SFTP foi implementada de forma incremental e idempotente.

Infraestrutura AWS -> AWS LAMBDA (PYTHON FUNCTION) -> S3 (Read: Function lambda Extract)

Foi adotado os metodos de STG (Stage) alem da criação de medalhas no S3 AWS.

Leitura dos arquivos SFTP, consolidado no S3;

Arquivos e buckets:
- Associados.csv >> Amazon S3/Buckets/ b2list /Associados/*.csv
 (https://b2list.s3.us-east-2.amazonaws.com/Associados/)
- Terceros.csv >> Amazon S3/Buckets / b2list /Terceros/*.csv
 (https://b2list.s3.us-east-2.amazonaws.com/Terceiros/)
- Maestro.csv >> Amazon S3/Buckets/ b2list /Maestro/*.csv 
(https://b2list.s3.us-east-2.amazonaws.com/Maestro/)


Lambda AWS Codigo Python:
Lambda >> Funções >> Extraction 
Hash SHA256 = 'KS11PMDn60cSDtDtV6P5QKj2LNX7RoVClXzt5X4yo70='


AWS Monitoring Cloud Watch:
(https://us-east-2.console.aws.amazon.com/cloudwatch/home?region=us-east-2#logStream:group=/aws/lambda/Extraction)





### 2. Mapeamento de Dados

O arquivo `pharmacy.csv` contem as seguintes colunas extraídas/derivadas dos arquivos fonte:

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


#### Estratégia adotada:

🔄 Estratégia de Processamento de Sobrescrita

Infraestrutura AWS -> AWS LAMBDA (PYTHON FUNCTION) -> S3 (Read: Function Transaction)

Foi adotado os metodos de STG (Stage) alem da criação de medalhas no S3 AWS.

Leitura dos arquivos STG, Consolidado no S3 (GOLD):

Arquivos e buckets:
- Amazon S3/Buckets/b2list/Associados/*.csv >> AmazonS3/Buckets/b2list/Associados-gold/*csv 
((https://b2list.s3.us-east-2.amazonaws.com/Associados-Gold/))
- Amazon S3/Buckets/b2list/Terceiros/*.csv >> AmazonS3/Buckets/b2list/Terceiros-gold/*csv 
((https://b2list.s3.us-east-2.amazonaws.com/Terceiros-Gold/))
- Amazon S3/Buckets/b2list/Maestro/*.csv >> Amazon S3/Buckets/b2list/Maestro-Gold/*.csv 
(https://b2list.s3.us-east-2.amazonaws.com/Maestro-Gold/)

 Pré-Postgres S3:
  Amazon S3/Buckets/b2list/Pharmacy-gold/*csv ((https://b2list.s3.us-east-2.amazonaws.com/Pharmacy-Gold/))


Lambda AWS Codigo Python:
Lambda >> Funções >> Transaction
Hash SHA256 = 'JrBeDlxfxncvZ1GB7hxtIKInnw8y7GFkNCjLHfDtMPY='

AWS Monitoring Cloud Watch:
(https://us-east-2.console.aws.amazon.com/cloudwatch/home?region=us-east-2#logStream:group=/aws/lambda/Transaction)

### 3. Carga no PostgreSQL

Function Lambda Criar um serviço que:

- Le o arquivo `pharmacy.csv` do S3
- Insere os dados na tabela `pharmacy` com o seguinte schema:

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


#### Estratégia adotada:

🔄 Estratégia de Processamento incremental

Foi adotado o metodo load, utilizando os dados previamente armazenados no AWS S3 para o Postgres via lambda;

Infraestrutura AWS -> AWS LAMBDA (PYTHON FUNCTION) -> S3 (Read: Function LoadPostgres)

Arquivos e buckets:

- Amazon S3/Buckets/b2list/Pharmacy-gold/*csv >> Postgres B2list.STG_Pharmacy.db


Lambda AWS Codigo Python:
Lambda >> Funções >> LoadPostgres
Hash SHA256 = '4Ja7J7fA1kVbUlztWgAh6qTTEMQEVyx2SzZBIqpWrZc='


AWS Monitoring Cloud Watch:
(https://us-east-2.console.aws.amazon.com/cloudwatch/home?region=us-east-2#logStream:group=/aws/lambda/Load_Postgres312)


### 4. Sincronização Incremental

Implementado rotinas no postgres que:

- Execute a cada **5 minutos**
- Verifique se houve alterações no arquivo `pharmacy.csv` no S3 pareado com `b2list.stg_pharmacy.db`
-  **atualização incremental** no banco de dados:
  - Novos registros → INSERT com `created_at` = timestamp atual e `enabled = true`
  - Registros alterados → UPDATE com `last_modified` = timestamp atual
  - Registros removidos → UPDATE `enabled = false` (exclusão lógica / soft delete)

Procedure Postgres:

```sql

CREATE OR REPLACE PROCEDURE sync_pharmacy()
LANGUAGE plpgsql
AS $$
BEGIN
    -- MERGE update/insert
    MERGE INTO public.pharmacy AS target
    USING (
        SELECT *
        FROM (
            SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY data_transf DESC) AS rn
            FROM public.stg_pharmacy
        ) ranked
        WHERE rn = 1 AND data_transf > current_date
    ) AS stage
    ON target.code_pharmacy = stage.id
    WHEN MATCHED THEN
        UPDATE SET
            trade_name = stage.NOME,
            category = stage.category,
            corporate_name = stage.NOME_FANTASIA,
            nit = stage.NIT,
            lat = stage.LATITUDE,
            lon = stage.LONGITUDE,
            enabled = stage.enabled,
			last_modified = data_transf
			
    WHEN NOT MATCHED THEN
        INSERT (code_pharmacy, trade_name, category, corporate_name, nit, lat, lon, enabled)
        VALUES (stage.id, stage.NOME, stage.category, stage.NOME_FANTASIA, stage.NIT, stage.LATITUDE, stage.LONGITUDE, TRUE);

    -- DELETE registros obsoletos
    DELETE FROM public.pharmacy
    WHERE code_pharmacy NOT IN (
        SELECT id
        FROM (
            SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY data_transf DESC) AS rn
            FROM public.stg_pharmacy
        ) ranked
        WHERE rn = 1 AND data_transf > current_date
    );
END;
$$;
```


**Importante sobre Soft Delete:**
- A coluna `enabled` (BOOLEAN) controla a exclusão lógica
- `enabled = true` → Registro ativo
- `enabled = false` → Registro excluído logicamente
- Registros que existiam no banco mas não estão mais no arquivo fonte devem ter `enabled` alterado para `false`
- NUNCA realizar DELETE físico dos registros

## Entrega

1. Repositório Git (GitHub) : https://github.com/davidoval74/pharmacy-challenge
2. Acesso de leitura para o avaliador :  <mislene.dalila@b2list.com>
3. Branch `main` com a solução final no Github
4. README com instruções claras de execução

## Dúvidas

Em caso de dúvidas sobre o desafio, entre em contato através do e-mail fornecido pelo recrutador.

---

**Obrigado B2list! 🚀**

