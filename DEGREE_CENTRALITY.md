# 📊 Degree Centrality for Super Node Detection

> **Identify highly connected nodes that may represent data quality issues rather than genuine fraud patterns**

## 📖 What Is Degree Centrality?

Degree centrality measures the **number of direct connections** a node has in your graph. In fraud detection, nodes with abnormally high degree centrality often indicate placeholder values or test data rather than actual fraudulent activity.

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

When a single phone number, email, or SSN has hundreds of connections, it's typically **test data or defaults** - not a fraud ring.

## 🎯 Why This Matters for Fraud Detection

### The Problem with Raw Connection Counts

| Scenario | What You See | What It Actually Is | Impact |
|----------|--------------|---------------------|--------|
| **Default Phone** | 500+ connections | `000-000-0000` placeholder | False fraud alerts |
| **Test Email** | 200+ connections | `test@test.com` | Noise in analysis |
| **Corporate Phone** | 50+ connections | Legitimate shared line | Mixed signals |
| **Fraud Ring** | 5-10 connections | Actual criminal network | Lost in the noise |

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
        WHEN 'Phone' THEN node.number
        WHEN 'SSN' THEN node.value
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
        WHEN e.degreeScore > 100 THEN 'Auto-exclude from all fraud queries'
        WHEN e.address CONTAINS 'test' THEN 'Known test pattern'
        ELSE 'Manual review required'
    END AS Action
ORDER BY e.degreeScore DESC
LIMIT 25;
```

#### 📱 Phone Super Nodes
```cypher
MATCH (p:Phone)
WHERE p.degreeScore > 10
RETURN
    p.number AS PhoneNumber,
    p.degreeScore AS Connections,
    CASE
        WHEN p.degreeScore > 100 THEN '🔴 Super Node - Exclude'
        WHEN p.number CONTAINS '000000' THEN '🟠 Default - Exclude'
        WHEN p.number CONTAINS '999999' THEN '🟠 Test - Exclude'
        WHEN p.number CONTAINS '555' THEN '🟠 Fictional - Exclude'
        WHEN p.degreeScore >= 10 AND p.degreeScore <= 50 THEN '🟡 High Share - Review'
        ELSE '⚠️ Investigate'
    END AS Classification
ORDER BY p.degreeScore DESC
LIMIT 25
```

#### 🆔 SSN Super Nodes
```cypher
MATCH (s:SSN)
WHERE s.degreeScore > 5
RETURN
    // Mask SSN for security
    substring(s.snnNumber, 0, 3) + '-XX-XXXX' AS MaskedSSN,
    s.degreeScore AS Connections,
    CASE
        WHEN s.degreeScore > 50 THEN '🔴 Critical - Synthetic Identity'
        WHEN s.snnNumber STARTS WITH '000' THEN '🟠 Invalid SSN'
        WHEN s.degreeScore >= 10 THEN '⚠️ High Risk - Investigate'
        ELSE '✅ Normal'
    END AS Classification
ORDER BY s.degreeScore DESC
LIMIT 25;
```

## 🛡️ Applying Results to Fraud Detection

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
// Find fraud patterns excluding super nodes
MATCH p=(c1:Customer)-[:HAS_PHONE]->(phone:!SuperConnector)<-[:HAS_PHONE]-(c2:Customer)
WHERE c1 <> c2
RETURN p;

// Include monitored nodes but flag them
MATCH p=(c1:Customer)-[:HAS_EMAIL]->(email)<-[:HAS_EMAIL]-(c2:Customer)
WHERE c1 <> c2
RETURN p,
    CASE WHEN email:MonitorNode THEN 'Warning: Shared Email' ELSE 'Normal' END AS Flag;
```

## 📊 Ongoing Management Strategy

### Weekly Maintenance Tasks

