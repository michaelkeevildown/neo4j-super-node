# 📊 Degree Centrality Without GDS - Pure Cypher Approach

> **Identify highly connected nodes using only Cypher queries - no GDS installation required**

## ⚠️ Important: GDS is the Superior Approach

> **Neo4j Graph Data Science (GDS) library is BY FAR the better approach for degree centrality calculations.**

### Why GDS is Better:
- **🚀 Performance**: 10-100x faster on large graphs through optimized parallel algorithms
- **💾 Memory Efficiency**: Uses efficient graph projections and compressed data structures
- **⚡ Scalability**: Handles millions of nodes and relationships with ease
- **🔧 Built-in Optimizations**: Native implementations with C++ performance
- **📊 Additional Algorithms**: Access to 60+ graph algorithms beyond just degree centrality
- **🎯 Production Ready**: Battle-tested for enterprise workloads

### When to Use GDS Instead:
- Graphs with >100K nodes
- Real-time or near real-time requirements
- Production environments
- When you need other centrality measures (PageRank, Betweenness, etc.)
- Regular batch processing of large datasets

### Why This Guide Exists:
This pure Cypher approach is provided for:
- **Environments where GDS cannot be installed** (restricted systems, cloud limitations)
- **Quick exploratory analysis** on small graphs (<10K nodes)
- **Learning and understanding** the concepts before using GDS
- **Proof of concepts** where GDS licensing is not yet available

**👉 If you can install GDS, use [DEGREE_CENTRALITY.md](./DEGREE_CENTRALITY.md) instead for significantly better performance.**

---

## 📖 Overview

This guide provides pure Cypher alternatives to GDS degree centrality for detecting super nodes in your fraud detection graph. Perfect for environments where GDS is not available or when you need quick, lightweight analysis.

### What You'll Learn
- Calculate degree centrality using COUNT aggregations
- Detect and mark super nodes efficiently
- Exclude placeholder/test data from fraud analysis
- Maintain degree scores without graph projections

## 🎯 Quick Start: Find Your Super Nodes

### Instant Super Node Detection
```cypher
// Find top connected nodes across all types
CYPHER runtime=parallel
CALL () {
  // Process Email nodes
  MATCH (n:Email)<-[:HAS_EMAIL]-(c:Customer)
  WITH n, COUNT(DISTINCT c) as degree
  WHERE degree > 1
  RETURN 
    n,
    'Email' as nodeType,
    n.address as identifier,
    degree
  
  UNION ALL
  
  // Process Phone nodes  
  MATCH (n:Phone)<-[:HAS_PHONE]-(c:Customer)
  WITH n, COUNT(DISTINCT c) as degree
  WHERE degree > 1
  RETURN 
    n,
    'Phone' as nodeType,
    n.phoneNumber as identifier,
    degree
    
  UNION ALL
  
  // Process SSN nodes
  MATCH (n:SSN)<-[:HAS_SSN]-(c:Customer)
  WITH n, COUNT(DISTINCT c) as degree
  WHERE degree > 1
  RETURN 
    n,
    'SSN' as nodeType,
    n.ssnNumber as identifier,
    degree
}

RETURN 
    nodeType,
    identifier,
    degree as connections,
    CASE
        WHEN degree > 100 THEN '🔴 Super Node - Exclude'
        WHEN degree > 50 THEN '🟠 High Risk - Review'
        WHEN degree > 5 THEN '🟡 Monitor'
        ELSE '✅ Normal'
    END as classification
ORDER BY degree DESC
LIMIT 50;
```

## 🚀 Core Implementation Methods

### Method 1: On-the-Fly Calculation (No Storage)

#### 📧 Email Super Nodes
```cypher
// Real-time email degree calculation
CYPHER runtime=parallel
CALL () {
  // Process Email nodes
  MATCH (n:Email)<-[:HAS_EMAIL]-(c:Customer)
  WITH n, COUNT(DISTINCT c) as degree
  WHERE degree > 1
  RETURN 
    n,
    'Email' as nodeType,
    n.address as identifier,
    degree
}

RETURN 
    nodeType,
    identifier,
    degree as connections,
    CASE
        WHEN degree > 100 THEN '🔴 Super Node - Exclude'
        WHEN degree > 50 THEN '🟠 High Risk - Review'
        WHEN degree > 5 THEN '🟡 Monitor'
        ELSE '✅ Normal'
    END as classification
ORDER BY degree DESC
LIMIT 50;
```

#### 📱 Phone Super Nodes
```cypher
// Real-time phone degree calculation
CYPHER runtime=parallel
CALL () {
  // Process Phone nodes  
  MATCH (n:Phone)<-[:HAS_PHONE]-(c:Customer)
  WITH n, COUNT(DISTINCT c) as degree
  WHERE degree > 1
  RETURN 
    n,
    'Phone' as nodeType,
    n.phoneNumber as identifier,
    degree
}

RETURN 
    nodeType,
    identifier,
    degree as connections,
    CASE
        WHEN degree > 100 THEN '🔴 Super Node - Exclude'
        WHEN degree > 50 THEN '🟠 High Risk - Review'
        WHEN degree > 5 THEN '🟡 Monitor'
        ELSE '✅ Normal'
    END as classification
ORDER BY degree DESC
LIMIT 50;
```

