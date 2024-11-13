<!-- <script> tag includes JavaScript code -->
<script>
    import Icon from '@iconify/svelte'
    import * as turf from '@turf/turf'
    import { onMount } from 'svelte'
    import Geolocation from 'svelte-geolocation'

    import {
        Control,
        ControlButton,
        ControlGroup,
        DefaultMarker,
        GeoJSON,
        MapLibre,
        Marker,
        MarkerLayer,
        Popup,
        SymbolLayer,
    } from 'svelte-maplibre'

    // Button-related
    let disableGeolocation = false
    let disableTracking = true

    let prisms, ats

    // Map Variables
    let bounds

    /**
     * Declaring a function
     *
     * Functions declared in <script> can only be used in this component
     */

    // Geolocation API related
    const options = {
        enableHighAccuracy: true,
        timeout: Infinity, // milliseconds
        maximumAge: 0, // milliseconds, 0 disables cached positions
    }
    let getPosition = false
    let success = false
    let error = ''
    let position = {}
    let coords = []
    let watchPosition = false
    let watchedPosition = {}
    let watchedMarker = {}
    let accuracy = ''
    let heading = ''
    let speed = ''
    /**
     * $: indicates a reactive statement, meaning that this block of code is
     * executed whenever the variable used as the condition changes its value
     *
     * In this case: whenever success is set to true, a Position object
     * has been successfully obtained. Immediately update the relevant variables
     */
    $: if (success || error) {
        // reset the flag
        getPosition = false
    }

    $: if (success) {
        coords = [position.coords.longitude, position.coords.latitude]
    }

    /**
     * Trigger an action when getting close to a marker
     */
    $: if (watchedPosition.coords) { // this block is triggered when watchedPosition is updated
        // The tracked position in marker format
        watchedMarker = {
            lngLat: {
                lng: watchedPosition.coords.longitude,
                lat: watchedPosition.coords.latitude,
            },
        }
    }

    /**
     * Variables can be initialised without a value and populated later
     * WARNING: this can lead to errors if the variable is used before being
     * assigned a value
     */

    /**
     * onMount is executed immediately after the component is mounted, it can be
     * used to load large datasets or to execute code required after the page
     * has been loaded
     *
     * async/await indicate an asynchronous function (i.e., program is paused
     * when a line marked with await starts and resumes when it is resolved)
     *
     * Asset files (e.g., data files, images) can be put in static folder
     *
     * Another way to load data files is to use a URL to the file hosted
     * on a remote server. Try this by replacing 'melbourne.geojson' with
     * 'https://raw.githubusercontent.com/codeforgermany/click_that_hood/main/public/data/melbourne.geojson'
     */
    onMount(async () => {
        try {
            const prismsResponse = await fetch('neln_instruments_v2.json')
            prisms = await prismsResponse.json()
        }
        catch (error) {
            console.error('Error loading Prisms:', error)
            debugInfo = `Error: ${error.message}`
        }
        try {
            const atsResponse = await fetch('neln_ats_v2.json')
            ats = await atsResponse.json()
        }
        catch (error) {
            console.error('Error loading ATS Units:', error)
            debugInfo = `Error: ${error.message}`
        }
    })
</script>

<!-- Everything after <script> will be HTML for rendering -->

