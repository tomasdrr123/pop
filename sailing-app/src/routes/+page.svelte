<script>
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';
  import { tick } from 'svelte';
  
  let mapContainer;
  let map;
  let searchInput;
  let searchResults = [];
  let loading = false;
  let showResults = false;
  let cityFeatures = []; // Store all city features when map loads
  let popup; // Variable to hold the popup instance
  
  // Trip planner state
  let tripPorts = [];
  
  // Route line IDs for removal
  let routeSourceId = 'trip-route-source';
  let routeLayerId = 'trip-route-layer';
  
  // Define the grid resolution directly in pixels instead of meters
  const BASE_GRID_SIZE = 10; // Base pixels per grid cell (was GRID_RESOLUTION_METERS)
  let GRID_SIZE = BASE_GRID_SIZE; // Current grid size adjusted by zoom
  const COASTAL_PADDING = 3; // Padding cells around detected land
  
  // Algorithm selection
  let pathfindingAlgorithm = 'direct'; // Routing algorithm: 'dijkstra', 'astar', or 'direct'
  
  // Function to calculate the pixel size that corresponds to a given distance in meters
  function getPixelSizeForMeters(meters, map) {
    if (!map) return 4; // Default fallback if map not loaded
    
    try {
      // Get the map's current center
      const center = map.getCenter();
      
      // Create two points the specified meters apart
      // Convert meters to km for the calculation
      const km = meters / 1000;
      
      // Calculate the approximate longitude difference for the given distance
      // 111.32 km is approximately 1 degree at the equator
      // Adjust for the cosine of latitude
      const latRad = center.lat * (Math.PI / 180);
      const lngDiff = (km / (111.32 * Math.cos(latRad))) * 1;
      
      // Create a point at the calculated distance
      const point1 = center;
      const point2 = new mapboxgl.LngLat(center.lng + lngDiff, center.lat);
      
      // Convert both to pixels
      const pixel1 = map.project(point1);
      const pixel2 = map.project(point2);
      
      // Calculate pixel distance
      const pixelSize = Math.max(2, Math.round(Math.abs(pixel2.x - pixel1.x)));
      console.log(`Grid resolution: ${meters}m = ${pixelSize}px at current zoom`);
      return pixelSize;
    } catch (e) {
      console.error('Error calculating grid size based on meters:', e);
      return 4; // Fallback to default
    }
  }
  
  // Search method selection
  let searchMethod = 'local'; // 'local' or 'tilequery'
  
  // Tileset ID for Tilequery API - replace with your actual tileset ID
  const tilesetId = 'tomasdrr.5p3av8a2'; // Your actual tileset ID

  // Mapbox configuration
  const mapboxToken = 'pk.eyJ1IjoidG9tYXNkcnIiLCJhIjoiY202OWNjNDRyMDZ4ejJ2cXQxNnc5ZTJ5biJ9.gZVHMIpx8dsRNFsbfuoUHw';
  const mapboxStyle = 'mapbox://styles/tomasdrr/cm8etfz9z00tz01se50pycmdp';

  // Initialize map
  onMount(async () => {
    if (!browser) return;
    
    // Import Mapbox CSS
    await import('mapbox-gl/dist/mapbox-gl.css');
    
    // Initialize map
    initMap();
  });

  // Initialize map
  async function initMap() {
    if (!browser || !mapContainer) return;
    
    const mapboxgl = (await import('mapbox-gl')).default;
    mapboxgl.accessToken = mapboxToken;
    
    map = new mapboxgl.Map({
      container: mapContainer,
      style: mapboxStyle,
      center: [0, 0],
      zoom: 2
    });

    // Initialize popup
    popup = new mapboxgl.Popup({
      closeButton: true,
      closeOnClick: false,
      offset: 15
    });

    // Wait for map to load before setting up search
    map.on('load', () => {
      // Log all sources and layers to console for debugging
      console.log("Map sources:", Object.keys(map.getStyle().sources));
      console.log("Map layers:", map.getStyle().layers.map(layer => layer.id));
      
      // Find all symbol layers that might contain city data
      const symbolLayers = map.getStyle().layers
        .filter(layer => layer.type === 'symbol')
        .map(layer => layer.id);
      
      console.log("Symbol layers that might contain cities:", symbolLayers);
      
      // Try to identify tileset ID for cities
      const sources = map.getStyle().sources;
      Object.entries(sources).forEach(([id, source]) => {
        if (source.type === 'vector' && source.url) {
          console.log(`Source ${id} URL:`, source.url);
        }
      });
      
      // Initial calculation of grid size based on real-world meters
      GRID_SIZE = getPixelSizeForMeters(BASE_GRID_SIZE, map);
      console.log(`Initial grid size set to ${GRID_SIZE}px (representing ${BASE_GRID_SIZE}m at current zoom)`);
      
      // Set up the search functionality once we have layers info
      setupSearch();
      
      // Setup port click interaction
      setupPortClickHandler();
      
      // Initialize the route line source and layer
      initializeRouteLine();
    });
    
    // Update grid size when zoom level changes to maintain consistent real-world resolution
    map.on('zoom', () => {
      // Only recalculate occasionally to avoid performance issues
      if (Math.random() < 0.1) { // 10% chance to update on each zoom event
        GRID_SIZE = getPixelSizeForMeters(BASE_GRID_SIZE, map);
      }
    });
  }

  // Initialize route line source and layer
  function initializeRouteLine() {
    // Add an empty GeoJSON source for the route line
    map.addSource(routeSourceId, {
      type: 'geojson',
      data: {
        type: 'Feature',
        properties: {},
        geometry: {
          type: 'LineString',
          coordinates: []
        }
      }
    });
    
    // Add a line layer using the empty source
    map.addLayer({
      id: routeLayerId,
      type: 'line',
      source: routeSourceId,
      layout: {
        'line-join': 'round',
        'line-cap': 'round'
      },
      paint: {
        'line-color': '#0070f3',
        'line-width': 3,
        'line-dasharray': [0, 2, 1], // Creates a dashed line
        'line-opacity': 0.8
      }
    });
  }
  
  // Update route line when trip ports change
  async function updateRouteLine() {
    if (!map || !map.getSource(routeSourceId) || tripPorts.length < 2) {
      // If map isn't loaded, source doesn't exist, or not enough ports, clear the line
      if (map && map.getSource(routeSourceId)) {
        map.getSource(routeSourceId).setData({
          type: 'Feature',
          properties: {},
          geometry: {
            type: 'LineString',
            coordinates: []
          }
        });
      }
      return;
    }
    
    // Create an array to store all route segments
    let allRouteCoordinates = [];
    
    // For each consecutive pair of ports, find a water-only path
    for (let i = 0; i < tripPorts.length - 1; i++) {
      const startPort = tripPorts[i].coordinates;
      const endPort = tripPorts[i + 1].coordinates;
      
      // Use direct lines or calculate water path based on algorithm choice
      let pathCoordinates;
      if (pathfindingAlgorithm === 'direct') {
        // For direct lines, just use the start and end coordinates
        pathCoordinates = [startPort, endPort];
        console.log('Using direct line for route segment');
      } else {
        // Find water path between these two ports using pathfinding
        pathCoordinates = await findWaterPath(startPort, endPort);
      }
      
      // Add the path segment to our full route
      // If it's not the first segment, remove the first point to avoid duplication
      if (i === 0) {
        allRouteCoordinates = allRouteCoordinates.concat(pathCoordinates);
      } else {
        allRouteCoordinates = allRouteCoordinates.concat(pathCoordinates.slice(1));
      }
    }
    
    // Update the line source with water-only paths
    map.getSource(routeSourceId).setData({
      type: 'Feature',
      properties: {},
      geometry: {
        type: 'LineString',
        coordinates: allRouteCoordinates
      }
    });
  }

  // Setup port click handler with popup
  function setupPortClickHandler() {
    let selectedFeature = null;
    
    // Add click interaction for ports layer
    map.on('click', 'list-of-ports-new-march-she-7vyiom', (e) => {
      // If we already have a selected feature, reset its state
      if (selectedFeature) {
        map.setFeatureState(
          { source: selectedFeature.source, sourceLayer: selectedFeature.sourceLayer, id: selectedFeature.id },
          { state: false }
        );
      }
      
      // If there's a clicked feature
      if (e.features && e.features.length > 0) {
        const clickedFeature = e.features[0];
        
        // Toggle selection state
        if (selectedFeature && selectedFeature.id === clickedFeature.id) {
          // If it's the same feature, deselect it
          selectedFeature = null;
          popup.remove();
        } else {
          // Select the new feature
          selectedFeature = clickedFeature;
          
          // Set the feature state to true for styling
          map.setFeatureState(
            { source: clickedFeature.source, sourceLayer: clickedFeature.sourceLayer, id: clickedFeature.id },
            { state: true }
          );
          
          // Get port name from feature properties
          const portName = clickedFeature.properties.name || 
                           clickedFeature.properties.port_name || 
                           clickedFeature.properties.title || 
                           'Unknown Port';
          
          // Check if port is already in trip
          const isInTrip = tripPorts.some(port => port.id === clickedFeature.id);
          
          // Create popup HTML with Add button
          const popupHtml = `
            <div class="port-popup">
              <h3>${portName}</h3>
              <button class="add-port-btn" ${isInTrip ? 'disabled' : ''}>
                ${isInTrip ? 'Added to Trip' : 'Add to Trip'}
              </button>
            </div>
          `;
          
          // Display popup with port name and add button
          popup
            .setLngLat(e.lngLat)
            .setHTML(popupHtml)
            .addTo(map);
          
          // Add event listener to the Add button
          const addButton = document.querySelector('.add-port-btn');
          if (addButton && !isInTrip) {
            addButton.addEventListener('click', () => {
              addPortToTrip(clickedFeature, portName, e.lngLat);
            });
          }
        }
      }
    });
    
    // Change cursor to pointer when hovering over port features
    map.on('mouseenter', 'list-of-ports-new-march-she-7vyiom', () => {
      map.getCanvas().style.cursor = 'pointer';
    });
    
    // Change cursor back when leaving port features
    map.on('mouseleave', 'list-of-ports-new-march-she-7vyiom', () => {
      map.getCanvas().style.cursor = '';
    });
  }

  // Add port to trip planner
  function addPortToTrip(feature, portName, coordinates) {
    // Create port object
    const port = {
      id: feature.id,
      name: portName,
      coordinates: [coordinates.lng, coordinates.lat],
      properties: feature.properties
    };
    
    // Add to trip if not already included
    if (!tripPorts.some(p => p.id === port.id)) {
      tripPorts = [...tripPorts, port];
      
      // Update popup button to show "Added to Trip"
      const addButton = document.querySelector('.add-port-btn');
      if (addButton) {
        addButton.textContent = 'Added to Trip';
        addButton.disabled = true;
      }
      
      // Update the route line
      updateRouteLine();
    }
  }
  
  // Remove port from trip
  function removePortFromTrip(portId) {
    tripPorts = tripPorts.filter(port => port.id !== portId);
    
    // Update the route line after removing a port
    updateRouteLine();
  }
  
  // Go to port on map
  function goToPort(coordinates) {
    map.flyTo({
      center: coordinates,
      zoom: 10,
      essential: true
    });
  }
  
  // Clear all ports from trip
  function clearTrip() {
    tripPorts = [];
    
    // Clear the route line
    updateRouteLine();
  }

  // Find water routes using Mapbox's built-in water data
  async function findWaterPath(startCoord, endCoord) {
    // Ensure map is loaded
    if (!map) return [startCoord, endCoord];
    
    // Get current zoom level to adjust grid size
    const zoomLevel = map.getZoom();
    
    // Use a finer grid for better accuracy - reduced from 100 to 25
    GRID_SIZE = 25; // Smaller grid size for more accurate land/water detection
    
    console.log(`Using fixed grid size of ${GRID_SIZE}px at zoom level ${zoomLevel}`);
    
    // Make sure mapboxgl is available
    const mapboxgl = (await import('mapbox-gl')).default;

    console.log('Finding water path between', startCoord, 'and', endCoord);
    
    // Debug option: Visualize the grid with a temporary marker layer
    const debugGrid = true; // Keep debug visualization on to see water/land detection
    
    // Create a grid over the visible map area between the two points
    // First, convert geographic coordinates to pixel coordinates
    const startPoint = map.project(new mapboxgl.LngLat(startCoord[0], startCoord[1]));
    const endPoint = map.project(new mapboxgl.LngLat(endCoord[0], endCoord[1]));
    
    // Calculate distance for more generous padding calculation
    const pixelDistance = Math.sqrt(
      Math.pow(endPoint.x - startPoint.x, 2) + 
      Math.pow(endPoint.y - startPoint.y, 2)
    );
    
    // More generous padding to ensure we have enough water area to route around islands
    const padding = Math.min(1000, Math.max(500, pixelDistance * 0.5));
    
    const minX = Math.min(startPoint.x, endPoint.x) - padding;
    const maxX = Math.max(startPoint.x, endPoint.x) + padding;
    const minY = Math.min(startPoint.y, endPoint.y) - padding;
    const maxY = Math.max(startPoint.y, endPoint.y) + padding;
    
    // Calculate grid dimensions
    const gridWidth = Math.ceil((maxX - minX) / GRID_SIZE);
    const gridHeight = Math.ceil((maxY - minY) / GRID_SIZE);
    
    // Set a reasonable upper limit on grid size to prevent performance issues
    const MAX_GRID_SIZE = 300; // Increased from 250 to allow more detailed grids
    
    // Don't process extremely large grids that would be too slow
    if (gridWidth > MAX_GRID_SIZE || gridHeight > MAX_GRID_SIZE) {
      console.warn(`Grid too large (${gridWidth}x${gridHeight}) for A* pathfinding, using a coarser grid`);
      
      // Try again with a coarser grid
      const scaleFactor = Math.max(gridWidth, gridHeight) / MAX_GRID_SIZE;
      GRID_SIZE = Math.ceil(GRID_SIZE * scaleFactor);
      
      // Recalculate grid dimensions
      const newGridWidth = Math.ceil((maxX - minX) / GRID_SIZE);
      const newGridHeight = Math.ceil((maxY - minY) / GRID_SIZE);
      
      if (newGridWidth > MAX_GRID_SIZE || newGridHeight > MAX_GRID_SIZE) {
        console.warn('Still too large, falling back to direct line');
        return [startCoord, endCoord];
      }
      
      console.log(`Adjusted to coarser grid: ${newGridWidth}x${newGridHeight} with cell size ${GRID_SIZE}px`);
    }
    
    console.log(`Creating grid: ${gridWidth}x${gridHeight}`);
    
    // Grid features for visualization
    const gridFeatures = [];
    
    // Create our grid
    const grid = [];
    
    // Pre-populate grid with nodes (all land initially)
    for (let y = 0; y < gridHeight; y++) {
      const row = [];
      for (let x = 0; x < gridWidth; x++) {
        // Get the canvas pixel coordinates
        const pixelX = minX + (x * GRID_SIZE);
        const pixelY = minY + (y * GRID_SIZE);
        
        // Convert pixel coordinates back to geographic for water testing
        const lngLat = map.unproject([pixelX, pixelY]);
        
        row.push({
          x, y, // Grid coordinates
          pixelX, pixelY, // Canvas coordinates
          lngLat, // Geographic coordinates
          isWater: false, // Default to land, will check later
          isLand: true,   // Start with the assumption it's land
          isChecked: false, // Track if we've checked this cell
          f: 0, g: 0, h: 0, // A* scoring values
          visited: false,
          parent: null
        });
      }
      grid.push(row);
    }
    
    // Find the corresponding grid nodes for start and end points
    const startGridX = Math.floor((startPoint.x - minX) / GRID_SIZE);
    const startGridY = Math.floor((startPoint.y - minY) / GRID_SIZE);
    const endGridX = Math.floor((endPoint.x - minX) / GRID_SIZE);
    const endGridY = Math.floor((endPoint.y - minY) / GRID_SIZE);
    
    // Make sure start and end nodes are within grid bounds
    if (startGridX < 0 || startGridX >= gridWidth || startGridY < 0 || startGridY >= gridHeight ||
        endGridX < 0 || endGridX >= gridWidth || endGridY < 0 || endGridY >= gridHeight) {
      console.warn('Start or end point outside grid bounds, falling back to direct line');
      return [startCoord, endCoord];
    }
    
    const startNode = grid[startGridY][startGridX];
    const endNode = grid[endGridY][endGridX];
    
    // Get all rendered features for better feature detection
    const allMapFeatures = map.queryRenderedFeatures();
    
    // Get all map layers
    const mapLayers = map.getStyle().layers.map(layer => layer.id);
    
    // Find water and land layers
    const waterLayerIds = mapLayers.filter(id => 
      id.includes('water') || id === 'water' || id === 'waterway'
    );
    
    const landLayerIds = mapLayers.filter(id => 
      id.includes('land') || id === 'land' || id === 'landuse' || 
      id === 'landcover' || id.includes('fill')
    );
    
    console.log('Water layers:', waterLayerIds);
    console.log('Land layers:', landLayerIds);
    
    // Process the grid with more aggressive detection
    console.log('Detecting water and land across grid...');
    
    for (let y = 0; y < gridHeight; y++) {
      for (let x = 0; x < gridWidth; x++) {
        const node = grid[y][x];
        node.isChecked = true;
        
        // Sample 9 points within each cell for more accurate detection
        const samplePoints = 3; // 3x3 grid within each cell
        let waterCount = 0;
        let landCount = 0;
        
        for (let sy = 0; sy < samplePoints; sy++) {
          for (let sx = 0; sx < samplePoints; sx++) {
            // Calculate sample point coordinates with more even distribution
            const sampleX = node.pixelX + (sx * GRID_SIZE / (samplePoints - 1));
            const sampleY = node.pixelY + (sy * GRID_SIZE / (samplePoints - 1));
            
            const point = [sampleX, sampleY];
            
            // First check explicitly for land
            let isOnLand = false;
            
            // Check for land using all rendered features (most reliable)
            // This catches many land areas even if not in specific land layers
            const features = map.queryRenderedFeatures(point);
            
            // Comprehensive land check first (higher priority)
            for (const feature of features) {
              // Check layer ID
              if (feature.layer && feature.layer.id && (
                feature.layer.id.includes('land') || 
                feature.layer.id === 'land' || 
                feature.layer.id === 'landuse' ||
                feature.layer.id === 'landcover'
              )) {
                landCount++;
                isOnLand = true;
                break;
              }
              
              // Check properties
              if (feature.properties) {
                if (feature.properties.class === 'land' || 
                    feature.properties.type === 'land') {
                  landCount++;
                  isOnLand = true;
                  break;
                }
                
                // Some maps use different property names
                if (feature.properties.landcover || 
                    feature.properties.landuse) {
                  landCount++;
                  isOnLand = true;
                  break;
                }
              }
            }
            
            // Only check for water if not on land
            if (!isOnLand) {
              // Check for water using water layers
              if (waterLayerIds.length > 0) {
                const waterFeatures = map.queryRenderedFeatures(point, {
                  layers: waterLayerIds
                });
                
                if (waterFeatures.length > 0) {
                  waterCount++;
                  continue;
                }
              }
              
              // Alternative water check - background might be water
              let isWater = false;
              for (const feature of features) {
                // Check for water in layer ID
                if (feature.layer && feature.layer.id && (
                  feature.layer.id.includes('water') || 
                  feature.layer.id === 'water' || 
                  feature.layer.id === 'waterway'
                )) {
                  isWater = true;
                  waterCount++;
                  break;
                }
                
                // Check for water in properties
                if (feature.properties && (
                  feature.properties.class === 'water' ||
                  feature.properties.type === 'water'
                )) {
                  isWater = true;
                  waterCount++;
                  break;
                }
              }
              
              // If no layers found but we're on a blue area, might be water
              // This is a fallback for maps without explicit water layers
              if (!isWater) {
                // If feature count is 0 or very low, likely water (empty space)
                if (features.length <= 1) {
                  // In marine maps, empty areas are often water
                  waterCount++;
                }
              }
            }
          }
        }
        
        const totalSamples = samplePoints * samplePoints;
        
        // ANY land detection means the cell is considered land - zero tolerance
        if (landCount > 0) {
          node.isWater = false;
          node.isLand = true;
        }
        // Require significant majority water samples to mark as water (at least 2/3)
        else if (waterCount >= Math.ceil(totalSamples * 0.66)) {
          node.isWater = true;
          node.isLand = false;
        }
        // By default, treat as land - safest approach for navigation
        else {
          node.isWater = false;
          node.isLand = true;
        }
        
        // Add to visualization
        if (debugGrid) {
          gridFeatures.push({
            type: 'Feature',
            properties: {
              isWater: node.isWater,
              isLand: node.isLand,
              waterCount,
              landCount,
              isStart: (x === startGridX && y === startGridY),
              isEnd: (x === endGridX && y === endGridY)
            },
            geometry: {
              type: 'Point',
              coordinates: [node.lngLat.lng, node.lngLat.lat]
            }
          });
        }
      }
    }
    
    // Force the start and end nodes to be water since they're ports
    startNode.isWater = true;
    startNode.isLand = false;
    endNode.isWater = true;
    endNode.isLand = false;
    
    // Update visualization for start/end
    if (debugGrid) {
      const startIndex = startGridY * gridWidth + startGridX;
      const endIndex = endGridY * gridWidth + endGridX;
      
      if (startIndex >= 0 && startIndex < gridFeatures.length) {
        gridFeatures[startIndex].properties.isWater = true;
        gridFeatures[startIndex].properties.isLand = false;
      }
      
      if (endIndex >= 0 && endIndex < gridFeatures.length) {
        gridFeatures[endIndex].properties.isWater = true;
        gridFeatures[endIndex].properties.isLand = false;
      }
    }
    
    // Apply padding around land to keep routes offshore
    // Use larger padding for safer navigation
    const COASTAL_PADDING = 2;
    
    for (let y = 0; y < gridHeight; y++) {
      for (let x = 0; x < gridWidth; x++) {
        if (grid[y][x].isLand) {
          // Apply buffer around land in a circular pattern
          for (let dy = -COASTAL_PADDING; dy <= COASTAL_PADDING; dy++) {
            for (let dx = -COASTAL_PADDING; dx <= COASTAL_PADDING; dx++) {
              const distSq = dx*dx + dy*dy;
              
              // Only apply buffer within the radius (circular pattern)
              if (distSq <= COASTAL_PADDING * COASTAL_PADDING) {
                const ny = y + dy;
                const nx = x + dx;
                
                // Make sure we're within grid bounds
                if (ny >= 0 && ny < gridHeight && nx >= 0 && nx < gridWidth) {
                  // Don't modify start or end points
                  if ((nx === startGridX && ny === startGridY) || 
                      (nx === endGridX && ny === endGridY)) {
                    continue;
                  }
                  
                  // Mark as coastal buffer (not water)
                  grid[ny][nx].isWater = false;
                  grid[ny][nx].isCoastal = true;
                  
                  // Update visualization
                  if (debugGrid) {
                    const index = ny * gridWidth + nx;
                    if (index >= 0 && index < gridFeatures.length) {
                      gridFeatures[index].properties.isWater = false;
                      gridFeatures[index].properties.isCoastal = true;
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
    
    // Update debug visualization
    if (debugGrid) {
      // Create or update debug grid source
      if (!map.getSource('debug-grid-source')) {
        map.addSource('debug-grid-source', {
          type: 'geojson',
          data: {
            type: 'FeatureCollection',
            features: gridFeatures
          }
        });
      } else {
        map.getSource('debug-grid-source').setData({
          type: 'FeatureCollection',
          features: gridFeatures
        });
      }
      
      // Add or update visualization layer
      if (!map.getLayer('debug-grid-layer')) {
        map.addLayer({
          id: 'debug-grid-layer',
          type: 'circle',
          source: 'debug-grid-source',
          paint: {
            'circle-radius': 2,
            'circle-color': [
              'case',
              ['get', 'isStart'], '#00ff00', // Start = green
              ['get', 'isEnd'], '#ff00ff',   // End = magenta
              ['get', 'isWater'], '#0088ff', // Water = blue
              ['get', 'isCoastal'], '#ff8800', // Coastal = orange
              '#ff0000' // Land = red
            ],
            'circle-opacity': 0.7
          }
        });
      }
    }
    
    // Use Dijkstra's algorithm instead of A* for more reliable water-only paths
    // Dijkstra's doesn't rely on a heuristic which can sometimes lead to cutting corners
    console.log('Running Dijkstra search for water-only path...');
    const path = findDijkstraPath(grid, startNode, endNode);
    
    // If no path found with standard algorithm, try relaxing the coastal buffer
    if (!path || path.length === 0) {
      console.warn('No path found, trying with reduced buffer');
      
      // Reset coastal buffer cells to water (keep explicit land as land)
      for (let y = 0; y < gridHeight; y++) {
        for (let x = 0; x < gridWidth; x++) {
          // If it's a coastal buffer but not explicitly land, reset to water
          if (grid[y][x].isCoastal && !grid[y][x].isLand) {
            grid[y][x].isWater = true;
            grid[y][x].isCoastal = false;
            
            // Update visualization
            if (debugGrid) {
              const index = y * gridWidth + x;
              if (index >= 0 && index < gridFeatures.length) {
                gridFeatures[index].properties.isWater = true;
                gridFeatures[index].properties.isCoastal = false;
              }
            }
          }
        }
      }
      
      // Try with reduced buffer
      console.log('Trying with reduced buffer...');
      const reducedPath = findDijkstraPath(grid, startNode, endNode);
      
      if (!reducedPath || reducedPath.length === 0) {
        console.warn('Still no path found, falling back to direct line');
        return [startCoord, endCoord];
      }
      
      // Update visualization with found path
      if (debugGrid) {
        showPathOnGrid(reducedPath);
      }
      
      console.log(`Found water path with reduced buffer: ${reducedPath.length} points`);
      return reducedPath.map(node => [node.lngLat.lng, node.lngLat.lat]);
    }
    
    // Update visualization with found path
    if (debugGrid) {
      showPathOnGrid(path);
    }
    
    console.log(`Found water path: ${path.length} points`);
    
    // Convert path to geographic coordinates
    return path.map(node => [node.lngLat.lng, node.lngLat.lat]);
  }

  // Helper to visualize the path on the grid
  function showPathOnGrid(path) {
    // Create path feature
    const pathFeature = {
      type: 'Feature',
      properties: {},
      geometry: {
        type: 'LineString',
        coordinates: path.map(node => [node.lngLat.lng, node.lngLat.lat])
      }
    };
    
    // Create or update path source
    if (!map.getSource('debug-path-source')) {
      map.addSource('debug-path-source', {
        type: 'geojson',
        data: pathFeature
      });
    } else {
      map.getSource('debug-path-source').setData(pathFeature);
    }
    
    // Add or update path layer
    if (!map.getLayer('debug-path-layer')) {
      map.addLayer({
        id: 'debug-path-layer',
        type: 'line',
        source: 'debug-path-source',
        layout: {
          'line-join': 'round',
          'line-cap': 'round'
        },
        paint: {
          'line-color': '#00ff00',
          'line-width': 3
        }
      });
    }
  }

  // Dijkstra's algorithm for finding water-only paths
  function findDijkstraPath(grid, start, end) {
    // Reset visited state and distance for all nodes
    for (let y = 0; y < grid.length; y++) {
      for (let x = 0; x < grid[0].length; x++) {
        const node = grid[y][x];
        node.visited = false;
        node.distance = Infinity;
        node.parent = null;
      }
    }
    
    // Set start node
    start.distance = 0;
    
    // Priority queue - nodes with lowest distance first
    const queue = [start];
    
    while (queue.length > 0) {
      // Sort queue by distance (lowest first) - simple priority queue
      queue.sort((a, b) => a.distance - b.distance);
      
      // Get the node with lowest distance
      const current = queue.shift();
      
      // If we've already visited this node, skip
      if (current.visited) continue;
      
      // Mark as visited
      current.visited = true;
      
      // If we reached the end, reconstruct and return the path
      if (current === end) {
        const path = [];
        let temp = current;
        while (temp) {
          path.unshift(temp);
          temp = temp.parent;
        }
        return path;
      }
      
      // Get water-only neighbors
      const neighbors = getWaterNeighbors(grid, current);
      
      // Process each neighbor
      for (const neighbor of neighbors) {
        // Skip if already visited
        if (neighbor.visited) continue;
        
        // Calculate distance to this neighbor
        // Use distance between points as cost
        const dx = neighbor.x - current.x;
        const dy = neighbor.y - current.y;
        const moveCost = Math.sqrt(dx*dx + dy*dy);
        
        // Calculate total distance to reach this neighbor
        const distance = current.distance + moveCost;
        
        // If this path is better than any previous one, update
        if (distance < neighbor.distance) {
          neighbor.distance = distance;
          neighbor.parent = current;
          
          // Add to queue for processing
          queue.push(neighbor);
        }
      }
    }
    
    // No path found
    return null;
  }
  
  // Get only water neighbors - strict water-only checking
  function getWaterNeighbors(grid, node) {
    const neighbors = [];
    
    // 8-direction neighborhood
    const dirs = [
      [-1, -1], [0, -1], [1, -1],
      [-1, 0],           [1, 0],
      [-1, 1],  [0, 1],  [1, 1]
    ];
    
    for (const [dx, dy] of dirs) {
      const newY = node.y + dy;
      const newX = node.x + dx;
      
      // Check if the new position is within the grid
      if (newY >= 0 && newY < grid.length && newX >= 0 && newX < grid[0].length) {
        const neighbor = grid[newY][newX];
        
        // STRICT WATER-ONLY POLICY: Only consider water cells
        if (neighbor.isWater) {
          // Diagonal movement needs extra checks to prevent corner cutting
          if (Math.abs(dx) === 1 && Math.abs(dy) === 1) {
            // Check both cardinal neighbors to ensure we're not cutting across land
            const horizontalNeighbor = grid[node.y][node.x + dx];
            const verticalNeighbor = grid[node.y + dy][node.x];
            
            // Only allow diagonal if BOTH adjacent cells are water
            // This prevents cutting corners around coastlines
            if (horizontalNeighbor.isWater && verticalNeighbor.isWater) {
              neighbors.push(neighbor);
            }
          } else {
            // Non-diagonal movement to water is always allowed
            neighbors.push(neighbor);
          }
        }
      }
    }
    
    return neighbors;
  }

  // Debug function to visualize the water detection using Mapbox's water layers
  function visualizeWaterDetection() {
    if (!map) return;
    
    console.log('Visualizing water detection...');
    
    // Debug: log all layers and sources to help diagnose the issue
    const mapStyle = map.getStyle();
    console.log('Current map style:', mapStyle.name || 'unnamed style');
    console.log('Available sources:', Object.keys(mapStyle.sources));
    
    // Log all layers in the style
    console.log('All layers in style:');
    mapStyle.layers.forEach(layer => {
      console.log(`Layer: ${layer.id}, Type: ${layer.type}, Source: ${layer.source}, Source-layer: ${layer['source-layer'] || 'none'}`);
    });
    
    // Print water and land layers specifically
    const waterLayerNames = mapStyle.layers
      .filter(layer => layer.id.includes('water') || (layer.id === 'water'))
      .map(layer => layer.id);
    console.log('Found water layers:', waterLayerNames);
    
    const landLayerNames = mapStyle.layers
      .filter(layer => layer.id.includes('land') || (layer.id === 'land'))
      .map(layer => layer.id);
    console.log('Found land layers:', landLayerNames);
    
    // Is debug grid already active?
    if (map.getLayer('debug-grid-layer')) {
      // Remove debug grid
      if (map.getLayer('debug-grid-layer')) {
        map.removeLayer('debug-grid-layer');
      }
      if (map.getSource('debug-grid-source')) {
        map.removeSource('debug-grid-source');
      }
      
      // If a test path is displayed, remove it
      if (map.getLayer('debug-path-layer')) {
        map.removeLayer('debug-path-layer');
      }
      if (map.getSource('debug-path-source')) {
        map.removeSource('debug-path-source');
      }
      
      console.log('Removed debug visualization');
    } else {
      // Generate a test grid between two points to visualize water detection
      console.log('Generating test visualization grid...');
      
      // Create a test grid near current view
      const center = map.getCenter();
      const bounds = map.getBounds();
      
      // Find two points for testing - one near left edge and one near right edge
      const testStartCoord = [bounds.getWest() + (bounds.getEast() - bounds.getWest()) * 0.25, center.lat];
      const testEndCoord = [bounds.getWest() + (bounds.getEast() - bounds.getWest()) * 0.75, center.lat];
      
      // Run a test path calculation (without actually updating the route)
      createTestGrid(testStartCoord, testEndCoord);
    }
    
    // Is debug mode already active?
    if (window.waterDebugActive) {
      // Turn off debug mode
      console.log('Turning off water debug visualization');
      
      // Restore original water and land colors
      if (window.originalWaterStyles) {
        Object.entries(window.originalWaterStyles).forEach(([propKey, originalValue]) => {
          // Check if this is an opacity property
          if (propKey.endsWith('-opacity')) {
            const layerId = propKey.replace('-opacity', '');
            if (map.getLayer(layerId)) {
              map.setPaintProperty(layerId, 'fill-opacity', originalValue);
            }
          } else {
            // It's a color property
            if (map.getLayer(propKey)) {
              map.setPaintProperty(propKey, 'fill-color', originalValue);
            }
          }
        });
      }
      
      // Remove land outline layer if it exists
      if (map.getLayer('debug-land-outline')) {
        map.removeLayer('debug-land-outline');
      }
      
      // Remove message
      const msgElement = document.getElementById('water-debug-message');
      if (msgElement) msgElement.remove();
      
      // Reset flag
      window.waterDebugActive = false;
      return;
    }
    
    // Activate debug mode
    window.waterDebugActive = true;
    
    // Store original styles
    window.originalWaterStyles = {};
    
    // Find all water and land layers
    const mapLayers = map.getStyle().layers;
    
    // Enhance water visibility
    const waterLayers = mapLayers.filter(layer => 
      layer.id.includes('water') && layer.type === 'fill'
    );
    
    console.log(`Found ${waterLayers.length} water layers to highlight`);
    
    // Make water bright blue
    waterLayers.forEach(layer => {
      const origColor = map.getPaintProperty(layer.id, 'fill-color');
      window.originalWaterStyles[layer.id] = origColor;
      map.setPaintProperty(layer.id, 'fill-color', 'rgba(0, 120, 255, 0.7)');
    });
    
    // Instead of trying to outline land, let's directly color land areas red
    // This is more reliable than trying to add an outline layer
    const landLayers = mapLayers.filter(layer => 
      (layer.id.includes('land') || layer.id === 'land') && 
      layer.type === 'fill'
    );
    
    console.log(`Found ${landLayers.length} land layers to highlight`);
    
    // Store original land colors
    landLayers.forEach(layer => {
      const origColor = map.getPaintProperty(layer.id, 'fill-color');
      window.originalWaterStyles[layer.id] = origColor;
      
      // Set land areas to red with transparency
      map.setPaintProperty(layer.id, 'fill-color', 'rgba(255, 0, 0, 0.5)');
      
      // Increase opacity for better visibility
      const origOpacity = map.getPaintProperty(layer.id, 'fill-opacity');
      if (origOpacity) {
        window.originalWaterStyles[layer.id + '-opacity'] = origOpacity;
        map.setPaintProperty(layer.id, 'fill-opacity', 0.7);
      }
    });
    
    // If we didn't find any land layers, try a fallback approach with custom layer
    if (landLayers.length === 0) {
      console.log('No standard land layers found, adding custom debug layer');
      
      // Check if we can add a fill-extrusion layer to highlight land
      try {
        // Find a good source for terrain/land
        const possibleSources = Object.keys(map.getStyle().sources)
          .filter(source => map.getStyle().sources[source].type === 'vector');
        
        if (possibleSources.length > 0) {
          const firstSource = possibleSources[0];
          
          // Add a temporary layer that fills all land with red
          map.addLayer({
            'id': 'debug-land-outline',
            'type': 'fill',
            'source': firstSource,
            'source-layer': 'land', // Most vector tilesets use 'land'
            'paint': {
              'fill-color': 'rgba(255, 0, 0, 0.5)',
              'fill-opacity': 0.7
            }
          });
        }
      } catch (e) {
        console.error('Error adding fallback land highlight layer:', e);
      }
    }
    
    // Add message to screen
    const messageDiv = document.createElement('div');
    messageDiv.id = 'water-debug-message';
    messageDiv.style.position = 'absolute';
    messageDiv.style.top = '20px';
    messageDiv.style.left = '50%';
    messageDiv.style.transform = 'translateX(-50%)';
    messageDiv.style.backgroundColor = 'rgba(0,0,0,0.8)';
    messageDiv.style.color = 'white';
    messageDiv.style.padding = '10px 20px';
    messageDiv.style.borderRadius = '4px';
    messageDiv.style.zIndex = '999';
    messageDiv.style.fontFamily = 'Arial, sans-serif';
    messageDiv.style.cursor = 'pointer';
    messageDiv.innerHTML = 'Water Debug Mode - Blue: Water, Red: Land<br>' + 
                         '<small>Click to exit debug mode</small>';
    
    // Click to turn off debug mode
    messageDiv.onclick = visualizeWaterDetection;
    
    document.body.appendChild(messageDiv);
    console.log('Water visualization active - click message to disable');
  }

  // Function to create and visualize a test grid
  async function createTestGrid(startCoord, endCoord) {
    // Ensure map is loaded
    if (!map) return;
    
    // Make sure mapboxgl is available
    const mapboxgl = (await import('mapbox-gl')).default;
    
    // Use smaller grid size for more detailed visualization
    const testGridSize = 50;
    
    // Create a grid over the visible map area between the two points
    const startPoint = map.project(new mapboxgl.LngLat(startCoord[0], startCoord[1]));
    const endPoint = map.project(new mapboxgl.LngLat(endCoord[0], endCoord[1]));
    
    // Calculate padding
    const padding = 100;
    const minX = Math.min(startPoint.x, endPoint.x) - padding;
    const maxX = Math.max(startPoint.x, endPoint.x) + padding;
    const minY = Math.min(startPoint.y, endPoint.y) - padding;
    const maxY = Math.max(startPoint.y, endPoint.y) + padding;
    
    // Calculate grid dimensions
    const gridWidth = Math.ceil((maxX - minX) / testGridSize);
    const gridHeight = Math.ceil((maxY - minY) / testGridSize);
    
    console.log(`Creating test grid: ${gridWidth}x${gridHeight}`);
    
    // Create grid
    const grid = [];
    
    // Pre-populate grid
    for (let y = 0; y < gridHeight; y++) {
      const row = [];
      for (let x = 0; x < gridWidth; x++) {
        // Get pixel coordinates
        const pixelX = minX + (x * testGridSize);
        const pixelY = minY + (y * testGridSize);
        
        // Convert to geographic coordinates
        const lngLat = map.unproject([pixelX, pixelY]);
        
        row.push({
          x, y,
          pixelX, pixelY,
          lngLat,
          isWater: false,
          isLand: true
        });
      }
      grid.push(row);
    }
    
    // Find start and end nodes
    const startGridX = Math.floor((startPoint.x - minX) / testGridSize);
    const startGridY = Math.floor((startPoint.y - minY) / testGridSize);
    const endGridX = Math.floor((endPoint.x - minX) / testGridSize);
    const endGridY = Math.floor((endPoint.y - minY) / testGridSize);
    
    // Ensure in bounds
    if (startGridX < 0 || startGridX >= gridWidth || startGridY < 0 || startGridY >= gridHeight ||
        endGridX < 0 || endGridX >= gridWidth || endGridY < 0 || endGridY >= gridHeight) {
      console.warn('Test points out of grid bounds');
      return;
    }
    
    const startNode = grid[startGridY][startGridX];
    const endNode = grid[endGridY][endGridX];
    
    // Get all layers
    const mapLayers = map.getStyle().layers.map(layer => layer.id);
    
    // Find water and land layers
    const waterLayerIds = mapLayers.filter(id => 
      id.includes('water') || id === 'water' || id === 'waterway'
    );
    
    const landLayerIds = mapLayers.filter(id => 
      id.includes('land') || id === 'land' || id === 'landuse' || id === 'landcover'
    );
    
    // Detect water for each cell
    const gridFeatures = [];
    
    // Process the grid
    for (let y = 0; y < gridHeight; y++) {
      for (let x = 0; x < gridWidth; x++) {
        const node = grid[y][x];
        
        // Check center point
        const point = [node.pixelX + testGridSize/2, node.pixelY + testGridSize/2];
        
        // Check for land 
        let isLandPoint = false;
        if (landLayerIds.length > 0) {
          const landFeatures = map.queryRenderedFeatures(point, {
            layers: landLayerIds
          });
          
          if (landFeatures.length > 0) {
            isLandPoint = true;
          }
        }
        
        // Check for water if not land
        let isWaterPoint = false;
        if (!isLandPoint && waterLayerIds.length > 0) {
          const waterFeatures = map.queryRenderedFeatures(point, {
            layers: waterLayerIds
          });
          
          if (waterFeatures.length > 0) {
            isWaterPoint = true;
          }
        }
        
        // Mark cell
        node.isWater = isWaterPoint;
        node.isLand = isLandPoint || !isWaterPoint;
        
        // Special case for start and end
        if ((x === startGridX && y === startGridY) || (x === endGridX && y === endGridY)) {
          node.isWater = true;
          node.isLand = false;
        }
        
        // Add to visualization
        gridFeatures.push({
          type: 'Feature',
          properties: {
            isWater: node.isWater,
            isLand: node.isLand,
            isStart: (x === startGridX && y === startGridY),
            isEnd: (x === endGridX && y === endGridY)
          },
          geometry: {
            type: 'Point',
            coordinates: [node.lngLat.lng, node.lngLat.lat]
          }
        });
      }
    }
    
    // Apply coastal padding
    for (let y = 0; y < gridHeight; y++) {
      for (let x = 0; x < gridWidth; x++) {
        if (grid[y][x].isLand) {
          // Apply a 1-cell buffer
          const padding = 1;
          for (let dy = -padding; dy <= padding; dy++) {
            for (let dx = -padding; dx <= padding; dx++) {
              const ny = y + dy;
              const nx = x + dx;
              if (ny >= 0 && ny < gridHeight && nx >= 0 && nx < gridWidth &&
                  !(nx === startGridX && ny === startGridY) && 
                  !(nx === endGridX && ny === endGridY)) {
                grid[ny][nx].isWater = false;
                grid[ny][nx].isNearLand = true;
                
                // Update visualization
                const index = ny * gridWidth + nx;
                if (index >= 0 && index < gridFeatures.length) {
                  gridFeatures[index].properties.isWater = false;
                  gridFeatures[index].properties.isNearLand = true;
                }
              }
            }
          }
        }
      }
    }
    
    // Find path
    const path = aStarSearch(grid, startNode, endNode);
    
    // Create visualization
    if (!map.getSource('debug-grid-source')) {
      map.addSource('debug-grid-source', {
        type: 'geojson',
        data: {
          type: 'FeatureCollection',
          features: gridFeatures
        }
      });
    } else {
      map.getSource('debug-grid-source').setData({
        type: 'FeatureCollection',
        features: gridFeatures
      });
    }
    
    // Add grid visualization layer if it doesn't exist
    if (!map.getLayer('debug-grid-layer')) {
      map.addLayer({
        id: 'debug-grid-layer',
        type: 'circle',
        source: 'debug-grid-source',
        paint: {
          'circle-radius': 3,
          'circle-color': [
            'case',
            ['get', 'isStart'], '#00ff00',  // Start = green
            ['get', 'isEnd'], '#ff00ff',    // End = magenta
            ['get', 'isWater'], '#0088ff',  // Water = blue
            ['get', 'isNearLand'], '#ff8800', // Near land = orange
            '#ff0000'  // Land = red
          ],
          'circle-opacity': 0.7
        }
      });
    }
    
    // Visualize path if found
    if (path && path.length > 0) {
      const pathFeature = {
        type: 'Feature',
        properties: {},
        geometry: {
          type: 'LineString',
          coordinates: path.map(node => [node.lngLat.lng, node.lngLat.lat])
        }
      };
      
      if (!map.getSource('debug-path-source')) {
        map.addSource('debug-path-source', {
          type: 'geojson',
          data: pathFeature
        });
      } else {
        map.getSource('debug-path-source').setData(pathFeature);
      }
      
      if (!map.getLayer('debug-path-layer')) {
        map.addLayer({
          id: 'debug-path-layer',
          type: 'line',
          source: 'debug-path-source',
          layout: {
            'line-join': 'round',
            'line-cap': 'round'
          },
          paint: {
            'line-color': '#00ff00',
            'line-width': 3
          }
        });
      }
      
      console.log(`Found test path with ${path.length} points`);
    } else {
      console.warn('No test path found between points');
    }
  }

  // A* search algorithm implementation
  function aStarSearch(grid, start, end) {
    // Open set is a priority queue of nodes to be evaluated
    const openSet = [start];
    // Closed set contains nodes already evaluated
    const closedSet = [];
    
    // Initialize start node
    start.g = 0; // Cost from start to start is 0
    start.h = heuristic(start, end); // Estimated cost from start to end
    start.f = start.g + start.h; // Total cost
    
    while (openSet.length > 0) {
      // Find the node in openSet with lowest f value
      let lowestIndex = 0;
      for (let i = 0; i < openSet.length; i++) {
        if (openSet[i].f < openSet[lowestIndex].f) {
          lowestIndex = i;
        }
      }
      
      const current = openSet[lowestIndex];
      
      // If we've reached the end
      if (current === end) {
        // Reconstruct the path
        const path = [];
        let temp = current;
        path.push(temp);
        while (temp.parent) {
          path.push(temp.parent);
          temp = temp.parent;
        }
        
        // Smooth the path to make it more natural for sailing
        return smoothPath(path.reverse()); // Reverse to get path from start to end
      }
      
      // Remove current from openSet and add to closedSet
      openSet.splice(lowestIndex, 1);
      closedSet.push(current);
      
      // Get all neighbors of current node
      const neighbors = getNeighbors(grid, current);
      
      for (const neighbor of neighbors) {
        // Skip if already evaluated or not water
        if (closedSet.includes(neighbor) || !neighbor.isWater) continue;
        
        // Calculate tentative g score (cost from start to this neighbor through current)
        // We use distance as the cost
        const tentativeG = current.g + distance(current, neighbor);
        
        // If this neighbor is not in openSet, add it
        const notInOpenSet = !openSet.includes(neighbor);
        if (notInOpenSet) {
          openSet.push(neighbor);
        } else if (tentativeG >= neighbor.g) {
          // If we've already found a better path to this neighbor, skip
          continue;
        }
        
        // This path to neighbor is better than any previous one
        neighbor.parent = current;
        neighbor.g = tentativeG;
        neighbor.h = heuristic(neighbor, end);
        neighbor.f = neighbor.g + neighbor.h;
      }
    }
    
    // If we get here, no path was found
    return [];
  }
  
  // Calculate distance between two nodes (weighted Euclidean)
  function distance(a, b) {
    // Diagonal movement on water costs more to prioritize straighter paths
    // This makes the routes look more natural for sailing
    const isDiagonal = a.x !== b.x && a.y !== b.y;
    const baseDist = Math.sqrt(Math.pow(a.x - b.x, 2) + Math.pow(a.y - b.y, 2));
    
    // Slight preference for non-diagonal movement
    return isDiagonal ? baseDist * 1.1 : baseDist;
  }
  
  // Heuristic function for A* (weighted Euclidean distance)
  function heuristic(a, b) {
    // Euclidean distance with very strong water-favoring bias
    const directDist = Math.sqrt(Math.pow(a.x - b.x, 2) + Math.pow(a.y - b.y, 2));
    
    // Use an even stronger weighting to force the algorithm to go around land
    return directDist * 0.65; // Reduced from 0.75 to 0.65
  }
  
  // Get all valid neighbors of a node
  function getNeighbors(grid, node) {
    const neighbors = [];
    
    // 8-direction neighborhood
    const dirs = [
      [-1, -1], [0, -1], [1, -1],
      [-1, 0],           [1, 0],
      [-1, 1],  [0, 1],  [1, 1]
    ];
    
    for (const [dx, dy] of dirs) {
      const newY = node.y + dy;
      const newX = node.x + dx;
      
      // Check if the new position is within the grid
      if (newY >= 0 && newY < grid.length && newX >= 0 && newX < grid[0].length) {
        const neighbor = grid[newY][newX];
        
        // Only add water cells as neighbors
        if (neighbor.isWater) {
          // For diagonal movement, check diagonal path safety
          if (Math.abs(dx) === 1 && Math.abs(dy) === 1) {
            const verticalNeighbor = grid[node.y + dy][node.x];
            const horizontalNeighbor = grid[node.y][node.x + dx];
            
            // Only allow diagonal if BOTH adjacent cells are water
            // This is more restrictive than before and prevents cutting corners
            if (verticalNeighbor.isWater && horizontalNeighbor.isWater) {
              neighbors.push(neighbor);
            }
          } else {
            // Non-diagonal movement to water cells is always allowed
            neighbors.push(neighbor);
          }
        }
      }
    }
    
    return neighbors;
  }
  
  // Path smoothing function to make routes look more natural
  function smoothPath(path) {
    if (path.length <= 2) return path;
    
    const result = [path[0]];
    
    // Douglas-Peucker algorithm simplified for path smoothing
    // Start with just the endpoints and add points in the middle if needed
    const stack = [[0, path.length - 1]];
    
    while (stack.length > 0) {
      const [startIdx, endIdx] = stack.pop();
      
      if (endIdx - startIdx <= 1) continue;
      
      // Find the point farthest from the line
      let maxDist = 0;
      let maxIdx = 0;
      
      const startPoint = path[startIdx];
      const endPoint = path[endIdx];
      
      for (let i = startIdx + 1; i < endIdx; i++) {
        // Skip intermediate nodes if there's a valid water path
        if (i % 2 !== 0) continue;
        
        const point = path[i];
        
        // Check if the path between startPoint and endPoint goes over land
        let crosses_land = false;
        for (let t = 0.1; t <= 0.9; t += 0.1) {
          // Sample several points along the line
          const checkX = Math.floor(startPoint.x + t * (endPoint.x - startPoint.x));
          const checkY = Math.floor(startPoint.y + t * (endPoint.y - startPoint.y));
          
          // Check if this point is within grid bounds
          if (checkY >= 0 && checkY < path[0].grid.length && 
              checkX >= 0 && checkX < path[0].grid[0].length) {
            // If any sampled point is land, we can't smooth this section
            if (!path[0].grid[checkY][checkX].isWater) {
              crosses_land = true;
              break;
            }
          }
        }
        
        if (crosses_land) {
          // If path crosses land, then we need to include intermediate points
          maxDist = Number.MAX_VALUE;
          maxIdx = i;
          break;
        }
      }
      
      if (maxDist > 0) {
        // Add the point and recursively smooth both segments
        result.push(path[maxIdx]);
        stack.push([startIdx, maxIdx]);
        stack.push([maxIdx, endIdx]);
      } else {
        // The segment is smooth enough, just add the endpoint
        result.push(path[endIdx]);
      }
    }
    
    return result;
  }

  // Setup search functionality
  function setupSearch() {
    // We need to identify the source ID and source layer for the city data
    map.on('idle', () => {
      // Only fetch once
      if (cityFeatures.length === 0) {
        // Log all sources and their properties to help identify your tileset
        const sources = map.getStyle().sources;
        console.log("Available sources details:", sources);
        
        // Get all vector tile sources (most likely where your city data is)
        const vectorSources = Object.entries(sources)
          .filter(([_, source]) => source.type === 'vector')
          .map(([id]) => id);
        
        console.log("Vector sources:", vectorSources);
        
        // Find source layers in each vector source
        let allSourceLayers = [];
        vectorSources.forEach(sourceId => {
          try {
            // Try to get source layers for composite sources
            if (sourceId === 'composite') {
              // Composite sources need special handling
              // We'll need to check the layers for their source-layer property
              const layersWithSourceLayer = map.getStyle().layers
                .filter(layer => layer.source === 'composite' && layer['source-layer'])
                .map(layer => layer['source-layer']);
              
              console.log(`Source layers in ${sourceId}:`, [...new Set(layersWithSourceLayer)]);
              allSourceLayers.push(...layersWithSourceLayer);
            }
          } catch (e) {
            console.error(`Error examining source ${sourceId}:`, e);
          }
        });
        
        console.log("All source layers found:", [...new Set(allSourceLayers)]);
        
        // Try to find the city data by looking at each source and source-layer
        let allFeatures = [];
        
        // Loop through vector sources
        vectorSources.forEach(sourceId => {
          try {
            // Get all source layers that we found for this source
            const sourceLayers = map.getStyle().layers
              .filter(layer => layer.source === sourceId && layer['source-layer'])
              .map(layer => layer['source-layer']);
            
            // Query each source layer for features
            [...new Set(sourceLayers)].forEach(sourceLayer => {
              try {
                const features = map.querySourceFeatures(sourceId, {
                  sourceLayer: sourceLayer
                });
                
                if (features.length > 0) {
                  console.log(`Found ${features.length} features in ${sourceId}/${sourceLayer}`);
                  
                  // Check if these features look like cities (have name properties)
                  const citiesInLayer = features.filter(f => 
                    f.properties && 
                    (f.properties.name || f.properties.city || f.properties.place_name)
                  );
                  
                  if (citiesInLayer.length > 0) {
                    console.log(`Found ${citiesInLayer.length} possible cities in ${sourceId}/${sourceLayer}`);
                    allFeatures.push(...citiesInLayer);
                  }
                }
              } catch (e) {
                console.error(`Error querying ${sourceId}/${sourceLayer}:`, e);
              }
            });
          } catch (e) {
            console.error(`Error processing source ${sourceId}:`, e);
          }
        });
        
        // Now let's check the symbol layers that might have cities
        const symbolLayers = map.getStyle().layers
          .filter(layer => layer.type === 'symbol')
          .map(layer => layer.id);
        
        // Query features from symbol layers as backup
        symbolLayers.forEach(layerId => {
          try {
            const layerFeatures = map.queryRenderedFeatures({ layers: [layerId] });
            
            // Check if these features look like cities
            const citiesInLayer = layerFeatures.filter(f => 
              f.properties && 
              (f.properties.name || f.properties.city || f.properties.place_name)
            );
            
            if (citiesInLayer.length > 0) {
              console.log(`Found ${citiesInLayer.length} possible cities in symbol layer ${layerId}`);
              allFeatures.push(...citiesInLayer);
            }
          } catch (e) {
            console.error(`Error querying layer ${layerId}:`, e);
          }
        });
        
        // Remove duplicates based on name and coordinates
        const uniqueFeatures = [];
        const seen = new Set();
        
        allFeatures.forEach(feature => {
          const name = feature.properties.name || feature.properties.city || feature.properties.place_name;
          if (name && feature.geometry && feature.geometry.coordinates) {
            const key = `${name}-${feature.geometry.coordinates[0]}-${feature.geometry.coordinates[1]}`;
            if (!seen.has(key)) {
              seen.add(key);
              uniqueFeatures.push(feature);
            }
          }
        });
        
        cityFeatures = uniqueFeatures;
        console.log(`Found ${cityFeatures.length} unique city features across all sources`);
      }
    });
  }

  // Handle search input - Determines which search method to use
  function handleSearch() {
    if (searchMethod === 'local') {
      handleLocalSearch();
    } else {
      handleTilequerySearch();
    }
  }

  // Local search using pre-loaded features
  function handleLocalSearch() {
    const query = searchInput.value.toLowerCase().trim();
    if (!query) {
      searchResults = [];
      showResults = false;
      return;
    }

    loading = true;
    showResults = true;
    
    try {
      // Filter city features based on query
      searchResults = cityFeatures
        .filter(feature => {
          const name = feature.properties.name || feature.properties.city || feature.properties.place_name || '';
          return name.toLowerCase().includes(query);
        })
        .slice(0, 5); // Limit to 5 results for performance
      
      console.log(`Found ${searchResults.length} matching cities for query "${query}" using local search`);
    } catch (error) {
      console.error("Local search error:", error);
      searchResults = [];
    } finally {
      loading = false;
    }
  }

  // Tilequery API search - searches the entire tileset regardless of what's loaded
  async function handleTilequerySearch() {
    const query = searchInput.value.toLowerCase().trim();
    if (!query) {
      searchResults = [];
      showResults = false;
      return;
    }

    loading = true;
    showResults = true;
    searchResults = [];
    
    try {
      // Current map center as a starting point for search
      const [lng, lat] = map.getCenter().toArray();
      
      // Use Mapbox Tilequery API to search around the current map center
      // Documentation: https://docs.mapbox.com/api/maps/tilequery/
      const apiUrl = `https://api.mapbox.com/v4/${tilesetId}/tilequery/${lng},${lat}.json?radius=50000&limit=10&dedupe=true&access_token=${mapboxToken}`;
      
      console.log(`Querying Tilequery API with URL: ${apiUrl}`);
      const response = await fetch(apiUrl);
      
      if (!response.ok) {
        console.error(`Tilequery API error status: ${response.status}, text: ${response.statusText}`);
        const errorText = await response.text();
        console.error(`Error response body: ${errorText}`);
        throw new Error(`Tilequery API error: ${response.statusText}`);
      }
      
      const data = await response.json();
      console.log("Full Tilequery API response:", data);
      console.log(`Tilequery API returned ${data.features?.length || 0} features`);
      
      // Log the first feature to see the property structure
      if (data.features && data.features.length > 0) {
        console.log("Sample feature properties:", data.features[0].properties);
        
        // Try to find what properties we should be searching
        const sampleFeature = data.features[0];
        console.log("All available properties in the first feature:", Object.keys(sampleFeature.properties));
      } else {
        console.log("No features returned from Tilequery API. Check that:");
        console.log("1. Your tileset ID is correct (currently using:", tilesetId, ")");
        console.log("2. The tileset contains data in the area around:", [lng, lat]);
        console.log("3. Your access token has access to this tileset");
        
        // Try with a larger radius as a test
        console.log("Trying with a larger radius (100km) as a test...");
        const testApiUrl = `https://api.mapbox.com/v4/${tilesetId}/tilequery/${lng},${lat}.json?radius=100000&limit=50&dedupe=true&access_token=${mapboxToken}`;
        const testResponse = await fetch(testApiUrl);
        
        if (testResponse.ok) {
          const testData = await testResponse.json();
          console.log(`Test with larger radius returned ${testData.features?.length || 0} features`);
          if (testData.features && testData.features.length > 0) {
            console.log("Sample feature from larger radius search:", testData.features[0]);
          }
        }
      }
      
      // If we got data but it's empty, try to give helpful debugging info
      if (data.features && data.features.length === 0) {
        console.log("The Tilequery API returned successfully but found no features in the area.");
        console.log("Consider zooming to an area where you know data exists in your tileset.");
      }
      
      // Filter the features for those that match the query
      const filteredFeatures = data.features ? data.features.filter(feature => {
        // Check all properties for matching text
        for (const [key, value] of Object.entries(feature.properties)) {
          if (typeof value === 'string' && value.toLowerCase().includes(query)) {
            console.log(`Match found in property "${key}" with value "${value}" for query "${query}"`);
            return true;
          }
        }
        return false;
      }) : [];
      
      searchResults = filteredFeatures.slice(0, 5); // Limit to 5 results
      console.log(`Found ${searchResults.length} matching cities for query "${query}" using Tilequery API`);
    } catch (error) {
      console.error("Tilequery search error:", error);
    } finally {
      loading = false;
    }
  }

  // Go to selected city
  function goToCity(feature) {
    if (!feature || !feature.geometry) return;
    
    // Get coordinates from feature
    const [lng, lat] = feature.geometry.coordinates;
    
    // Fly to location
    map.flyTo({
      center: [lng, lat],
      zoom: 10,
      essential: true
    });
    
    // Close search results
    searchResults = [];
    showResults = false;
  }

  // Handle click outside search results
  function handleClickOutside(event) {
    if (showResults && !event.target.closest('.search-container')) {
      showResults = false;
    }
  }

  // Toggle search method
  function toggleSearchMethod(method) {
    searchMethod = method;
    // Clear previous results
    searchResults = [];
    showResults = false;
    // If there's text in the search box, re-run the search with the new method
    if (searchInput && searchInput.value.trim()) {
      handleSearch();
    }
  }

  // Change the pathfinding algorithm
  function changePathfindingAlgorithm(algorithm) {
    pathfindingAlgorithm = algorithm;
    console.log(`Changed pathfinding algorithm to: ${algorithm}`);
    
    // Update routes if we have any
    if (tripPorts.length >= 2) {
      updateRouteLine();
    }
  }
</script>

<svelte:window on:click={handleClickOutside} />

<div id="map" bind:this={mapContainer}></div>

<div class="left-sidebar">
  <div class="search-container">
    <div class="search-bar">
      <input
        type="text"
        placeholder="Search for a city..."
        bind:this={searchInput}
        on:input={handleSearch}
        on:focus={() => showResults = searchResults.length > 0}
      />
      
      <div class="search-method-buttons">
        <button 
          class={searchMethod === 'local' ? 'active' : ''} 
          on:click={() => toggleSearchMethod('local')}
        >
          Local
        </button>
        <button 
          class={searchMethod === 'tilequery' ? 'active' : ''} 
          on:click={() => toggleSearchMethod('tilequery')}
        >
          API
        </button>
      </div>
      
      {#if loading}
        <div class="loading-indicator">
          <div class="spinner"></div>
        </div>
      {/if}
    </div>
    
    {#if showResults && searchResults.length > 0}
      <div class="search-results">
        {#each searchResults as result}
          <div class="search-result-item" on:click={() => goToCity(result)}>
            <div class="result-name">
              {result.properties.name || result.properties.city || result.properties.place_name || 'Unknown location'}
            </div>
            {#if result.properties.type || result.properties.place_type}
              <div class="result-type">
                {result.properties.type || result.properties.place_type}
              </div>
            {/if}
          </div>
        {/each}
      </div>
    {:else if showResults && searchInput?.value && !loading}
      <div class="search-results">
        <div class="no-results">No cities found</div>
      </div>
    {/if}
  </div>

  <!-- Trip Planner Panel (always visible) -->
  <div class="trip-planner-panel">
    <div class="trip-planner-header">
      <h2>Trip Planner</h2>
      <div class="trip-actions">
        {#if tripPorts.length}
          <button class="clear-trip-btn" on:click={clearTrip}>Clear All</button>
        {/if}
        <button class="debug-btn" on:click={visualizeWaterDetection}>Debug Waters</button>
      </div>
    </div>
    
    <!-- Algorithm Selection Dropdown -->
    <div class="algorithm-selector">
      <div class="algorithm-label">Routing Algorithm:</div>
      <div class="algorithm-options">
        <button 
          class={pathfindingAlgorithm === 'dijkstra' ? 'algorithm-btn active' : 'algorithm-btn'} 
          on:click={() => changePathfindingAlgorithm('dijkstra')}
        >
          Dijkstra
        </button>
        <button 
          class={pathfindingAlgorithm === 'astar' ? 'algorithm-btn active' : 'algorithm-btn'} 
          on:click={() => changePathfindingAlgorithm('astar')}
        >
          A*
        </button>
        <button 
          class={pathfindingAlgorithm === 'direct' ? 'algorithm-btn active' : 'algorithm-btn'} 
          on:click={() => changePathfindingAlgorithm('direct')}
        >
          Direct
        </button>
      </div>
    </div>
    
    {#if tripPorts.length === 0}
      <div class="empty-trip">
        <p>Click on ports in the map and add them to plan your trip</p>
      </div>
    {:else}
      <div class="trip-ports-list">
        {#each tripPorts as port, index}
          <div class="trip-port-item">
            <div class="port-number">{index + 1}</div>
            <div class="port-info">
              <div class="port-name">{port.name}</div>
            </div>
            <div class="port-actions">
              <button class="port-goto-btn" on:click={() => goToPort(port.coordinates)}>
                <span class="icon">📍</span>
              </button>
              <button class="port-remove-btn" on:click={() => removePortFromTrip(port.id)}>
                <span class="icon">✕</span>
              </button>
            </div>
          </div>
        {/each}
      </div>
    {/if}
  </div>
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    overflow: hidden;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  }
  
  #map {
    position: absolute;
    width: 100%;
    height: 100%;
  }

  /* Left sidebar containing search and trip planner */
  .left-sidebar {
    position: absolute;
    top: 20px;
    left: 20px;
    z-index: 1;
    width: 350px;
    display: flex;
    flex-direction: column;
    gap: 12px;
    max-height: calc(100vh - 40px);
  }

  .search-container {
    width: 100%;
  }
  
  .search-bar {
    display: flex;
    position: relative;
    width: 100%;
    align-items: center;
    background: white;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    overflow: hidden;
  }
  
  input[type="text"] {
    flex-grow: 1;
    padding: 12px 15px;
    border: none;
    font-size: 16px;
    outline: none;
  }

  .search-method-buttons {
    display: flex;
    border-left: 1px solid #eee;
  }
  
  .search-method-buttons button {
    padding: 12px 15px;
    background: none;
    border: none;
    cursor: pointer;
    font-size: 14px;
    color: #555;
    transition: all 0.2s ease;
  }
  
  .search-method-buttons button:hover {
    background: #f5f5f5;
  }
  
  .search-method-buttons button.active {
    background: #0070f3;
    color: white;
  }

  .loading-indicator {
    position: absolute;
    right: 110px; /* Position to not interfere with the buttons */
    top: 50%;
    transform: translateY(-50%);
  }
  
  .spinner {
    width: 20px;
    height: 20px;
    border: 2px solid rgba(0, 112, 243, 0.2);
    border-top-color: #0070f3;
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
  }
  
  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  .search-results {
    margin-top: 8px;
    background: white;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    max-height: 300px;
    overflow-y: auto;
  }

  .search-result-item {
    padding: 12px 15px;
    cursor: pointer;
    border-bottom: 1px solid #eee;
    transition: background-color 0.2s ease;
  }

  .search-result-item:last-child {
    border-bottom: none;
  }

  .search-result-item:hover {
    background-color: #f5f5f5;
  }
  
  .result-name {
    font-weight: 500;
    color: #333;
  }
  
  .result-type {
    font-size: 12px;
    color: #666;
    margin-top: 4px;
  }

  .no-results {
    padding: 15px;
    color: #888;
    text-align: center;
    font-size: 14px;
  }
  
  /* Port Popup Styles */
  :global(.port-popup) {
    text-align: center;
  }
  
  :global(.port-popup h3) {
    margin: 0 0 10px 0;
    font-size: 16px;
  }
  
  :global(.add-port-btn) {
    background-color: #0070f3;
    color: white;
    border: none;
    border-radius: 4px;
    padding: 6px 12px;
    font-size: 14px;
    cursor: pointer;
    transition: background-color 0.2s;
  }
  
  :global(.add-port-btn:hover) {
    background-color: #0051ab;
  }
  
  :global(.add-port-btn:disabled) {
    background-color: #a0c4e7;
    cursor: not-allowed;
  }
  
  /* Trip Planner Styles */
  .trip-planner-panel {
    background: white;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    overflow: hidden;
    display: flex;
    flex-direction: column;
    flex: 1;
    min-height: 200px;
    max-height: calc(100vh - 120px);
  }
  
  .trip-planner-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 15px;
    border-bottom: 1px solid #eee;
  }
  
  .trip-planner-header h2 {
    margin: 0;
    font-size: 18px;
  }
  
  .trip-actions {
    display: flex;
    gap: 8px;
    align-items: center;
  }

  .clear-trip-btn {
    background: none;
    border: none;
    color: #e53e3e;
    font-size: 14px;
    cursor: pointer;
    padding: 5px;
  }
  
  .clear-trip-btn:hover {
    text-decoration: underline;
  }

  .debug-btn {
    background: none;
    border: 1px solid #3b82f6;
    color: #3b82f6;
    font-size: 12px;
    border-radius: 4px;
    cursor: pointer;
    padding: 4px 8px;
    transition: all 0.2s ease;
  }
  
  .debug-btn:hover {
    background-color: #3b82f6;
    color: white;
  }
  
  .empty-trip {
    padding: 20px;
    text-align: center;
    color: #666;
  }
  
  .trip-ports-list {
    overflow-y: auto;
    flex: 1;
  }
  
  .trip-port-item {
    display: flex;
    align-items: center;
    padding: 12px 15px;
    border-bottom: 1px solid #eee;
  }
  
  .port-number {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 28px;
    height: 28px;
    background: #0070f3;
    color: white;
    border-radius: 50%;
    font-size: 14px;
    font-weight: bold;
    margin-right: 12px;
  }
  
  .port-info {
    flex-grow: 1;
  }
  
  .port-name {
    font-weight: 500;
    color: #333;
  }
  
  .port-actions {
    display: flex;
    gap: 8px;
  }
  
  .port-goto-btn, .port-remove-btn {
    background: none;
    border: none;
    cursor: pointer;
    width: 28px;
    height: 28px;
    border-radius: 4px;
    display: flex;
    justify-content: center;
    align-items: center;
    transition: background-color 0.2s;
  }
  
  .port-goto-btn:hover {
    background-color: #f0f0f0;
  }
  
  .port-remove-btn:hover {
    background-color: #fee2e2;
    color: #e53e3e;
  }
  
  .icon {
    font-size: 14px;
  }

  /* Algorithm Selector Styles */
  .algorithm-selector {
    padding: 10px 15px;
    border-bottom: 1px solid #eee;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  
  .algorithm-label {
    font-size: 14px;
    color: #555;
    font-weight: 500;
  }
  
  .algorithm-options {
    display: flex;
    gap: 6px;
  }
  
  .algorithm-btn {
    flex: 1;
    padding: 8px 0;
    background: #f5f5f5;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 13px;
    cursor: pointer;
    transition: all 0.2s ease;
  }
  
  .algorithm-btn:hover {
    background: #e5e5e5;
  }
  
  .algorithm-btn.active {
    background: #0070f3;
    color: white;
    border-color: #0070f3;
  }
</style>


