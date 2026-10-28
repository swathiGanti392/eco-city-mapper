# Eco City Mapper Architecture

## Overview

This document details the technical architecture for a comprehensive environmental mapping system that correlates Air Quality Index (AQI) with land use patterns, water bodies, and vehicle density to identify urban greening opportunities.

## System Architecture

### High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      Data Sources                            │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   AQI        │  │  Satellite   │  │  Vehicle     │     │
│  │   Stations   │  │  Imagery     │  │  Registries  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                 │                 │               │
│         └─────────────────┼─────────────────┘               │
│                           │                                 │
└───────────────────────────┼─────────────────────────────────┘
                            │ API/Crawlers
┌───────────────────────────┼─────────────────────────────────┐
│                      Data Ingestion                          │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  AQI         │  │  Satellite   │  │  Vehicle     │     │
│  │  Ingestor    │  │  Ingestor    │  │  Ingestor    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                 │                 │               │
│         └─────────────────┼─────────────────┘               │
│                           │                                 │
└───────────────────────────┼─────────────────────────────────┘
                            │
┌───────────────────────────┼─────────────────────────────────┐
│                      Data Processing                         │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Land Use    │  │  Spatial     │  │  Quality     │     │
│  │  Classifier  │  │  Processor   │  │  Control     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                 │                 │               │
│         └─────────────────┼─────────────────┘               │
│                           │                                 │
└───────────────────────────┼─────────────────────────────────┘
                            │
┌───────────────────────────┼─────────────────────────────────┐
│                      Data Storage                             │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  PostgreSQL  │  │  Object      │  │  Time Series │     │
│  │  (PostGIS)   │  │  Storage     │  │  Database   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────┼─────────────────────────────────┐
│                      Analysis Engine                          │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Correlation │  │  Opportunity │  │  Impact      │     │
│  │  Analyzer    │  │  Identifier  │  │  Calculator  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────┼─────────────────────────────────┐
│                      User Interface                           │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Web         │  │  Interactive │  │  API         │     │
│  │  Dashboard   │  │  Maps        │  │  Endpoints   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## Data Ingestion

### AQI Data Ingestor

#### OpenAQ Integration
```python
class OpenAQIngestor:
    def __init__(self):
        self.api_url = "https://api.openaq.org/v2"
        self.client = OpenAQ()

    def fetch_measurements(self, city, date_from, date_to):
        """Fetch AQI measurements for a city"""
        measurements = self.client.measurements(
            city=city,
            date_from=date_from,
            date_to=date_to,
            parameter=["pm25", "pm10", "o3", "no2", "so2", "co"]
        )
        return self.normalize_measurements(measurements)

    def normalize_measurements(self, measurements):
        """Normalize measurements to standard format"""
        normalized = []
        for m in measurements:
            normalized.append({
                'city': m['city'],
                'country': m['country'],
                'parameter': m['parameter'],
                'value': m['value'],
                'unit': m['unit'],
                'timestamp': m['date']['utc'],
                'coordinates': {
                    'lat': m['coordinates']['latitude'],
                    'lon': m['coordinates']['longitude']
                }
            })
        return normalized
```

#### Local Monitoring Stations
```python
class StationIngestor:
    def __init__(self, station_config):
        self.stations = station_config['stations']
        self.parsers = {
            'cpcb': self.parse_cpcb,
            'airnow': self.parse_airnow,
            'eea': self.parse_eea
        }

    def fetch_all_stations(self):
        """Fetch data from all configured stations"""
        data = []
        for station in self.stations:
            parser = self.parsers[station['type']]
            station_data = parser(station['url'])
            data.extend(station_data)
        return data
```

### Satellite Imagery Ingestor

#### Sentinel-2 Integration
```python
class SentinelIngestor:
    def __init__(self):
        self.sentinel_hub = SentinelHub()

    def fetch_imagery(self, bbox, date_from, date_to):
        """Fetch Sentinel-2 imagery for area"""
        request = SentinelHubRequest(
            evalscript=self.evalscript,
            input_data=[
                SentinelHubRequest.input_data(
                    data_collection=DataCollection.SENTINEL2_L2A,
                    time_interval=(date_from, date_to),
                    mosaicking_order=MosaickingOrder.LEAST_RECENT
                )
            ],
            responses=[SentinelHubRequest.output_response('default', MIMEType.TIFF)],
            bbox=bbox,
            size=(512, 512)
        )
        return request.get_data()
```

