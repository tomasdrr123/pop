<script>
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';
  
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
  
  // Search method selection
  let searchMethod = 'local'; // 'local' or 'tilequery'
  
  // Tileset ID for Tilequery API - replace with your actual tileset ID
  const tilesetId = 'tomasdrr.5p3av8a2'; // Your actual tileset ID

  // Mapbox configuration
  const mapboxToken = 'pk.eyJ1IjoidG9tYXNkcnIiLCJhIjoiY202OWNjNDRyMDZ4ejJ2cXQxNnc5ZTJ5biJ9.gZVHMIpx8dsRNFsbfuoUHw';
  const mapboxStyle = 'mapbox://styles/tomasdrr/cm82uq9xd00y201qsdqqn40um';

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
      
      // Set up the search functionality once we have layers info
      setupSearch();
      
      // Setup port click interaction
      setupPortClickHandler();
      
      // Initialize the route line source and layer
      initializeRouteLine();
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
  function updateRouteLine() {
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
    
    // Extract coordinates from tripPorts in order
    const coordinates = tripPorts.map(port => port.coordinates);
    
    // Update the line source with new coordinates
    map.getSource(routeSourceId).setData({
      type: 'Feature',
      properties: {},
      geometry: {
        type: 'LineString',
        coordinates: coordinates
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
      {#if tripPorts.length}
        <button class="clear-trip-btn" on:click={clearTrip}>Clear All</button>
      {/if}
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
</style>


