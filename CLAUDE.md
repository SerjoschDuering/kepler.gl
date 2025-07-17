# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Goals and Technical Approach

This project is being analyzed to understand how to build custom applications on top of deck.gl and mapbox with the following specific requirements:

- **State Management**: Using Zustand (NOT Redux) for application state
- **Data Integration**: Fetching from Speckle streams in the background
- **Rich Geometry Data**: Handling geometry objects with many attributes (KPIs, datapoints)
- **Interactive Filtering**: Working patterns for filtering layers/objects via attribute ranges (numeric, categorical)
- **Real-time Visualization**: 2D charts that update in real-time, on-the-fly KPI computation for changing selections
- **Map Enhancements**: Annotations on canvas/map, terrain and satellite image layers
- **Version Control**: Switching between versions of the data
- **Custom UI**: Custom widgets that set filters when clicked or customized

## Common Development Commands

### Build and Development
```bash
# Development server
yarn start

# Build for production
yarn build

# Build UMD bundle
yarn build:umd

# Build TypeScript declarations
yarn build:types

# Run tests
yarn test            # All tests
yarn test-jest       # Jest tests only
yarn test-tape       # Tape tests only
yarn test-headless   # Headless browser tests
```

### Code Quality
```bash
# Linting and TypeScript checking
yarn lint            # ESLint + TypeScript check
yarn typescript      # TypeScript check only
yarn prettier-all   # Format all code

# Coverage
yarn cover          # Full coverage report
```

### Example Applications
```bash
# Various example applications
yarn start:deck                    # Demo with deck.gl
yarn start:custom-reducer          # Custom reducer example
yarn start:replace-component       # Component replacement example
yarn start:custom-theme           # Custom theme example
yarn start:node-app               # Node.js server example
```

## Architecture Overview

### State Management Architecture (Redux → Zustand Migration Context)

kepler.gl uses a sophisticated Redux-based state management system with the following key patterns:

**Current Redux Structure:**
```typescript
KeplerGlState {
  visState: {
    layers: Layer[],
    filters: Filter[],
    datasets: {[id: string]: Dataset},
    layerData: {[id: string]: LayerData},
    interactionConfig: InteractionConfig
  },
  mapState: MapState,
  mapStyle: MapStyle,
  uiState: UiState
}
```

**Key Patterns for Zustand Migration:**
- **Functional Updates**: Each state slice uses pure updater functions
- **Immutable Patterns**: Extensive use of spread operators and helper functions
- **Cross-Cutting Concerns**: "Composer" functions handle actions affecting multiple state slices
- **Action Patterns**: Consistent action structure with typed payloads

### Layer System Architecture

**Base Layer Structure:**
- All layers inherit from `BaseLayer` class
- **Visual Channels**: Standardized mapping of data attributes to visual properties (color, size, height)
- **Column Modes**: Support for different data formats (lat/lng, GeoJSON, GeoArrow)
- **Update Triggers**: Efficient change detection for re-rendering

**Key Layer Types:**
- **Point Layer**: Basic point visualization with 3D support
- **Aggregation Layers**: Grid, Hexagon, Heatmap for spatial aggregation
- **Arc/Line Layers**: Connection-based visualization
- **Polygon Layers**: Complex geometry support with extrusion

### Filtering and Data Processing

**Filter Types:**
- **Range Filter**: Numeric ranges with histogram visualization
- **Multi-Select Filter**: Categorical value selection
- **Time Range Filter**: Temporal data with animation support
- **Polygon Filter**: Spatial filtering with drawn polygons

**Data Processing Pipeline:**
1. Raw data → Datasets (via file processors)
2. Datasets → Filtered datasets (via filters)
3. Filtered datasets → Layer data (via layer processors)
4. Layer data → Rendered visualization (via deck.gl)

**GPU vs CPU Filtering:**
- GPU filtering for real-time performance with large datasets
- CPU filtering for complex logic
- Hybrid approach for optimal performance

### Visual Channel System

Critical for mapping rich Speckle attributes to visual properties:

```typescript
visualChannels = {
  color: {
    field: 'colorField',      // Data attribute
    scale: 'colorScale',      // Scale type (linear, log, quantile, ordinal)
    domain: 'colorDomain',    // Data range
    range: 'colorRange',     // Visual range (colors)
  },
  size: {
    field: 'sizeField',
    scale: 'sizeScale',
    domain: 'sizeDomain',
    range: 'sizeRange',
  }
}
```

## Working with Rich Geometry Data (Speckle Integration)