#### Landsat Integration
```python
class LandsatIngestor:
    def __init__(self):
        self.usgs_eros = USGSEROS()

    def fetch_imagery(self, bbox, date_from, date_to):
        """Fetch Landsat imagery for area"""
        scenes = self.usgs_eros.search(
            bbox=bbox,
            date_range=(date_from, date_to),
            dataset='LANDSAT_8_C1'
        )
        return [self.download_scene(scene) for scene in scenes]
```

### Vehicle Data Ingestor

#### National Registry API
```python
class VehicleIngestor:
    def __init__(self):
        self.apis = {
            'usa': self.fetch_usa_data,
            'india': self.fetch_india_data,
            'eu': self.fetch_eu_data
        }

    def fetch_by_city(self, country, city):
        """Fetch vehicle data for a city"""
        fetcher = self.apis[country]
        return fetcher(city)

    def aggregate_data(self, vehicle_data):
        """Aggregate vehicle data by city"""
        aggregated = {}
        for record in vehicle_data:
            city = record['city']
            if city not in aggregated:
                aggregated[city] = {
                    'total': 0,
                    'electric': 0,
                    'diesel': 0,
                    'gasoline': 0
                }
            aggregated[city]['total'] += record['count']
            aggregated[city][record['fuel_type']] += record['count']
        return aggregated
```

## Data Processing

### Land Use Classification

#### Machine Learning Pipeline
```python
class LandUseClassifier:
    def __init__(self):
        self.model = self.load_model()
        self.preprocessor = ImagePreprocessor()

    def classify_imagery(self, image_path):
        """Classify land use from satellite imagery"""
        # Preprocess image
        processed = self.preprocessor.process(image_path)

        # Predict land use
        predictions = self.model.predict(processed)

        # Post-process
        classified = self.postprocess(predictions)

        return classified

    def postprocess(self, predictions):
        """Convert predictions to land use classes"""
        classes = {
            0: 'concrete',
            1: 'barren',
            2: 'water',
            3: 'green',
            4: 'mixed'
        }
        return [classes[p] for p in predictions]
```

#### Feature Extraction
```python
class FeatureExtractor:
    def extract_ndvi(self, image):
        """Calculate NDVI (Normalized Difference Vegetation Index)"""
        nir = image[:, :, 3]  # Near-infrared band
        red = image[:, :, 2]  # Red band
        ndvi = (nir - red) / (nir + red)
        return ndvi

    def extract_ndwi(self, image):
        """Calculate NDWI (Normalized Difference Water Index)"""
        green = image[:, :, 1]  # Green band
        nir = image[:, :, 3]  # Near-infrared band
        ndwi = (green - nir) / (green + nir)
        return ndwi

    def extract_nir_swir(self, image):
        """Calculate NIR-SWIR ratio for built-up areas"""
        nir = image[:, :, 3]
        swir = image[:, :, 4]
        ratio = nir / swir
        return ratio
```

### Spatial Processing

#### Geospatial Alignment
```python
class SpatialProcessor:
    def __init__(self):
        self.crs = 'EPSG:4326'  # WGS84

    def align_datasets(self, datasets):
        """Align all datasets to common CRS and resolution"""
        aligned = {}
        for name, data in datasets.items():
            # Reproject to common CRS
            reprojected = self.reproject(data, self.crs)
            # Resample to common resolution
            resampled = self.resample(reprojected, target_resolution=10)
            aligned[name] = resampled
        return aligned

    def reproject(self, data, target_crs):
        """Reproject data to target CRS"""
        return data.to_crs(target_crs)

    def resample(self, data, target_resolution):
        """Resample data to target resolution"""
        return data.rio.reproject(
            data.rio.crs,
            shape=(data.rio.height * 10 // target_resolution,
                   data.rio.width * 10 // target_resolution)
        )
```

#### Zonal Statistics
```python
class ZonalStatistics:
    def calculate_land_use_percentages(self, land_use_raster, city_boundary):
        """Calculate percentage of each land use type within city boundary"""
        # Clip raster to city boundary
        clipped = self.clip_raster(land_use_raster, city_boundary)

        # Count pixels by class
        counts = self.count_by_class(clipped)

        # Calculate percentages
        total = sum(counts.values())
        percentages = {k: v/total*100 for k, v in counts.items()}

        return percentages

    def clip_raster(self, raster, boundary):
        """Clip raster to boundary polygon"""
        return raster.clip(boundary)
```

## Analysis Engine

### Correlation Analyzer

