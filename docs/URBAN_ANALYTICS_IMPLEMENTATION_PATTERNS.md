# Urban Analytics Platform - Implementation Patterns

## Overview

This document provides concrete implementation patterns and code snippets extracted from kepler.gl analysis, specifically adapted for building the Urban Analytics Presentation Platform. These patterns are production-ready and based on kepler.gl's proven architecture.

## 1. Zustand State Management Patterns

### Core Store Structure

```typescript
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface UrbanAnalyticsStore {
  // Visualization state
  visualization: {
    layers: Layer[];
    filters: Filter[];
    datasets: Record<string, Dataset>;
    layerData: Record<string, LayerData>;
    selectedObjects: string[];
    interactionConfig: InteractionConfig;
  };

  // Map state
  map: {
    viewport: Viewport;
    mapStyle: MapStyle;
    bounds: Bounds;
    // Pre-generated annotations (loaded with Speckle data)
    annotations: PreGeneratedAnnotation[];
    regionSummaries: RegionSummary[];
  };

  // UI state with stable interface for visualization interactions
  ui: {
    scorecards: ScorecardState[];
    activePanels: string[];
    modals: ModalState;
    // Stable interface for UI components affecting visualization
    viewState: {
      cameraPosition: CameraPosition;
      selectedPanels: string[];
      layoutMode: 'comparison' | 'single' | 'overview';
    };
    // AI chat state (real-time interactions)
    ai: {
      activeChatBubbles: ChatBubble[];     // Currently open chat interfaces
      chatHistory: ChatMessage[];         // Conversation history
      isGenerating: boolean;               // LLM response in progress
      uiContext: UIContextSnapshot;       // Current state for AI context
    };
  };

  // Speckle integration
  speckle: {
    streams: SpeckleStream[];
    currentVersions: Record<string, string>;
    loadingStates: Record<string, LoadingState>;
  };

  // Actions
  actions: {
    visualization: VisualizationActions;
    map: MapActions;
    ui: UIActions;
    speckle: SpeckleActions;
  };
}

const useUrbanAnalyticsStore = create<UrbanAnalyticsStore>()(
  subscribeWithSelector(
    immer((set, get) => ({
      // Initial state
      visualization: {
        layers: [],
        filters: [],
        datasets: {},
        layerData: {},
        selectedObjects: [],
        interactionConfig: DEFAULT_INTERACTION_CONFIG
      },

      map: {
        viewport: DEFAULT_VIEWPORT,
        mapStyle: DEFAULT_MAP_STYLE,
        bounds: null,
        annotations: []
      },

      ui: {
        scorecards: [],
        activePanels: ['overview'],
        modals: { isOpen: false, type: null }
      },

      speckle: {
        streams: [],
        currentVersions: {},
        loadingStates: {}
      },

      // Actions implementation
      actions: {
        visualization: {
          updateLayer: (layerId: string, updates: Partial<Layer>) =>
            set((state) => {
              const layer = state.visualization.layers.find(l => l.id === layerId);
              if (layer) {
                Object.assign(layer, updates);
              }
            }),

          addFilter: (filter: Filter) =>
            set((state) => {
              state.visualization.filters.push(filter);
            }),

          updateFilter: (filterId: string, updates: Partial<Filter>) =>
            set((state) => {
              const filter = state.visualization.filters.find(f => f.id === filterId);
              if (filter) {
                Object.assign(filter, updates);
                // Trigger layer data recalculation
                state.visualization.layerData = recalculateLayerData(
                  state.visualization.layers,
                  state.visualization.filters
                );
              }
            }),

          setSelectedObjects: (objectIds: string[]) =>
            set((state) => {
              state.visualization.selectedObjects = objectIds;
            })
        },

        speckle: {
          addSpeckleStream: async (streamUrl: string) => {
            const { speckle } = get();

            set((state) => {
              state.speckle.loadingStates[streamUrl] = 'loading';
            });

            try {
              const speckleData = await fetchSpeckleStream(streamUrl);
              const datasets = processSpeckleToDatasets(speckleData);

              set((state) => {
                state.speckle.streams.push(speckleData);
                state.speckle.currentVersions[speckleData.id] = speckleData.commitId;
                state.speckle.loadingStates[streamUrl] = 'success';

                // Update visualization state
                Object.assign(state.visualization.datasets, datasets);
                state.visualization.layers.push(...createDefaultLayers(datasets));
              });
            } catch (error) {
              set((state) => {
                state.speckle.loadingStates[streamUrl] = 'error';
              });
            }
          }
        }
      }
    }))
  )
);
```

