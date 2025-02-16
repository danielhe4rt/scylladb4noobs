
## 1.2 Wide Column

O paradigma **Wide Column** (ou **Wide-Column Store**) é uma abordagem de armazenamento de dados que organiza as informações predominantemente em **colunas**, em vez de linhas, como ocorre nos bancos relacionais tradicionais. Esse modelo é amplamente adotado por bancos de dados **NoSQL** como **Cassandra**, **ScyllaDB**, **Bigtable**, **HBase**, entre outros. Cada um deles pode ter suas peculiaridades e linguagens de consulta específicas, mas todos compartilham a ideia fundamental de armazenamento orientado a colunas.

Como nosso foco aqui é o **ScyllaDB**, usaremos o **CQL** (*Cassandra Query Language*) para ilustrar como interagir com esse tipo de banco.

---

### 1.2.1 Modelagem Relacional

Para entender o Wide Column, vale primeiro relembrar a forma de modelagem em um banco **relacional**.  
Abaixo, temos um exemplo de tabela e consultas simples usando SQL:

```sql
$id1 = '550e8400-e29b-41d4-a716-446655440000';
$id2 = '550e8400-e29b-41d4-a716-446655440001';
$id3 = '550e8400-e29b-41d4-a716-446655440002';

CREATE TABLE users (
    user_id uuid,
    username text,
    email text,
    PRIMARY KEY (user_id)
);

INSERT INTO users (user_id, username, email) VALUES ($id1, 'Rafael', 'rafael@example.com');
INSERT INTO users (user_id, username, email) VALUES ($id2, 'Daniel', 'daniel@example.com');
INSERT INTO users (user_id, username, email) VALUES ($id3, 'Gabriel', 'gabriel@example.com');

SELECT * FROM users WHERE user_id = $id1;
```

Nesse contexto, cada linha representa um registro (usuário) e cada coluna representa um atributo (username, email etc.). Se quisermos buscar todos os usuários, basta:

```sql
SELECT * FROM users;
```

Porém, se precisarmos filtrar por colunas que não façam parte da **chave primária**, podemos ter problemas de desempenho:

```sql
SELECT * FROM users WHERE username = 'Gabriel';
```

Nesse caso, o banco precisa verificar todas as linhas para encontrar as que satisfazem o filtro. A solução em um banco relacional costuma ser criar um **índice**:

```sql
CREATE INDEX ON users (username);

SELECT * FROM users WHERE username = 'Gabriel'; -- Já funciona melhor
```

Ainda assim, mesmo com índices, o custo de manutenção e o desempenho em grande escala podem não ser ideais para cenários de alto volume e distribuição. É aqui que entra o modelo Wide Column.

---

### 1.2.2 Modelagem Wide-Column

No modelo **Wide Column**, a ideia é que cada **chave de partição** represente um “gaveteiro” (ou “armário de arquivos”), e dentro dele existam diversas “fichas” (ou linhas) organizadas por chaves de **cluster**.

**Exemplo:**  
Podemos imaginar um sistema de rede social onde armazenamos todos os posts que um usuário “curtiu” ou “não curtiu” (likes e dislikes). Uma tabela no ScyllaDB poderia ficar assim:

```sql
CREATE TABLE bluesky.timeline (
    user_id uuid,
    post_id uuid,
    liked boolean,
    post_created_at timestamp,
    PRIMARY KEY ((user_id, liked), post_created_at, post_id)
) WITH CLUSTERING ORDER BY (post_created_at DESC, post_id ASC);
```

Observe que agora a **chave primária** está estruturada em duas partes:
1. **Chave de partição**: `(user_id, liked)`.  
2. **Chaves de cluster**: `post_created_at, post_id`.

Isso significa que, para cada usuário, teremos **duas partições** possíveis: uma para `liked = true` e outra para `liked = false`. Dentro de cada partição, os registros (posts) serão ordenados de forma decrescente por `post_created_at` (e, em caso de empates, por `post_id` em ordem crescente).

#### Analogia com gaveteiros

Podemos visualizar assim:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                               Gaveteiro (Partition Key)                       │
│ ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐         │
│ │ user_id=... │   │ user_id=... │   │ user_id=... │   │ user_id=... │         │
│ │   (fechado) │   │   (fechado) │   │   (fechado) │   │   (fechado) │         │
│ └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘         │
│                                                                                 │
│ ┌─────────────────────────────────────────────────────────────────────────────┐  │
│ │ user_id='daniel' and liked=true (aberto)                                    │  │
│ │   ┌────────────────────────────────────────────────┐                       │  │
│ │   │ post_created_at DESC, post_id ASC             │ ← Clustering          │  │
│ │   ├────────────────────────────────────────────────┤                       │  │
│ │   │ Ficha 1: (post_id=..., liked=true, ...)       │                       │  │
│ │   ├────────────────────────────────────────────────┤                       │  │
│ │   │ Ficha 2: (post_id=..., liked=true, ...)       │                       │  │
│ │   ├────────────────────────────────────────────────┤                       │  │
│ │   │ Ficha 3: (post_id=..., liked=true, ...)       │                       │  │
│ │   └────────────────────────────────────────────────┘                       │  │
│ └─────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

Para cada usuário, abrimos o “gaveteiro” (partição) correspondente, que já está organizado por “likes” e “dislikes” (ou seja, `liked = true` e `liked = false`). Dentro desse gaveteiro, as “fichas” (linhas) estão ordenadas por data de criação, facilitando consultas cronológicas.

#### Vantagens

1. **Escalabilidade Horizontal**: Bancos NoSQL como ScyllaDB e Cassandra são projetados para distribuir dados em múltiplos nós, aumentando a capacidade de armazenamento e de processamento à medida que adicionamos máquinas.
2. **Acesso Direto à Partição**: Ao consultar, por exemplo, `WHERE user_id = 'daniel' AND liked = true`, o banco acessa diretamente a partição (gaveteiro) correta, em vez de percorrer toda a tabela.
3. **Otimização para Consultas Específicas**: A modelagem é pensada para responder a perguntas frequentes de forma rápida, evitando scans completos ou a necessidade de índices adicionais em muitos casos.

Claro que, para isso, a modelagem de dados no Wide Column deve ser muito bem planejada de acordo com as **queries** mais comuns da sua aplicação. Ou seja, ao contrário do modelo relacional, em que muitas vezes definimos a estrutura de dados de forma genérica e depois criamos índices para diferentes consultas, no modelo Wide Column é normal “desenhar” a tabela já pensando nos principais cenários de leitura (queries) que serão executados.


### Quando Usar Wide Column?

O principal objetivo do NoSQL (e, em especial, do Wide Column) é lidar com **grandes volumes de dados** e **altas taxas de leitura/escrita** de forma distribuída. Se o seu banco relacional atual (ex.: MySQL) não consegue escalar para atender ao crescimento de usuários e operações, migrar para um banco distribuído como o ScyllaDB pode ser uma ótima solução.

Em resumo, **Wide Column** é um modelo que prioriza a **organização dos dados** de acordo com as principais consultas de leitura/escrita, distribuindo eficientemente as informações entre nós do cluster e facilitando a escalabilidade horizontal.