#### Statistical Correlation
```python
class CorrelationAnalyzer:
    def __init__(self):
        self.correlation_methods = {
            'pearson': self.pearson_correlation,
            'spearman': self.spearman_correlation,
            'kendall': self.kendall_correlation
        }

    def analyze(self, data, method='pearson'):
        """Analyze correlations between variables"""
        correlations = {}
        variables = ['aqi', 'concrete_percent', 'barren_percent',
                    'water_percent', 'vehicles_per_capita']

        for var1 in variables:
            for var2 in variables:
                if var1 != var2:
                    corr = self.correlation_methods[method](
                        data[var1], data[var2]
                    )
                    correlations[f"{var1}_vs_{var2}"] = corr

        return correlations

    def pearson_correlation(self, x, y):
        """Calculate Pearson correlation coefficient"""
        from scipy.stats import pearsonr
        return pearsonr(x, y)
```

#### Spatial Correlation
```python
class SpatialCorrelation:
    def morans_i(self, data, weights):
        """Calculate Moran's I for spatial autocorrelation"""
        from esda.moran import Moran
        moran = Moran(data, weights)
        return {
            'I': moran.I,
            'p_value': moran.p_norm,
            'z_score': moran.z_norm
        }

    def geary_c(self, data, weights):
        """Calculate Geary's C for spatial autocorrelation"""
        from esda.geary import Geary
        geary = Geary(data, weights)
        return {
            'C': geary.C,
            'p_value': geary.p_norm
        }
```

### Opportunity Identifier

#### Barren Land Detection
```python
class OpportunityIdentifier:
    def __init__(self):
        self.classifier = LandUseClassifier()
        self.scorer = PriorityScorer()

    def identify_opportunities(self, city_land_use, city_boundary):
        """Identify barren land areas for greening"""
        # Find barren land pixels
        barren_mask = city_land_use == 'barren'

        # Group into contiguous areas
        areas = self.group_contiguous(barren_mask)

        # Calculate area size
        areas_with_size = [(area, self.calculate_area(area))
                          for area in areas]

        # Filter by minimum size
        large_areas = [(area, size) for area, size in areas_with_size
                       if size > 1000]  # Minimum 1000 sqm

        # Score each area
        scored = [(area, size, self.scorer.score(area, city_boundary))
                 for area, size in large_areas]

        # Sort by priority
        scored.sort(key=lambda x: x[2], reverse=True)

        return scored

    def group_contiguous(self, mask):
        """Group contiguous pixels into areas"""
        from scipy.ndimage import label
        labeled, num_features = label(mask)
        areas = []
        for i in range(1, num_features + 1):
            areas.append(labeled == i)
        return areas
```

#### Priority Scoring
```python
class PriorityScorer:
    def score(self, area, city_boundary):
        """Score an area for greening priority"""
        score = 0

        # Distance to pollution sources
        distance_score = self.distance_to_pollution(area, city_boundary)
        score += distance_score * 0.3

        # Population density nearby
        population_score = self.population_density(area, city_boundary)
        score += population_score * 0.3

        # Accessibility
        accessibility_score = self.accessibility(area, city_boundary)
        score += accessibility_score * 0.2

        # Size (larger areas get higher score)
        size_score = self.size_score(area)
        score += size_score * 0.2

        return score
```

### Impact Calculator

#### AQI Improvement Estimation
```python
class ImpactCalculator:
    def __init__(self):
        self.coefficients = self.load_coefficients()

    def estimate_aqi_improvement(self, area, proposed_type):
        """Estimate AQI improvement from greening"""
        current_type = self.get_current_type(area)

        if current_type == 'barren' and proposed_type == 'green':
            # Base improvement for barren to green
            base_improvement = 15  # AQI points

            # Adjust based on area size
            area_factor = self.area_factor(area)

            # Adjust based on surrounding AQI
            aqi_factor = self.aqi_factor(area)

            estimated = base_improvement * area_factor * aqi_factor

            return estimated
        else:
            return 0

    def area_factor(self, area):
        """Calculate impact factor based on area size"""
        size = self.calculate_area(area)
        # Logarithmic scaling - diminishing returns
        return np.log(size / 1000 + 1) / np.log(10)
```

