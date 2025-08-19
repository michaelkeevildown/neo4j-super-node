# 🔗 Articulation Points for Super Node Detection

> **Identify critical bridge nodes that artificially connect unrelated customer clusters**

## 📖 What Are Articulation Points?

Articulation points are nodes whose removal would **disconnect parts of your graph**. In fraud detection, these critical nodes often reveal data quality issues rather than genuine fraud patterns.

### Real Example
```
Cluster A (100 customers) ←→ [test@test.com] ←→ Cluster B (200 customers)
                                     ↑
                            ARTICULATION POINT
                          (removal splits graph)
```

When a single email, phone, or SSN connects unrelated customer groups, it's typically **dirty data** - not fraud.

## 🎯 Why This Matters for Fraud Detection

### The Problem with Traditional Approaches

| Approach | What It Finds | Reality |
|----------|--------------|----------|
| **High Degree Nodes** | Most connected nodes | Often just dirty data |
| **Community Detection** | Customer clusters | Misses bridge nodes |
| **Pattern Matching** | Known fraud patterns | Can't adapt to new schemes |

### What Articulation Points Reveal

| Type | Description | Example | Action |
|------|-------------|---------|--------|
| 🗑️ **Dirty Data** | Placeholder values bridging clusters | `test@test.com` | Exclude |
| ✅ **Legitimate** | Real shared services | Corporate phone | Monitor |
| 🚨 **Fraud** | Intentional identity reuse | Synthetic SSN | Investigate |

## 🚀 Implementation Guide

### Step 1: Create Graph Projection

> 📝 **Note:** Relationships must be UNDIRECTED for articulation point detection

```cypher
CALL gds.graph.project(
    'customerArticulationGraph',
    // Include all relevant node types
    ['Customer', 'Email', 'Phone', 'SSN'],
    // Project relationships as undirected for connectivity analysis
    {
        HAS_EMAIL: {orientation: 'UNDIRECTED'},
        HAS_PHONE: {orientation: 'UNDIRECTED'}, 
        HAS_SSN: {orientation: 'UNDIRECTED'}
    }
);
```

### Step 2: Run Detection Algorithm

```cypher
CALL gds.articulationPoints.write('customerArticulationGraph', { 
    writeProperty: 'articulationPoint'
})
YIELD articulationPointCount;
```

✅ **Expected Output:** Typically 50-200 articulation points in a million-node graph

### Step 3: Analyze Detected Super Nodes

#### 📧 Email Super Nodes
```cypher
MATCH (e:Email)
WHERE e.articulationPoint = 1
WITH e, COUNT { (e)<-[:HAS_EMAIL]-() } AS connections
RETURN 
    e.address AS Address,
    connections AS ConnectedCustomers,
    CASE 
        WHEN e.address CONTAINS 'test' THEN '🗑️ Test Data'
        WHEN e.address STARTS WITH 'no' THEN '🗑️ Placeholder'
        WHEN connections > 3 THEN '⚠️ Suspicious'
        ELSE '✅ Review'
    END AS Classification
ORDER BY connections DESC
LIMIT 10;
```

#### 📱 Phone Super Nodes
```cypher
MATCH (p:Phone)
WHERE p.articulationPoint = 1
WITH p, COUNT { (p)<-[:HAS_PHONE]-() } AS connections
RETURN 
    p.number AS PhoneNumber,
    connections AS ConnectedCustomers,
    CASE 
        WHEN p.number CONTAINS '000' THEN '🗑️ Default'
        WHEN p.number CONTAINS '999' THEN '🗑️ Test'
        WHEN connections > 3 THEN '⚠️ Investigate'
        ELSE '✅ Legitimate'
    END AS Classification
ORDER BY connections DESC
LIMIT 10;
```

## 🛡️ Applying Results to Fraud Detection

### Option 1: WHERE Clause Filtering

```cypher
MATCH p=(c1:Customer)-[:HAS_EMAIL]->(email)<-[:HAS_EMAIL]-(c2:Customer)
WHERE email.articulationPoint IS NULL OR email.articulationPoint = 0
RETURN p;
```

**Pros:** Simple to implement  
**Cons:** Slower performance, cluttered queries

### Option 2: Label-Based Exclusion ✅ **RECOMMENDED**

#### Setup: Mark Super Nodes
```cypher
MATCH (n)
WHERE n.articulationPoint = 1
SET n:SuperConnector
RETURN COUNT(n) AS "Nodes Labeled";
```

#### Use in Queries: Clean & Fast
```cypher
MATCH p=(c1:Customer)-[:HAS_EMAIL]->(email:!SuperConnector)<-[:HAS_EMAIL]-(c2:Customer)
RETURN p;
```

## 📊 Ongoing Management Strategy

