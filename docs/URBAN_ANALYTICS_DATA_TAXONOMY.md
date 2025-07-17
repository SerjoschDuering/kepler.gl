# Urban Analytics Platform - Data Structure & Taxonomy Mapping

## Overview

This document defines the data structure and taxonomy mapping for the Urban Analytics Platform, establishing how Speckle BIM data should be organized, accessed, and visualized. The taxonomy is designed to be compatible with kepler.gl's data processing patterns while supporting rich urban analytics workflows.

## Core Data Taxonomy

### 1. Geometry Types Classification

```typescript
enum GeometryType {
  // Primary Analysis Geometries (carry analysis results)
  BUILDING = 'building',
  TREE = 'tree',
  ROAD = 'road',
  URBAN_FURNITURE = 'urban_furniture',

  // Reference Geometries (contextual elements)
  POI = 'poi',
  INFRASTRUCTURE = 'infrastructure',
  BOUNDARY = 'boundary',

  // Annotation Layers (interactive elements)
  ANNOTATION_POINT = 'annotation_point',
  ANNOTATION_AREA = 'annotation_area',

  // Regional Summaries (aggregated data)
  GRID_CELL = 'grid_cell',
  DISTRICT = 'district'
}
```

### 2. Analysis Categories

```typescript
enum AnalysisCategory {
  // Environmental Analysis
  MICROCLIMATE = 'microclimate',
  SOLAR_ACCESS = 'solar_access',
  WIND_PATTERNS = 'wind_patterns',
  NOISE_LEVELS = 'noise_levels',
  AIR_QUALITY = 'air_quality',

  // Biodiversity Analysis
  HABITAT_QUALITY = 'habitat_quality',
  SPECIES_DIVERSITY = 'species_diversity',
  ECOLOGICAL_CONNECTIVITY = 'ecological_connectivity',
  GREEN_COVERAGE = 'green_coverage',

  // Accessibility Analysis
  WALKABILITY = 'walkability',
  PUBLIC_TRANSPORT = 'public_transport',
  CYCLING_ACCESS = 'cycling_access',
  BARRIER_FREE_ACCESS = 'barrier_free_access',

  // Social & Economic
  DEMOGRAPHICS = 'demographics',
  ECONOMIC_ACTIVITY = 'economic_activity',
  SOCIAL_COHESION = 'social_cohesion',

  // Infrastructure
  UTILITIES = 'utilities',
  DIGITAL_CONNECTIVITY = 'digital_connectivity',
  EMERGENCY_ACCESS = 'emergency_access'
}
```

### 3. Standard KPI Naming Convention

```typescript
interface KPIField {
  category: AnalysisCategory;
  metric: string;
  unit?: string;
  analysisType: 'quantitative' | 'qualitative' | 'categorical';
  // Extended attributes for better UX and AI understanding
  shortName?: string;                    // e.g., "Temp" for "Temperature"
  description: string;                   // Full description for tooltips/AI
  descriptionShort: string;              // Brief description for UI
  defaultAggregation: AggregationMethod; // Default aggregation method
  interpretationHighLow: {               // How to interpret high/low values
    high: 'good' | 'bad' | 'neutral';
    low: 'good' | 'bad' | 'neutral';
    explanation: string;
  };
  aiContext?: string;                    // Additional context for LLM understanding
}

// Standard naming pattern: analysis_{category}_{metric}_{unit?}
const KPI_NAMING_PATTERNS = {
  // Microclimate
  'analysis_microclimate_temperature_c': {
    category: AnalysisCategory.MICROCLIMATE,
    metric: 'temperature',
    unit: 'celsius',
    analysisType: 'quantitative'
  },
  'analysis_microclimate_humidity_percent': {
    category: AnalysisCategory.MICROCLIMATE,
    metric: 'humidity',
    unit: 'percent',
    analysisType: 'quantitative'
  },

  // Solar Access
  'analysis_solar_annual_hours': {
    category: AnalysisCategory.SOLAR_ACCESS,
    metric: 'annual_hours',
    unit: 'hours',
    analysisType: 'quantitative'
  },
  'analysis_solar_winter_access_percent': {
    category: AnalysisCategory.SOLAR_ACCESS,
    metric: 'winter_access',
    unit: 'percent',
    analysisType: 'quantitative'
  },

  // Biodiversity
  'analysis_biodiversity_species_count': {
    category: AnalysisCategory.SPECIES_DIVERSITY,
    metric: 'species_count',
    analysisType: 'quantitative'
  },
  'analysis_biodiversity_habitat_quality_score': {
    category: AnalysisCategory.HABITAT_QUALITY,
    metric: 'quality_score',
    analysisType: 'quantitative'
  },

  // Accessibility
  'analysis_accessibility_walk_score': {
    category: AnalysisCategory.WALKABILITY,
    metric: 'walk_score',
    analysisType: 'quantitative'
  },
  'analysis_accessibility_transit_minutes': {
    category: AnalysisCategory.PUBLIC_TRANSPORT,
    metric: 'transit_time',
    unit: 'minutes',
    analysisType: 'quantitative'
  }
};
```