### Cross-slice Coordination Pattern

```typescript
// Subscription-based side effects
useUrbanAnalyticsStore.subscribe(
  (state) => state.visualization.filters,
  (filters, prevFilters) => {
    if (filters !== prevFilters) {
      // Recalculate KPIs when filters change
      const { visualization, ui } = useUrbanAnalyticsStore.getState();
      const updatedKPIs = computeKPIsFromFilters(
        visualization.datasets,
        filters,
        ui.scorecards
      );

      useUrbanAnalyticsStore.setState((state) => ({
        ...state,
        ui: {
          ...state.ui,
          scorecards: state.ui.scorecards.map(scorecard => ({
            ...scorecard,
            kpis: updatedKPIs[scorecard.id] || scorecard.kpis
          }))
        }
      }));
    }
  },
  { equalityFn: shallow }
);
```

## 2. Speckle Data Container Implementation

```typescript
import { LRUCache } from 'lru-cache';

interface SpeckleObject {
  id: string;
  speckle_type: string;
  parameters?: Record<string, any>;
  properties?: Record<string, any>;
  geometry?: any;
  [key: string]: any;
}

interface SpeckleField {
  name: string;
  path: string;
  type: string;
  frequency: number;
}

class SpeckleDataContainer implements DataContainerInterface {
  private _speckleObjects: SpeckleObject[];
  private _fields: SpeckleField[];
  private _fieldMap: Map<string, string>;
  private _propertyCache: LRUCache<string, any>;
  private _geometryCache: LRUCache<string, any>;

  constructor(config: {
    objects: SpeckleObject[];
    fields: SpeckleField[];
    commitInfo: CommitInfo;
  }) {
    this._speckleObjects = config.objects;
    this._fields = config.fields;
    this._fieldMap = new Map(config.fields.map(f => [f.name, f.path]));
    this._propertyCache = new LRUCache({ max: 10000 });
    this._geometryCache = new LRUCache({ max: 1000 });
  }

  numRows(): number {
    return this._speckleObjects.length;
  }

  numColumns(): number {
    return this._fields.length;
  }

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

  private _extractNestedProperty(obj: any, path: string): any {
    const segments = path.split('.');
    let current = obj;

    for (const segment of segments) {
      if (current == null) return null;

      // Handle Speckle parameter access: parameters.{paramName}
      if (segment.startsWith('{') && segment.endsWith('}')) {
        const paramName = segment.slice(1, -1);
        current = current.parameters?.[paramName]?.displayValue ??
                 current.parameters?.[paramName];
      }
      // Handle array indices
      else if (segment.includes('[') && segment.includes(']')) {
        const [prop, indexStr] = segment.split('[');
        const index = parseInt(indexStr.replace(']', ''));
        current = current[prop]?.[index];
      }
      // Standard property access
      else {
        current = current[segment];
      }
    }

    return current;
  }

  // Version switching capability
  switchToVersion(newObjects: SpeckleObject[]): SpeckleDataContainer {
    return new SpeckleDataContainer({
      objects: newObjects,
      fields: this._fields, // Reuse field schema
      commitInfo: { /* new commit info */ }
    });
  }

  // KPI computation for selected objects
  computeKPIs(objectIds: string[], kpiFields: string[]): Record<string, KPIValue> {
    const selectedIndices = objectIds.map(id =>
      this._speckleObjects.findIndex(obj => obj.id === id)
    ).filter(idx => idx !== -1);

    return kpiFields.reduce((kpis, field) => {
      const fieldIndex = this._fields.findIndex(f => f.name === field);
      if (fieldIndex === -1) return kpis;

      const values = selectedIndices
        .map(idx => this.valueAt(idx, fieldIndex))
        .filter(v => v != null && typeof v === 'number');

      kpis[field] = {
        sum: values.reduce((a, b) => a + b, 0),
        avg: values.length ? values.reduce((a, b) => a + b, 0) / values.length : 0,
        min: values.length ? Math.min(...values) : 0,
        max: values.length ? Math.max(...values) : 0,
        count: values.length
      };

      return kpis;
    }, {});
  }
}
```

