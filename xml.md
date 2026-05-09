# COMP0178 — Semi-Structured Data, XML & NoSQL Databases

---

## Part 1: Semi-Structured Data & XML

### 1.1 What is Semi-Structured Data?

Relational databases work best when data is **well-structured**: the schema is known in advance, unlikely to change, and every record of a given type conforms to the same shape (a rectangular table). Semi-structured data relaxes these assumptions.

**Semi-structured (schema-less / self-describing) data** has structure, but that structure may be irregular, incomplete, or change unpredictably. There is either no schema at all (the data itself defines its own shape), or a schema exists but permits wide variation.

**Example — why relational struggles:**
A property listing site might store rent as `monthlyRent` for some properties and `annualRent` for others. In a relational table you'd need both columns (with NULLs), or a separate design. In a semi-structured format each record simply includes whichever field applies.

---

### 1.2 The Object Exchange Model (OEM)

OEM is an early semi-structured database model with a **tree (nested) structure**. Each object has:

- A **label** (e.g. `"DreamHome"`)
- A **unique ID** (e.g. `&1`)
- Optionally a **value** (e.g. `"Ann Beech"`)
- **Edges** to child objects

Data can be irregular: one staff member's name might be stored as a single string `"Ann Beech"`, while another's is split into `fName` and `lName` sub-objects.

**Querying OEM** uses path-based access. Lorel (Lightweight Object REpository Language) is an SQL-like language:

```
SELECT object/path FROM path WHERE condition
```

**DataGuides** are auxiliary summary structures of the OEM tree, annotated with the IDs of nodes matching each path. They speed up queries by letting you check whether a path is valid and jump directly to matching data without scanning the whole tree.

**Key principles that carry over to XML:**
1. Flexible, tree-like structure.
2. Querying is based on **paths** through the tree.
3. Efficient querying requires a summary/understanding of the data's structure.

---

### 1.3 XML Fundamentals

**XML (eXtensible Markup Language)** generalises HTML by letting users define their own tags and document structure.

An XML document has:
- An `<?xml ... ?>` declaration
- An optional reference to a stylesheet or schema (DTD / XML Schema)
- A single **root element** containing nested elements

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!DOCTYPE STAFFLIST SYSTEM "staff_list.dtd">
<STAFFLIST>
  <STAFF branchNo="B005">
    <STAFFNO>SL21</STAFFNO>
    <NAME>
      <FNAME>John</FNAME>
      <LNAME>White</LNAME>
    </NAME>
    <POSITION>Manager</POSITION>
    <DOB>1-Oct-45</DOB>
    <SALARY>30000</SALARY>
  </STAFF>
  <STAFF branchNo="B003">
    <STAFFNO>SG37</STAFFNO>
    <NAME>
      <FNAME>Ann</FNAME>
      <LNAME>Beech</LNAME>
    </NAME>
    <POSITION>Assistant</POSITION>
    <SALARY>12000</SALARY>
  </STAFF>
