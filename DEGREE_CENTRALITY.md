# 📊 Degree Centrality for Super Node Detection

> **Identify highly connected nodes that may represent data quality issues rather than genuine patterns**

## 📖 What Is Degree Centrality?

Degree centrality measures the **number of direct connections** a node has in your graph. In network analysis, nodes with abnormally high degree centrality often indicate placeholder values or test data rather than actual meaningful relationships.

### Real Example
```
        [000-000-0000]
       /      |      \
      /       |       \
Customer1  Customer2  Customer3 ... Customer500
                                        ↓
                              HIGH DEGREE CENTRALITY
                            (500+ connections = red flag)
```

When a single phone number, email, or SSN has hundreds of connections, it's typically **test data or defaults** - not a real pattern.

## 🎯 Why This Matters for Data Quality

### The Problem with Raw Connection Counts

| Scenario | What You See | What It Actually Is | Impact |
|----------|--------------|---------------------|--------|
| **Default Phone** | 500+ connections | `000-000-0000` placeholder | False pattern alerts |
| **Test Email** | 200+ connections | `test@test.com` | Noise in analysis |
| **Corporate Phone** | 50+ connections | Legitimate shared line | Mixed signals |
| **Data Cluster** | 5-10 connections | Actual related entities | Lost in the noise |

### What Degree Centrality Reveals

| Degree Range | Node Type | Typical Cause | Action |
|--------------|-----------|---------------|--------|
| 🔴 **100+** | Ultra-high degree | System defaults/test data | Auto-exclude |
| 🟠 **20-99** | High degree | Possible dirty data | Review & classify |
| 🟡 **5-19** | Medium degree | Mixed legitimate/suspicious | Monitor closely |
| 🟢 **2-4** | Low degree | Normal sharing patterns | Include in analysis |


## 🚀 Implementation Guide

### Step 1: Create Graph Projection

> 📝 **Note:** Use UNDIRECTED orientation to catch all connections regardless of direction

```cypher
CALL gds.graph.project(
    'customerDegreeGraph',
    // Include all node types that share identifying information
    ['Customer', 'Email', 'Phone', 'SSN'],
    // Project relationships as undirected for complete connectivity view
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
CALL gds.degree.stream('customerDegreeGraph', {
    orientation: 'UNDIRECTED'
})
YIELD nodeId, score
WITH gds.util.asNode(nodeId) AS node, score
WHERE score > 10  // Focus on high-degree nodes
RETURN 
    labels(node)[0] AS nodeType,
    CASE labels(node)[0]
        WHEN 'Email' THEN node.address
        WHEN 'Phone' THEN node.phoneNumber
        WHEN 'SSN' THEN node.ssnNumber
        ELSE node.id
    END AS identifier,
    score AS connections
ORDER BY score DESC
LIMIT 20;
```

#### Write Mode (for Persistence)
```cypher
CALL gds.degree.write('customerDegreeGraph', {
    orientation: 'UNDIRECTED',
    writeProperty: 'degreeScore'
})
YIELD centralityDistribution
RETURN 
    centralityDistribution.min AS minDegree,
    centralityDistribution.mean AS avgDegree,
    centralityDistribution.max AS maxDegree,
    centralityDistribution.p99 AS p99Degree;
```

✅ **Expected Output:**
- Min: 1 (unique identifiers)
- Mean: 1.5-2.5 (normal sharing)
- P99: 10-20 (threshold for investigation)
- Max: 100-1000+ (definite super nodes)

### Step 3: Analyze Detected Super Nodes

#### 📧 Email Super Nodes
```cypher
MATCH (e:Email)
WHERE e.degreeScore > 10
RETURN
    e.address AS EmailAddress,
    e.degreeScore AS Connections,
    CASE
        WHEN e.degreeScore > 100 THEN '🔴 Super Node - Exclude'
        WHEN e.address CONTAINS 'test' THEN '🟠 Test Data - Exclude'
        WHEN e.address STARTS WITH 'no' THEN '🟠 Placeholder - Exclude'
        WHEN e.address CONTAINS '@company.com' AND e.degreeScore < 50 THEN '🟡 Corporate - Monitor'
        WHEN e.degreeScore > 15 THEN '⚠️ Suspicious - Investigate'
        ELSE '✅ Review'
    END AS Classification,
    CASE
        WHEN e.degreeScore > 100 THEN 'Auto-exclude from all data queries'
        WHEN e.address CONTAINS 'test' THEN 'Known test pattern'
        ELSE 'Manual review required'
    END AS Action
ORDER BY e.degreeScore DESC
LIMIT 25;
```

