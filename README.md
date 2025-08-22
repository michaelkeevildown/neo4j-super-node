# 🔍 Super Node Detection

> **Enhance data accuracy by identifying and filtering super nodes in Neo4j graphs**

## 🎯 Quick Start

Super nodes are highly connected nodes that often represent dirty data. This project helps you identify and manage these nodes to improve financial networks.

## 📊 The Problem

### Why Super Nodes Matter

In production databases, data quality issues create artificial connections that obscure real patterns:

| Data Type | Common Default Values | Impact |
|-----------|----------------------|--------|
| **Email** | `test@test.com`, `noemail@test.com` | False customer links |
| **Phone** | `000-000-0000`, `999-999-9999` | Artificial connections |
| **Address** | `123 Main St`, `NULL`, `N/A` | Geographic clustering |
| **SSN** | `000-00-0000`, `123-45-6789` | Identity confusion |

These placeholder values become **super nodes** - creating thousands of false connections between unrelated entities.

## 🗂️ Graph Data Model

![Graph Schema](diagrams/schema.png)

### Node Types

| Node | Color | Description |
|------|-------|-------------|
| **Customer** | 🔴 Pink | Individual customers |
| **Account** | 🟣 Purple | Bank accounts |
| **Transaction** | 🟢 Green | Financial transactions |
| **Bank** | 🟦 Teal | Financial institutions |
| **Merchant** | 🟠 Orange | Transaction recipients |
| **Email** | 🔵 Light Blue | Email addresses |
| **Phone** | 🟪 Lavender | Phone numbers |
| **SSN** | 🔷 Blue | Social Security Numbers |

### Relationship Types

```cypher
(:Customer)-[:HAS_EMAIL]->(:Email)
(:Customer)-[:HAS_PHONE]->(:Phone)
(:Customer)-[:HAS_SSN]->(:SSN)
(:Customer)-[:HAS_ACCOUNT]->(:Account)
(:Account)-[:PERFORM]->(:Transaction)
(:Transaction)-[:BENEFITS_TO]->(:Account|:Bank|:Merchant)
```

## 🔬 Detection Strategy

### 🎯 Primary Targets

We focus on three critical super node types in this demo:

| Priority | Node Type | Why It Matters |
|----------|-----------|----------------|
| **1** | 📱 Phone Numbers | Most commonly reused/defaulted |
| **2** | 🆔 SSNs | Shared across synthetic identities |
| **3** | 📧 Email Addresses | Frequent placeholder values |

### 📈 Algorithm Suite

#### 1. **Degree Centrality** 
> Identifies nodes with abnormally high connections

```cypher
CALL gds.degree.write('customerDegreeGraph', {
    orientation: 'UNDIRECTED',
    writeProperty: 'degreeScore'
})
YIELD centralityDistribution;
```

- **Output**: Nodes exceeding normal connection patterns (typically >50 connections indicates super node)
- [📖 Full Implementation Guide](DEGREE_CENTRALITY.md) | [Neo4j Documentation](https://neo4j.com/docs/graph-data-science/current/algorithms/degree-centrality/)

#### 2. **Articulation Points**
> Finds critical bridge nodes in the network

```cypher
CALL gds.articulationPoints.write('customerArticulationGraph', { 
    writeProperty: 'articulationPoint'
})
YIELD articulationPointCount;
```

- **Output**: Boolean property marking nodes whose removal would disconnect the graph (super nodes often act as bridges between unrelated customer clusters)
- [📖 Neo4j Documentation](https://neo4j.com/docs/graph-data-science/current/algorithms/articulation-points/)

#### 3. **Closeness Centrality**
> Finds how quickly a node can reach all other nodes in your graph.

```cypher
CALL gds.closeness.write('customerClosenessGraph', {
    writeProperty: 'closenessScore'
})
YIELD centralityDistribution;
```

- **Output**: Closeness score (0-1) measuring how central a node is to the entire network - super nodes score higher due to their extensive connections
- [📖 Full Implementation Guide](CLOSENESS_CENTRALITY.md) | [Neo4j Documentation](https://neo4j.com/docs/graph-data-science/current/algorithms/closeness-centrality/)

### Information Flow Risk Matrix

| Closeness | Degree | Articulation | Risk Level | Action |
|-----------|--------|--------------|------------|--------|
| **High** | High | Yes | 🔴 Critical | Auto-exclude |
| **High** | High | No | 🟠 Very High | Manual review |
| **High** | Low | Yes | 🟠 High | Investigate |
| **High** | Low | No | 🟡 Medium | Monitor |
| **Low** | High | Yes | 🟡 Medium | Monitor |
| **Low** | Low | No | 🟢 Low | Include |

## 🔄 Combining with Other Centrality Measures

### The Power Trinity: Closeness + Degree + Articulation

```cypher
// Create comprehensive super node detection
MATCH (n)
WHERE n.closenessScore IS NOT NULL 
   OR n.degreeScore IS NOT NULL 
   OR n.articulationPoint IS NOT NULL
WITH n,
     COALESCE(n.closenessScore, 0) AS closeness,
     COALESCE(n.degreeScore, 0) AS degree,
     COALESCE(n.articulationPoint, 0) AS articulation
SET n.superNodeRisk = 
    (CASE WHEN closeness > 0.6 THEN 3 ELSE 0 END) +
    (CASE WHEN degree > 50 THEN 3 ELSE 0 END) +
    (CASE WHEN articulation = 1 THEN 2 ELSE 0 END)
WITH n
WHERE n.superNodeRisk >= 4
SET n:HighRiskSuperNode
RETURN COUNT(n) AS HighRiskNodesIdentified;
```

## 📚 Resources

- [Financial Services Use Cases](https://neo4j.com/developer/industry-use-cases/)
- [Neo4j Graph Data Science Documentation](https://neo4j.com/docs/graph-data-science/)
- [Fraud Detection Best Practices](https://neo4j.com/use-cases/fraud-detection/)
- [Community Forum](https://community.neo4j.com/)