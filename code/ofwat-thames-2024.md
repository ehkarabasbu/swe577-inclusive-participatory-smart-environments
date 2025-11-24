# Code Dive: Thames Water (2024) - When GIS Meets Graph Databases

**Paper:** Unlocking Digital Twins Toolkit <br>
**Authors:** Thames Water & Ofwat Initiative <br>
[**Official Repository**](https://github.com/Sand-EnterpriseAI/udt-clean-water-toolkit) <br>
**Internal Wiki Analysis:** [Paper-Analysis-ThamesWater-2024](https://github.com/ehkarabasbu/swe577-inclusive-participatory-smart-environments/wiki/Related-Work#thameswater-2024---unlocking-digital-twins-toolkit)

---

## 1. The Problem Nobody Talks About

Water infrastructure lives in PostGIS. Fine. You need spatial queries ("where is this pipe?"), GIS handles that. But ask "what happens if valve V123 fails?" ( that's not spatial anymore ). That's network analysis. Flow direction, downstream assets, affected customers. PostGIS isn't built for that.

So they use two databases. PostGIS for location, Neo4j for relationships. Dual-database pattern.

### `cwm/transformers/gis_to_graph.py`

Here's the conversion - turning geospatial data into graph topology.

```python
# Conceptual structure from CWM library
class NetworkTransformer:
    """Transforms GIS data to graph representation"""
    
    def transform_to_graph(self, gis_data):
        # Create nodes from point features (junctions, valves, hydrants)
        nodes = self._create_network_nodes(gis_data.points)
        
        # Create edges from line features (pipes) with topology
        edges = self._create_network_edges(gis_data.lines, nodes)
        
        # Keep the spatial reference so we can map it later
        return GraphNetwork(nodes, edges, spatial_ref=gis_data.crs)
    
    def _create_network_edges(self, lines, nodes):
        """Build edges from pipes, maintaining flow direction"""
        edges = []
        for line in lines:
            start_node = self._find_nearest_node(line.start_point, nodes)
            end_node = self._find_nearest_node(line.end_point, nodes)
            
            edges.append({
                'from': start_node.id,
                'to': end_node.id,
                'pipe_id': line.id,
                'material': line.material,
                'diameter': line.diameter,
                'flow_direction': 'DOWNSTREAM'
            })
        return edges
```

Once it's in Neo4j, you can run graph queries that would take hours in traditional GIS. That's the whole point.

---

## 2. Cypher Queries: Why This Actually Works

Neo4j uses Cypher. Readable syntax with ASCII art like `()-[]->()`. Here's what you can do.

### Finding Downstream Assets

```cypher
// Find everything downstream of a valve
MATCH (valve:Valve {id: 'V123'})-[:CONNECTED_TO*]->(downstream)
RETURN downstream

// Translation: Start at V123, follow CONNECTED_TO any number of hops
```

This is from Screenshot 2 in the paper. Answers "if I close this valve, what breaks?" in milliseconds. Try that in PostGIS - you'd be writing procedural code for hours.

### Material-Based Analysis

```cypher
// Find pipe connections by material
MATCH (n1)-[r:pipe_main]->(n2)
WHERE r.material = "Copper"
RETURN n1, n2
```

Super useful for planning copper pipe replacements across the network. Screenshot 2 example.

---

## 3. Modular Split: CWM + CWA

They separated the library from the app. Smart.

**CWM (Clean Water Module)**: Just the algorithms. Pure Python, no web stuff.
- GIS-to-graph transformation
- Network analysis (NetworkX)
- Water flow simulation (WNTR)

**CWA (Clean Water Application)**: Django wrapper.
- REST API
- Database connections
- Docker deployment

Why split? Small utilities might just want the library. Large ones deploy the full stack. Flexible adoption.

---

## 4. Synthetic Data Generator

Here's the clever part. Real water network data is sensitive (security risk). So they generate realistic fake networks. This enables:
- Academic research without data access barriers
- Third-party contributions
- Open-source community growth

```python
# Conceptual from their approach
class SyntheticNetworkGenerator:
    def generate_network(self, size='medium', topology='grid'):
        # Create topology
        network = self._create_topology(size, topology)
        
        # Add realistic constraints (pipe sizing, pressure)
        network = self._add_realistic_constraints(network)
        
        # Export both formats
        return {
            'gis': network.to_postgis(),
            'graph': network.to_neo4j()
        }
```

Unlike Mendonça's IDS (needs real building data), anyone can contribute here without security clearance. Participatory infrastructure development.

---

## Infrastructure Scale, Revolutionary Governance

The tech is solid. But the governance model? That's the real story. Competing water companies (Thames Water, Severn Trent Water) collaborating on open-source tools funded by public money (Ofwat). Sanders' "user as partner" at national infrastructure scale.

The gap: This handles water beautifully but doesn't connect to building BIM (Dave) or urban twins (Snap4City). A water pipe running through a building exists in three separate models with zero integration.
