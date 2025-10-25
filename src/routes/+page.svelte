<script>
	import { onMount, onDestroy } from 'svelte';

	import '$lib/leaflet/dist/leaflet.css';
	import '$lib/leaflet-routing-machine/dist/leaflet-routing-machine.css';

	let mapContainer;
	let map;
	let L;

	// --- State Management ---
	let state = 'initial';

	let userLocation; // To store [lat, lng]
	let destination; // To store L.latLng object

	// --- References ---
	let userMarker;
	let routingControl;
	let watchId;
	let destinationMarker;

	onMount(async () => {
		L = (await import('leaflet')).default;
		await import('leaflet-routing-machine');

		// --- Geolocation (watchPosition) ---
		if (navigator.geolocation) {
			watchId = navigator.geolocation.watchPosition(
				(position) => {
					const newLatLng = [position.coords.latitude, position.coords.longitude];
					userLocation = newLatLng;

					if (!map) {
						// First time: Init map and create blue dot
						initMap(newLatLng, 15);

						const blueDotIcon = L.divIcon({
							className: 'user-location-dot',
							html: '<div class="pulse"></div>',
							iconSize: [20, 20],
							iconAnchor: [10, 10]
						});

						userMarker = L.marker(newLatLng, { icon: blueDotIcon }).addTo(map);
					} else {
						// Subsequent updates: Move dot and update route
						userMarker.setLatLng(newLatLng);

						if (state === 'routing' && routingControl) {
							routingControl.setWaypoints([L.latLng(userLocation), destination]);
						}
					}
				},
				(err) => {
					console.warn(`Could not get location: ${err.message}`);
					if (!map) {
						userLocation = [51.505, -0.09]; // London Fallback
						initMap(userLocation, 10);
						L.marker(userLocation).addTo(map).bindPopup('Fallback Location');
					}
				},
				{
					enableHighAccuracy: true,
					timeout: 5000,
					maximumAge: 0
				}
			);
		} else {
			console.warn('Geolocation is not supported by this browser.');
			userLocation = [51.505, -0.09];
			initMap(userLocation, 10);
		}
	});

	// Clean up the watcher
	onDestroy(() => {
		if (watchId) {
			navigator.geolocation.clearWatch(watchId);
		}
	});

	// Initialize the map
	function initMap(centerCoords, zoom) {
		map = L.map(mapContainer, {
			zoomControl: false
		}).setView(centerCoords, zoom);

		L.control.zoom({ position: 'topright' }).addTo(map);

		L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
			attribution:
				'&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
		}).addTo(map);

		// --- NEW ---
		// Call the function to load and display data pins 📍
		loadDataPins();
	}

	// --- NEW: Function to load and plot data from CSV ---
	async function loadDataPins() {
		if (!map || !L) return; // Make sure map and Leaflet are ready

		const csvUrl =
			'https://docs.google.com/spreadsheets/d/e/2PACX-1vSo00v_Jl9XeD5piLhChE-vW0boIHAPua61YTltj-u7SHA8CTYz08ws8uEKerWe3Q-rhE0vnb8Uo706/pub?gid=0&single=true&output=csv';

		try {
			const response = await fetch(csvUrl);
			if (!response.ok) {
				throw new Error(`Failed to fetch: ${response.statusText}`);
			}
			const csvText = await response.text();

			// --- Simple CSV Parsing ---
			const rows = csvText
				.split('\n')
				.map((row) => row.trim())
				.filter(Boolean); // Split by line, trim, remove empty lines

			if (rows.length < 2) {
				console.warn('No data rows found in CSV.');
				return; // No data or only header
			}

			const headers = rows.shift().split(','); // Get header row

			// Find column indices
			const latIndex = headers.indexOf('latitude');
			const lngIndex = headers.indexOf('longitude');
			const nameIndex = headers.indexOf('accident');
			const descIndex = headers.indexOf('location');

			if (latIndex === -1 || lngIndex === -1) {
				console.error("CSV must have 'latitude' and 'longitude' columns.");
				return;
			}

			// Create a layer group to hold all the new pins
			const pinLayerGroup = L.layerGroup().addTo(map);

			// --- Loop and Create Markers ---
			for (const row of rows) {
				// Note: This is a simple parser. It will break if your 'description'
				// or 'name' fields contain a comma.
				const values = row.split(',');

				const lat = parseFloat(values[latIndex]);
				const lng = parseFloat(values[lngIndex]);

				// Check for valid coordinates
				if (!isNaN(lat) && !isNaN(lng)) {
					const name = values[nameIndex] || 'No Name';
					const description = values[descIndex] || 'No Description';

					// Create the popup content
					const popupContent = `<b>${name}</b><br>${description}`;

					// Create marker, add to group, and bind the popup
					L.marker([lat, lng]).addTo(pinLayerGroup).bindPopup(popupContent);
				}
			}

			console.log('Successfully loaded all data pins! 🥳');
		} catch (error) {
			console.error('Failed to load data pins:', error);
		}
	}

	// --- Button Handlers ---

	function handleChooseDestination() {
		state = 'selecting';

		map.once('click', (e) => {
			destination = e.latlng;
			handleSubmit();
		});
	}

	function handleSubmit() {
		if (!userLocation || !destination) return;

		// Clean up old marker if it exists from a previous route
		if (destinationMarker) {
			map.removeLayer(destinationMarker);
		}

		routingControl = L.Routing.control({
			waypoints: [L.latLng(userLocation[0], userLocation[1]), destination],
			routeWhileDragging: true,
			show: false, // Keep hiding the text panel
			createMarker: function (i, waypoint, n) {
				if (i === 0) {
					return null;
				}
				if (i === n - 1) {
					destinationMarker = L.marker(waypoint.latLng);
					return destinationMarker;
				}
				return L.marker(waypoint.latLng);
			}
		}).addTo(map);

		state = 'routing';
	}

	function handleCancel() {
		// Remove the routing line
		if (routingControl) {
			map.removeControl(routingControl);
			routingControl = null;
		}

		// NEW: Also remove the destination marker
		if (destinationMarker) {
			map.removeLayer(destinationMarker);
			destinationMarker = null;
		}

		destination = null;
		state = 'initial';
	}

	function panToUser() {
		if (map && userLocation) {
			// Use setView for a smooth pan and zoom
			map.setView(userLocation, 15);
		}
	}