#### 🆔 SSN Super Nodes
```cypher
// Real-time SSN degree calculation (with masking)
CYPHER runtime=parallel
CALL () {
  // Process SSN nodes
  MATCH (n:SSN)<-[:HAS_SSN]-(c:Customer)
  WITH n, COUNT(DISTINCT c) as degree
  WHERE degree > 1
  RETURN 
    n,
    'SSN' as nodeType,
    n.ssnNumber as identifier,
    degree
}

RETURN 
    nodeType,
    identifier,
    degree as connections,
    CASE
        WHEN degree > 100 THEN '🔴 Super Node - Exclude'
        WHEN degree > 50 THEN '🟠 High Risk - Review'
        WHEN degree > 5 THEN '🟡 Monitor'
        ELSE '✅ Normal'
    END as classification
ORDER BY degree DESC
LIMIT 50;
```

### Method 2: Store Degree as Property

#### Step 1: Calculate and Store All Degrees
```cypher
// Calculate degree for all Email nodes
MATCH (e:Email)
OPTIONAL MATCH (e)<-[:HAS_EMAIL]-(c:Customer)
WITH e, COUNT(DISTINCT c) as degree
SET e.degreeScore = degree
RETURN COUNT(e) as emailsProcessed, MAX(degree) as maxDegree, AVG(degree) as avgDegree;
```

```cypher
// Calculate degree for all Phone nodes
MATCH (p:Phone)
OPTIONAL MATCH (p)<-[:HAS_PHONE]-(c:Customer)
WITH p, COUNT(DISTINCT c) as degree
SET p.degreeScore = degree
RETURN COUNT(p) as phonesProcessed, MAX(degree) as maxDegree, AVG(degree) as avgDegree;
```

```cypher
// Calculate degree for all SSN nodes
MATCH (s:SSN)
OPTIONAL MATCH (s)<-[:HAS_SSN]-(c:Customer)
WITH s, COUNT(DISTINCT c) as degree
SET s.degreeScore = degree
RETURN COUNT(s) as ssnsProcessed, MAX(degree) as maxDegree, AVG(degree) as avgDegree;
```

#### Step 2: Query Using Stored Scores
```cypher
// Fast lookup of high-degree nodes
MATCH (n:Email | Phone | SSN)
WHERE n.degreeScore > 5
RETURN 
    labels(n)[0] as type,
    n.degreeScore as connections,
    CASE labels(n)[0]
        WHEN 'Email' THEN n.address
        WHEN 'Phone' THEN n.phoneNumber
        WHEN 'SSN' THEN n.ssnNumber
    END as identifier
ORDER BY n.degreeScore DESC
LIMIT 30;
```

### Method 3: Label-Based Classification ✅ **RECOMMENDED**

#### Setup: Mark Nodes by Degree Thresholds
```cypher
// Step 1: Clear existing labels
MATCH (n:SuperConnector)
REMOVE n:SuperConnector
RETURN COUNT(n) as removedSuperConnector;

MATCH (n:MonitorNode)
REMOVE n:MonitorNode
RETURN COUNT(n) as removedMonitor;

// Step 2: Label high-degree nodes (50+ connections)
MATCH (n:Email | Phone | SSN)
OPTIONAL MATCH (n)<--(c:Customer)
WITH n, COUNT(DISTINCT c) as degree
WHERE degree >= 5
SET n:SuperConnector
RETURN COUNT(n) as markedAsSuperConnector;

// Step 3: Label medium-degree nodes (20-49 connections)
MATCH (n:Email | Phone | SSN)
OPTIONAL MATCH (n)<--(c:Customer)
WITH n, COUNT(DISTINCT c) as degree
WHERE 1 < degree < 5 
SET n:MonitorNode
RETURN COUNT(n) as markedForMonitoring;
```

#### Use Labels for Efficient Filtering
```cypher
// Find shared phones excluding super connectors
MATCH path=(c1:Customer)-[:HAS_PHONE]->(p:Phone & !SuperConnector)<-[:HAS_PHONE]-(c2:Customer)
RETURN path
LIMIT 100;

// Include monitor nodes but flag them
MATCH (c1:Customer)-[:HAS_EMAIL]->(e:Email)<-[:HAS_EMAIL]-(c2:Customer)
RETURN 
    c1.id as customer1, 
    c2.id as customer2, 
    e.address as sharedEmail,
    CASE 
        WHEN e:SuperConnector THEN '🔴 EXCLUDED - Super Node'
        WHEN e:MonitorNode THEN '🟡 WARNING - High Degree'
        ELSE '✅ Normal Sharing'
    END as status
LIMIT 100;
```


## 📚 Next Steps

1. Run the Quick Start query to identify your super nodes
2. Choose between on-the-fly or stored degree calculation
3. Implement the label-based classification for best performance
4. Set up daily maintenance procedures
5. Adjust thresholds based on your data patterns

Remember: The goal is to filter out noise (test data, placeholders) so you can focus on genuine fraud patterns in your graph.