## Speckle Object Structure Mapping

### 1. BIM Parameter Mapping

**Note**: The BIM parameter structure below provides future extensibility. For the initial implementation, we'll use a simplified Speckle data structure that you'll provide, but the framework supports full BIM parameter access when needed.

```typescript
interface SpeckleParameterStructure {
  // Revit Parameters
  parameters?: {
    [parameterName: string]: {
      displayValue: string | number;
      units?: string;
      type?: string;
    }
  };

  // Properties (generic key-value pairs)
  properties?: {
    [key: string]: any;
  };

  // IFC Property Sets
  psets?: {
    [psetName: string]: {
      [propertyName: string]: any;
    }
  };

  // Quantity Sets
  qsets?: {
    [qsetName: string]: {
      [quantityName: string]: {
        value: number;
        unit: string;
      }
    }
  };
}

// Property path mapping for different BIM platforms
const BIM_PROPERTY_PATHS = {
  // Revit parameter access
  revit: {
    'Area': 'parameters.Area.displayValue',
    'Volume': 'parameters.Volume.displayValue',
    'Height': 'parameters.Height.displayValue',
    'Material': 'parameters.Material.displayValue',
    'Type Comments': 'parameters.Type Comments.displayValue'
  },

  // IFC property access
  ifc: {
    'NominalArea': 'psets.Pset_SpaceCommon.NominalArea',
    'GrossFloorArea': 'psets.Pset_BuildingCommon.GrossFloorArea',
    'FireRating': 'psets.Pset_WallCommon.FireRating'
  },

  // Generic properties
  generic: {
    'category': 'properties.category',
    'level': 'properties.level',
    'phase': 'properties.phase'
  }
};
```

### 2. Geometry Data Structure

```typescript
interface GeometryDataStructure {
  // Basic object identification
  id: string;
  speckle_type: string;

  // Geometry representation
  geometry?: {
    type: 'Mesh' | 'Brep' | 'Curve' | 'Point';
    vertices?: number[];
    faces?: number[];
    bbox?: [number, number, number, number, number, number]; // [minX, minY, minZ, maxX, maxY, maxZ]
  };

  // Spatial properties (auto-calculated)
  centroid?: [number, number, number];
  area?: number;
  volume?: number;

  // Analysis results (our taxonomy)
  analysis_results?: {
    [kpiName: string]: number | string | boolean;
  };

  // BIM parameters and properties
  parameters?: SpeckleParameterStructure['parameters'];
  properties?: SpeckleParameterStructure['properties'];

  // Hierarchical relationships
  parent?: string; // Parent object ID
  children?: string[]; // Child object IDs

  // Version control
  commitId: string;
  branchName: string;
  versionDate: Date;
}
```

### 3. Field Detection and Mapping

