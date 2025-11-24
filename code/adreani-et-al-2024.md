# Code Dive: Adreani et al. (2024) - How to Build a City-Scale Digital Twin

**Paper:** Smart City Digital Twin Framework for Real-Time Multi-Data Integration <br>
**Authors:** Adreani, L., Bellini, P., Benigni, M., & Nesi, P. <br>
[**Official Repository**](https://github.com/disit/snap4city) <br>
**Internal Wiki Analysis:** [Paper-Analysis-Adreani-2024](https://github.com/ehkarabasbu/swe577-inclusive-participatory-smart-environments/wiki/Related-Work#adreani-et-al-2024---smart-city-digital-twin-framework)

---

## 1. The Federation API: One Door to All City Data

Snap4City has data coming in from everywhere - IoT sensors, BIM models, social media, traffic cameras, weather stations. How do you make sense of that chaos? The Federation API is their answer. It's a single access point that hides all the complexity.

### `Snap4CityCore/api/federation.js`

This is the conceptual structure based on their architecture diagrams. One API to rule them all.

```javascript
// Conceptual structure from Snap4City architecture
class FederationAPI {
    constructor() {
        this.dataSources = {
            iot: new FIWAREBroker(),
            bim: new BIMServerClient(),
            gis: new GeoServerClient(),
            opendata: new CKANClient()
        };
    }
    
    async getEntityData(entityType, filters) {
        // Query across multiple data sources
        // Returns normalized city entity data
        const results = await Promise.all([
            this.dataSources.iot.query(entityType, filters),
            this.dataSources.gis.query(entityType, filters)
        ]);
        
        return this.normalize(results);
    }
    
    async runWhatIfScenario(changes) {
        // Citizen proposes: "Close Via Cavour for bike lane"
        // System reasons: affects 12 roads, impacts 3 bus routes
        // Simulates: +15% bike traffic, -8% car traffic, -12% local pollution
        // Returns: 3D visualization of the impact
        const simulation = new TrafficSimulator();
        const impact = await simulation.run(changes);
        
        return {
            trafficFlow: impact.traffic,
            pollutionChange: impact.pollution,
            visualization: this.generate3D(impact)
        };
    }
}
```

This is Sanders' co-creation philosophy meeting actual code. Citizens can propose changes through the web interface, the API runs the simulation, and they see the results in 3D. That's participatory urban planning, not just participatory design.

---

## 2. The Semantic Backbone: Why RDF Matters

Figure 2 in their paper shows the road graph data model. It's not just storing "Road" objects in a database - it's building a knowledge graph where the data knows its own meaning.

### `KM4City/ontology/road_network.ttl`

This is what makes the magic happen. RDF triples that let the system reason about relationships.

```turtle
# Conceptual RDF structure for KM4City road network
@prefix km4c: <http://www.disit.org/km4city/schema#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .

:Road_Via_Cavour a km4c:Road ;
    km4c:belongToRoad :RoadNetwork_Florence ;
    km4c:hasMunicipality :Municipality_Florence ;
    km4c:containsElement :RoadElement_1, :RoadElement_2 ;
    km4c:hasRestriction :SpeedLimit_50kmh .

:RoadElement_1 a km4c:RoadElement ;
    km4c:startsAtNode :Node_123 ;
    km4c:endsAtNode :Node_124 ;
    km4c:lanesCount 2 ;
    km4c:where [
        a km4c:Lanes ;
        rdf:_1 :Lane_1 ;
        rdf:_2 :Lane_2
    ] ;
    km4c:restrictions [
        a rdf:Bag ;
        rdf:_1 :TurnRestriction_NoLeftTurn ;
        rdf:_2 :AccessRestriction_BusOnly
    ] .
```

Why does this matter? Because now you can ask questions like "show me all bike lanes affected by closing Via Cavour" and the system can reason through the graph to find the answer. Dave's O-MI/O-DF was simpler - just property values. This is full semantic reasoning.

---

## 3. BIM Meets City: The Snap4BIM Integration

The Snap4BIM folder in their repo shows how they bridge architectural models into the city digital twin. This is the piece that extends Dave's building-level work to urban scale.

### `Snap4BIM/viewer/bim_integration.js`

Here's how they merge BIM models with real-time city data.

```javascript
// Conceptual from Snap4BIM architecture
class DigitalTwinViewer {
    constructor() {
        this.bimsurfer = new BIMSurfer();  // 3D WebGL viewer
        this.geoserver = new GeoServerClient();
        this.iot = new FIWAREClient();
    }
    
    async loadCityModel(cityId) {
        // 1. Load 3D buildings from BIMserver
        const buildings = await this.loadBIMModels(cityId);
        this.bimsurfer.load(buildings);
        
        // 2. Load GIS context from GeoServer
        const context = await this.geoserver.getCityContext(cityId);
        this.bimsurfer.addContext(context);
        
        // 3. Overlay real-time IoT data
        const sensorData = await this.iot.subscribe(cityId);
        this.updateRealTimeData(sensorData);
    }
    
    updateRealTimeData(sensorData) {
        // Temperature sensors in buildings
        // Traffic flow on roads
        // Air quality at monitoring stations
        // All updating in real-time on the 3D model
        this.bimsurfer.updateLayers(sensorData);
    }
}
```

This is exactly what we need for our framework. Buildings (from BIM/IFC) in their spatial context (from GIS) with live sensor data (from IoT). And citizens can interact with all of it through a web browser.

---

## The Multi-Scale Picture

Aavild gave us building-scale distribution. Snap4City gives us urban-scale integration. The pattern is the same at both levels - containerized deployment (Docker), open standards (FIWARE, BIMserver), and web-based participation. What's missing is the bridge between them. How does data from Aavild's distributed home sensors feed into Snap4City's urban twin? That connection doesn't exist yet.
