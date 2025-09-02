# 🎯 Closeness Centrality for Information Flow Analysis

> **Identify nodes positioned to rapidly spread information through your data network**

## 📖 What Is Closeness Centrality?

Closeness centrality measures how **quickly a node can reach all other nodes** in your graph. In network analysis, nodes with high closeness centrality are positioned at the "center" of information flow - they're the shortest average distance from everyone else.

### Real Example
```
         [Central Hub SSN]
        /    |    |    \
       /     |    |     \
   Cluster1 Cluster2 Cluster3 Cluster4
      |        |        |        |
   5 nodes  8 nodes  6 nodes  7 nodes
                ↓
        HIGH CLOSENESS CENTRALITY
    (shortest average path to all nodes)
```

When an identifier has high closeness centrality, it can **rapidly propagate data patterns** or quality issues throughout your network.

## 🎯 Why This Matters for Data Quality

### The Information Spread Problem

| Centrality Type | What It Reveals | Risk Factor |
|-----------------|-----------------|-------------|
| **High Closeness** | Central position in network | Information spreads fast |
| **Low Closeness** | Peripheral position | Isolated, limited impact |
| **Medium Closeness** | Bridge between clusters | Controls information flow |

### What Closeness Centrality Reveals

| Score Range | Position | Typical Pattern | Action |
|-------------|----------|-----------------|--------|
| 🔴 **0.8-1.0** | Network center | System defaults/test data | Auto-exclude |
| 🟠 **0.6-0.8** | Well-connected | Possible duplicate identity | Deep investigation |
| 🟡 **0.4-0.6** | Moderate reach | Normal business patterns | Monitor |
| 🟢 **< 0.4** | Peripheral | Genuine isolated cases | Include in analysis |

## 🚀 Implementation Guide

### Step 1: Create Graph Projection

> 📝 **Note:** Use UNDIRECTED for complete connectivity analysis

```cypher
CALL gds.graph.project(
    'customerClosenessGraph',
    // Include all node types involved in identity sharing
    ['Customer', 'Email', 'Phone', 'SSN'],
    // Project relationships as undirected for distance calculations
    {
        HAS_EMAIL: {orientation: 'UNDIRECTED'},
        HAS_PHONE: {orientation: 'UNDIRECTED'}, 
        HAS_SSN: {orientation: 'UNDIRECTED'}
    }
);
```

### Step 2: Run Detection Algorithm

#### Stream Mode (for Analysis)
```cypher
CALL gds.closeness.stream('customerClosenessGraph')
YIELD nodeId, score
WITH gds.util.asNode(nodeId) AS node, score
WHERE score > 0.5  // Focus on high-centrality nodes
RETURN 
    labels(node)[0] AS nodeType,
    CASE labels(node)[0]
        WHEN 'Email' THEN node.address
        WHEN 'Phone' THEN node.phoneNumber
        WHEN 'SSN' THEN node.ssnNumber
        ELSE node.id
    END AS identifier,
    round(score, 3) AS closenessScore,
    CASE
        WHEN score > 0.8 THEN '🔴 Central Hub'
        WHEN score > 0.6 THEN '🟠 Well Connected'
        WHEN score > 0.4 THEN '🟡 Moderate'
        ELSE '🟢 Peripheral'
    END AS riskLevel
ORDER BY score DESC
LIMIT 25;
```

#### Write Mode (for Persistence)
```cypher
CALL gds.closeness.write('customerClosenessGraph', {
    writeProperty: 'closenessScore'
})
YIELD centralityDistribution, nodePropertiesWritten, preProcessingMillis, computeMillis
RETURN 
    nodePropertiesWritten,
    round(centralityDistribution.min, 3) AS minCloseness,
    round(centralityDistribution.mean, 3) AS avgCloseness,
    round(centralityDistribution.max, 3) AS maxCloseness,
    round(centralityDistribution.p90, 3) AS p90Closeness,
    round(centralityDistribution.p99, 3) AS p99Closeness,
    computeMillis AS computationTimeMs;
```

✅ **Expected Output:**
- Min: 0.0 (disconnected nodes)
- Mean: 0.2-0.3 (typical distribution)
- P90: 0.4-0.5 (investigation threshold)
- P99: 0.6-0.8 (super node threshold)
- Max: 0.8-1.0 (definite super nodes)

