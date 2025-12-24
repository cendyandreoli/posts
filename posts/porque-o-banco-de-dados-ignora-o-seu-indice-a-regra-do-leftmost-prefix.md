---
id: 1bd83eec-e59c-48ec-9697-2d0bc8fee5ae
title: "Porque o Banco de Dados Ignora o seu Índice: A Regra do "Leftmost Prefix""
slug: porque-o-banco-de-dados-ignora-o-seu-indice-a-regra-do-leftmost-prefix
summary: "Criar índices compostos sem entender B-Trees é como organizar uma lista telefônica pela última letra do sobrenome"
tags: ["SQL", "Database", "Performance", "Postgres", "MySQL", "Indexing"]
cover_image_url: ""
published_at: 2025-12-24T01:01:14.94+00:00
reading_time_minutes: 6
featured: false
status: published
view_count: 0
created_at: 2025-12-24T01:01:17.69577+00:00
updated_at: 2025-12-24T01:01:18.658816+00:00
---

# Porque o Banco de Dados Ignora o seu Índice: A Regra do "Leftmost Prefix"

Criar índices compostos sem entender B-Trees é como organizar uma lista telefônica pela última letra do sobrenome

---

Você tem uma query lenta. Você decide ser proativo.
Você olha para o `WHERE` da query:

```sql
SELECT * FROM orders
WHERE status = 'PENDING'
  AND created_at > '2023-01-01';
```

Você pensa: *"Vou criar um índice que cobre tudo!"* e corre:

```sql
CREATE INDEX idx_all_orders ON orders (tenant_id, status, created_at);
```

Você roda a query novamente. Ela continua lenta.
Você roda o `EXPLAIN ANALYZE` e vê o temido **Seq Scan** (ou **Full Table Scan**). O banco de dados ignorou completamente o seu índice novo.

Porquê? Porque você violou a regra mais importante da indexação em B-Trees: a regra do **Leftmost Prefix** (Prefixo Mais à Esquerda).

---

### A Anatomia de um Índice Composto

Para entender o erro, precisamos entender como o banco guarda os dados.
Um índice composto `(A, B, C)` **não** são três listas separadas. É **uma** lista ordenada por A, depois por B, depois por C.

A melhor analogia do mundo real é uma **Lista Telefônica Impressa**.
Imagine uma lista telefônica organizada por: `(Sobrenome, Nome)`.

1.  Silva, Ana
2.  Silva, Beto
3.  Souza, Carlos

#### O Cenário de Sucesso
Se eu pedir: *"Procure todos os **Silva**"*.
Você vai direto para a letra "S". O índice funciona perfeitamente.

#### O Cenário de Fracasso (A Violação)
Se eu pedir: *"Procure todos os **Beto**"*.
Você **não consegue** usar a organização da lista. Você não sabe se é "Almeida, Beto" ou "Zambrota, Beto".
Você é obrigado a ler **a lista inteira**, página por página, linha por linha, para encontrar todos os "Betos".

Isso é um **Full Table Scan**.

---

### A Regra do Leftmost Prefix

A regra é binária: **O banco de dados só pode usar o índice se a sua query usar as colunas na ordem exata da definição, começando da esquerda, sem pular colunas.**

Vamos analisar o índice: `idx_orders (tenant_id, status, created_at)`.

Aqui está a "Tabela da Verdade" da performance:

| Query WHERE ... | Usa o Índice? | O que acontece? |
| :--- | :---: | :--- |
| `tenant_id = 1` | ✅ **SIM** | Começa pelo prefixo (1ª coluna). Perfeito. |
| `tenant_id = 1 AND status = 'P'` | ✅ **SIM** | Segue a ordem: 1ª ➔ 2ª coluna. |
| `tenant_id = 1 AND status = 'P' AND created_at > ...` | ✅ **SIM** | Full Match! Usa as 3 colunas para filtrar. |
| `status = 'P'` | ❌ **NÃO** | Pulou o `tenant_id` (o início). Index Scan impossível. |
| `created_at > ...` | ❌ **NÃO** | Pulou `tenant_id` e `status`. Full Table Scan. |
| `tenant_id = 1 AND created_at > ...` | ⚠️ **PARCIAL** | Filtra pelo `tenant_id`, mas escaneia o resto. |

---

### O Erro Clássico do Multi-Tenant

Em sistemas SaaS, quase todas as tabelas têm `tenant_id`.
É muito comum os desenvolvedores criarem índices começando com `tenant_id` para "segurança".

```sql
-- Índice: (tenant_id, created_at)
```

E a query do dashboard administrativo geral (Super Admin) é:
```sql
-- "Mostre os últimos usuários criados na plataforma toda"
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;
```

Como essa query não filtra por um `tenant_id` específico, o índice `(tenant_id, created_at)` é **inútil**. O banco vai ler a tabela inteira para achar os registros mais recentes, mesmo que eles estejam ordenados *dentro* de cada tenant.

---

### Como Corrigir?

Você tem duas estratégias:

#### 1. Crie Índices Específicos para os Padrões de Acesso
Se você tem queries que buscam apenas por `status`, você precisa de um índice que comece com `status`.

```sql
CREATE INDEX idx_status ON orders (status);
```