## 3. Simplified Geometry Layer Implementation

**Note**: This implementation uses simplified geometry data rather than full BIM models for the initial version:

```typescript
import { GeoJsonLayer } from '@deck.gl/layers';
import { BaseLayer } from 'kepler.gl/src/layers';

// Simplified Speckle Geometry Layer (not full BIM)
class SpeckleGeometryLayer extends BaseLayer {
  constructor(props) {
    super(props);
    this.registerVisConfig(speckleBIMVisConfigs);
  }

  get type(): 'speckle-geometry' {
    return 'speckle-geometry';
  }

  get visualChannels(): VisualChannels {
    return {
      color: {
        property: 'color',
        field: 'colorField',
        scale: 'colorScale',
        domain: 'colorDomain',
        range: 'colorRange',
        channelScaleType: CHANNEL_SCALES.color,
        accessor: 'getFillColor',
        nullValue: NO_VALUE_COLOR
      },
      height: {
        property: 'height',
        field: 'heightField',
        scale: 'heightScale',
        domain: 'heightDomain',
        range: 'heightRange',
        channelScaleType: CHANNEL_SCALES.size,
        accessor: 'getElevation',
        nullValue: 0
      },
      strokeColor: {
        property: 'strokeColor',
        field: 'strokeColorField',
        scale: 'strokeColorScale',
        domain: 'strokeColorDomain',
        range: 'strokeColorRange',
        channelScaleType: CHANNEL_SCALES.color,
        accessor: 'getLineColor'
      }
    };
  }

  calculateDataAttribute(keplerTable: KeplerTable, getPosition) {
    const { dataContainer } = keplerTable;
    const data = [];

    for (let i = 0; i < dataContainer.numRows(); i++) {
      const speckleObject = dataContainer.row(i);
      const position = getPosition(speckleObject);

      if (position.every(Number.isFinite)) {
        data.push({
          position,
          index: i,
          speckleId: speckleObject.id,
          speckleType: speckleObject.speckle_type,
          properties: this.extractProperties(speckleObject)
        });
      }
    }

    return data;
  }

  renderLayer(opts) {
    const { data, mapState } = opts;
    const accessors = this.getAttributeAccessors({
      dataContainer: data.dataContainer
    });

    return [
      new GeoJsonLayer({
        ...this.getDefaultDeckLayerProps(opts),
        data: data.data,

        // 3D visualization
        extruded: this.config.visConfig.enable3d,
        getElevation: accessors.getElevation,

        // Visual encoding
        getFillColor: accessors.getFillColor,
        getLineColor: accessors.getLineColor,
        getLineWidth: d => this.config.visConfig.thickness,

        // Interactions
        onHover: this.handleObjectHover.bind(this),
        onClick: this.handleObjectClick.bind(this),

        updateTriggers: {
          getFillColor: this.config.colorTrigger,
          getElevation: this.config.heightTrigger,
          getLineColor: this.config.strokeColorTrigger
        }
      })
    ];
  }

  handleObjectClick(info) {
    if (info.object) {
      // Update store with selected object
      useUrbanAnalyticsStore.getState().actions.visualization.setSelectedObjects([
        info.object.speckleId
      ]);

      return {
        type: 'speckle-object',
        object: info.object,
        tooltip: this.formatTooltip(info.object)
      };
    }
  }

  formatTooltip(speckleObject): string {
    const properties = speckleObject.properties || {};
    return `
      <div class="speckle-tooltip">
        <h4>${speckleObject.speckleType}</h4>
        <div>ID: ${speckleObject.speckleId}</div>
        ${Object.entries(properties).map(([key, value]) =>
          `<div>${key}: ${value}</div>`
        ).join('')}
      </div>
    `;
  }
}
```