### Weekly Maintenance Tasks

```cypher
// -- 1. Drop existing projection (if exists)
CALL gds.graph.exists('customerArticulationGraph') YIELD exists
WHERE exists
CALL gds.graph.drop('customerArticulationGraph') YIELD graphName
RETURN graphName + ' dropped' AS status;

// -- 2. Create fresh graph projection with current data
CALL gds.graph.project(
    'customerArticulationGraph',
    ['Customer', 'Email', 'Phone', 'SSN'],
    {
        HAS_EMAIL: {orientation: 'UNDIRECTED'},
        HAS_PHONE: {orientation: 'UNDIRECTED'}, 
        HAS_SSN: {orientation: 'UNDIRECTED'}
    }
)
YIELD nodeCount, relationshipCount
RETURN nodeCount, relationshipCount;

// -- 3. Clear ALL previous articulation point flags
MATCH (n)
WHERE n.articulationPoint IS NOT NULL
SET n.articulationPoint = NULL
RETURN COUNT(n) AS "Articulation flags cleared";

// -- 4. Run fresh articulation point detection
CALL gds.articulationPoints.write('customerArticulationGraph', {
    writeProperty: 'articulationPoint'
})
YIELD articulationPointCount
RETURN articulationPointCount;

// -- 5. Remove SuperConnector label from non-articulation points
MATCH (n:SuperConnector)
WHERE n.articulationPoint IS NULL OR n.articulationPoint = 0
REMOVE n:SuperConnector
RETURN COUNT(n) AS "Labels removed from non-articulation points";

// -- 6. Add SuperConnector label to new articulation points
MATCH (n)
WHERE n.articulationPoint = 1 AND NOT n:SuperConnector
SET n:SuperConnector
RETURN COUNT(n) AS "New articulation points labeled";

// -- 7. Generate comprehensive status report
MATCH (n)
WHERE n.articulationPoint = 1
WITH labels(n)[0] AS nodeType,
     n,
     SIZE([(n)-[]-() | 1]) AS connections
WITH nodeType,
     COUNT(n) AS articulationPointCount,
     AVG(connections) AS avgConnections,
     MAX(connections) AS maxConnections,
     COLLECT(n)[0..3] AS examples
RETURN nodeType, 
       articulationPointCount,
       round(avgConnections, 1) AS avgConnections,
       maxConnections,
       [ex IN examples | 
           CASE labels(ex)[0]
               WHEN 'Email' THEN ex.address
               WHEN 'Phone' THEN ex.number
               WHEN 'SSN' THEN substring(ex.value, 0, 3) + '-XX-XXXX'
               ELSE ex.id
           END + ' (' + SIZE([(ex)-[]-() | 1]) + ' connections)'
       ] AS topExamples
ORDER BY articulationPointCount DESC;

// -- 8. Compare with previous week
MATCH (n)
WHERE n.articulationPoint = 1
WITH n, COALESCE(n.previousArticulation, 0) AS wasArticulation
SET n.previousArticulation = 1
WITH labels(n)[0] AS nodeType,
     SUM(CASE WHEN wasArticulation = 0 THEN 1 ELSE 0 END) AS newArticulationPoints,
     COUNT(n) AS totalArticulationPoints
RETURN nodeType, newArticulationPoints, totalArticulationPoints,
       round(100.0 * newArticulationPoints / totalArticulationPoints, 1) AS percentNew
ORDER BY newArticulationPoints DESC;

// -- 9. Clean up previous articulation markers for nodes no longer articulation points
MATCH (n)
WHERE n.previousArticulation = 1 AND (n.articulationPoint IS NULL OR n.articulationPoint = 0)
SET n.previousArticulation = 0
RETURN COUNT(n) AS "No longer articulation points";
```

### Data Quality Pipeline

| Stage | Action | Owner | Frequency |
|-------|--------|-------|-----------||
| **1. Detect** | Run articulation point algorithm | Data Science | Weekly |
| **2. Classify** | Review and categorize super nodes | Fraud Team | Weekly |
| **3. Report** | Send dirty data list upstream | Data Quality | Monthly |
| **4. Clean** | Fix at source systems | IT/Business | Ongoing |
| **5. Monitor** | Track improvement metrics | Analytics | Monthly |


## 🎉 Expected Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **False Positive Rate** | 40% | 5% | 87.5% reduction |
| **Query Performance** | 5-10s | <1s | 10x faster |
| **Alert Volume** | 1000/day | 150/day | 85% reduction |
| **Investigation Time** | 30 min/alert | 10 min/alert | 67% reduction |

---

## 📚 Additional Resources

- [Neo4j GDS Articulation Points Documentation](https://neo4j.com/docs/graph-data-science/current/algorithms/articulation-points/)