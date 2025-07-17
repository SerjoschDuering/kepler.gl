# Urban Analytics Platform - Architectural Insights from Kepler.gl

## Executive Summary

This document provides architectural insights extracted from kepler.gl analysis for building the Urban Analytics Presentation Platform. The platform will transform complex simulation data (microclimate, biodiversity, accessibility) into engaging client experiences through 3D visualizations and AI-powered explanations.

## Core Architecture Recommendations

### 1. State Management Architecture (Zustand Migration from Redux)

**Recommended Structure:**
```typescript
interface UrbanAnalyticsStore {
  // Core visualization state (equivalent to kepler.gl's visState)
  visualization: {
    layers: Layer[];
    filters: Filter[];
    datasets: Record<string, Dataset>;
    layerData: Record<string, LayerData>;
    interactionConfig: InteractionConfig;
    selectedObjects: string[];
  };

  // Map and viewport state
  map: {
    viewport: Viewport;
    mapStyle: MapStyle;
    bounds: Bounds;
    annotations: Annotation[];
  };

  // UI state management with stable interface for canvas/map/visualization interactions
  ui: {
    scorecards: ScorecardState[];
    sidePanel: SidePanelState;
    modals: ModalState;
    notifications: Notification[];
    // Stable interface for UI components affecting visualization
    viewState: {
      cameraPosition: CameraPosition;
      selectedPanels: string[];
      layoutMode: 'comparison' | 'single' | 'overview';
    };
    // AI integration as part of UI state
    ai: {
      annotations: AIAnnotation[];
      insights: RegionalInsight[];
      chatHistory: ChatMessage[];
      isGenerating: boolean;
    };
  };

  // Speckle data integration
  speckle: {
    streams: SpeckleStream[];
    currentVersions: Record<string, string>;
    geometryObjects: GeometryObject[];
    loadingStates: Record<string, LoadingState>;
  };
}
```

**Key Patterns from Kepler.gl:**
- **Immutable Updates**: Use Immer for complex nested state updates
- **Cross-slice Coordination**: Implement composer functions for actions affecting multiple slices
- **Subscription-based Side Effects**: Replace Redux middleware with Zustand subscriptions
- **Memoized Selectors**: Use shallow equality checks for expensive computations

### 2. Data Container Architecture for Speckle Integration

**Architecture Overview:**
Kepler.gl uses a DataContainer abstraction that separates data storage from data access. This allows different data formats (CSV, Arrow, Parquet) to be handled uniformly. For Speckle integration, we implement a custom container that handles BIM object hierarchies and nested properties efficiently.

**Core Concept:**
- **Rows**: Each Speckle object (building, tree, etc.) is a row
- **Columns**: Each property/KPI (temperature, area, type) is a column
- **Nested Access**: Properties like `parameters.Area.displayValue` are flattened to columns
- **Caching**: Frequently accessed properties are cached for performance

**Recommended Implementation:**
```typescript
class SpeckleDataContainer implements DataContainerInterface {
  private _speckleObjects: SpeckleObject[];
  private _fieldMap: Map<string, string>; // field name to property path
  private _propertyCache: LRU<string, any>; // property path cache
  private _geometryCache: LRU<string, any>; // geometry cache

  // Efficient nested property access with caching
  valueAt(rowIndex: number, columnIndex: number): any {
    const field = this._fields[columnIndex];
    const propertyPath = this._fieldMap.get(field.name);
    const cacheKey = `${rowIndex}-${propertyPath}`;

    if (this._propertyCache.has(cacheKey)) {
      return this._propertyCache.get(cacheKey);
    }

    const value = this._extractNestedProperty(
      this._speckleObjects[rowIndex],
      propertyPath
    );
    this._propertyCache.set(cacheKey, value);
    return value;
  }
}
```

**Benefits:**
- **Memory Efficiency**: LRU caching for frequently accessed properties
- **Performance**: Direct property access without data transformation
- **BIM-Aware**: Handles Revit parameters, IFC properties, and nested structures
- **Version Control**: Support for switching between Speckle commits