### Step 3: Analyze Information Flow Hubs

#### 📧 Email Information Hubs
```cypher
MATCH (e:Email)
WHERE e.closenessScore > 0.5
WITH e, 
     e.closenessScore AS score,
     SIZE([(e)<-[:HAS_EMAIL]-() | 1]) AS directConnections
RETURN
    e.address AS EmailAddress,
    round(score, 3) AS ClosenessScore,
    directConnections AS DirectLinks,
    CASE
        WHEN score > 0.8 THEN '🔴 Central Hub - Exclude'
        WHEN e.address CONTAINS 'test' THEN '🟠 Test Data - Exclude'
        WHEN e.address CONTAINS 'noreply' THEN '🟠 System Email - Exclude'
        WHEN score > 0.6 AND directConnections > 10 THEN '⚠️ Data Quality Risk'
        WHEN score > 0.5 THEN '🟡 Monitor Closely'
        ELSE '✅ Normal'
    END AS Classification,
    CASE
        WHEN score > 0.8 THEN 'Auto-exclude from data analysis'
        WHEN score > 0.6 THEN 'Flag for manual review'
        ELSE 'Include with monitoring'
    END AS Action
ORDER BY score DESC
LIMIT 25;
```

Find top connected email address
```
WITH "EMAIL ADDRESS IN HERE" AS emailAddress
MATCH path=(:Email {address: emailAddress})<-[:HAS_EMAIL]-(:Customer)
RETURN path;
```

#### 📱 Phone Information Hubs
```cypher
MATCH (p:Phone)
WHERE p.closenessScore > 0.5
WITH p,
     p.closenessScore AS score,
     SIZE([(p)<-[:HAS_PHONE]-() | 1]) AS directConnections
RETURN
    p.phoneNumber AS PhoneNumber,
    round(score, 3) AS ClosenessScore,
    directConnections AS DirectLinks,
    CASE
        WHEN score > 0.8 THEN '🔴 Central Hub - Exclude'
        WHEN p.phoneNumber CONTAINS '000000' THEN '🟠 Default - Exclude'
        WHEN p.phoneNumber CONTAINS '555' THEN '🟠 Fictional - Exclude'
        WHEN score > 0.6 AND directConnections > 10 THEN '⚠️ Data Hub Center'
        WHEN score > 0.5 THEN '🟡 Information Broker'
        ELSE '✅ Normal'
    END AS Classification
ORDER BY score DESC
LIMIT 25;
```

Find top connected phone number
```
WITH "PHONE NUMBER IN HERE" AS phoneNumber
MATCH path=(:Phone {phoneNumber: phoneNumber})<-[:HAS_PHONE]-(:Customer)
RETURN path;
```

#### 🆔 SSN Information Hubs
```cypher
MATCH (s:SSN)
WHERE s.closenessScore > 0.4  // Lower threshold for SSNs
WITH s,
     s.closenessScore AS score,
     SIZE([(s)<-[:HAS_SSN]-() | 1]) AS directConnections
RETURN
    // Mask SSN for security
    substring(s.ssnNumber, 0, 3) + '-XX-XXXX' AS MaskedSSN,
    round(score, 3) AS ClosenessScore,
    directConnections AS DirectLinks,
    CASE
        WHEN score > 0.6 THEN '🔴 Critical - Data Quality Hub'
        WHEN s.ssnNumber STARTS WITH '000' THEN '🟠 Invalid SSN'
        WHEN score > 0.4 AND directConnections > 5 THEN '⚠️ Duplicate Identity'
        WHEN score > 0.3 THEN '🟡 Data Anomaly'
        ELSE '✅ Normal'
    END AS Classification,
    'Immediate investigation required' AS Priority
ORDER BY score DESC
LIMIT 25;
```

---

## 📚 Additional Resources

- [Neo4j GDS Closeness Centrality Documentation](https://neo4j.com/docs/graph-data-science/current/algorithms/closeness-centrality/)
- [Understanding Centrality Metrics](https://neo4j.com/docs/graph-data-science/current/algorithms/centrality/)
- [Information Flow in Networks](https://neo4j.com/developer/graph-algorithms/)