```typescript
class UrbanAnalyticsFieldMapper {
  private static readonly FIELD_TYPE_PATTERNS = {
    // Numeric fields (for quantitative analysis)
    numeric: [
      /^analysis_.*_(temperature|humidity|score|count|hours|minutes|percent|area|volume)$/,
      /^(area|volume|height|width|length)$/i,
      /parameters\..*\.(area|volume|height)/i
    ],

    // Categorical fields (for filtering and grouping)
    categorical: [
      /^(type|category|level|phase|material|status)$/i,
      /^analysis_.*_(quality|rating|class)$/,
      /speckle_type$/
    ],

    // Temporal fields
    temporal: [
      /^(date|time|timestamp|created|modified)$/i,
      /^analysis_.*_(date|time)$/
    ],

    // Spatial fields
    spatial: [
      /^(geometry|centroid|bbox|coordinates)$/,
      /^(latitude|longitude|x|y|z)$/i
    ]
  };

  static detectFieldType(fieldName: string, sampleValues: any[]): string {
    // Check against KPI naming patterns first
    if (KPI_NAMING_PATTERNS[fieldName]) {
      return KPI_NAMING_PATTERNS[fieldName].analysisType;
    }

    // Pattern-based detection
    for (const [type, patterns] of Object.entries(this.FIELD_TYPE_PATTERNS)) {
      if (patterns.some(pattern => pattern.test(fieldName))) {
        return type;
      }
    }

    // Value-based analysis
    return this.analyzeValueTypes(sampleValues);
  }

  static extractFieldsFromSpeckleObjects(objects: SpeckleObject[]): FieldDefinition[] {
    const fieldMap = new Map<string, {
      type: string;
      frequency: number;
      sampleValues: any[];
    }>();

    // Sample-based field extraction for performance
    const sampleSize = Math.min(1000, objects.length);
    const sample = objects.slice(0, sampleSize);

    sample.forEach(obj => {
      this.traverseObject(obj, '', (path, value) => {
        if (!fieldMap.has(path)) {
          fieldMap.set(path, {
            type: 'unknown',
            frequency: 0,
            sampleValues: []
          });
        }

        const field = fieldMap.get(path)!;
        field.frequency++;
        if (field.sampleValues.length < 100) {
          field.sampleValues.push(value);
        }
      });
    });

    // Generate field definitions
    return Array.from(fieldMap.entries()).map(([path, data]) => ({
      name: this.pathToFieldName(path),
      path,
      type: this.detectFieldType(path, data.sampleValues),
      frequency: data.frequency / sampleSize,
      category: this.categorizeField(path)
    }));
  }

  private static traverseObject(
    obj: any,
    currentPath: string,
    callback: (path: string, value: any) => void,
    maxDepth: number = 3
  ): void {
    if (maxDepth <= 0 || obj == null || typeof obj !== 'object') {
      return;
    }

    for (const [key, value] of Object.entries(obj)) {
      const path = currentPath ? `${currentPath}.${key}` : key;

      if (value != null && typeof value !== 'object') {
        callback(path, value);
      } else if (typeof value === 'object' && !Array.isArray(value)) {
        this.traverseObject(value, path, callback, maxDepth - 1);
      }
    }
  }
  private static categorizeField(path: string, configMapping?: FieldCategoryConfig): AnalysisCategory | 'metadata' | 'geometry' {
    // Use configuration-based field categorization when available
    if (configMapping?.fieldCategories[path]) {
      return configMapping.fieldCategories[path];
    }
    // Analysis fields
    if (path.startsWith('analysis_microclimate')) return AnalysisCategory.MICROCLIMATE;
    if (path.startsWith('analysis_solar')) return AnalysisCategory.SOLAR_ACCESS;
    if (path.startsWith('analysis_biodiversity')) return AnalysisCategory.SPECIES_DIVERSITY;
    if (path.startsWith('analysis_accessibility')) return AnalysisCategory.WALKABILITY;

    // Geometry fields
    if (path.startsWith('geometry') || path.includes('centroid') || path.includes('bbox')) {
      return 'geometry';
    }

    // Default to metadata
    return 'metadata';
  }
}
```

## Configuration Management

### 1. Field Category Configuration

