<template>
    <div class="tasks-page">
        <md-tabs md-alignment="fixed" md-sync-route>
            <template slot="md-tab" slot-scope="{tab}">
                <span class="tab-label">
                    <md-icon class="tab-icon" v-if="tab.data.icon">{{tab.data.icon}}</md-icon> {{tab.label}}
                </span>
            </template>

            <md-tab id="tab-map" class="tab-one" md-label="Map" :md-template-data="{icon: 'map'}"
                    to="/find/tabOne">
                <GmapMap
                        Map
                        :center="position"
                        :zoom="16"
                        :options="{
                    disableDefaultUI: true,
                    draggable: false,
                    clickableIcons: false,
                    streetViewControl: true
                }"
                        map-type-id="terrain"
                        class="map"
                        ref="mapRef"
                >
                </GmapMap>
            </md-tab>
            <md-tab id="tab-two" md-label="Street View" :md-template-data="{icon: 'streetview'}"
                    to="/find/tabTwo">
                <div ref="pano" class="map"></div>
            </md-tab>
        </md-tabs>
        <hr class="tab-divider"/>
        <md-snackbar md-position="center" :md-duration="4000" :md-active.sync="showSnackbar" md-persistent>
            <span style="width: 100%; text-align: center;">Please stay within the allocated area!</span>
        </md-snackbar>
    </div>
</template>
<script>
    import {gmapApi} from "vue2-google-maps";
    import {mapGetters, mapMutations} from 'vuex'

    export default {
        name: 'Find',
        components: {},
        data: function () {
            return {
                position: {lat: 0, lng: 0},
                panoPosition: {},
                previousPosition: this.position,
                polygons: [],
                outerCoords: [{lat: 90, lng: -90}, {lat: 90, lng: 90}, {lat: 90, lng: 180}, {lat: 90, lng: -90},
                    {lat: -90, lng: -90}, {lat: -90, lng: 180}, {lat: -90, lng: 90}, {lat: -90, lng: -90}],
                innerCoords: [],
                showSnackbar: false
            }
        },
        computed: {
            google: gmapApi,
            location: function () {
                return this.getLocation()
            }
        },
        watch: {
            location: {
                immediate: true,
                handler: function() {
                    if (this.location !== null) {
                        this.position = this.getCoordinates();
                        this.$nextTick(() => {
                            this.initMap();
                        });
                    }
                }
            }
        },
        mounted: function () {
            this.$nextTick(() => {
                if (this.$route.path === "/find/") {
                    this.$router.push("/find/tabOne")
                }
            });
        },
        methods: {
            ...mapGetters(["getLocation", "getCoordinates", "getPosition"]),
            ...mapMutations(["setPosition"]),
            initMap: function () {
                this.$refs.mapRef.$mapPromise.then((map) => {
                    var bounds = map.getBounds();
                    var latBounds = bounds.Ya;
                    var lngBounds = bounds.Ua;
                    var latDiff = Math.abs(latBounds.i - latBounds.j);
                    var lngDiff = Math.abs(lngBounds.i - lngBounds.j);

                    var quadrants = [{lat: latBounds.i, lng: lngBounds.i},
                        {lat: latBounds.i + latDiff / 2, lng: lngBounds.i},
                        {lat: latBounds.i, lng: lngBounds.i + lngDiff / 2},
                        {lat: latBounds.i + latDiff / 2, lng: lngBounds.i + lngDiff / 2}];
                    for (var i = 0; i < 2; i++) {
                        for (var j = i; j < i + 2; j++) {
                            if (i + j < 1 || Math.random() + Math.random() >= 1) {
                                var boxLatDiff = latDiff / 10;
                                var boxLngDiff = lngDiff / 10;

                                var latRatio = Math.random();
                                var lngRatio = Math.random();
                                var startLat = Math.min(quadrants[i + j].lat + ((latDiff / 2) * latRatio),
                                    quadrants[i + j].lat + (latDiff / 2) - boxLatDiff);
                                var startLng = Math.min(quadrants[i + j].lng + ((lngDiff / 2) * lngRatio),
                                    quadrants[i + j].lng + (lngDiff / 2) - boxLngDiff);

                                this.innerCoords[this.innerCoords.length] = [{lat: startLat, lng: startLng},
                                    {lat: startLat + boxLatDiff, lng: startLng},
                                    {lat: startLat + boxLatDiff, lng: startLng + boxLngDiff},
                                    {lat: startLat, lng: startLng + boxLngDiff},
                                    {lat: startLat, lng: startLng}];

                                this.polygons[this.polygons.length] = new this.google.maps.Polygon({
                                    paths: this.innerCoords[this.innerCoords.length - 1],
                                    strokeColor: '#0000FF',
                                    strokeOpacity: 0.3,
                                    strokeWeight: 1,
                                    map: map,
                                    fillOpacity: 0.0
                                });

                                if (i + j === 0) {
                                    this.panoPosition = {lat: startLat + (boxLatDiff / 2),
                                        lng: startLng + (boxLngDiff / 2)}
                                }
                            }
                        }
                    }

                    var paths = [this.outerCoords];
                    this.innerCoords.forEach((o) => {
                        paths.push(o.reverse())
                    });

                    new this.google.maps.Polygon({
                        paths: paths,
                        strokeWeight: 0,
                        map: map,
                        fillColor: '#FF0000',
                        fillOpacity: 0.35
                    });

                    var pano = new this.google.maps.StreetViewPanorama(this.$refs.pano, {
                        position: this.panoPosition,
                        source: this.google.maps.StreetViewSource.OUTDOOR
                    });
                    map.setStreetView(pano);

                    pano.addListener('position_changed', () => {
                        var pos = pano.getPosition();

                        if (pos !== this.previousPosition) {
                            var isInBounds = false;
                            for (var i = 0; i < this.polygons.length; i++) {
                                if (this.google.maps.geometry.poly.containsLocation(pos, this.polygons[i])) {
                                    isInBounds = true;
                                    break;
                                }
                            }

                            if (!isInBounds) {
                                this.showSnackbar = true;
                                pano.setPosition(this.previousPosition);
                            } else {
                                this.previousPosition = pos;
                            }
                        }
                    });
                });
            }
        }
    };
</script>
<style>
    .tasks-page {
        width: 100%;
        height: calc(100vh - 48px);
    }

    .tab-divider {
        position: absolute;
        width: 100vw;
        margin-top: -1px;
        top: 96px;
    }

    .tab-icon {
        margin-top: -5px;
    }

    .tab-label {
        font-size: 16px !important;
    }

    .md-tab {
        height: calc(100vh - 96px);
        max-height: 100%;
    }

    .tab-one {
        overflow-y: hidden;
    }

    .md-content.md-tabs-content {
        padding-bottom: 8px;
        height: calc(100vh - 96px) !important;
    }

    .md-button.md-tab-nav-button {
        max-width: 100vh !important;
    }

    .md-button.md-tab-nav-button.md-active {
        color: var(--forest-green) !important;
    }

    .md-tabs-indicator {
        background-color: var(--forest-green) !important;
    }

    .md-tabs .md-tabs-navigation .md-button .md-icon {
        color: var(--forest-green) !important;
    }

    .map {
        margin: -16px 0 0 -16px;
        height: calc(100% + 32px);
        width: calc(100% + 32px);
    }
</style>