# Distâncias Geográficas Brasil

Projeto em SQL para calcular a distância geográfica entre empresas e pontos de atendimento, otimizando a escolha do ponto mais próximo. Utiliza a fórmula de Haversine para permitir a análise espacial de dados com latitude e longitude, oferecendo soluções logísticas e de atendimento eficientes.

## Objetivo

Este projeto visa calcular a distância geográfica entre empresas e pontos de atendimento utilizando SQL. Através de consultas SQL, realiza-se a análise das localizações, considerando o endereço da empresa e os pontos de atendimento próximos, otimizando o processo de escolha do ponto de atendimento mais próximo para cada empresa.

## Estrutura do Projeto

O projeto é dividido em três partes principais:

1. **Criação das tabelas de empresas e pontos de atendimento**: Inclui dados de latitude e longitude.
2. **Cálculo das distâncias**: Utiliza a fórmula de Haversine para determinar a distância entre empresas e pontos de atendimento.
3. **Seleção do ponto de atendimento mais próximo**: Baseado na distância geográfica calculada.

Esta solução é particularmente útil para sistemas que exigem otimização logística, como empresas que precisam determinar o ponto de atendimento mais próximo para seus clientes.

## Funcionalidades

- **Criação e populamento de tabelas**: Com dados de empresas e pontos de atendimento.
- **Cálculo de distâncias geográficas**: Utilizando SQL.
- **Identificação do ponto de atendimento mais próximo**: Para cada empresa.

## Tecnologias Utilizadas

- **SQL**: Consultas e cálculos de distâncias geográficas.
- **Geolocalização**: Dados de latitude e longitude.

## Fontes de Dados