Find top connected email address
```
WITH "EMAIL ADDRESS IN HERE" AS emailAddress
MATCH path=(:Email {address: emailAddress})<-[:HAS_EMAIL]-(:Customer)
RETURN path;
```

#### 📱 Phone Super Nodes
```cypher
MATCH (p:Phone)
WHERE p.degreeScore > 10
RETURN
    p.phoneNumber AS PhoneNumber,
    p.degreeScore AS Connections,
    CASE
        WHEN p.degreeScore > 100 THEN '🔴 Super Node - Exclude'
        WHEN p.phoneNumber CONTAINS '000000' THEN '🟠 Default - Exclude'
        WHEN p.phoneNumber CONTAINS '999999' THEN '🟠 Test - Exclude'
        WHEN p.phoneNumber CONTAINS '555' THEN '🟠 Fictional - Exclude'
        WHEN p.degreeScore >= 10 AND p.degreeScore <= 50 THEN '🟡 High Share - Review'
        ELSE '⚠️ Investigate'
    END AS Classification
ORDER BY p.degreeScore DESC
LIMIT 25
```

Find top connected phone number
```
WITH "PHONE NUMBER IN HERE" AS phoneNumber
MATCH path=(:Phone {phoneNumber: phoneNumber})<-[:HAS_PHONE]-(:Customer)
RETURN path;
```

#### 🆔 SSN Super Nodes
```cypher
MATCH (s:SSN)
WHERE s.degreeScore > 5
RETURN
    // Mask SSN for security
    substring(s.ssnNumber, 0, 3) + '-XX-XXXX' AS MaskedSSN,
    s.degreeScore AS Connections,
    CASE
        WHEN s.degreeScore > 50 THEN '🔴 Critical - Duplicate Identity'
        WHEN s.ssnNumber STARTS WITH '000' THEN '🟠 Invalid SSN'
        WHEN s.degreeScore >= 10 THEN '⚠️ High Risk - Investigate'
        ELSE '✅ Normal'
    END AS Classification
ORDER BY s.degreeScore DESC
LIMIT 25;
```

## 🛡️ Applying Results

### Option 1: WHERE Clause Filtering

```cypher
// Find connections between customers, excluding super nodes
MATCH p=(c1:Customer)-[:HAS_EMAIL]->(email)<-[:HAS_EMAIL]-(c2:Customer)
WHERE email.degreeScore < 10
RETURN p;
```

**Pros:** Flexible threshold adjustment
**Cons:** Requires degree score calculation, slower queries

### Option 2: Label-Based Exclusion ✅ **RECOMMENDED**

#### Setup: Mark Super Nodes by Degree
```cypher
// Mark high-degree nodes as SuperConnectors
MATCH (n)
WHERE n.degreeScore >= 10
SET n:SuperConnector
RETURN COUNT(n) AS HighDegreeNodesLabeled;

// Mark medium-degree nodes for monitoring
MATCH (n)
WHERE n.degreeScore >= 20 AND n.degreeScore <= 50
SET n:MonitorNode
RETURN COUNT(n) AS MediumDegreeNodesLabeled;
```

#### Use in Queries: Efficient Filtering
```cypher
// Find data patterns excluding super nodes
MATCH p=(c1:Customer)-[:HAS_PHONE]->(phone:!SuperConnector)<-[:HAS_PHONE]-(c2:Customer)
WHERE c1 <> c2
RETURN p;
```

---

## 📚 Additional Resources

- [Neo4j GDS Degree Centrality Documentation](https://neo4j.com/docs/graph-data-science/current/algorithms/degree-centrality/)
- [Understanding Centrality Metrics](https://neo4j.com/docs/graph-data-science/current/algorithms/centrality/)