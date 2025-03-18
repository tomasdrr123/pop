# Sailing Trip Planner

An interactive web application for planning sailing trips by selecting ports on a map. Built with Svelte and Mapbox GL JS.

## Features

- Interactive map interface using Mapbox GL JS
- Search for ports and cities (both local and API-based search)
- Add ports to your trip plan
- Visualize your route on the map
- Responsive design with a modern UI

## Setup

1. Clone the repository:
```bash
git clone [your-repository-url]
cd sailing-app
```

2. Install dependencies:
```bash
npm install
```

3. Create a `.env` file in the root directory and add your Mapbox token:
```
MAPBOX_TOKEN=your_mapbox_token_here
```

4. Start the development server:
```bash
npm run dev
```

5. Open your browser and navigate to `http://localhost:5173`

## Environment Variables

- `MAPBOX_TOKEN`: Your Mapbox access token (required)

## Technologies Used

- Svelte
- Mapbox GL JS
- JavaScript/TypeScript
- CSS3

## License

MIT 