- **Informações de Empresas**: [Receita Federal do Brasil](https://arquivos.receitafederal.gov.br/dados/cnpj/dados_abertos_cnpj/?C=N;O=D)
- **Dados de Latitude e Longitude**: [IBGE - Cadastro Nacional de Endereços para Fins Estatísticos](https://www.ibge.gov.br/estatisticas/sociais/populacao/38734-cadastro-nacional-de-enderecos-para-fins-estatisticos.html?=&t=downloads)

## Como Usar

1. Faça o clone ou download deste repositório.
2. Baixe os dados das empresas do site da Receita Federal [aqui](https://arquivos.receitafederal.gov.br/dados/cnpj/dados_abertos_cnpj/?C=N;O=D).
3. Baixe os dados de latitude e longitude do site do IBGE [aqui](https://www.ibge.gov.br/estatisticas/sociais/populacao/38734-cadastro-nacional-de-enderecos-para-fins-estatisticos.html?=&t=downloads).
4. Execute as consultas SQL em seu banco de dados para gerar as tabelas e calcular as distâncias.
5. Consulte os resultados para identificar os pontos de atendimento mais próximos de cada empresa.

## Exemplo de Uso

### Relacionamentos e Calcúlos de Distância `Tabela IBGE / Receita Federal - RFB`

```sql
-- Cria uma nova tabela chamada Empresas_Endereco_Lat_Long
CREATE TABLE Empresas_Endereco_Lat_Long AS
-- Subconsulta com o nome RankedEmpresas para classificar as empresas
WITH RankedEmpresas AS (
    SELECT t1.CNPJ_BASICO,
           t1.RAZAO_SOCIAL_NOME_EMPRESARIAL,
           t1.PORTE_DA_EMPRESA,
           -- Concatena o tipo de logradouro, logradouro e número para formar o endereço completo
           CONCAT(t2.TIPO_LOGRADOURO, ' ', t2.LOGRADOURO, ' ', t2.NUMERO) AS ENDERECO,
           t3.LATITUDE,
           t3.LONGITUDE,
           -- Utiliza a função ROW_NUMBER para numerar as linhas particionadas pelo CNPJ_BASICO e ordenadas pelo logradouro
           ROW_NUMBER() OVER (PARTITION BY t1.CNPJ_BASICO ORDER BY t2.LOGRADOURO) AS rn
      FROM EMPRESAS t1
      -- Junta a tabela EMPRESAS com a tabela ESTABELECIMENTOS usando o CNPJ_BASICO como chave
      LEFT JOIN ESTABELECIMENTOS t2 ON t1.CNPJ_BASICO = t2.CNPJ_BASICO
      -- Junta a tabela ESTABELECIMENTOS com a tabela COORDENADAS_IBGE usando o logradouro como chave
      LEFT JOIN COORDENADAS_IBGE t3 ON t2.LOGRADOURO = t3.NOM_SEGLOGR
)
-- Seleciona as colunas desejadas da subconsulta RankedEmpresas onde a linha é a primeira (rn = 1)
SELECT CNPJ_BASICO,
       RAZAO_SOCIAL_NOME_EMPRESARIAL,
       PORTE_DA_EMPRESA,
       ENDERECO,
       LATITUDE,
       LONGITUDE
  FROM RankedEmpresas
 WHERE rn = 1;

 -- Cria uma nova tabela chamada pontos_atendimento para armazenar os pontos de atendimento
CREATE TABLE pontos_atendimento (
    id INT PRIMARY KEY,             -- Chave primária
    logradouro VARCHAR(255),        -- Nome do logradouro
    latitude DECIMAL(9, 6),         -- Latitude com precisão de até 6 casas decimais
    longitude DECIMAL(9, 6)         -- Longitude com precisão de até 6 casas decimais
);

-- Insere dados na tabela pontos_atendimento (Troque pela tabela dos seus pontos designados contendo Latitude e Longitude)
INSERT INTO pontos_atendimento (id, logradouro, latitude, longitude) VALUES
(1, 'Rua X, Bairro Y', -9.0232, -70.5065),
(2, 'Av. Brasil, Centro', -8.7742, -70.3415),
(3, 'Rua da Paz, Bairro Bom Jesus', -8.7711, -70.5123),
(4, 'Rua Rio Branco, Centro', -8.7718, -70.5577),
(5, 'Rua Boa Vista, Bairro Y', -8.8333, -70.3001);

-- Subconsulta para calcular as distâncias entre empresas e pontos de atendimento
WITH Distancias AS (
    SELECT e.CNPJ_BASICO,
           e.RAZAO_SOCIAL_NOME_EMPRESARIAL,
           e.PORTE_DA_EMPRESA,
           e.ENDERECO,
           e.LATITUDE AS lat_empresa,
           e.LONGITUDE AS lon_empresa,
           p.id AS ponto_id,
           p.logradouro,
           p.latitude AS lat_ponto,
           p.longitude AS lon_ponto,
           -- Calcula a distância em quilômetros usando a fórmula de Haversine
           6371 * ACOS(
               COS(RADIANS(e.LATITUDE)) * COS(RADIANS(p.latitude)) *
               COS(RADIANS(p.longitude) - RADIANS(e.LONGITUDE)) +
               SIN(RADIANS(e.LATITUDE)) * SIN(RADIANS(p.latitude))
           ) AS distancia_km
    FROM Empresas_Endereco_Lat_Long e
    -- Faz um CROSS JOIN para combinar cada empresa com todos os pontos de atendimento
    CROSS JOIN pontos_atendimento p
),
-- Subconsulta para encontrar a menor distância para cada empresa
DistanciaMinima AS (
    SELECT CNPJ_BASICO,
           RAZAO_SOCIAL_NOME_EMPRESARIAL,
           PORTE_DA_EMPRESA,
           ENDERECO,
           lat_empresa,
           lon_empresa,
           ponto_id,
           logradouro,
           distancia_km,
           -- Utiliza a função ROW_NUMBER para numerar as linhas particionadas pelo CNPJ_BASICO e ordenadas pela distância
           ROW_NUMBER() OVER (PARTITION BY CNPJ_BASICO ORDER BY distancia_km) AS rn
    FROM Distancias
)
-- Seleciona os resultados onde a linha é a primeira (rn = 1), que representa o ponto de atendimento mais próximo
SELECT *
FROM DistanciaMinima
WHERE rn = 1;