### 3. Layer System for Rich Geometry Visualization

**Simplified Approach for Initial Implementation:**
For the first version, we'll use simplified geometry data rather than full BIM/IFC models:
- **Point/Polygon Geometries**: Speckle objects converted to GeoJSON-like structures
- **Attribute Mapping**: KPI values mapped to visual properties (color, height, opacity)
- **Progressive Enhancement**: Framework ready for future BIM model integration

**Custom Speckle Geometry Layer:**
```typescript
class SpeckleBIMLayer extends BaseLayer {
  get visualChannels(): VisualChannels {
    return {
      color: {
        property: 'color',
        field: 'colorField', // Maps to KPI like 'carbon_footprint'
        scale: 'colorScale',
        accessor: 'getFillColor'
      },
      height: {
        property: 'height',
        field: 'heightField', // Maps to KPI like 'energy_usage'
        scale: 'heightScale',
        accessor: 'getElevation'
      },
      // Custom channels for BIM-specific attributes
      opacity: {
        property: 'opacity',
        field: 'opacityField', // Maps to 'construction_phase'
        scale: 'opacityScale',
        accessor: 'getOpacity'
      }
    };
  }

  // Version switching capability
  switchToVersion(version: string, datasets: Datasets) {
    const dataset = this.getDataset(datasets);
    const versionedData = dataset.getVersionData(version);

    this.updateLayerConfig({ currentVersion: version });
    return this.updateData({...datasets, [dataset.id]: versionedData}, null);
  }
}
```

### 4. Interactive Scorecard System

**JSON-Configurable Architecture:**
```typescript
interface ScorecardConfig {
  id: string;
  title: string;
  datasets: string[];
  charts: ChartConfig[];
  filters: FilterConfig[];
  kpis: KPIConfig[];
  interactions: InteractionConfig[];
}

interface ChartConfig {
  type: 'histogram' | 'lineChart' | 'bar' | 'pie';
  dataId: string;
  field: string;
  aggregation: 'sum' | 'average' | 'count' | 'max' | 'min';
  binCount?: number;
  clickAction: 'filter' | 'highlight' | 'select';
}
```

**Real-time KPI Computation:**

**Kepler.gl Aggregation Functions:**
Yes! Kepler.gl provides optimized aggregation functions from d3-array:
- `sum()`, `mean()`, `max()`, `min()`, `median()` - Fast numeric aggregations
- `mode()`, `countUnique()` - Categorical aggregations
- GPU-accelerated for large datasets when possible
- Efficient change detection to avoid unnecessary recalculations

```typescript
// Leveraging kepler.gl's aggregation patterns
export const computeKPIsFromSelection = (
  selectedObjects: string[],
  datasets: Datasets,
  kpiFields: string[]
): Record<string, KPIValue> => {
  return kpiFields.reduce((kpis, field) => {
    const values = selectedObjects
      .map(id => getKPIValue(datasets, id, field))
      .filter(v => v != null);

    kpis[field] = {
      sum: aggregate(values, AGGREGATION_TYPES.sum),
      avg: aggregate(values, AGGREGATION_TYPES.average),
      min: aggregate(values, AGGREGATION_TYPES.minimum),
      max: aggregate(values, AGGREGATION_TYPES.maximum),
      count: values.length
    };
    return kpis;
  }, {});
};
```

### 5. Performance Optimization Strategy

**For 3-20k Geometry Objects (Preset-Based Approach):**

**Performance Presets:**
- **Small Dataset (<1k objects)**: Standard processing, all features enabled
- **Medium Dataset (1-5k objects)**: Selective GPU filtering, moderate caching
- **Large Dataset (5-20k objects)**: Aggressive caching, chunked processing, simplified interactions
- **Auto-Detection**: System automatically selects optimal preset based on object count
1. **Data Container Selection:**
   - Use `ArrowDataContainer` for large datasets (>5k objects)
   - Use `IndexedDataContainer` for filtered views
   - Implement `SpeckleDataContainer` for BIM-specific optimizations