```typescript
interface FieldCategoryConfig {
  version: string;
  fieldCategories: Record<string, AnalysisCategory | 'metadata' | 'geometry'>;
  kpiDefinitions: Record<string, KPIFieldDefinition>;
  chartCategories: Record<string, CategoryDefinition>;
}

interface CategoryDefinition {
  name: string;
  ranges: Array<{
    label: string;
    min: number;
    max: number;
    color?: string;
  }>;
}

// Example configuration
const urbanAnalyticsConfig: FieldCategoryConfig = {
  version: "1.0",
  fieldCategories: {
    "analysis_microclimate_temperature_c": AnalysisCategory.MICROCLIMATE,
    "analysis_solar_annual_hours": AnalysisCategory.SOLAR_ACCESS,
    "building_type": "metadata",
    "geometry.centroid": "geometry"
  },
  kpiDefinitions: {
    "analysis_microclimate_temperature_c": {
      shortName: "Temp",
      description: "Ambient temperature measured at object location",
      descriptionShort: "Temperature (°C)",
      defaultAggregation: "average",
      interpretationHighLow: {
        high: "bad",
        low: "good",
        explanation: "Lower temperatures indicate better thermal comfort"
      }
    }
  },
  chartCategories: {
    "comfort_level": {
      name: "Comfort Level",
      ranges: [
        { label: "Comfortable", min: 70, max: 100, color: "#22c55e" },
        { label: "Moderate", min: 40, max: 70, color: "#f59e0b" },
        { label: "Uncomfortable", min: 0, max: 40, color: "#ef4444" }
      ]
    }
  }
};
```

## Scorecard Configuration Schema

### 1. JSON Configuration Structure

```typescript
interface ScorecardConfiguration {
  version: string;
  scorecards: {
    [scorecardId: string]: {
      title: string;
      description: string;
      category: AnalysisCategory;
      datasets: string[];

      charts: {
        [chartId: string]: {
          type: 'histogram' | 'pie' | 'line' | 'bar' | 'scatter';
          title: string;
          field: string;
          aggregation?: 'sum' | 'average' | 'count' | 'max' | 'min';
          binCount?: number;
          colorScheme?: string;
          clickAction: {
            type: 'filter' | 'highlight' | 'select';
            field?: string;
          };
        }
      };

      kpis: {
        [kpiId: string]: {
          title: string;
          field: string;
          aggregation: 'sum' | 'average' | 'count' | 'max' | 'min';
          format: string; // e.g., ".2f", ".0%", "$,.0f"
          threshold?: {
            warning: number;
            critical: number;
          };
          trend?: {
            enabled: boolean;
            period: 'month' | 'quarter' | 'year';
          };
        }
      };

      filters: {
        [filterId: string]: {
          type: 'range' | 'multiSelect' | 'timeRange' | 'category';
          field: string;
          defaultValue?: any;
          options?: string[];
        }
      };

      layout: {
        columns: number;
        kpiPosition: 'top' | 'left' | 'right';
        chartOrder: string[];
      };
    }
  };
}
```

### 2. Example Configuration

```json
{
  "version": "1.0",
  "scorecards": {
    "microclimate_overview": {
      "title": "Microclimate Analysis",
      "description": "Temperature, humidity, and comfort analysis",
      "category": "microclimate",
      // Yes! datasets = spatial entity groups. Each dataset contains:
      // - Rows: Individual geometric objects (building, tree, etc.)
      // - Columns: Attributes/KPIs (temperature, area, type, etc.)
      "datasets": ["buildings", "outdoor_spaces"],

      "charts": {
        "temperature_distribution": {
          "type": "histogram",
          "title": "Temperature Distribution",
          "field": "analysis_microclimate_temperature_c",
          "binCount": 20,
          "colorScheme": "thermal",
          "clickAction": {
            // Non-interactive for initial implementation
            "type": "none",
            "field": "analysis_microclimate_temperature_c"
          }
        },
        "comfort_by_type": {
          "type": "pie",
          "title": "Comfort by Building Type",
          "field": "building_type", // Categorical field for pie charts
          "aggregation": "count",
          // For numeric data in pie charts, define categories:
          "categories": {
            "Comfortable": {"field": "comfort_score", "range": [70, 100]},
            "Moderate": {"field": "comfort_score", "range": [40, 70]},
            "Uncomfortable": {"field": "comfort_score", "range": [0, 40]}
          },
          "clickAction": {
            "type": "highlight"
          }
        }
      },

      "kpis": {
        "avg_temperature": {
          "title": "Average Temperature",
          "field": "analysis_microclimate_temperature_c",
          "aggregation": "average",
          "format": ".1f°C",
          "threshold": {
            "warning": 28,
            "critical": 35
          }
        },
        "comfort_zones": {
          "title": "Comfort Zones",
          "field": "analysis_microclimate_comfort_score",
          "aggregation": "count",
          "format": ".0f zones"
        }
      },

      "filters": {
        // Only these filters are active for this scorecard, others are disabled
        "building_type": {
          "type": "multiSelect", // Determines filter UI component and function
          "field": "building_type",
          "options": ["Residential", "Commercial", "Mixed Use"]
        },
        "temperature_range": {
          "type": "range",
          "field": "analysis_microclimate_temperature_c",
          "defaultValue": [20, 30]
        }
      },

      "layout": {
        "columns": 2,
        "kpiPosition": "top",
        "chartOrder": ["temperature_distribution", "comfort_by_type"]
      }
    }
  }
}
```