## 4. Interactive Scorecard System

```typescript
interface ScorecardConfig {
  id: string;
  title: string;
  datasets: string[];
  charts: ChartConfig[];
  kpis: KPIConfig[];
  filters: FilterConfig[];
  interactions: InteractionConfig[];
}

interface ChartConfig {
  type: 'histogram' | 'lineChart' | 'bar' | 'pie';
  field: string;
  aggregation: string;
  binCount?: number;
  clickAction: 'filter' | 'highlight' | 'select';
}

interface KPIConfig {
  id: string;
  title: string;
  field: string;
  aggregation: string;
  format: string;
  threshold?: {
    warning: number;
    critical: number;
  };
}

// Scorecard React Component
const ScorecardWidget: React.FC<{
  config: ScorecardConfig;
  onChartClick: (chartId: string, value: any) => void;
}> = ({ config, onChartClick }) => {
  const datasets = useUrbanAnalyticsStore(state => state.visualization.datasets);
  const filters = useUrbanAnalyticsStore(state => state.visualization.filters);

  // Real-time KPI computation
  const kpis = useMemo(() => {
    return computeKPIsFromConfig(datasets, filters, config.kpis);
  }, [datasets, filters, config.kpis]);

  const chartData = useMemo(() => {
    return generateChartsFromConfig(datasets, filters, config.charts);
  }, [datasets, filters, config.charts]);

  return (
    <ScorecardContainer>
      <ScorecardHeader>
        <Title>{config.title}</Title>
      </ScorecardHeader>

      <KPIGrid>
        {kpis.map(kpi => (
          <KPICard key={kpi.id} kpi={kpi} />
        ))}
      </KPIGrid>

      <ChartsGrid>
        {chartData.map(chart => (
          <ChartComponent
            key={chart.id}
            chart={chart}
            onClick={(value) => onChartClick(chart.id, value)}
          />
        ))}
      </ChartsGrid>
    </ScorecardContainer>
  );
};

// Pre-made Fast Aggregation Functions from Kepler.gl
import {
  sum, mean, max, min, median,
  mode, countUnique
} from 'd3-array';

const AGGREGATION_FUNCTIONS = {
  sum: (values: number[]) => sum(values),
  average: (values: number[]) => mean(values),
  max: (values: number[]) => max(values),
  min: (values: number[]) => min(values),
  median: (values: number[]) => median(values),
  count: (values: any[]) => values.length,
  countUnique: (values: any[]) => countUnique(values),
  mode: (values: any[]) => mode(values)
};

// KPI computation utility
const computeKPIsFromConfig = (
  datasets: Record<string, Dataset>,
  filters: Filter[],
  kpiConfigs: KPIConfig[]
): KPIValue[] => {
  return kpiConfigs.map(config => {
    const relevantDatasets = config.datasets || Object.keys(datasets);
    let aggregatedValue = 0;
    let count = 0;

    relevantDatasets.forEach(datasetId => {
      const dataset = datasets[datasetId];
      if (!dataset) return;

      const filteredData = applyFiltersToDataset(dataset, filters);
      const fieldValues = extractFieldValues(filteredData, config.field);

      // Use optimized aggregation functions
      const aggregationFn = AGGREGATION_FUNCTIONS[config.aggregation];
      if (aggregationFn) {
        const result = aggregationFn(fieldValues);
        if (config.aggregation === 'average') {
          aggregatedValue += result * fieldValues.length;
          count += fieldValues.length;
        } else if (['sum', 'count'].includes(config.aggregation)) {
          aggregatedValue += result;
        } else {
          aggregatedValue = count === 0 ? result :
                           (config.aggregation === 'max' ?
                            Math.max(aggregatedValue, result) :
                            Math.min(aggregatedValue, result));
        }
      }
    });

    if (config.aggregation === 'average' && count > 0) {
      aggregatedValue = aggregatedValue / count;
    }

    return {
      id: config.id,
      title: config.title,
      value: aggregatedValue,
      format: config.format,
      threshold: config.threshold,
      status: getKPIStatus(aggregatedValue, config.threshold)
    };
  });
};
```