2. **GPU/CPU Processing Balance:**
   - GPU filtering for up to 4 numeric KPIs simultaneously
   - CPU filtering for complex categorical/string filters
   - Automatic fallback when GPU channels exhausted

3. **Memory Management (Automatic Under-the-Hood Optimizations):**
   - **Automatic**: Shared row objects to reduce GC pressure
   - **Automatic**: LRU caching for frequently accessed properties
   - **Automatic**: Lazy loading of geometry data
   - **Configurable**: Cache sizes based on available memory

4. **Rendering Optimizations:**
   - Viewport-based culling for large models
   - Level-of-detail based on zoom level
   - Aggregation layers for overview visualization

### 6. Component Extensibility Architecture

**Factory Pattern Implementation:**
```typescript
// Custom scorecard widget factory
function ScoreCardWidgetFactory(
  MetricsDisplayFactory,
  ChartComponentFactory,
  FilterIntegrationFactory
) {
  const ScoreCardWidget = ({ metrics, filters, onMetricClick }) => (
    <WidgetContainer>
      <MetricsDisplay metrics={metrics} onClick={onMetricClick} />
      <ChartComponent data={metrics.chartData} />
      <FilterIntegration filters={filters} />
    </WidgetContainer>
  );
  return ScoreCardWidget;
}

// Component injection
const UrbanKeplerGl = injectComponents([
  [CustomPanelsFactory, UrbanAnalyticsPanelFactory],
  [BottomWidgetFactory, UrbanBottomWidgetFactory],
  [SidebarFactory, UrbanSidebarFactory]
]);
```

### 7. AI Integration Architecture

**User Journey for AI Integration:**
1. **Context-Aware Annotations**: User clicks on map → AI explains what they're seeing
2. **Insight Generation**: User selects objects → AI summarizes KPIs and relationships
3. **Comparative Analysis**: User switches between versions → AI highlights key differences
4. **Smart Explanations**: User hovers over KPI → AI provides contextual interpretation

**Stateless Annotation System:**
```typescript
interface AIIntegration {
  // Annotation chat with external API
  generateAnnotationInsight: (
    annotation: Annotation,
    context: AnalysisContext
  ) => Promise<string>;

  // Regional summary insights
  generateRegionalSummary: (
    geometryData: GeometryObject[],
    kpis: KPIData[]
  ) => Promise<RegionalInsight>;

  // Contextual explanations
  explainKPIResults: (
    kpi: string,
    value: number,
    context: SpatialContext
  ) => Promise<string>;
}
```

## Implementation Roadmap

### Phase 1: Core Architecture (Weeks 1-2)
1. Set up Zustand store with state slices
2. Implement Speckle data container
3. Create base layer system with visual channels
4. Build component factory system

### Phase 2: Scorecard System (Weeks 3-4)
1. Implement JSON-configurable scorecards
2. Build real-time KPI computation
3. Create interactive chart components
4. Implement chart-to-3D scene interactions

### Phase 3: Performance & Polish (Weeks 5-6)
1. Optimize data containers for large datasets
2. Implement viewport-based optimizations
3. Add background processing capabilities
4. Performance testing with 3-20k objects

### Phase 4: AI Integration (Weeks 7-8)
1. Implement annotation system
2. Integrate external AI API
3. Build insight generation pipeline
4. Add contextual explanations

## Success Metrics

- **Performance**: Smooth interaction with 3-20k geometry objects
- **Extensibility**: Easy addition of new KPIs and visualization types
- **User Experience**: Intuitive scorecard interactions driving 3D exploration
- **Technical Debt**: Clean separation between UI, data, and visualization logic

## Risk Mitigation

1. **Performance Risk**: Implement progressive loading and chunked processing
2. **Complexity Risk**: Start with simple scorecard types, expand gradually
3. **Integration Risk**: Build adapter patterns for external dependencies
4. **Scalability Risk**: Design for horizontal scaling of data processing

This architectural foundation leverages kepler.gl's proven patterns while adapting them for the specific needs of urban analytics visualization with Speckle data integration.
