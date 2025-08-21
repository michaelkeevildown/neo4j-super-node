# 🎯 Closeness Centrality for Information Flow Analysis

> **Identify nodes positioned to rapidly spread information through your fraud network**

## 📖 What Is Closeness Centrality?

Closeness centrality measures how **quickly a node can reach all other nodes** in your graph. In fraud detection, nodes with high closeness centrality are positioned at the "center" of information flow - they're the shortest average distance from everyone else.

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

When an identifier has high closeness centrality, it can **rapidly propagate fraud patterns** or data quality issues throughout your network.

## 🎯 Why This Matters for Fraud Detection

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
| 🟠 **0.6-0.8** | Well-connected | Possible synthetic identity | Deep investigation |
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
        WHEN score > 0.6 AND directConnections > 10 THEN '⚠️ Synthetic Identity Risk'
        WHEN score > 0.5 THEN '🟡 Monitor Closely'
        ELSE '✅ Normal'
    END AS Classification,
    CASE
        WHEN score > 0.8 THEN 'Auto-exclude from fraud detection'
        WHEN score > 0.6 THEN 'Flag for manual review'
        ELSE 'Include with monitoring'
    END AS Action
ORDER BY score DESC
LIMIT 25;
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
        WHEN score > 0.6 AND directConnections > 10 THEN '⚠️ Fraud Ring Center'
        WHEN score > 0.5 THEN '🟡 Information Broker'
        ELSE '✅ Normal'
    END AS Classification
ORDER BY score DESC
LIMIT 25;
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
        WHEN score > 0.6 THEN '🔴 Critical - Identity Theft Hub'
        WHEN s.ssnNumber STARTS WITH '000' THEN '🟠 Invalid SSN'
        WHEN score > 0.4 AND directConnections > 5 THEN '⚠️ Synthetic Identity'
        WHEN score > 0.3 THEN '🟡 Suspicious Activity'
        ELSE '✅ Normal'
    END AS Classification,
    'Immediate investigation required' AS Priority
ORDER BY score DESC
LIMIT 25;
```

## 🛡️ Applying Results to Fraud Detection

### Option 1: Score-Based Filtering

```cypher
// Find connections excluding high-centrality nodes
MATCH p=(c1:Customer)-[:HAS_EMAIL]->(email)<-[:HAS_EMAIL]-(c2:Customer)
WHERE email.closenessScore < 0.5
AND c1 <> c2
RETURN p;
```

**Pros:** Granular control with score thresholds
**Cons:** Requires score calculation, slower queries

### Option 2: Combined Centrality Analysis ✅ **RECOMMENDED**

#### Setup: Multi-Metric Super Node Detection
```cypher
// Combine closeness with degree centrality for robust detection
MATCH (n)
WHERE n.closenessScore > 0.6 OR n.degreeScore > 50
SET n:InformationHub
RETURN COUNT(n) AS InformationHubsLabeled;

// Mark extreme cases for automatic exclusion
MATCH (n)
WHERE n.closenessScore > 0.8 AND n.degreeScore > 100
SET n:SuperConnector
RETURN COUNT(n) AS SuperConnectorsLabeled;
```

#### Use in Queries: Efficient Multi-Level Filtering
```cypher
// Find fraud patterns excluding information hubs
MATCH p=(c1:Customer)-[:HAS_PHONE]->(phone:!InformationHub)<-[:HAS_PHONE]-(c2:Customer)
WHERE c1 <> c2
AND NOT phone:SuperConnector
RETURN p;

// Include hubs but flag risk level
MATCH p=(c1:Customer)-[:HAS_EMAIL]->(email)<-[:HAS_EMAIL]-(c2:Customer)
WHERE c1 <> c2
RETURN p,
    CASE 
        WHEN email:SuperConnector THEN '🔴 Excluded Hub'
        WHEN email:InformationHub THEN '🟠 Information Hub'
        WHEN email.closenessScore > 0.4 THEN '🟡 Well Connected'
        ELSE '🟢 Normal'
    END AS RiskFlag;
```

## 📊 Ongoing Management Strategy

### Weekly Maintenance Tasks

```cypher
// -- 1. Drop existing projection (if exists)
CALL gds.graph.exists('customerClosenessGraph') YIELD exists
WHERE exists
CALL gds.graph.drop('customerClosenessGraph') YIELD graphName
RETURN graphName + ' dropped' AS status;

// -- 2. Create fresh graph projection with current data
CALL gds.graph.project(
    'customerClosenessGraph',
    ['Customer', 'Email', 'Phone', 'SSN'],
    {
        HAS_EMAIL: {orientation: 'UNDIRECTED'},
        HAS_PHONE: {orientation: 'UNDIRECTED'}, 
        HAS_SSN: {orientation: 'UNDIRECTED'}
    }
)
YIELD nodeCount, relationshipCount, projectMillis
RETURN nodeCount, relationshipCount, 
       projectMillis + ' ms' AS projectionTime;