## 5. Component Factory Pattern

```typescript
// Custom component factory for urban analytics
function UrbanAnalyticsPanelFactory() {
  const panels = [
    {
      id: 'overview',
      label: 'City Overview',
      iconComponent: Icons.Dashboard,
      component: OverviewPanelFactory
    },
    {
      id: 'scorecard',
      label: 'Score Cards',
      iconComponent: Icons.BarChart,
      component: ScorecardPanelFactory
    },
    {
      id: 'annotations',
      label: 'Annotations',
      iconComponent: Icons.Pin,
      component: AnnotationPanelFactory
    }
  ];

  const getProps = (sidePanelProps) => ({
    selectedObjects: sidePanelProps.selectedObjects,
    timeRange: sidePanelProps.timeRange,
    comparisonMode: sidePanelProps.comparisonMode
  });

  return { panels, getProps };
}

// Component injection setup
const UrbanKeplerGl = injectComponents([
  [CustomPanelsFactory, UrbanAnalyticsPanelFactory],
  [SidebarFactory, UrbanSidebarFactory],
  [BottomWidgetFactory, UrbanBottomWidgetFactory]
]);
```

## 6. Performance Optimization Patterns

```typescript
// Efficient data processing for large datasets
class PerformanceOptimizedProcessor {
  private static readonly CHUNK_SIZE = 1000;
  private static readonly MAX_SAMPLE_SIZE = 5000;

  static processLargeDataset(
    speckleObjects: SpeckleObject[],
    processingFunction: (chunk: SpeckleObject[]) => any
  ): any[] {
    const results = [];

    for (let i = 0; i < speckleObjects.length; i += this.CHUNK_SIZE) {
      const chunk = speckleObjects.slice(i, i + this.CHUNK_SIZE);
      results.push(processingFunction(chunk));
    }

    return results.flat();
  }

  static sampleDataForAnalysis(
    speckleObjects: SpeckleObject[],
    sampleSize: number = this.MAX_SAMPLE_SIZE
  ): SpeckleObject[] {
    if (speckleObjects.length <= sampleSize) {
      return speckleObjects;
    }

    const step = Math.floor(speckleObjects.length / sampleSize);
    return speckleObjects.filter((_, index) => index % step === 0);
  }

  static createFilteredIndex(
    dataContainer: SpeckleDataContainer,
    filters: Filter[]
  ): Uint8ClampedArray {
    const filteredIndex = new Uint8ClampedArray(dataContainer.numRows());

    for (let i = 0; i < dataContainer.numRows(); i++) {
      filteredIndex[i] = filters.every(filter =>
        this.evaluateFilter(dataContainer, i, filter)
      ) ? 1 : 0;
    }

    return filteredIndex;
  }
}

// Memoized selectors for expensive computations
const useOptimizedSelectors = () => {
  const layers = useUrbanAnalyticsStore(
    useCallback((state) => state.visualization.layers, []),
    shallow
  );

  const filteredLayers = useUrbanAnalyticsStore(
    useCallback((state) => {
      const { layers, filters } = state.visualization;
      return layers.filter(layer =>
        filters.every(filter => evaluateLayerAgainstFilter(layer, filter))
      );
    }, []),
    shallow
  );

  const selectedObjectKPIs = useUrbanAnalyticsStore(
    useCallback((state) => {
      const { selectedObjects, datasets } = state.visualization;
      return computeKPIsForSelection(selectedObjects, datasets);
    }, []),
    shallow
  );

  return { layers, filteredLayers, selectedObjectKPIs };
};
```

