# Urban Analytics Platform - User Flows

## Overview

This document describes key user flows for the Urban Analytics Platform, demonstrating how the data taxonomy and scorecard system work together to create engaging client experiences.

## User Flow 1: Scorecard Exploration - Wind Comfort

### Narrative
As a planner evaluating a design variant, I want to open the Wind-Comfort scorecard, examine low-performance areas, and ask questions in context so that I understand where wind comfort fails and why.

### Pre-conditions
- The application is in Presentation mode
- The side-panel lists available scorecards
- Speckle data loaded with wind comfort analysis results

### Main Flow

**1. Open the scorecard**
- User clicks "Wind Comfort" in the side-panel
- The card turns active, showing:
  - Large KPI: "Avg. Windspeed" (from KPI registry)
  - Five-slice Lawson pie-chart (categories from KPI definitions)
  - Static histogram (non-interactive, for context)

**2. Canvas updates**
- 3D map colors the wind-comfort grid by category
- Areas below comfort threshold receive yellow call-outs
- Preset filters applied: `geometry_type: "grid_cell"`, `analysis_type: "wind_comfort"`

**3. Read annotations**
- Hovering an annotation reveals tooltip:
  *"Low shading near tower – strong gusts 11 AM – 3 PM"*
- AI-generated based on spatial context and KPI values

**4. Ask an in-place question**
- Small "+" on call-out opens chat bubble anchored to that spot
- User types: "Why is this spot so windy?"
- AI receives: annotation position, local KPI values, nearby geometry
- Inline AI answer appears with spatial context

**5. Hover exploration**
- Moving cursor across grid shows live tooltips
- Displays: wind speed, comfort category, any extra configured metrics
- Real-time KPI computation using optimized aggregation functions

**6. Contextual chat anywhere**
- User right-clicks any point on canvas
- Text box appears
- Question + answer anchored to that location
- AI context includes: clicked position, visible KPIs, active filters

**7. Filter via pie-chart**
- User clicks pie slice "Comfortable while running"
- Chart interaction triggers filter override
- Non-matching cells fade out, selected category highlighted
- KPI values recalculate for filtered subset

**8. Toggle context layers**
- Right-hand layer toolbar shows: Trees, Buildings, Grid
- User hides Trees and Buildings for cleaner view
- Layer visibility persists within current scorecard

**9. Switch to another KPI**
- User clicks "Thermal Heat-Intake" in side-panel
- Canvas recolors buildings by heat-intake values
- Trees reappear (defined in thermal scorecard configuration)
- Previous filters reset to new scorecard's presets

### Post-conditions
- User understands wind-comfort hotspots and mitigation reasons
- Chosen filters and layer visibility persist until new scorecard selected
- AI chat history available for reference

### Technical Implementation Notes
- **Data Flow**: `grid_cells` dataset → `wind_comfort_category` field → pie chart categories
- **KPI Computation**: Uses `withinBounds` aggregation for Lawson categories
- **Filter Management**: Scorecard presets override user selections when switching
- **AI Context**: Spatial position + active KPIs + visible geometry types

---

## User Flow 2: Overview/Summary Mode

### Narrative
As a client reviewing design options, I want to see a high-level overview of all key performance indicators across multiple analysis categories so that I can quickly understand the overall project performance and identify areas needing attention.

### Pre-conditions
- Application loaded with complete analysis dataset
- Multiple analysis categories available (wind, solar, biodiversity, accessibility)
- Comparison mode enabled with baseline/alternative scenarios

### Main Flow

**1. Enter overview mode**
- User clicks "Overview" in the main navigation
- Interface switches to dashboard layout
- Multiple mini-scorecards displayed in grid format

**2. Multi-category dashboard**
- Grid shows 6 mini-scorecards:
  - Wind Comfort (avg comfort score + status indicator)
  - Solar Access (total sunshine hours + percentage above threshold)
  - Biodiversity (species count + habitat quality index)
  - Accessibility (walkability score + transit access)
  - Thermal Comfort (avg temperature + heat stress zones)
  - Air Quality (pollution levels + health impact score)

**3. Status indicators**
- Each mini-scorecard shows:
  - Primary KPI value (large number)
  - Status indicator (green/yellow/red based on thresholds)
  - Trend arrow (improvement/decline vs. baseline)
  - Sparkline chart showing distribution

**4. Comparative analysis**
- Toggle between "Current Design" and "Baseline Scenario"
- Delta indicators show percentage changes
- Color coding: green (improvement), red (decline), gray (neutral)

**5. Drill-down interaction**
- Click any mini-scorecard to open full scorecard view
- Maintains overview context (breadcrumb navigation)
- "Back to Overview" button preserves previous state

**6. Spatial overview**
- Canvas shows aggregated performance heatmap
- Color intensity represents overall performance index
- Hover reveals top 3 performing/underperforming KPIs for that area

**7. AI summary generation**
- "Generate Summary" button triggers AI analysis
- AI processes all KPIs across all categories
- Returns executive summary with:
  - Top 3 strengths of the design
  - Top 3 areas for improvement
  - Specific recommendations with spatial references

**8. Export and presentation**
- "Export Dashboard" creates PDF summary
- Includes all KPI values, charts, and AI insights
- Formatted for client presentation

### Post-conditions
- User has comprehensive understanding of project performance
- Key insights identified for detailed investigation
- Summary materials prepared for stakeholder communication

### Technical Implementation Notes
- **Data Aggregation**: Cross-category KPI computation using multiple datasets
- **Performance Index**: Weighted average of normalized KPI scores
- **AI Processing**: Batch analysis of all KPIs with spatial context
- **State Management**: Overview state preserved when drilling down to individual scorecards

---

## Data Taxonomy Alignment

### Scorecard Flow Alignment
✅ **Geometry Grouping**: Wind comfort uses `grid_cell` geometry type
✅ **KPI Definitions**: Lawson categories from unified KPI registry
✅ **Chart Configuration**: Pie chart with category-based aggregation
✅ **Filter Presets**: Automatic scorecard-specific filtering
✅ **AI Integration**: Contextual annotations with spatial awareness

### Overview Flow Alignment
✅ **Multi-Dataset**: Processes buildings, trees, roads, grid_cells simultaneously
✅ **Cross-Category KPIs**: Aggregates across different analysis categories
✅ **Comparison Mode**: Version-based comparison using Speckle commit IDs
✅ **Status Thresholds**: Uses flexible threshold system (absolute/relative/divergence)
✅ **AI Summarization**: Batch processing of spatial and analytical context

### Complexity Management
Both flows maintain simplicity while providing depth:
- **Scorecard Flow**: One analysis type per view, progressive disclosure
- **Overview Flow**: High-level aggregation with drill-down capability
- **Consistent Navigation**: Clear mental model for switching between modes
- **Preset-Based Configuration**: Reduces cognitive load, maintains focus

This design ensures that complex urban analytics remain accessible to non-technical stakeholders while providing the depth needed for informed decision-making.