</STAFFLIST>
```

Notice that `DOB` is present for the first staff member but absent for the second — XML naturally tolerates this kind of irregularity.

#### Elements vs Attributes

| Feature | Elements | Attributes |
|---------|----------|------------|
| Syntax | `<TAG>value</TAG>` | `<TAG attr="value">` |
| Order | Ordered (sequence matters) | Unordered |
| Multiplicity | Can have multiple children with the same tag | Only one attribute of a given name per tag |
| Nesting | Can contain further nested elements | Cannot nest |

**Rule of thumb:** if you might need more than one value, or nesting, use a child element. Otherwise either works.

---

### 1.4 Defining XML Structure — DTD and XML Schema

#### DTD (Document Type Definition)

DTDs define allowed elements, attributes, and nesting rules.

```
<!ELEMENT STAFFLIST (STAFF)*>
<!ELEMENT STAFF (NAME, POSITION, DOB?, SALARY)>
<!ELEMENT NAME (FNAME, LNAME)>
<!ELEMENT FNAME (#PCDATA)>
<!ELEMENT LNAME (#PCDATA)>
<!ELEMENT POSITION (#PCDATA)>
<!ELEMENT DOB (#PCDATA)>
<!ELEMENT SALARY (#PCDATA)>
<!ATTLIST STAFF branchNo CDATA #IMPLIED>
```

**Key notation:**
- `*` = zero or more
- `+` = one or more
- `?` = optional (zero or one)
- `#PCDATA` = parsed character data (text content)
- `#REQUIRED` = attribute must be present; `#IMPLIED` = optional

DTDs also support `ID` (unique identifier for an element) and `IDREF` / `IDREFS` (references to another element's ID), which let you model relationships outside the nesting hierarchy.

#### XML Schema

XML Schema is the modern replacement for DTD. Unlike DTD, XML Schema documents are themselves valid XML. Key advantages over DTD:

- Richer built-in data types (`xs:string`, `xs:date`, `xs:decimal`, etc.)
- Ability to define custom types
- Support for uniqueness and primary key constraints

```xml
<xs:element name="STAFFLIST">
  <xs:complexType>
    <xs:sequence>
      <xs:element name="STAFF" maxOccurs="unbounded">
        <xs:complexType>
          <xs:sequence>
            <xs:element name="STAFFNO" type="xs:string"/>
            <xs:element name="POSITION" type="xs:string"/>
            <xs:element name="DOB" type="xs:date" minOccurs="0"/>
            <xs:element name="SALARY" type="xs:decimal"/>
          </xs:sequence>
          <xs:attribute name="branchNo" type="xs:string"/>
        </xs:complexType>
      </xs:element>
    </xs:sequence>
  </xs:complexType>
</xs:element>
```

---

### 1.5 Querying XML — XPath

XPath lets you navigate the tree structure of an XML document using **path expressions**.

| Notation | Meaning |
|----------|---------|
| `doc("file.xml")` | Opens an XML file |
| `/` | Selects direct children |
| `//` | Selects descendants at any depth |
| `..` | Selects parent node |
| `@` | Refers to an attribute |
| `[cond]` | Applies a filter condition |
| `*` | Wildcard — matches any element |
| `[n]` | Selects the nth element (1-indexed) |

**Example 1:** Find the staff number of the first staff member.

```
doc("staff_list.xml")/STAFFLIST/STAFF[1]/STAFFNO
```

Step by step: open the file → select root `STAFFLIST` → select the first `STAFF` child → select its `STAFFNO` child. Returns `SL21`.

**Example 2:** Find all staff at branch B003.

```
doc("staff_list.xml")//STAFF[@branchNo="B003"]
```

The `//` means "search at any depth", and `[@branchNo="B003"]` filters on the attribute.

**Multiple equivalent paths** can reach the same data:
- `doc("staff_list.xml")/STAFFLIST/STAFF[1]/STAFFNO`
- `doc("staff_list.xml")//STAFF[1]/STAFFNO`
- `doc("staff_list.xml")/STAFFLIST/STAFF[1]//STAFFNO`

---

### 1.6 Querying XML — XQuery (FLWOR)

FLWOR (pronounced "flower") expressions are the XQuery equivalent of SQL's SELECT-FROM-WHERE. The clauses are:

| Clause | Purpose | Required? |
|--------|---------|-----------|
| **F**OR | Iterates over a sequence, binding each item to a variable | At least one FOR or LET |
| **L**ET | Binds a variable to an entire sequence (no iteration) | |
| **W**HERE | Filters the bound variables | Optional |
| **O**RDER BY | Sorts the results | Optional |
| **R**ETURN | Specifies what to output for each binding | Required |

**Example 1 — Simple filter:**

*Task: List staff numbers of all staff at branch B005 earning more than £15,000.*

```xquery
FOR $S IN doc("staff_list.xml")//STAFF
WHERE $S/SALARY > 15000
  AND $S/@branchNo = "B005"
RETURN $S/STAFFNO
```

**Example 2 — Nested FOR/LET with XML construction:**

*Task: List branches that have more than 20 staff.*

```xquery
<LARGEBRANCHES> {
  FOR $B IN distinct-values(doc("staff_list.xml")//@branchNo)
  LET $S := doc("staff_list.xml")//STAFF[@branchNo = $B]
  WHERE count($S) > 20
  RETURN <BRANCHNO>{ $B }</BRANCHNO>
} </LARGEBRANCHES>
```

Here `FOR` iterates over each unique branch number; `LET` binds `$S` to *all* staff at that branch (as a set, not iterating); `WHERE` filters; `RETURN` constructs new XML.

**Example 3 — Exam-style (2024/25 Q5a):**

Given this XML structure:
```xml
<PROPERTIES>
  <PROPERTY>
    <ADDRESS>
      <NUMBER>123</NUMBER>
      <STREET>Main Street</STREET>
      <CITY>London</CITY>
    </ADDRESS>
    <MONTHLYRENT>1000</MONTHLYRENT>
    <TIMESVISITED>12</TIMESVISITED>
  </PROPERTY>
  ...
</PROPERTIES>
```

*Task: Get a list of properties that have not been visited.*

**XPath approach:**
```
doc("properties.xml")//PROPERTY[TIMESVISITED = 0]
```

**FLWOR approach:**
```xquery
FOR $P IN doc("properties.xml")//PROPERTY
WHERE $P/TIMESVISITED = 0
RETURN $P
```

---

### 1.7 XML Data Models (DOM and XDM)

Just as OEM uses DataGuides, XML querying relies on in-memory tree representations:

**DOM (Document Object Model):** An in-memory tree where each node represents an element, attribute, or text value. Supports navigation (parent, child, sibling) and updates. Used by programming languages to manipulate XML.

**XDM (XPath and XQuery Data Model):** The input format for XQuery engines. A flattened list of nodes in document order, where each node records its parent/child relationships and attributes.

---

### 1.8 XML and Relational Databases Together

*Exam relevance: "Can relational databases and semi-structured databases be used together?" (2024/25 Q5b)*

**Yes.** There are several approaches:

**Data-centric model:** XML is used purely as a **data interchange format**. Different relational databases export/import data as XML to communicate with each other. The actual storage remains relational.

**Document-centric model:** A **native XML database (NXD)** stores data directly in XML format, using XML-specific query engines.

**Hybrid / storing XML in a relational DB:**

1. **Store as a single column** — the entire XML document is stored as a CLOB (Character Large Object) or `XMLType` in one attribute of a relational table. Simple but limits querying.

2. **Shredding** — the XML structure is analysed and distributed across multiple relational tables/columns. A `STAFF` XML document might map to a `Staff` table with columns `staffNo`, `fName`, `lName`, `position`, `salary`. Enables full SQL querying but loses flexibility.

3. **Schema-independent storage** — the XML tree is parsed into a generic relational table recording every node:

| nodeID | nodeType | nodeName | nodeData | parentID |
|--------|----------|----------|----------|----------|
| 0 | Document | STAFFLIST | | |
| 1 | Element | STAFFLIST | | 0 |
| 2 | Element | STAFF | | 1 |
| 3 | Element | STAFFNO | | 2 |
| 4 | Text | | SL21 | 3 |

This works for any XML schema but queries become complex joins.

**Key exam answer:** Relational and semi-structured databases can absolutely be used together. Common approaches include using XML as a data interchange format between relational systems, storing XML documents inside relational tables (as CLOBs or by shredding into columns), and many modern RDBMSs natively support XML/JSON data types with built-in querying functions.

---

### 1.9 Relational vs Semi-Structured — Comparison

*Exam relevance: frequently tested (2022/23 Q5a, 2023/24 Q5a)*

| Aspect | Relational | Semi-Structured (XML/JSON) |
|--------|-----------|---------------------------|
| **Structure** | Fixed schema; rectangular tables | Flexible/no schema; nested trees |
| **Schema** | Defined before data entry; enforced by DBMS | Self-describing or loosely defined |
| **Data regularity** | All rows in a table have the same columns | Records can have different fields |
| **Querying** | SQL — declarative, set-based, powerful joins | Path-based (XPath/XQuery); tree traversal |
| **Joins** | Native and optimised | No native join; must navigate paths |
| **Integrity** | Strong constraints (PK, FK, CHECK, etc.) | Limited (some via DTD/XML Schema) |
| **Multi-valued data** | Requires separate tables (1NF) | Natural nesting (arrays, child elements) |
| **Strengths** | Data integrity, complex queries, mature optimisation | Flexibility, handles irregular data, good for interchange |
| **Weaknesses** | Rigid schema, poor fit for irregular data | Querying is harder, less optimised, weaker integrity |

**Querying differences (exam-style answer):**
Querying semi-structured databases differs from relational databases in that semi-structured queries rely on navigating paths through a tree structure (e.g. `/STAFFLIST/STAFF/NAME/FNAME`) rather than performing set-based operations on flat tables. Because semi-structured data has no guaranteed schema, queries must handle missing or variable fields, whereas relational queries operate on a known, fixed structure. Semi-structured queries also lack native join operations — relationships must be expressed through nesting or explicit ID/IDREF references rather than foreign keys.

---

## Part 2: NoSQL Databases

### 2.1 Motivation — Why NoSQL?

Relational databases prioritise **consistency**: ACID properties, stable schemas, normalisation, integrity constraints. This emphasis can hinder **flexibility** and **scalability**, especially when:

- The database is enormous and must be distributed across many machines
- The schema needs to evolve frequently or is inherently irregular
- High availability matters more than perfect consistency at every instant
- Read/write throughput is extremely high

NoSQL ("Not Only SQL") databases depart from the relational model to address these scenarios. The key trade-off: relaxing consistency in favour of **availability** and **partition tolerance**.

---

### 2.2 ACID vs BASE

| Property | ACID (Relational) | BASE (NoSQL) |
|----------|------------------|--------------|
| Focus | Consistency first | Availability first |
| **A** | Atomicity — all or nothing | **B**asically **A**vailable — system always responds |
| **C** | Consistency — DB always in valid state | |
| **I** | Isolation — transactions don't interfere | **S**oft state — data may change over time without input |
| **D** | Durability — committed data survives failure | **E**ventual consistency — system converges to consistent state |

**Eventual consistency** means that if no new updates are made, all replicas will eventually hold the same data — but at any given moment, different nodes might return different values.

---

### 2.3 The CAP Theorem

The CAP theorem states that a distributed system can guarantee at most **two of three** properties simultaneously:

- **C**onsistency — every read returns the most recent write
- **A**vailability — every request receives a (non-error) response
- **P**artition tolerance — the system works even when network partitions occur

In practice, network partitions are unavoidable in distributed systems, so the real choice is between **CP** (consistent but may be unavailable during partitions) and **AP** (available but may return stale data). NoSQL databases typically choose AP, accepting eventual consistency.

---

### 2.4 Types of NoSQL Databases

#### 2.4.1 Key-Value Stores

The simplest NoSQL model. Data is stored as `(key, value)` pairs where keys are unique and the DBMS treats values as opaque blobs.

**Operations** are basic: `get(key)`, `put(key, value)`, `delete(key)`.

**Example:**
```
put("user:1001", "{name: 'Alice', email: 'alice@example.com'}")
get("user:1001")  →  "{name: 'Alice', email: 'alice@example.com'}"
```

**Strengths:** Extremely fast lookups by key; easy horizontal scaling via sharding.
**Weaknesses:** No querying by value; no schema; no relationships or integrity constraints. The DBMS cannot inspect or index the contents of values.

**Use cases:** Session stores, caches, user preferences.
**Examples:** Redis, Amazon DynamoDB, Memcached.

Can also be used as a **memory cache** sitting in front of a relational database to speed up frequent reads.

---

#### 2.4.2 Sharding with Hashing

Key-value stores distribute data across nodes by **hashing keys**.

**Simple (modulus) hashing:** With *n* servers, assign key to server `hash(key) mod n`.

**Problem:** Adding or removing a server changes *n*, causing most keys to be reassigned. With 3→4 servers, roughly 75% of keys move.

**Consistent hashing** solves this using a **ring topology**:
- Both servers and keys are hashed onto a ring (interval [0, 1)).
- Each key is stored at the nearest server **clockwise** on the ring.
- Adding/removing a server only affects approximately `1/n` of the keys (those between the new/removed server and its predecessor).

This is used by Cassandra, DynamoDB, and many other distributed databases.

*Exam relevance: "True or false: Hashing the keys of a table can be a quick way of figuring out how to fragment it." (2022/23 Q5b) — True. Applying a hash function to each key and using the result (e.g. modulus) to assign rows to fragments provides an automatic, even distribution without needing to understand the data semantics. This is exactly how key-value stores distribute data across nodes.*

---

#### 2.4.3 Tuple Stores

An extension of key-value stores where the value is a **vector (tuple) of data** — like a row in a table, but with no requirement for tuples to have the same length or structure.

Some implementations allow grouping entries into "collections" or "tables" for organisational purposes, but this is optional and not enforced.

---

#### 2.4.4 Document Stores

The value associated with each key is a **semi-structured document**, typically JSON.

```json
{
  "title": "Harry Potter",
  "authors": ["J.K. Rowling"],
  "price": 32.00,
  "genres": ["fantasy"],
  "dimensions": {
    "width": 8.5,
    "height": 11.0
  },
  "pages": 234,
  "in_publication": true
}
```

**Advantages over key-value stores:** The DBMS understands the document structure and can index/query fields within documents.

**Challenges:**
- Filtering still requires scanning unless indexes are created on specific fields
- Joins are difficult without a schema — typically handled by **denormalisation** (embedding related data within documents) or manual application-level joining

**Examples:** MongoDB, CouchDB, Amazon DocumentDB.

---

#### 2.4.5 Graph Databases

Data is modelled as **nodes** (entities) and **edges** (relationships). Both can have properties (key-value pairs).

**Example — social network:**
```
(Alice)-[:FRIENDS_WITH]->(Bob)
(Bob)-[:POSTED]->(Post1)
```

**Querying with Cypher** (Neo4j's query language):

```cypher
-- Find all books written by a specific author
MATCH (b:Book)<-[:WRITTEN_BY]-(a:Author)
WHERE a.name = "Bart Baesens"
RETURN b.title
```

Cypher syntax:
- `()` = nodes (optionally labelled: `(b:Book)`)
- `--` = undirected edge; `->` = directed edge
- `[:TYPE]` = edge type filter
- `WHERE` / `RETURN` work similarly to SQL

**Strengths:** Traversing relationships is extremely fast (O(1) per hop vs expensive joins in SQL); natural fit for highly connected data.

**Use cases:** Social networks, recommendation engines, fraud detection, knowledge graphs, route planning.

**Examples:** Neo4j, Amazon Neptune, ArangoDB.

---

#### 2.4.6 Other Types

- **Column-family stores** (e.g. Apache Cassandra, HBase): store data by columns rather than rows, optimised for aggregations over large datasets.
- **XML databases** (e.g. MarkLogic): native storage and querying of XML.
- **Time-series databases** (e.g. InfluxDB): optimised for timestamped data (IoT, monitoring).
- **Geospatial databases** (e.g. PostGIS): specialised for location data and spatial queries.

---

### 2.5 MapReduce

MapReduce is a programming model for processing large datasets in parallel across a distributed system. Originally developed by Google; open-source implementation in Apache Hadoop.

**Two phases:**

1. **Map** — takes input records and emits intermediate `(key, value)` pairs
2. **Reduce** — groups intermediate pairs by key and aggregates the values

**Worked example — total pages per genre:**

**Input data (distributed across workers):**
```
{id: 1, genre: "education", nrPages: 120}
{id: 2, genre: "thriller",  nrPages: 100}
{id: 3, genre: "fantasy",   nrPages: 20}
{id: 4, genre: "drama",     nrPages: 500}
{id: 5, genre: "education", nrPages: 200}
{id: 6, genre: "education", nrPages: 20}
{id: 7, genre: "fantasy",   nrPages: 10}
```

**Map function** (runs on each worker independently):
```
function map(key, value):
    emit(value.genre, value.nrPages)
```

**Intermediate results** (after grouping by key):
```
education → [120, 200, 20]
thriller  → [100]
drama     → [500]
fantasy   → [20, 10]
```

**Reduce function:**
```
function reduce(key, values_list):
    emit(key, sum(values_list))
```

**Final output:**
```
education → 340
thriller  → 100
drama     → 500
fantasy   → 30
```

The power of MapReduce is that both phases can be **parallelised** across many machines — each worker handles its local data independently.

---

### 2.6 Request Coordination & Membership

In many NoSQL systems (Cassandra, DynamoDB), **any node** can serve as the request coordinator for a query. This requires every node to know what data is stored on every other node.

A **membership protocol** handles node joins and departures: a new node informs at least one existing node, and information propagates gradually. This means the network state may not be perfectly consistent at any moment — another manifestation of **eventual consistency**.

---

### 2.7 Relational vs NoSQL — Full Comparison

| Feature | Relational | NoSQL |
|---------|-----------|-------|
| Data model | Tables with fixed schema | Key-value, document, graph, column, etc. |
| Schema | Schema-first (rigid) | Schema-free or flexible |
| Scalability | Primarily vertical; horizontal is complex | Designed for horizontal scaling |
| Query language | SQL (standardised, powerful) | Varies: none, simple APIs, Cypher, MapReduce |
| Transactions | ACID | BASE (eventual consistency) |
| Joins | Native, optimised | Difficult or unsupported; use denormalisation |
| Integrity | Strong (PK, FK, constraints, triggers) | Minimal or application-enforced |
| Best for | Complex queries, transactions, data integrity | High volume, flexible schema, high availability |

---

### 2.8 NewSQL

The gap between relational and NoSQL is narrowing from both sides:

**NoSQL vendors** are adding robustness features (stronger consistency options, transaction support).

**Relational vendors** are adding NoSQL-like features: native JSON/XML column types, horizontal scaling, schema-free options, MapReduce support, geospatial types.

**NewSQL** systems aim to combine the scalable performance of NoSQL with the ACID guarantees of relational databases. Examples include Google Spanner, CockroachDB, and VoltDB.

---

### 2.9 JSON vs XML

| Feature | XML | JSON |
|---------|-----|------|
| Verbosity | Verbose (opening + closing tags) | Compact (curly braces, colons) |
| Data types | Everything is text (#PCDATA) unless schema used | Native numbers, strings, booleans, null, arrays |
| Schema support | DTD, XML Schema (rich) | JSON Schema (simpler) |
| Attributes | Supported (metadata on elements) | No attributes; everything is a key-value pair |
| Order | Element order is significant | Object key order is not guaranteed |
| Querying | XPath, XQuery (FLWOR) | JSONPath, or language-native parsing |
| Use today | Configuration, enterprise systems, SOAP | APIs (REST), web applications, document stores |

**JSON example of the same staff data:**
```json
{
  "staffList": [
    {
      "branchNo": "B005",
      "staffNo": "SL21",
      "name": { "fname": "John", "lname": "White" },
      "position": "Manager",
      "dob": "1945-10-01",
      "salary": 30000
    },
    {
      "branchNo": "B003",
      "staffNo": "SG37",
      "name": { "fname": "Ann", "lname": "Beech" },
      "position": "Assistant",
      "salary": 12000
    }
  ]
}
```

Notice: more compact, native number types, and the missing `dob` for the second record is simply omitted.

---

## Exam-Style Practice Questions with Outline Answers

**Q1: "Summarise the main differences between querying semi-structured databases vs querying relational databases." (5 marks)**

Relational queries use SQL, which operates on flat tables with a known schema using set-based operations (SELECT, JOIN, GROUP BY). Semi-structured queries navigate a tree structure using path expressions (e.g. XPath: `/STAFFLIST/STAFF/NAME`). Because semi-structured data has no guaranteed schema, queries must handle variable or missing fields. Relational databases support efficient native joins via foreign keys, whereas semi-structured databases express relationships through nesting or ID references, making cross-entity queries harder.

**Q2: "Can relational databases and semi-structured databases be used together? Briefly explain." (5 marks)**

Yes. XML/JSON can serve as a data interchange format between relational systems. Many RDBMSs support storing semi-structured data directly (e.g. XML columns, JSON columns) and provide functions to query within them. XML documents can also be "shredded" into relational tables for full SQL access. This hybrid approach lets applications use relational storage for structured core data while accommodating flexible or irregular data in semi-structured formats.

**Q3: "True or false: Hashing the keys of a table can be a quick way of figuring out how to fragment it. Explain." (5 marks)**

True. Applying a hash function to each row's key and using the hash value (e.g. `hash(key) mod n`) assigns each row to one of *n* fragments automatically. This provides roughly even distribution without requiring knowledge of data semantics. It is the standard approach in key-value stores and distributed NoSQL databases. However, it does not preserve range queries (nearby keys may end up on different nodes) and simple modulus hashing requires extensive data movement when nodes are added/removed (consistent hashing mitigates this).

**Q4: "What is one strength and one weakness that relational databases have compared to semi-structured databases?" (10 marks)**

*Strength:* Relational databases enforce strong data integrity through schemas, primary/foreign key constraints, and ACID transactions. This guarantees consistency and prevents invalid data from entering the system — critical for applications like banking or healthcare.

*Weakness:* Relational databases require a fixed schema defined upfront, making them inflexible when data structures vary between records or evolve frequently. Semi-structured databases naturally accommodate irregular, nested, or evolving data without schema migrations.

**Q5: Write a FLWOR query for properties with rent under £800.**

Given the properties XML from the 2024/25 exam:
```xquery
FOR $P IN doc("properties.xml")//PROPERTY
WHERE $P/MONTHLYRENT < 800
RETURN $P/ADDRESS
```

Or equivalently with XPath only:
```
doc("properties.xml")//PROPERTY[MONTHLYRENT < 800]/ADDRESS
```
