# Documentação Completa — Projeto SNBA (Sistema Nacional de Bens Apreendidos)

---

## Índice

1. [Estrutura Geral das Consultas](#1-estrutura-geral-das-consultas)
2. [Aliases de Tabela](#2-aliases-de-tabela)
3. [Tipos de JOIN utilizados](#3-tipos-de-join-utilizados)
4. [json_build_object() — Construção de JSON](#4-json_build_object--construção-de-json)
5. [Consultas de Exploração](#5-consultas-de-exploração)
6. [Consultas de Apreensão](#6-consultas-de-apreensão)
7. [Consultas de Especificação](#7-consultas-de-especificação)
8. [Consulta Final Consolidada](#8-consulta-final-consolidada)
9. [Tabela Temporária](#9-tabela-temporária)
10. [Estrutura das Tabelas do Schema `snba`](#10-estrutura-das-tabelas-do-schema-snba)
11. [Funções PostgreSQL Utilizadas](#11-funções-postgresql-utilizadas)
12. [Erros Comuns Corrigidos no Projeto](#12-erros-comuns-corrigidos-no-projeto)
13. [Notas Finais](#13-notas-finais)

---

## 1. Estrutura Geral das Consultas

Todas as consultas seguem o padrão básico do **PostgreSQL**:

```sql
SELECT colunas
FROM tabela_principal alias
JOIN tabela_relacionada alias ON condicao
WHERE filtro;

Cláusula	Função
SELECT	Define quais colunas/expressões serão retornadas
FROM	Tabela base da consulta
JOIN	Junta dados de tabelas relacionadas
WHERE	Filtra os registros
AS	Cria um alias (apelido) para tabela ou coluna


2. Aliases de Tabela

Aliases encurtam referências a tabelas longas:


sql
sql
FROM snba.bem AS b
JOIN snba.situacao AS s ON b.situacao_id = s.id

Alias	Tabela real
b	snba.bem
be	snba.bem_especificacao
pe	snba.padrao_especificacao
s	snba.situacao
c	snba.categoria
cl	snba.classe
sub	snba.subclasse
a	snba.apreensao
apv	snba.apreensao_processo_vinculado
ta	snba.tipo_apreensao
o	snba.orgao
u	snba.unidade
la	snba.local_armazenamento
t	snba.tipo_local
e	snba.endereco
uf	snba.uf
p	snba.pais


3. Tipos de JOIN utilizados

3.1 JOIN (INNER JOIN)

Retorna somente registros que existem em ambas as tabelas:


sql
sql
JOIN snba.situacao AS s ON b.situacao_id = s.id

3.2 FULL JOIN (FULL OUTER JOIN)

Retorna todos os registros de ambas as tabelas, mesmo sem correspondência:


sql
sql
FROM snba.bem_especificacao AS be
FULL JOIN snba.bem AS b ON be.bem_id = b.id
FULL JOIN snba.situacao AS s ON b.situacao_id = s.id

3.3 Cadeia de JOINs hierárquicos

Usada para navegar a hierarquia de classificação:


sql
sql
JOIN snba.subclasse AS sub ON b.subclasse_id = sub.id
JOIN snba.classe    AS cl ON sub.classe_id   = cl.id
JOIN snba.categoria AS c  ON cl.categoria_id  = c.id

Fluxo: subclasse → classe → categoria



4. json_build_object() — Construção de JSON

Função nativa do PostgreSQL que constrói objetos JSON dinamicamente:


sql
sql
json_build_object(
    'chave1', valor1,
    'chave2', valor2
)

4.1 Estrutura simples

sql
sql
json_build_object(
    'id', b.id,
    'versao', '0.3.0',
    'dataRestricao', NULL
)

Resultado:


json
json
{
    "id": 530792,
    "versao": "0.3.0",
    "dataRestricao": null
}

4.2 Estrutura aninhada (nested)

Objetos dentro de objetos:


sql
sql
'situacao', json_build_object(
    'idSngb', s.id,
    'nome', s.nome_situacao
)

4.3 Estrutura com múltiplos níveis

sql
sql
'classificacao', json_build_object(
    'categoria', json_build_object(
        'idSngb', c.id,
        'codigo', c.codigo_categoria,
        'nome', c.nome_categoria
    ),
    'classe', json_build_object(
        'idSngb', cl.id,
        'codigo', cl.codigo_classe,
        'nome', cl.nome_classe
    ),
    'subclasse', json_build_object(
        'idSngb', sub.id,
        'codigo', sub.codigo_subclasse,
        'nome', sub.nome_subclasse
    )
)

Resultado:


json
json
{
    "classificacao": {
        "categoria": { "idSngb": 1, "codigo": "01", "nome": "Bens Móveis" },
        "classe":    { "idSngb": 5, "codigo": "01.02", "nome": "Equipamentos" },
        "subclasse": { "idSngb": 12, "codigo": "01.02.03", "nome": "Celulares" }
    }
}


5. Consultas de Exploração

5.1 Listar todos os bens

sql
sql
SELECT * FROM snba.bem;

5.2 Bens com JSON básico

sql
sql
SELECT
    *,
    json_build_object(
        'versao', '1.0.0'
    ) AS json_processo
FROM snba.bem tr;

5.3 Bens com ID e JSON básico

sql
sql
SELECT
    tr.id,
    json_build_object(
        'versao', '1.0.0'
    ) AS json_processo
FROM snba.bem tr;

5.4 Explorar padrão de especificação

sql
sql
SELECT
    pe.id,
    json_build_object(
        'id', pe.id,
        'versao', '0.3.0',
        'dataHoraAtualizacao', '2025-09-04 19:07:22.881',
        'dataHoraInclusaoSngb', '2025-06-12 18:24:13.598',
        'dataHoraAtualizacaoSngb', '2025-06-12 18:24:13.598',
        'dataRestricao', null
    ) AS json_processo
FROM snba.padrao_especificacao AS pe;

5.5 Explorar padrão de especificação com JOIN

sql
sql
SELECT *
FROM snba.padrao_especificacao AS pe
JOIN snba.bem AS b ON pe.id = b.id;

5.6 Explorar hierarquia completa (categoria → classe → subclasse)

sql
sql
SELECT
    c.id AS categoria_id,
    cla.categoria_id AS classe_categoria_id,
    cla.id AS classe_id,
    *
FROM snba.categoria AS c
FULL JOIN snba.classe AS cla ON c.id = cla.categoria_id
FULL JOIN snba.subclasse AS subcla ON cla.id = subcla.classe_id
FULL JOIN snba.bem AS b ON b.subclasse_id = subcla.id;

5.7 Explorar categoria → classe → subclasse (versão corrigida)

sql
sql
SELECT *
FROM snba.categoria AS c
FULL JOIN snba.classe AS cla ON c.id = cla.categoria_id
FULL JOIN snba.subclasse AS subcla ON cla.id = subcla.classe_id
FULL JOIN snba.bem AS b ON b.subclasse_id = subcla.id;

5.8 Explorar tabela de classes

sql
sql
SELECT * FROM snba.classe AS cl;

5.9 Explorar tabela de subclasses

sql
sql
SELECT * FROM snba.subclasse AS subcla;

5.10 Explorar tabela de situações

sql
sql
SELECT * FROM snba.situacao AS s;

5.11 Explorar tabela de especificações

sql
sql
SELECT * FROM snba.bem_especificacao;

5.12 Explorar tabela de categorias

sql
sql
SELECT * FROM snba.categoria AS c;


6. Consultas de Apreensão

6.1 Listar todas as apreensões

sql
sql
SELECT * FROM snba.apreensao AS a;

6.2 Apreensão com JOINs

sql
sql
SELECT *
FROM snba.apreensao AS a
JOIN snba.apreensao_processo_vinculado AS apv ON a.id = apv.id
JOIN snba.tipo_apreensao AS ta ON apv.id = ta.id;

6.3 Tabela temporária de apreensão

sql
sql
CREATE TEMP TABLE tmp_apreensao_join AS
SELECT a.id
FROM snba.apreensao AS a
JOIN snba.apreensao_processo_vinculado AS apv
    ON a.id = apv.apreensao_id
JOIN snba.tipo_apreensao AS ta
    ON a.tipo_apreensao_id = ta.id;

SELECT * FROM tmp_apreensao_join;

DROP TABLE IF EXISTS tmp_apreensao_join;


7. Consultas de Especificação

7.1 Especificação simples (versão inicial)

sql
sql
SELECT
    json_build_object(
        'especificacao', json_build_object(
            'padrao', json_build_object(
                'idSngb', b.subclasse_id,
                'nome', b.id
            ),
            'texto', b.texto_especificacao_bem,
            'observacoes', b.observacoes
        )
    ) AS json_especificacao
FROM snba.bem AS b;

7.2 Especificação com JOINs (versão intermediária)

sql
sql
SELECT
    json_build_object(
        'id', b.id,
        'versao', '0.3.0',
        'dataHoraAtualizacao', '2025-09-04 19:07:22.881',
        'dataHoraInclusaoSngb', '2025-06-12 18:24:13.598',
        'dataHoraAtualizacaoSngb', '2025-06-12 18:24:13.598',

        'especificacao', json_build_object(
            'padrao', json_build_object(
                'idSngb', b.padrao_especificacao_id,
                'nome', be.valor_atributo
            )
        ),

        'situacao', json_build_object(
            'idSngb', s.id,
            'nome', s.nome_situacao
        )
    ) AS json_final
FROM snba.bem AS b
FULL JOIN snba.bem_especificacao AS be ON b.id = be.bem_id
FULL JOIN snba.situacao AS s ON s.id = b.situacao_id
JOIN snba.subclasse AS subcla ON subcla.id = b.subclasse_id
JOIN snba.classe AS cla ON cla.id = subcla.classe_id
JOIN snba.categoria AS c
WHERE b.id = 530792;

7.3 Consulta com WHERE por be.id

sql
sql
SELECT
    be.id,
    json_build_object(
        'id', b.id,
        'versao', '0.3.0',
        'dataHoraAtualizacao', '2025-09-04 19:07:22.881',
        'dataHoraInclusaoSngb', '2025-06-12 18:24:13.598',
        'dataHoraAtualizacaoSngb', '2025-06-12 18:24:13.598',
        'especificacao', json_build_object(
            'padrao', json_build_object(
                'idSngb', b.subclasse_id,
                'nome', be.nome_atributo
            ),
            'texto', b.texto_especificacao_bem,
            'observacoes', b.observacoes
        ),
        'situacao', json_build_object(
            'idSngb', s.id,
            'nome', s.nome_situacao
        ),
        'classificacao', json_build_object(
            'categoria', json_build_object(
                'idSngb', c.id,
                'codigo', c.codigo_categoria,
                'nome', c.nome_categoria
            ),
            'classe', json_build_object(
                'idSngb', cl.id,
                'codigo', cl.codigo_classe,
                'nome', cl.nome_classe
            ),
            'subclasse', json_build_object(
                'idSngb', sub.id,
                'codigo', sub.codigo_subclasse,
                'nome', sub.nome_subclasse
            )
        )
    ) AS json_final
FROM snba.bem_especificacao AS be
JOIN snba.bem AS b ON be.id = b.padrao_especificacao_id
JOIN snba.situacao AS s ON b.situacao_id = s.id
JOIN snba.subclasse AS sub ON b.subclasse_id = sub.id
JOIN snba.classe AS cl ON sub.classe_id = cl.id
JOIN snba.categoria AS c ON cl.categoria_id = c.id
WHERE be.id = 530792;


8. Consulta Final Consolidada

8.1 Versão corrigida e completa

sql
sql
SELECT
    be.id,
    json_build_object(
        'id', b.id,
        'versao', '0.3.0',
        'dataHoraAtualizacao', '2025-09-04 19:07:22.881',
        'dataHoraInclusaoSngb', '2025-06-12 18:24:13.598',
        'dataHoraAtualizacaoSngb', '2025-06-12 18:24:13.598',
        'dataRestricao', NULL,

        'especificacao', json_build_object(
            'padrao', json_build_object(
                'idSngb', b.subclasse_id,
                'nome', be.nome_atributo
            ),
            'texto', b.texto_especificacao_bem,
            'observacoes', b.observacoes
        ),

        'situacao', json_build_object(
            'idSngb', s.id,
            'nome', s.nome_situacao
        ),

        'classificacao', json_build_object(
            'categoria', json_build_object(
                'idSngb', c.id,
                'codigo', c.codigo_categoria,
                'nome', c.nome_categoria
            ),
            'classe', json_build_object(
                'idSngb', cl.id,
                'codigo', cl.codigo_classe,
                'nome', cl.nome_classe
            ),
            'subclasse', json_build_object(
                'idSngb', sub.id,
                'codigo', sub.codigo_subclasse,
                'nome', sub.nome_subclasse
            )
        )
    ) AS json_final
FROM snba.bem_especificacao AS be
JOIN snba.bem       AS b   ON be.bem_id       = b.id
JOIN snba.situacao  AS s   ON b.situacao_id    = s.id
JOIN snba.subclasse AS sub ON b.subclasse_id   = sub.id
JOIN snba.classe    AS cl  ON sub.classe_id    = cl.id
JOIN snba.categoria AS c   ON cl.categoria_id   = c.id
WHERE be.id = 530792;

8.2 Mapa de JOINs

text
text
bem_especificacao (be)
        │
        │ be.bem_id = b.id
        ▼
      bem (b)
        │
        ├── b.situacao_id = s.id ──────► situacao (s)
        │
        └── b.subclasse_id = sub.id ──► subclasse (sub)
                                           │
                                           │ sub.classe_id = cl.id
                                           ▼
                                         classe (cl)
                                           │
                                           │ cl.categoria_id = c.id
                                           ▼
                                         categoria (c)

8.3 JSON de saída esperado

json
json
{
    "id": 530792,
    "versao": "0.3.0",
    "dataHoraAtualizacao": "2025-09-04 19:07:22.881",
    "dataHoraInclusaoSngb": "2025-06-12 18:24:13.598",
    "dataHoraAtualizacaoSngb": "2025-06-12 18:24:13.598",
    "dataRestricao": null,
    "especificacao": {
        "padrao": {
            "idSngb": 261,
            "nome": "Celular ou smartphone"
        },
        "texto": "Celular ou smartphone: Celulares;",
        "observacoes": "15 celulares; 197"
    },
    "situacao": {
        "idSngb": 1,
        "nome": "Disponível"
    },
    "classificacao": {
        "categoria": {
            "idSngb": 1,
            "codigo": "01",
            "nome": "Bens Móveis"
        },
        "classe": {
            "idSngb": 5,
            "codigo": "01.02",
            "nome": "Equipamentos"
        },
        "subclasse": {
            "idSngb": 12,
            "codigo": "01.02.03",
            "nome": "Celulares"
        }
    }
}

8.4 Estrutura futura — Detentor (bloco planejado)

sql
sql
'detentor', json_build_object(
    'orgao', json_build_object(
        'idSngb', o.id,
        'nome', o.nome
    ),
    'unidade', json_build_object(
        'idSngb', u.id,
        'nome', u.nome
    ),
    'localArmazenamento', json_build_object(
        'idSngb', la.id,
        'nome', la.nome,
        'tipo', json_build_object(
            'idSngb', t.id,
            'codigo', t.codigo,
            'nome', t.nome
        ),
        'endereco', json_build_object(
            'idSngb', e.id,
            'idEndereco', e.id_endereco,
            'logradouro', e.logradouro,
            'numero', e.numero,
            'complemento', e.complemento,
            'bairro', e.bairro,
            'cep', e.cep,
            'pais', e.pais,
            'localidade', json_build_object(
                'codigo', e.codigo_localidade,
                'descricao', e.descricao_localidade
            ),
            'uf', json_build_object(
                'idSngb', uf.id,
                'sigla', uf.sigla,
                'nome', uf.nome
            ),
            'pais', json_build_object(
                'idSngb', p.id,
                'sigla', p.sigla,
                'nome', p.nome
            ),
            'excluido', e.excluido
        ),
        'excluido', la.excluido
    )
)


9. Tabela Temporária

Usada para armazenar resultados intermediários durante a sessão:


sql
sql
-- Criar
CREATE TEMP TABLE tmp_apreensao_join AS
SELECT a.id
FROM snba.apreensao AS a
JOIN snba.apreensao_processo_vinculado AS apv
    ON a.id = apv.apreensao_id
JOIN snba.tipo_apreensao AS ta
    ON a.tipo_apreensao_id = ta.id;

-- Consultar
SELECT * FROM tmp_apreensao_join;

-- Remover
DROP TABLE IF EXISTS tmp_apreensao_join;

Comando	Função
CREATE TEMP TABLE ... AS	Cria tabela temporária visível apenas na sessão atual
DROP TABLE IF EXISTS	Remove a tabela sem erro caso não exista


10. Estrutura das Tabelas do Schema snba

10.1 Tabela bem

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único do bem
padrao_especificacao_id	INTEGER	FK → bem_especificacao.id
subclasse_id	INTEGER	FK → subclasse.id
situacao_id	INTEGER	FK → situacao.id
texto_especificacao_bem	TEXT	Descrição textual do bem
observacoes	TEXT	Observações adicionais

10.2 Tabela bem_especificacao

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único
bem_id	INTEGER	FK → bem.id
nome_atributo	TEXT	Nome do atributo de especificação
valor_atributo	TEXT	Valor do atributo

10.3 Tabela situacao

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único
nome_situacao	TEXT	Nome da situação do bem

10.4 Tabela categoria

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único
codigo_categoria	TEXT	Código da categoria
nome_categoria	TEXT	Nome da categoria

10.5 Tabela classe

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único
categoria_id	INTEGER	FK → categoria.id
codigo_classe	TEXT	Código da classe
nome_classe	TEXT	Nome da classe

10.6 Tabela subclasse

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único
classe_id	INTEGER	FK → classe.id
codigo_subclasse	TEXT	Código da subclasse
nome_subclasse	TEXT	Nome da subclasse

10.7 Tabela apreensao

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único da apreensão
tipo_apreensao_id	INTEGER	FK → tipo_apreensao.id

10.8 Tabela apreensao_processo_vinculado

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único
apreensao_id	INTEGER	FK → apreensao.id

10.9 Tabela tipo_apreensao

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único

10.10 Tabela padrao_especificacao

Coluna	Tipo esperado	Descrição
id	INTEGER	Identificador único

10.11 Diagrama ER (texto)

text
text
┌──────────────┐       ┌──────────────────────┐
│  categoria   │       │        bem           │
│──────────────│       │──────────────────────│
│ id (PK)      │       │ id (PK)              │
│ codigo       │       │ padrao_especificacao_id (FK)
│ nome         │       │ subclasse_id (FK)    │
└──────┬───────┘       │ situacao_id (FK)     │
       │               │ texto_especificacao  │
       │               │ observacoes          │
       ▼               └───┬──────────┬───────┘
┌──────────────┐           │          │
│   classe     │           │          │
│──────────────│           │          ▼
│ id (PK)      │     ┌────┴────┐  ┌──────────────┐
│ categoria_id │     │bem_espe-│  │  situacao    │
│ codigo       │     │cificacao│  │──────────────│
│ nome         │     │─────────│  │ id (PK)      │
└──────┬───────┘     │ id (PK) │  │ nome         │
       │             │bem_id(FK)│ └──────────────┘
       ▼             │nome_attr│
┌──────────────┐     │valor    │
│  subclasse   │     └─────────┘
│──────────────│
│ id (PK)      │
│ classe_id(FK)│
│ codigo       │
│ nome         │
└──────────────┘


11. Funções PostgreSQL Utilizadas

Função	Exemplo	Resultado
json_build_object()	json_build_object('a', 1, 'b', 2)	{"a":1,"b":2}
NULL	NULL	Valor nulo no JSON → null
AS	b.id AS json_final	Renomeia a coluna no resultado


12. Erros Comuns Corrigidos no Projeto

Erro	Correção
JOIN snba.classe AS cl ON sub.classe_id = cl.id escrito como ON c.id = cla.categoria_id	Manter a cadeia: sub → cl → c
WHERE be.id = 530792 com JOIN em be.bem_id = b.padrao_especificacao_id	Corrigido para be.bem_id = b.id
FULL JOIN usado onde bastaria JOIN	Usar JOIN quando a correspondência é obrigatória
Vírgula faltando entre blocos json_build_object	Separar cada par 'chave', valor com vírgula
JOIN snba.categoria AS c sem cláusula ON	Adicionar ON c.id = cla.categoria_id
Subclasse duplicada (sub e subcla) na mesma query	Manter apenas um alias para subclasse


13. Notas Finais

Item	Valor
Schema utilizado	snba
SGBD	PostgreSQL
Funções JSON nativas	json_build_object() (PostgreSQL 9.4+)
Padrão de nomenclatura (SQL)	snake_case para colunas e tabelas
Padrão de nomenclatura (JSON)	camelCase para compatibilidade com APIs REST
Versão do JSON	0.3.0
Bem de exemplo	id = 530792