// -- 3. Clear ALL previous closeness scores
MATCH (n)
WHERE n.closenessScore IS NOT NULL
SET n.closenessScore = NULL
RETURN COUNT(n) AS ScoresCleared;

// -- 4. Calculate fresh closeness centrality scores
CALL gds.closeness.write('customerClosenessGraph', {
    writeProperty: 'closenessScore'
})
YIELD centralityDistribution, nodePropertiesWritten, computeMillis
RETURN 
    nodePropertiesWritten AS nodesScored,
    round(centralityDistribution.mean, 3) AS avgCloseness,
    round(centralityDistribution.p90, 3) AS p90Closeness,
    round(centralityDistribution.p99, 3) AS p99Closeness,
    round(centralityDistribution.max, 3) AS maxCloseness,
    computeMillis + ' ms' AS computationTime;

// -- 5. Remove InformationHub label from low-score nodes
MATCH (n:InformationHub)
WHERE n.closenessScore IS NULL OR n.closenessScore < 0.6
REMOVE n:InformationHub
RETURN COUNT(n) AS InformationHubLabelsRemoved;

// -- 6. Add InformationHub label to high-centrality nodes
MATCH (n)
WHERE n.closenessScore >= 0.6 AND NOT n:InformationHub
SET n:InformationHub
RETURN COUNT(n) AS InformationHubLabelsAdded;

// -- 7. Update SuperConnector labels for extreme cases
MATCH (n:SuperConnector)
WHERE n.closenessScore IS NULL OR n.closenessScore < 0.8
REMOVE n:SuperConnector
RETURN COUNT(n) AS SuperConnectorLabelsRemoved;

MATCH (n)
WHERE n.closenessScore >= 0.8 AND NOT n:SuperConnector
SET n:SuperConnector
RETURN COUNT(n) AS SuperConnectorLabelsAdded;

// -- 8. Generate comprehensive status report
MATCH (n)
WHERE n.closenessScore IS NOT NULL
WITH labels(n)[0] AS nodeType,
    CASE
        WHEN n.closenessScore >= 0.8 THEN '🔴 Central Hub (0.8+)'
        WHEN n.closenessScore >= 0.6 THEN '🟠 Well Connected (0.6-0.8)'
        WHEN n.closenessScore >= 0.4 THEN '🟡 Moderate (0.4-0.6)'
        WHEN n.closenessScore >= 0.2 THEN '🟢 Normal (0.2-0.4)'
        ELSE '⚫ Peripheral (<0.2)'
    END AS centralityCategory,
    COUNT(n) AS nodeCount,
    COLLECT(n)[0..3] AS examples
RETURN nodeType, centralityCategory, nodeCount,
       [ex IN examples | 
           CASE labels(ex)[0]
               WHEN 'Email' THEN ex.address
               WHEN 'Phone' THEN ex.phoneNumber
               WHEN 'SSN' THEN substring(ex.ssnNumber, 0, 3) + '-XX-XXXX'
               ELSE ex.id
           END + ' (' + round(ex.closenessScore, 3) + ')'
       ] AS topExamples
ORDER BY nodeType, centralityCategory;

// -- 9. Cross-reference with other centrality metrics
MATCH (n)
WHERE n.closenessScore > 0.6 OR n.degreeScore > 50 OR n.articulationPoint = 1
WITH n,
     n.closenessScore AS closeness,
     n.degreeScore AS degree,
     n.articulationPoint AS articulation
RETURN 
    labels(n)[0] AS nodeType,
    COUNT(n) AS suspiciousNodes,
    COUNT(CASE WHEN closeness > 0.6 THEN 1 END) AS highCloseness,
    COUNT(CASE WHEN degree > 50 THEN 1 END) AS highDegree,
    COUNT(CASE WHEN articulation = 1 THEN 1 END) AS articulationPoints,
    COUNT(CASE WHEN closeness > 0.6 AND degree > 50 THEN 1 END) AS multiMetricFlags
ORDER BY suspiciousNodes DESC;
```

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

---

## 📚 Additional Resources

- [Neo4j GDS Closeness Centrality Documentation](https://neo4j.com/docs/graph-data-science/current/algorithms/closeness-centrality/)
- [Understanding Centrality Metrics](https://neo4j.com/docs/graph-data-science/current/algorithms/centrality/)
- [Information Flow in Networks](https://neo4j.com/developer/graph-algorithms/)