</script>

<main>
	<div class="map-container" bind:this={mapContainer}></div>

	<div class="ui-container">
		<button on:click={panToUser} class="btn-location" title="My Location">
			<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="currentColor">
				<path
					d="M12 8c-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4-1.79-4-4-4zm8.94 3c-.46-4.17-3.77-7.48-7.94-7.94V1h-2v2.06C6.83 3.52 3.52 6.83 3.06 11H1v2h2.06c.46 4.17 3.77 7.48 7.94 7.94V23h2v-2.06c4.17-.46 7.48-3.77 7.94-7.94H23v-2h-2.06zM12 19c-3.87 0-7-3.13-7-7s3.13-7 7-7 7 3.13 7 7-3.13 7-7 7z"
				/>
			</svg>
		</button>

		<div class="main-button-wrapper">
			{#if state === 'routing'}
				<button on:click={handleCancel} class="btn btn-cancel"> ยกเลิก </button>
			{:else}
				<button
					on:click={handleChooseDestination}
					class="btn btn-choose"
					disabled={state === 'selecting'}
				>
					{#if state === 'selecting'}
						Click on the map...
					{:else}
						เลือกจุดหมาย
					{/if}
				</button>
			{/if}
		</div>

		<div class="location-placeholder"></div>
	</div>
</main>

<style>
	main {
		position: relative;
		width: 100%;
		height: 100vh;
	}

	.map-container {
		height: 100%;
		width: 100%;
	}

	.ui-container {
		/* This is now a flex container to align the 3 items */
		position: absolute;
		bottom: 0;
		left: 0;
		right: 0;
		z-index: 1000;

		display: flex;
		justify-content: space-between; /* [Left] [Center] [Right] */
		align-items: center;

		padding: 20px;
		box-sizing: border-box;
		/* Clicks should "pass through" the container, only buttons are clickable */
		pointer-events: none;
	}

	/* --- Main Button (Center) --- */
	.main-button-wrapper {
		/* This wrapper will hold the main button */
		width: 100%;
		max-width: 350px;
		margin: 0 10px; /* Add some space between buttons */
		pointer-events: auto; /* This element is clickable */
	}

	.btn {
		width: 100%; /* Button fills its wrapper */
		padding: 15px;
		font-size: 1.2rem;
		font-weight: bold;
		color: white;
		border: none;
		border-radius: 30px;
		cursor: pointer;
		box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
		transition: background-color 0.2s;
	}

	/* ... (rest of .btn styles are unchanged) ... */
	.btn-choose {
		background-color: #2998ff;
	}
	.btn-choose:hover {
		background-color: #1a7fdb;
	}
	.btn-choose:disabled {
		background-color: #999;
		cursor: not-allowed;
	}
	.btn-cancel {
		background-color: #e53935;
	}
	.btn-cancel:hover {
		background-color: #c62828;
	}

	/* --- NEW: Location Button (Left) --- */
	.btn-location {
		pointer-events: auto; /* This element is clickable */
		background-color: white;
		color: #333;
		width: 54px;
		height: 54px;
		flex-shrink: 0; /* Don't let it shrink */
		border: none;
		border-radius: 50%; /* Make it round */
		box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
		cursor: pointer;

		/* Center the SVG icon inside */
		display: flex;
		justify-content: center;
		align-items: center;
		padding: 0;
	}
	.btn-location:hover {
		background-color: #f4f4f4;
	}
	.btn-location svg {
		width: 24px;
		height: 24px;
	}

	/* --- NEW: Placeholder (Right) --- */
	.location-placeholder {
		/* This must have the same width as the .btn-location */
		width: 54px;
		flex-shrink: 0; /* Don't let it shrink */
	}

	/* --- Blue Dot Styles (Unchanged) --- */
	:global(.user-location-dot) {
		background-color: #1a73e8;
		border-radius: 50%;
		border: 2px solid white;
		box-shadow: 0 0 5px rgba(0, 0, 0, 0.5);
		transform: translate(-50%, -50%);
	}
	:global(.user-location-dot .pulse) {
		width: 100%;
		height: 100%;
		border-radius: 50%;
		background-color: #1a73e8;
		opacity: 0.4;
		animation: pulse 2s infinite ease-out;
	}
	@keyframes pulse {
		0% {
			transform: scale(1);
			opacity: 0.4;
		}
		100% {
			transform: scale(3);
			opacity: 0;
		}
	}
	:global(.leaflet-routing-container) {
		display: none;
	}
</style>
