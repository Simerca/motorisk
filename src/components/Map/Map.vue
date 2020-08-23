<template>
<div>
    <div id="map"></div>

<div class="map-overlay top">
        <div class="map-overlay-inner" v-if="synthese">
           
           Distance : {{synthese.road.routes[0].distance}} km <br>
           Durée : {{ Math.round(synthese.road.routes[0].duration / 60) }} min 
           
        </div>
    </div>
</div>
</template>

<script>
/* eslint-disable */
import mapboxgl from 'mapbox-gl';
import MapboxGeocoder from '@mapbox/mapbox-gl-geocoder';
import '@mapbox/mapbox-gl-geocoder/dist/mapbox-gl-geocoder.css';
import * as L from '@mapbox/polyline';
import axios from 'axios';
import moment from 'moment';
import 'moment-timezone';
import {
    Plugins
} from '@capacitor/core';

const {
    Geolocation
} = Plugins;

export default {
    data() {
        return {
            map: null,
            userCoord: null,
            layers: [],
            sliderValue: 100,
            markers: [],
            synthese:false,
        }
    },
    methods: {
        initMap() {
            mapboxgl.accessToken = 'pk.eyJ1IjoicG1jb20iLCJhIjoiY2lxajgxbHF3MDBjZWkwbWMza2pwYzFoayJ9.a1ywamqhc2BKAdxMdgcBpA';
            this.map = new mapboxgl.Map({
                container: 'map', // container id
                style: 'mapbox://styles/mapbox/dark-v10',
                center: [2, 47], // starting position
                zoom: 12 // starting zoom
            });

            // Add the control to the map.
            this.map.on('load', function () {

                let geocoder = new MapboxGeocoder({
                    accessToken: mapboxgl.accessToken,
                    mapboxgl: mapboxgl,
                })
                this.map.addControl(
                    geocoder
                );

                geocoder.on('result', function (results) {
                    this.markers.forEach(element => {
                        element.remove();
                    });
                    this.layers.forEach(element => {
                        console.log(element);
                        if (this.map.getLayer(element)) {
                            if (this.map.getSource(element)) {
                                console.log('get is done');
                                this.map.removeLayer(element);
                                this.map.removeSource(element);
                            }
                        }
                    })
                    console.log('finish');
                    this.howToNavigateTo(results.result.geometry.coordinates);

                }.bind(this))

                this.initRainViewer();

                this.map.resize();

            }.bind(this));

        },
        changeOpacity(e) {
            this.map.setPaintProperty(
                'radar-tiles',
                'raster-opacity',
                parseInt(e.target.value, 10) / 100
            );
        },
        initRainViewer() {
            axios.get('https://api.rainviewer.com/public/maps.json').then(res => {
                console.log('map json is');
                console.log(res.data);
                let ts = [];
                res.data.forEach(element => {
                    ts.push(moment(element * 1000).format('YYYY-MM-DD HH:mm'))
                });
                let day = ts[6];
                day = moment(day).unix();
                this.map.addSource('rainviewer', {
                    "type": 'raster',
                    "tiles": ['https://tilecache.rainviewer.com/v2/radar/' + day + '/256/{z}/{x}/{y}/2/1_1.png'],
                    "tileSize": 256
                });
                this.map.addLayer({
                    "id": "radar-tiles",
                    "type": "raster",
                    "source": "rainviewer",
                    "minzoom": 0,
                    "maxzoom": 22
                });
                this.map.setPaintProperty(
                    'radar-tiles',
                    'raster-opacity',
                    parseInt(50,10) / 100
                );
            })

        },
        async getCurrentPosition() {
            this.userCoord = await Geolocation.getCurrentPosition();
            let marker = new mapboxgl.Marker()
                .setLngLat([this.userCoord.coords.longitude, this.userCoord.coords.latitude])
                .addTo(this.map);
            this.map.flyTo({
                center: [
                    this.userCoord.coords.longitude,
                    this.userCoord.coords.latitude
                ],
                essential: true // this animation is considered essential with respect to prefers-reduced-motion
            })

        },
        watchPosition() {
            const wait = Geolocation.watchPosition({}, (position, err) => {})
        },
        async howToNavigateTo(coords) {

            let start = {
                lat: this.userCoord.coords.latitude,
                lon: this.userCoord.coords.longitude
            }
            let end = {
                lat: coords[1],
                lon: coords[0]
            }
            let url = `https://api.mapbox.com/directions/v5/mapbox/driving/${start.lon},${start.lat};${end.lon},${end.lat}?access_token=pk.eyJ1IjoicG1jb20iLCJhIjoiY2lxajgxbHF3MDBjZWkwbWMza2pwYzFoayJ9.a1ywamqhc2BKAdxMdgcBpA`
            axios.get(url).then(resp => {
                console.log('Driving');
                this.synthese = {};
                console.log(resp);
                this.synthese.road = resp.data;
                let coordinates = resp.data.routes[0].geometry;
                //let line = L.dec(coordinates, {color: 'red'}).addTo(this.map);
                let line = L.decode(coordinates);
                var flipped = [];
                for (var i = 0; i < line.length; i++) {
                    flipped.push(line[i].slice().reverse());
                }

                let risks = {};
                for (var i = 0; i < line.length; i += 1) {
                    let points = line[i];
                    let nextPoint = points;
                    if (i < line.length - 1) {
                        nextPoint = line[i + 1];
                    }
                    let b = i;
                    this.synthese.weather = [];
                    axios.get(`https://api.openweathermap.org/data/2.5/onecall?lat=${line[i][0]}&lon=${line[i][1]}&units=metric&appid=1c7695c1f16906559acd7fc9ef1eaab0`).then(res => {
                        console.log('weather')
                        console.log(res.data)
                        this.synthese.weather.push(res.data.hourly[3]);
                        risks = {
                            weather: res.data,
                            points: points
                        };
                        let geojson = {
                            'type': 'FeatureCollection',
                            'features': []
                        };
                        let color = "red";
                        // if(res.data.current.weather[0].main == 'Clouds'){

                        console.log(res.data.hourly);
                        if (res.data.hourly[3].rain) {
                            if (res.data.hourly[3].rain['1h'] > .7) {
                                color = '#F7455D'
                            } else if (res.data.hourly[3].rain['1h'] > .1) {
                                color = '#ed8200'
                            } else {
                                color = '#008211'
                            }
                        } else {
                            color = '#008211'

                        }
                        let linestring = {
                            'type': 'Feature',
                            'properties': {
                                'color': color // blue
                            },
                            'geometry': {
                                'type': 'LineString',
                                'coordinates': [
                                    [points[1], points[0]],
                                    [nextPoint[1], nextPoint[0]],
                                ]
                            }
                        };
                        geojson.features.push(linestring);
                        this.map.addSource(`route${b}`, {
                            'type': 'geojson',
                            'data': {
                                'type': 'Feature',
                                'properties': {},
                                'geometry': {
                                    'type': 'LineString',
                                    'coordinates': []
                                }
                            }
                        });

                        this.map.addLayer({
                            'id': `route${b}`,
                            'type': 'line',
                            'source': `route${b}`,
                            'layout': {
                                'line-join': 'round',
                                'line-cap': 'round'
                            },
                            'paint': {
                                'line-width': 8,
                                'line-color': ['get', 'color']
                            }
                        });
                        this.layers.push(`route${b}`);
                        this.map.getSource(`route${b}`).setData(geojson);
                        this.setRisks(risks);
                    });
                    console.log(this.synthese)
                }

            })
        },
        async getWeatherForecastOnPoints(points) {

        },
        setRisks(element) {
            let popupText = '';
            if (element.weather.current.rain) {
                popupText = 'Precip. : ' + element.weather.current.rain['1h'] + 'mm'
            } else {
                popupText = 'Pas ou faible precip.'
            }
            let popup = new mapboxgl.Popup().setText(popupText);
            // create a DOM element for the marker
            var el = document.createElement('div');
            el.className = 'marker';
            el.style.backgroundImage =
                `url(https://openweathermap.org/img/wn/${element.weather.current.weather[0].icon}@2x.png)`;
            el.style.backgroundSize = 'cover';
            el.style.width = '80px';
            el.style.height = '80px';

            let marker = new mapboxgl.Marker(el)
                .setLngLat([element.points[1], element.points[0]])
                .setPopup(popup)
                .addTo(this.map);

            this.markers.push(marker);
        }
    },
    mounted() {
        this.initMap();
        this.getCurrentPosition();
    }
}
</script>

<style>
#map {
    height: 100vh;
}

.marker {
    display: block;
    border: none;
    border-radius: 50%;
    cursor: pointer;
    padding: 0;
    background-size: cover;
    /* background-color: #7878785c; */
}

.map-overlay {
    font: bold 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
    position: absolute;
    z-index: 100000;
    width: 100%;
    bottom: 0;
    left: 0;
}

.map-overlay .map-overlay-inner {
    background-color: #fff;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
    border-radius: 3px;
    padding: 10px;
    margin-bottom: 10px;
}

.map-overlay label {
    display: block;
}

.map-overlay input {
    background-color: transparent;
    display: inline-block;
    width: 100%;
    position: relative;
    margin: 0;
    cursor: ew-resize;
}

.mapboxgl-ctrl-top-right {
    width: 100%;
}

.mapboxgl-ctrl-geocoder.mapboxgl-ctrl {
    margin: 0;
}
</style>