<!-- This section demonstrates how to get the current user location -->
<div class="flex flex-col h-[calc(100vh-80px)] w-full">
    <!-- This section demonstrates how to make a web map using MapLibre -->
    <!-- More basemap options -->
    <!-- "https://basemaps.cartocdn.com/gl/positron-gl-style/style.json" -->
    <!-- "https://tiles.basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json" -->
    <!-- "https://tiles.basemaps.cartocdn.com/gl/voyager-gl-style/style.json" -->
    <MapLibre
        center={[145.08921355211515, -37.70618385230228]}
        class="map flex-grow min-h-[500px]"
        standardControls
        style="https://tiles.basemaps.cartocdn.com/gl/voyager-gl-style/style.json"
        bind:bounds
        let:map
        zoom={15}
    >
        <!-- Custom control buttons -->
        <Control class="flex flex-col gap-y-2">
            <ControlGroup>
                <ControlButton
                    on:click={() => {
                        map.flyTo({
                            center: coords,
                            zoom: 4,
                        })
                    }}
                >
                    FLY
                </ControlButton>
                <ControlButton
                    on:click={() => {
                        map.fitBounds(turf.bbox(prisms))
                    }}
                >
                    SITE
                </ControlButton>
            </ControlGroup>
        </Control>

        {#if ats}
            {#each ats.features as feature (feature)}
                <Marker
                    lngLat={feature.geometry.coordinates}
                    class="place-items-center">
                    <span class="[text-shadow:_0_1px_0_rgb(255_255_255_/_40%)] tracking-tighter text-gray-500 text-xs font-bold">
                        {feature.properties.point}</span>
                    <Icon
                        icon="maki:observation-tower"
                        class="text-blue-600 w-8 h-8 "></Icon>
                </Marker>
            {/each}
        {/if}

        {#if prisms}
            <GeoJSON
                id="prisms"
                data={prisms}
                promoteId="point">
                <MarkerLayer
                    id="markerPrisms"
                    interactive
                    let:feature>
                    {#if Number(feature.properties.percent_visibility) > 50}
                        <Icon
                            icon="maki:circle"
                            class="text-green-600 w-3 h-3 "></Icon>
                    {:else}
                        <Icon
                            icon="maki:circle"
                            class="text-red-600 w-3 h-3 "></Icon>
                    {/if}
                    <Popup openOn="click">
                        {feature.properties.point} : {feature.properties.percent_visibility || 'NONE'}
                    </Popup>
                </MarkerLayer>
            </GeoJSON>

            <SymbolLayer
                source="prisms"
                layout={{
                    'text-field': [
                        'format',
                        ['get', 'point'],
                        { 'font-scale': 0.8 },
                        '\n',
                        { 'font-scale': 0.8 },
                    ],
                    'text-size': 12,
                    'text-offset': [0, -1],
                }}
                paint={{
                    'text-halo-color': 'white',
                    'text-halo-width': 3,
                }}
            />
        {/if}

        <!-- Display the watched position as a marker -->
        {#if watchedMarker.lngLat}
            <DefaultMarker lngLat={watchedMarker.lngLat}>
                <Popup offset={[0, -10]}>
                    <div class="text-lg font-bold">You</div>
                </Popup>
            </DefaultMarker>
        {/if}
    </MapLibre>

    <!-- grid, grid-cols-#, col-span-#, md:xxxx are some Tailwind utilities you can use for responsive design -->
    <div class="grid grid-cols-4">
        <div class="col-span-4 md:col-span-1 text-center">
            <!-- on:click declares what to do when the button is clicked -->
            <!-- In the HTML part, {} tells the framework to treat what's inside as code (variables or functions), instead of as strings -->
            <!-- () => {} is an arrow function, almost equivalent to function foo() {} -->
            <button
                class="btn btn-s sm:btn-sm md:btn-md lg:btn-lg btn-btn-primary"
                disabled={disableGeolocation}
                on:click={() => {
                    getPosition = true
                    disableGeolocation = true
                    disableTracking = false
                }}
            >
                Enable Position
            </button>

            <button
                class="btn btn-s sm:btn-sm md:btn-md lg:btn-lg btn-secondary"
                disabled={disableTracking}
                on:click={() => {
                    watchPosition = true
                }}
            >
                Track Location
            </button>

            <!-- <Geolocation> tag is used to access the Geolocation API -->
            <!-- {getPosition} is equivalent to getPosition={getPosition} -->
            <!-- bind:variable associates the parameter with the variable with the same name declared in <script> reactively -->
            <!-- let:variable creates a variable for use from the component's return -->
        </div>

        <!-- This section demonstrates how to get automatically updated user location -->
        <div class="col-span-4 md:col-span-1 text-center">

            <Geolocation
                getPosition={watchPosition}
                options={options}
                watch={true}
                bind:watchedPosition
                on:position={(e) => {
                    watchedPosition = e.detail
                }}
            />
            {#if watchedPosition.coords}
                <h1 class="font-bold text-left">Current Position: {Number.parseFloat(watchedPosition.coords.longitude)}, {Number.parseFloat(watchedPosition.coords.latitude)}</h1>
                <h1 class="font-bold text-left"> Accuracy: {Number.parseFloat(watchedPosition.coords.accuracy).toFixed(4)} m </h1>
                {#if Number.parseFloat(watchedPosition.coords.accuracy) > 100.0}
                    <p class="break-words text-left text-red-500">Device may be using WIFI/5G Network</p>
                {:else}
                    <p class="break-words text-left text-green-500">Device using GNSS</p>
                {/if}
                {#if watchedPosition.coords.altitude}
                    <h1 class="font-bold break-words text-left">Updated Altitude: {Number.parseFloat(watchedPosition.coords.altitude).toFixed(4)} | Altitude Accuracy: {Number.parseFloat(watchedPosition.coords.altitudeAccuracy).toFixed(4)} m </h1>
                {/if}
                {#if watchedPosition.coords.heading || watchedPosition.coords.speed}
                    <h1 class="font-bold break-words text-left">Heading: {Number.parseFloat(watchedPosition.coords.heading).toFixed(4)} | Speed: {Number.parseFloat(watchedPosition.coords.speed).toFixed(4)}</h1>
                    <!-- p class="break-words text-left">{JSON.stringify(watchedPosition)}</p -->
                {/if}
            {/if}

        </div>
        <div class="col-span-4 md:col-span-1 bg-white text-center">
            <!-- <Geolocation> tag is used to access the Geolocation API -->
            <!-- {getPosition} is equivalent to getPosition={getPosition} -->
            <!-- bind:variable associates the parameter with the variable with the same name declared in <script> reactively -->
            <!-- let:variable creates a variable for use from the component's return -->
            <Geolocation
                {getPosition}
                options={options}
                bind:position
                bind:accuracy
                bind:speed
                bind:heading
                bind:success
                bind:error
                let:loading
                let:notSupported
            >
                <!-- If-else block syntax -->
                {#if notSupported}
                    Your browser does not support the Geolocation API.
                {:else}
                    {#if loading}
                        Loading...
                    {/if}
                    {#if success}
                        Success!
                    {/if}
                    {#if error}
                        An error occurred. Error code {error.code}: {error.message}.
                    {/if}
                {/if}
            </Geolocation>

            <p class="break-words text-left">Initial Position: {coords}</p>
            <p class="break-words text-left">Accuracy: {Number.parseFloat(accuracy).toFixed(4)}m</p>
            {#if Number.parseFloat(accuracy) > 25.0}
                <p class="break-words text-left text-red-500">ONSTART: Device may be using WIFI/5G Network</p>
            {:else}
                <p class="break-words text-left text-green-500">ONSTART: Device using GNSS</p>
            {/if}
            {#if heading || speed}
                <p class="break-words text-left">Heading: {heading.toFixed(4)} | Speed: {speed.toFixed(4)} </p>
            {/if}
        </div>

    </div>

</div>