## 7. AI Integration Patterns - Speckle-Based Annotations

**Architecture Overview:**
AI annotations and region summaries are stored directly in Speckle geometry objects as JSON data, loaded with the main dataset.

```typescript
// Speckle geometry object with attached annotation data
interface SpeckleAnnotationGeometry {
  id: string;
  speckle_type: 'Point'; // Point geometry for annotations
  geometry: {
    coordinates: [number, number, number]; // 3D position
  };
  // JSON annotation data attached to geometry
  annotationData: {
    summaryText: string;                   // Pre-generated AI description
    analysisCategory: AnalysisCategory;    // Which analysis this relates to
    kpis: Record<string, number>;          // Associated KPI values
    visibility: {
      scorecards: string[];                // Which scorecards show this
      filterConditions: FilterCondition[]; // When to show/hide
    };
  };
}

// Speckle geometry object with region summary data
interface SpeckleRegionGeometry {
  id: string;
  speckle_type: 'Curve' | 'Polygon';     // Boundary geometry for regions
  geometry: {
    boundary: Polygon;                     // Region boundary
  };
  // JSON region summary attached to geometry
  regionData: {
    summaryKPIs: Record<string, number>;   // Aggregated stats for region
    aiDescription: string;                 // Pre-generated AI description
    analysisLayer: string;                 // Which analysis layer
  };
}

// Data extraction from Speckle objects
class SpeckleAIDataExtractor {
  extractAnnotations(speckleObjects: SpeckleObject[]): PreGeneratedAnnotation[] {
    return speckleObjects
      .filter(obj => obj.speckle_type === 'Point' && obj.annotationData)
      .map(obj => ({
        id: obj.id,
        position: obj.geometry.coordinates,
        summaryText: obj.annotationData.summaryText,
        analysisCategory: obj.annotationData.analysisCategory,
        kpis: obj.annotationData.kpis,
        visibility: obj.annotationData.visibility
      }));
  }

  extractRegionSummaries(speckleObjects: SpeckleObject[]): RegionSummary[] {
    return speckleObjects
      .filter(obj => 
        ['Curve', 'Polygon'].includes(obj.speckle_type) && 
        obj.regionData
      )
      .map(obj => ({
        regionId: obj.id,
        bbox: this.calculateBoundingBox(obj.geometry.boundary),
        summaryKPIs: obj.regionData.summaryKPIs,
        aiDescription: obj.regionData.aiDescription,
        analysisLayer: obj.regionData.analysisLayer,
        geometry: obj.geometry.boundary
      }));
  }

  private calculateBoundingBox(boundary: Polygon): BoundingBox {
    // Calculate bbox from polygon coordinates
    const coords = boundary.coordinates[0];
    return {
      minX: Math.min(...coords.map(c => c[0])),
      minY: Math.min(...coords.map(c => c[1])),
      maxX: Math.max(...coords.map(c => c[0])),
      maxY: Math.max(...coords.map(c => c[1]))
    };
  }
}

// Annotation management with Speckle integration
class SpeckleAnnotationManager {
  private annotations: PreGeneratedAnnotation[];
  private regionSummaries: RegionSummary[];

  constructor(speckleObjects: SpeckleObject[]) {
    const extractor = new SpeckleAIDataExtractor();
    this.annotations = extractor.extractAnnotations(speckleObjects);
    this.regionSummaries = extractor.extractRegionSummaries(speckleObjects);
  }

  updateVisibilityByScorecard(scorecardId: string, activeFilters: Filter[]) {
    this.annotations.forEach(annotation => {
      annotation.visible = 
        annotation.visibility.scorecards.includes(scorecardId) &&
        annotation.visibility.filterConditions.every(condition =>
          this.evaluateFilterCondition(condition, activeFilters)
        );
    });
  }

  getRegionSummaryAtPosition(position: [number, number, number]): RegionSummary | null {
    return this.regionSummaries.find(region =>
      this.isPointInRegion(position, region.geometry)
    );
  }

  getNearbyAnnotations(position: [number, number, number], radius = 100): PreGeneratedAnnotation[] {
    return this.annotations.filter(annotation =>
      this.calculateDistance(position, annotation.position) <= radius
    );
  }
}

// External LLM service with Speckle context
class ExternalLLMService {
  async askSpatialQuestion(
    question: string,
    context: SpatialChatContext
  ): Promise<string> {
    const payload = {
      question,
      context: {
        // Speckle-based data
        regionSummary: context.regionSummary?.aiDescription,
        regionKPIs: context.regionSummary?.summaryKPIs,
        nearbyAnnotations: context.nearbyAnnotations.map(a => a.summaryText),
        
        // Current UI state
        activeScorecard: context.uiState.activeScorecard,
        visibleLayers: context.uiState.visibleLayers,
        activeFilters: context.uiState.activeFilters,
        
        // Visual context
        screenshot: context.screenshot,
        queryPosition: context.queryPosition,
        localKPIs: context.kpiValues
      }
    };

    const response = await fetch(`${this.apiUrl}/spatial/chat`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.apiKey}`
      },
      body: JSON.stringify(payload)
    });

    return (await response.json()).response;
  }
}