#### Cost-Benefit Analysis
```python
class CostBenefitAnalyzer:
    def analyze(self, opportunity):
        """Perform cost-benefit analysis"""
        # Calculate costs
        land_cost = self.estimate_land_cost(opportunity)
        planting_cost = self.estimate_planting_cost(opportunity)
        maintenance_cost = self.estimate_maintenance_cost(opportunity)
        total_cost = land_cost + planting_cost + maintenance_cost

        # Calculate benefits
        aqi_improvement = opportunity['estimated_aqi_improvement']
        health_benefit = self.estimate_health_benefit(aqi_improvement)
        property_value_increase = self.estimate_property_increase(opportunity)
        total_benefit = health_benefit + property_value_increase

        # Calculate ROI
        roi = (total_benefit - total_cost) / total_cost

        return {
            'costs': {
                'land': land_cost,
                'planting': planting_cost,
                'maintenance': maintenance_cost,
                'total': total_cost
            },
            'benefits': {
                'health': health_benefit,
                'property': property_value_increase,
                'total': total_benefit
            },
            'roi': roi
        }
```

## Data Storage

### PostgreSQL with PostGIS

#### Schema Design
```sql
-- Cities table
CREATE TABLE cities (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    country VARCHAR(255) NOT NULL,
    geometry GEOMETRY(Polygon, 4326),
    area_sqkm FLOAT,
    population INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

-- AQI measurements table
CREATE TABLE aqi_measurements (
    id SERIAL PRIMARY KEY,
    city_id INTEGER REFERENCES cities(id),
    pm25 FLOAT,
    pm10 FLOAT,
    o3 FLOAT,
    no2 FLOAT,
    so2 FLOAT,
    co FLOAT,
    timestamp TIMESTAMP NOT NULL,
    geometry GEOMETRY(Point, 4326)
);

-- Land use table
CREATE TABLE land_use (
    id SERIAL PRIMARY KEY,
    city_id INTEGER REFERENCES cities(id),
    concrete_percent FLOAT,
    barren_percent FLOAT,
    water_percent FLOAT,
    green_percent FLOAT,
    date DATE NOT NULL,
    geometry GEOMETRY(Polygon, 4326)
);

-- Vehicle data table
CREATE TABLE vehicle_data (
    id SERIAL PRIMARY KEY,
    city_id INTEGER REFERENCES cities(id),
    total_vehicles INTEGER,
    vehicles_per_capita FLOAT,
    electric_percent FLOAT,
    diesel_percent FLOAT,
    year INTEGER NOT NULL
);

-- Opportunities table
CREATE TABLE opportunities (
    id SERIAL PRIMARY KEY,
    city_id INTEGER REFERENCES cities(id),
    geometry GEOMETRY(Polygon, 4326),
    current_type VARCHAR(50),
    proposed_type VARCHAR(50),
    area_sqkm FLOAT,
    priority_score FLOAT,
    estimated_aqi_improvement FLOAT,
    estimated_cost FLOAT,
    feasibility FLOAT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create spatial indexes
CREATE INDEX idx_cities_geometry ON cities USING GIST(geometry);
CREATE INDEX idx_aqi_geometry ON aqi_measurements USING GIST(geometry);
CREATE INDEX idx_land_use_geometry ON land_use USING GIST(geometry);
CREATE INDEX idx_opportunities_geometry ON opportunities USING GIST(geometry);
```

## API Design

### REST API Endpoints

#### Cities
```
GET /api/cities
GET /api/cities/:id
GET /api/cities/:id/aqi
GET /api/cities/:id/land-use
GET /api/cities/:id/vehicles
GET /api/cities/:id/opportunities
```

#### Analysis
```
GET /api/analysis/correlations/:city_id
GET /api/analysis/trends/:city_id
POST /api/analysis/custom
```

#### Map Data
```
GET /api/map/cities
GET /api/map/aqi/:city_id
GET /api/map/land-use/:city_id
GET /api/map/opportunities/:city_id
```

### API Implementation
```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"]
)

@app.get("/api/cities")
async def get_cities():
    """Get all cities"""
    query = "SELECT * FROM cities"
    cities = db.execute(query)
    return cities

@app.get("/api/cities/{city_id}/aqi")
async def get_city_aqi(city_id: int):
    """Get AQI data for a city"""
    query = """
        SELECT * FROM aqi_measurements
        WHERE city_id = $1
        ORDER BY timestamp DESC
        LIMIT 30
    """
    aqi = db.execute(query, city_id)
    return aqi

@app.get("/api/cities/{city_id}/opportunities")
async def get_opportunities(city_id: int, min_priority: float = 0.5):
    """Get greening opportunities for a city"""
    query = """
        SELECT * FROM opportunities
        WHERE city_id = $1 AND priority_score >= $2
        ORDER BY priority_score DESC
    """
    opportunities = db.execute(query, city_id, min_priority)
    return opportunities
```