#### 2. Reordene o Índice Composto (Ajuste de Cardinalidade)
A "Cardinalidade" é o número de valores únicos numa coluna.
Geralmente, queremos colocar colunas com **alta seletividade** (que filtram muito) no início, ou colunas que são **sempre** usadas (`tenant_id`) no início.

Se a sua query é **sempre** `WHERE tenant_id = ? AND status = ?`, a ordem `(tenant_id, status)` ou `(status, tenant_id)` funciona para a busca.
Mas se você às vezes busca **só** por `status` e **nunca** só por `tenant_id`, então o `status` deve vir primeiro: `(status, tenant_id)`.

---

### Conclusão

Índices não são mágicos; são estruturas de dados ordenadas rígidas.

Sempre que criar um índice composto, faça a pergunta da Lista Telefônica:
*"Se eu remover a primeira coluna da minha busca, eu ainda consigo achar o dado sem ler o livro todo?"*

Se a resposta for não, o seu índice está apenas ocupando espaço em disco e RAM.

---

## English Version

You have a slow query. You decide to be proactive.
You look at the `WHERE` clause of the query:

```sql
SELECT * FROM orders
WHERE status = 'PENDING'
  AND created_at > '2023-01-01';
```

You think: *"I'll create an index that covers everything!"* and run:

```sql
CREATE INDEX idx_all_orders ON orders (tenant_id, status, created_at);
```

You run the query again. It's still slow.
You run `EXPLAIN ANALYZE` and see the dreaded **Seq Scan** (or **Full Table Scan**). The database completely ignored your new index.

Why? Because you violated the most important rule of indexing in B-Trees: the **Leftmost Prefix** rule.

---

### The Anatomy of a Composite Index

To understand the error, we need to understand how the database stores the data.
A composite index `(A, B, C)` is **not** three separate lists. It's **one** list ordered by A, then by B, then by C.

The best real-world analogy is a **Printed Phone Book**.
Imagine a phone book organized by: `(Last Name, First Name)`.

1.  Silva, Ana
2.  Silva, Beto
3.  Souza, Carlos

#### The Success Scenario
If I ask: *"Find all the **Silvas**"*.
You go straight to the letter "S". The index works perfectly.

#### The Failure Scenario (The Violation)
If I ask: *"Find all the **Betos**"*.
You **cannot** use the organization of the list. You don't know if it's "Almeida, Beto" or "Zambrota, Beto".
You are forced to read **the entire list**, page by page, line by line, to find all the "Betos".

That's a **Full Table Scan**.

---

### The Leftmost Prefix Rule

The rule is binary: **The database can only use the index if your query uses the columns in the exact order of the definition, starting from the left, without skipping columns.**

Let's analyze the index: `idx_orders (tenant_id, status, created_at)`.

Here's the "Truth Table" of performance:

| Query WHERE ... | Uses the Index? | What happens? |
| :--- | :---: | :--- |
| `tenant_id = 1` | ✅ **YES** | Starts with the prefix (1st column). Perfect. |
| `tenant_id = 1 AND status = 'P'` | ✅ **YES** | Follows the order: 1st ➔ 2nd column. |
| `tenant_id = 1 AND status = 'P' AND created_at > ...` | ✅ **YES** | Full Match! Uses all 3 columns to filter. |
| `status = 'P'` | ❌ **NO** | Skipped the `tenant_id` (the beginning). Index Scan impossible. |
| `created_at > ...` | ❌ **NO** | Skipped `tenant_id` and `status`. Full Table Scan. |
| `tenant_id = 1 AND created_at > ...` | ⚠️ **PARTIAL** | Filters by `tenant_id`, but scans the rest. |

---

### The Classic Multi-Tenant Error

In SaaS systems, almost all tables have `tenant_id`.
It's very common for developers to create indexes starting with `tenant_id` for "security".

```sql
-- Index: (tenant_id, created_at)
```

And the query of the general administrative dashboard (Super Admin) is:
```sql
-- "Show the latest users created on the entire platform"
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;
```

Since this query does not filter by a specific `tenant_id`, the `(tenant_id, created_at)` index is **useless**. The database will read the entire table to find the most recent records, even if they are ordered *within* each tenant.

---

### How to Fix?

You have two strategies:

#### 1. Create Specific Indexes for Access Patterns
If you have queries that search only by `status`, you need an index that starts with `status`.

```sql
CREATE INDEX idx_status ON orders (status);
```

#### 2. Reorder the Composite Index (Cardinality Adjustment)
"Cardinality" is the number of unique values in a column.
Generally, we want to put columns with **high selectivity** (that filter a lot) at the beginning, or columns that are **always** used (`tenant_id`) at the beginning.

If your query is **always** `WHERE tenant_id = ? AND status = ?`, the order `(tenant_id, status)` or `(status, tenant_id)` works for the search.
But if you sometimes search **only** by `status` and **never** only by `tenant_id`, then the `status` should come first: `(status, tenant_id)`.

---

### Conclusion

Indexes are not magical; they are rigid ordered data structures.

Whenever you create a composite index, ask the Phone Book question:
*"If I remove the first column from my search, can I still find the data without reading the entire book?"*

If the answer is no, your index is just taking up disk and RAM space.


---

*This file is automatically generated and backed up from the blog system.*
*Last updated: 2025-12-24T01:01:20.258Z*