// Complete chat integration
class SpatialChatManager {
  constructor(
    private annotationManager: SpeckleAnnotationManager,
    private llmService: ExternalLLMService
  ) {}

  // Handle click on pre-existing Speckle annotation
  handleAnnotationClick(annotationId: string): ChatBubble {
    const annotation = this.annotationManager.getAnnotation(annotationId);
    
    return this.createChatBubble({
      position: annotation.position,
      initialContent: annotation.summaryText,
      context: {
        annotation,
        regionSummary: this.annotationManager.getRegionSummaryAtPosition(annotation.position),
        activeScorecard: this.getCurrentScorecard()
      }
    });
  }

  // Handle click anywhere on canvas
  async handleCanvasClick(
    position: [number, number, number], 
    question: string
  ): Promise<ChatBubble> {
    const context: SpatialChatContext = {
      // Speckle-based contextual data
      regionSummary: this.annotationManager.getRegionSummaryAtPosition(position),
      nearbyAnnotations: this.annotationManager.getNearbyAnnotations(position),
      
      // Current state
      uiState: this.getCurrentUIState(),
      queryPosition: position,
      screenshot: await this.captureViewport(),
      kpiValues: this.getKPIsAtPosition(position)
    };

    const response = await this.llmService.askSpatialQuestion(question, context);
    
    return this.createChatBubble({
      position,
      content: response,
      context
    });
  }
}

interface SpatialChatContext {
  regionSummary: RegionSummary | null;       // From Speckle curve/polygon geometry
  nearbyAnnotations: PreGeneratedAnnotation[]; // From Speckle point geometry
  uiState: UIState;
  queryPosition: [number, number, number];
  screenshot: string;
  kpiValues: Record<string, number>;
}
```

These implementation patterns provide a solid foundation for building the Urban Analytics Platform, leveraging kepler.gl's proven architecture while adapting it for BIM/Speckle data integration, real-time scorecard interactions, and AI-powered insights.