### Data Container Interface
- **RowDataContainer**: Standard array/object data
- **ArrowDataContainer**: Apache Arrow columnar data (recommended for large datasets)
- **IndexedDataContainer**: Memory-optimized indexed access

### Rich Attribute Support
- **Nested property access**: Support for `object.property.subproperty` paths
- **Field typing**: Automatic detection of numeric, categorical, temporal data
- **Custom value accessors**: Functions to extract values from complex objects
- **Metadata preservation**: Maintains original object structure

## Key Patterns for Custom Applications

### 1. Custom Filter Widgets
```typescript
// Filter component pattern
const CustomFilter = ({filter, setFilter, setFilterPlot}) => (
  <CustomSlider
    range={filter.domain}
    value={filter.value}
    onChange={setFilter}
  />
);
```

### 2. Real-time KPI Computation
- Use filter synchronization for cross-dataset filtering
- Implement computed values that update based on selection changes
- Leverage the interaction system for selection-based calculations

### 3. Custom Layer Types
- Extend base layers for specialized visualization needs
- Implement custom geometry processing for Speckle objects
- Add specialized interaction handlers for construction workflows

### 4. Performance Optimization
- Use Arrow data containers for large datasets
- Implement efficient data accessors for nested properties
- Leverage aggregation layers for overview visualizations

## File Structure and Key Locations

### Core Architecture
- `/src/reducers/` - Redux state management (study for Zustand patterns)
- `/src/actions/` - Action creators and types
- `/src/layers/` - Layer implementations and visual channels
- `/src/utils/filter-utils.ts` - Filtering logic and utilities
- `/src/components/` - UI components and interactions

### Examples for Reference
- `/examples/demo-app/` - Main demo application
- `/examples/custom-reducer/` - Custom state management example
- `/examples/replace-component/` - Component customization
- `/examples/custom-theme/` - Theming and styling

### Key Files for Custom Development
- `/src/layers/src/base-layer.ts` - Base layer class
- `/src/utils/src/filter-utils.ts` - Filter system implementation
- `/src/reducers/src/vis-state-updaters.ts` - State update patterns
- `/src/components/src/filters/` - Filter UI components

## Dependencies and Ecosystem

### Core Dependencies
- **deck.gl**: WebGL-based visualization framework
- **mapbox-gl**: Map rendering and interaction
- **react**: UI framework
- **redux**: State management (to be replaced with Zustand)
- **styled-components**: CSS-in-JS styling

### Data Processing
- **Apache Arrow**: Columnar data format for performance
- **d3-array**: Data manipulation and scaling
- **@loaders.gl**: Data loading and parsing

### Build and Development
- **TypeScript**: Type safety and development experience
- **Babel**: JavaScript transpilation
- **webpack**: Module bundling
- **jest**: Testing framework

## Testing Strategy

### Test Types
- **Unit Tests**: Individual component and utility testing
- **Integration Tests**: Layer and filter system testing
- **Browser Tests**: Full application testing with headless browsers

### Running Tests
```bash
yarn test-node      # Node.js unit tests
yarn test-browser   # Browser-based tests
yarn test-headless  # Automated browser testing
```

## Development Workflow

### 1. Setting up Development Environment
```bash
# Install dependencies
yarn install

# Start development server
yarn start
```

### 2. Creating Custom Layers
- Extend `BaseLayer` class
- Implement required methods (`initializeLayerConfig`, `updateLayerData`, etc.)
- Define visual channels for attribute mapping
- Add layer-specific UI components

### 3. Implementing Custom Filters
- Create filter configuration object
- Implement filter validation logic
- Build UI component for filter interaction
- Connect to state management system

### 4. Performance Considerations
- Use GPU filtering for large datasets
- Implement efficient data accessors
- Leverage deck.gl's update triggers system
- Consider using Arrow data containers for large datasets

## Common Pitfalls and Solutions

### 1. Performance Issues
- **Problem**: Slow rendering with large datasets
- **Solution**: Use GPU filtering, Arrow data containers, and efficient data accessors

### 2. Memory Usage
- **Problem**: High memory consumption with complex geometries
- **Solution**: Implement data virtualization and efficient data structures

### 3. State Management Complexity
- **Problem**: Complex state updates across multiple layers and filters
- **Solution**: Use composer functions and immutable update patterns

### 4. Custom UI Integration
- **Problem**: Difficulty integrating custom widgets
- **Solution**: Use the dependency injection system and follow component factory patterns

This documentation provides the foundation for building sophisticated visualization applications with Speckle's rich 3D geometry and attribute data, leveraging kepler.gl's proven patterns while adapting them for modern state management with Zustand.