## Frontend Architecture

### React Application

#### Component Structure
```
src/
├── components/
│   ├── Map/
│   │   ├── CityMap.jsx
│   │   ├── AQILayer.jsx
│   │   ├── LandUseLayer.jsx
│   │   └── OpportunityLayer.jsx
│   ├── Dashboard/
│   │   ├── CitySelector.jsx
│   │   ├── AQIChart.jsx
│   │   ├── LandUseChart.jsx
│   │   └── OpportunityList.jsx
│   └── Layout/
│       ├── Header.jsx
│       ├── Sidebar.jsx
│       └── Footer.jsx
├── services/
│   ├── api.js
│   └── dataProcessor.js
├── utils/
│   ├── mapUtils.js
│   └── chartUtils.js
└── App.jsx
```

#### Map Component
```jsx
import React, { useEffect, useState } from 'react';
import Map from 'react-map-gl';

const CityMap = ({ cityId }) => {
  const [viewState, setViewState] = useState({
    longitude: -74.0060,
    latitude: 40.7128,
    zoom: 10
  });

  const [layers, setLayers] = useState([]);

  useEffect(() => {
    // Load map layers
    loadLayers(cityId).then(setLayers);
  }, [cityId]);

  return (
    <Map
      {...viewState}
      onMove={evt => setViewState(evt.viewState)}
      style={{ width: '100%', height: '100%' }}
      mapStyle="mapbox://styles/mapbox/light-v10"
    >
      {layers.map(layer => (
        <Source key={layer.id} type="geojson" data={layer.data}>
          <Layer {...layer.config} />
        </Source>
      ))}
    </Map>
  );
};
```

## Performance Optimization

### Caching Strategy
```python
from functools import lru_cache
import redis

class CacheManager:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379)

    @lru_cache(maxsize=1000)
    def get_city_data(self, city_id):
        """Get city data with LRU cache"""
        return self.fetch_from_db(city_id)

    def cache_aqi_data(self, city_id, data, ttl=3600):
        """Cache AQI data with TTL"""
        key = f"aqi:{city_id}"
        self.redis.setex(key, ttl, json.dumps(data))

    def get_cached_aqi(self, city_id):
        """Get cached AQI data"""
        key = f"aqi:{city_id}"
        data = self.redis.get(key)
        return json.loads(data) if data else None
```

### Batch Processing
```python
class BatchProcessor:
    def process_cities_batch(self, city_ids):
        """Process multiple cities in batch"""
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = [
                executor.submit(self.process_city, city_id)
                for city_id in city_ids
            ]
            results = [f.result() for f in futures]
        return results
```

## Monitoring & Logging

### Logging Configuration
```python
import logging
from logging.handlers import RotatingFileHandler

def setup_logging():
    logger = logging.getLogger('eco_city_mapper')
    logger.setLevel(logging.INFO)

    handler = RotatingFileHandler(
        'app.log',
        maxBytes=10*1024*1024,
        backupCount=5
    )
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    return logger
```

### Metrics Collection
```python
from prometheus_client import Counter, Histogram, Gauge

# Define metrics
api_requests = Counter('api_requests_total', 'Total API requests')
api_latency = Histogram('api_latency_seconds', 'API request latency')
cities_processed = Gauge('cities_processed', 'Number of cities processed')

def track_api_request(endpoint):
    """Track API request metrics"""
    api_requests.labels(endpoint=endpoint).inc()

def track_api_latency(endpoint, duration):
    """Track API latency"""
    api_latency.labels(endpoint=endpoint).observe(duration)
```

## Deployment

### Docker Configuration
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eco-city-mapper
spec:
  replicas: 3
  selector:
    matchLabels:
      app: eco-city-mapper
  template:
    metadata:
      labels:
        app: eco-city-mapper
    spec:
      containers:
      - name: api
        image: eco-city-mapper:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

## Conclusion

The Eco City Mapper architecture provides a comprehensive framework for analyzing the relationship between land use, vehicle density, and air quality. By integrating multiple data sources and applying advanced analysis techniques, it enables data-driven urban planning decisions for environmental improvement.

Success requires collaboration between urban planners, environmental scientists, data engineers, and GIS specialists to ensure scientific validity, technical feasibility, and practical applicability.