```cypher
// -- 1. Drop existing projection (if exists)
CALL gds.graph.exists('customerDegreeGraph') YIELD exists
WHERE exists
CALL gds.graph.drop('customerDegreeGraph') YIELD graphName
RETURN graphName + ' dropped' AS status;

// -- 2. Create fresh graph projection with current data
CALL gds.graph.project(
    'customerDegreeGraph',
    ['Customer', 'Email', 'Phone', 'SSN'],
    {
        HAS_EMAIL: {orientation: 'UNDIRECTED'},
        HAS_PHONE: {orientation: 'UNDIRECTED'}, 
        HAS_SSN: {orientation: 'UNDIRECTED'}
    }
)
YIELD nodeCount, relationshipCount
RETURN nodeCount, relationshipCount;

// -- 3. Clear ALL previous degree scores (handles nodes that went to 0 connections)
MATCH (n)
WHERE n.degreeScore IS NOT NULL
SET n.degreeScore = NULL
RETURN COUNT(n) AS ScoresCleared;

// -- 4. Calculate fresh degree centrality scores
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

// -- 5. Remove SuperConnector label from nodes below threshold or with no score
MATCH (n:SuperConnector)
WHERE n.degreeScore IS NULL OR n.degreeScore < 50
REMOVE n:SuperConnector
RETURN COUNT(n) AS LabelsRemoved;

// -- 6. Remove MonitorNode label from nodes outside range
MATCH (n:MonitorNode)
WHERE n.degreeScore IS NULL OR n.degreeScore < 20 OR n.degreeScore >= 50
REMOVE n:MonitorNode
RETURN COUNT(n) AS MonitorLabelsRemoved;

// -- 7. Add SuperConnector label to high-degree nodes
MATCH (n)
WHERE n.degreeScore >= 10 AND NOT n:SuperConnector
SET n:SuperConnector
RETURN COUNT(n) AS SuperConnectorLabelsAdded;

// -- 8. Add MonitorNode label to medium-degree nodes
MATCH (n)
WHERE n.degreeScore >= 5 AND n.degreeScore < 10 AND NOT n:MonitorNode
SET n:MonitorNode
RETURN COUNT(n) AS MonitorNodeLabelsAdded;

// -- 9. Generate status report
MATCH (n)
WHERE n.degreeScore IS NOT NULL
WITH labels(n)[0] AS nodeType,
    CASE
        WHEN n.degreeScore >= 100 THEN 'Ultra High (100+)'
        WHEN n.degreeScore >= 50 THEN 'Very High (50-100)'
        WHEN n.degreeScore >= 20 THEN 'High (20-50)'
        WHEN n.degreeScore >= 2 THEN 'Investigate (2-19)'
        WHEN n.degreeScore = 1 THEN 'Unique (1)'
        ELSE 'Isolated (0)'
    END AS degreeCategory,
    COUNT(n) AS nodeCount,
    COLLECT(n)[0..3] AS examples
RETURN nodeType, degreeCategory, nodeCount, 
       [ex IN examples | 
           CASE labels(ex)[0]
               WHEN 'Email' THEN ex.address
               WHEN 'Phone' THEN ex.number
               WHEN 'SSN' THEN substring(ex.snnNumber, 0, 3) + '-XX-XXXX'
               ELSE ex.id
           END
       ] AS exampleIdentifiers
ORDER BY nodeType, degreeCategory DESC;
```

### Threshold Tuning Strategy

| Node Type | Initial Threshold | Review After | Adjust Based On |
|-----------|------------------|--------------|-----------------|
| **Email** | 50 connections | 1 week | False positive rate |
| **Phone** | 50 connections | 1 week | Business phone patterns |
| **SSN** | 10 connections | Daily | Fraud risk tolerance |
| **Address** | 100 connections | 2 weeks | Apartment buildings |

---

## 📚 Additional Resources

- [Neo4j GDS Degree Centrality Documentation](https://neo4j.com/docs/graph-data-science/current/algorithms/degree-centrality/)
- [Understanding Centrality Metrics](https://neo4j.com/docs/graph-data-science/current/algorithms/centrality/)