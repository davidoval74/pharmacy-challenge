# pharmacy-challenge
Desafio Técnico - Head de Dados (Solved - Davi Hora)

flowchart TD

%% =======================
%% CAMADA DE INGESTÃO
%% =======================

subgraph Ingestion Layer
    SFTP[SFTP Server\n(Associados.csv\nTerceros.csv\nMaestro.csv)]
    Extractor[Extractor Service\nPython + Paramiko]
end

SFTP --> Extractor


%% =======================
%% DATA LAKE
%% =======================

subgraph Data Lake (Amazon S3)
    Raw[Raw Zone\nArquivos Originais]
    Curated[Curated Zone\npharmacy.csv]
end

Extractor --> Raw


%% =======================
%% TRANSFORMAÇÃO
%% =======================

subgraph Processing Layer
    Transformer[Transformer\nPandas\nConsolidação + Regras de Negócio]
end

Raw --> Transformer
Transformer --> Curated


%% =======================
%% CARGA NO BANCO
%% =======================

subgraph Database Layer
    Loader[Loader Service\nIncremental Sync]
    Postgres[(PostgreSQL\nTabela pharmacy)]
end

Curated --> Loader
Loader --> Postgres


%% =======================
%% ORQUESTRAÇÃO
%% =======================

subgraph Orchestration
    Scheduler[Scheduler\nExecução a cada 15 minutos]
end

Scheduler --> Extractor
Scheduler --> Loader