## Data Processing Pipeline

### 1. Speckle to Kepler.gl Transformation

```typescript
interface DataProcessingPipeline {
  // Step 1: Extract and normalize Speckle data
  extractSpeckleData(commitUrl: string): Promise<SpeckleObject[]>;

  // Step 2: Apply field mapping and taxonomy
  mapFieldsToTaxonomy(objects: SpeckleObject[]): FieldDefinition[];

  // Step 3: Create optimized data container
  createDataContainer(
    objects: SpeckleObject[],
    fields: FieldDefinition[]
  ): SpeckleDataContainer;

  // Step 4: Generate layers based on geometry types
  createLayers(
    dataContainer: SpeckleDataContainer,
    fields: FieldDefinition[]
  ): Layer[];

  // Step 5: Configure default filters based on analysis categories
  createDefaultFilters(fields: FieldDefinition[]): Filter[];
}

class UrbanAnalyticsDataProcessor implements DataProcessingPipeline {
  async extractSpeckleData(commitUrl: string): Promise<SpeckleObject[]> {
    const speckleClient = new SpeckleApi();
    const commitData = await speckleClient.getCommit(commitUrl);
    return await speckleClient.getObjects(commitData.referencedObject);
  }

  mapFieldsToTaxonomy(objects: SpeckleObject[]): FieldDefinition[] {
    const detectedFields = UrbanAnalyticsFieldMapper.extractFieldsFromSpeckleObjects(objects);

    // Enhance with taxonomy information
    return detectedFields.map(field => ({
      ...field,
      category: UrbanAnalyticsFieldMapper.categorizeField(field.path),
      isKPI: field.name.startsWith('analysis_'),
      visualChannelCompatible: this.checkVisualChannelCompatibility(field)
    }));
  }

  createDataContainer(
    objects: SpeckleObject[],
    fields: FieldDefinition[]
  ): SpeckleDataContainer {
    return new SpeckleDataContainer({
      objects,
      fields,
      commitInfo: { /* commit metadata */ }
    });
  }

  createLayers(
    dataContainer: SpeckleDataContainer,
    fields: FieldDefinition[]
  ): Layer[] {
    const layers = [];

    // Group objects by geometry type
    const geometryGroups = this.groupByGeometryType(dataContainer);

    Object.entries(geometryGroups).forEach(([geometryType, indices]) => {
      const layer = new SpeckleBIMLayer({
        id: `${geometryType}_layer`,
        type: 'speckle-bim',
        config: {
          dataId: dataContainer.id,
          geometryType,
          visConfig: this.getDefaultVisConfig(geometryType),
          columns: this.getColumnMapping(fields, geometryType)
        }
      });

      layers.push(layer);
    });

    return layers;
  }
}
```

This data taxonomy provides a comprehensive framework for organizing and processing urban analytics data, ensuring consistency across different BIM platforms while enabling rich visualization and analysis capabilities.
