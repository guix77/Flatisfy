<template>
    <div>
        <template v-if="isLoading">
            <p>{{ $t("common.loading") }}</p>
        </template>
        <div class="grid" v-else-if="flat && timeToPlaces">
            <div class="left-panel">
                <h2>
                    (<!--
                        --><router-link :to="{ name: 'status', params: { status: flat.status }}"><!--
                            -->{{ flat.status ? capitalize($t("status." + flat.status)) : '' }}<!--
                        --></router-link><!--
                    -->) {{ flat.title }} [{{ flat.id.split("@")[1] }}]
                    <span class="expired">{{ flat.is_expired ? '[' + $t('common.expired') + ']' : '' }}</span>
                </h2>
                <div class="grid">
                    <div class="left-panel">
                        <p>
                            {{ flat.cost }} {{ flat.currency }}
                            <template v-if="flat.utilities === 'included'">
                                {{ $t("flatsDetails.utilities_included") }}
                            </template>
                            <template v-else-if="flat.utilities === 'excluded'">
                                {{ $t("flatsDetails.utilities_excluded") }}
                            </template>
                        </p>
                    </div>
                    <p class="right-panel right">
                        {{ flat.area ? flat.area : '?' }} m<sup>2</sup>,
                        {{ flat.rooms ? flat.rooms : '?' }} {{ $tc("flatsDetails.rooms", flat.rooms) }} /
                        {{ flat.bedrooms ? flat.bedrooms : '?' }} {{ $tc("flatsDetails.bedrooms", flat.bedrooms) }}
                    </p>
                </div>
                <div>
                    <template v-if="flat.photos && flat.photos.length > 0">
                        <Slider :photos="flat.photos"></Slider>
                    </template>
                </div>
                <div>
                    <h3>{{ $t("flatsDetails.Description") }}</h3>
                    <p>{{ flat.text }}</p>
                    <p class="right">{{ flat.location }}</p>
                    <p>{{ $t("flatsDetails.First_posted") }} {{ flat.date ? flat.date.fromNow() : '?' }}.</p>
                </div>
                <div>
                    <h3>{{ $t("flatsDetails.Details") }}</h3>
                    <table>
                        <tr v-for="(value, key) in flat.details">
                            <th>{{ key }}</th>
                            <td>{{ value }}</td>
                        </tr>
                    </table>
                </div>
                <div>
                    <h3>{{ $t("flatsDetails.Metadata") }}</h3>
                    <table>
                        <tr>
                            <th>
                                {{ $t("flatsDetails.postal_code") }}
                            </th>
                            <td>
                                <template v-if="flat.flatisfy_postal_code.postal_code">
                                    {{ flat.flatisfy_postal_code.name }} ({{ flat.flatisfy_postal_code.postal_code }})
                                </template>
                                <template v-else>
                                    ?
                                </template>
                            </td>
                        </tr>

                        <tr>
                            <th>
                                {{ $t("flatsDetails.nearby_stations") }}
                            </th>
                            <td>
                                <template v-if="displayedStations">
                                    {{ displayedStations }}
                                </template>
                                <template v-else>
                                    ?
                                </template>
                            </td>
                        </tr>
                        <tr>
                            <th>
                                {{ $t("flatsDetails.Times_to") }}
                            </th>
                            <td>
                                <template v-if="Object.keys(flat.flatisfy_time_to).length">
                                    <ul class="time_to_list">
                                        <li v-for="(time_to, place) in flat.flatisfy_time_to" :key="place">
                                            {{ place }}: {{ humanizeTimeTo(time_to["time"]) }}
                                        </li>
                                    </ul>
                                </template>
                                <template v-else>
                                    ?
                                </template>
                            </td>
                        </tr>
                        <tr>
                            <th>
                                {{ $t("flatsDetails.SquareMeterCost") }}
                            </th>
                            <td>
                                {{ flat.sqCost }} {{ flat.currency }}
                            </td>
                        </tr>
                    </table>
                </div>
                <div>
                    <h3>{{ $t("flatsDetails.Location") }}</h3>

                    <FlatsMap :flats="flatMarkers" :places="timeToPlaces" :journeys="journeys"></FlatsMap>
                </div>
                <div>
                    <h3>{{ $t("flatsDetails.Notes") }}</h3>

                    <form v-on:submit="updateFlatNotes">
                        <textarea ref="notesTextarea" rows="10">{{ flat.notes }}</textarea>
                        <p class="right"><input type="submit" :value="$t('flatsDetails.Save')"/></p>
                    </form>
                </div>
            </div>

            <div class="right-panel">
                <h3>{{ $t("flatsDetails.Contact") }}</h3>
                <div class="contact">
                    <p>
                        <template v-if="flat.phone">
                            <template v-for="phoneNumber in flat.phone.split(',')">
                                <a :href="'tel:+33' + normalizePhoneNumber(phoneNumber)">{{ phoneNumber }}</a><br/>
                            </template>
                        </template>
                        <template v-else>
                            {{ $t("flatsDetails.no_phone_found") }}
                        </template>
                    </p>
                    <p>{{ $tc("common.Original_post", 42) }}
                        <ul>
                            <li v-for="(url, index) in flat.urls">
                                <a :href="url">
                                    {{ $tc("common.Original_post", 1) }} {{ index + 1 }}
                                    <i class="fa fa-external-link" aria-hidden="true"></i>
                                </a>
                            </li>
                        </ul>
                    </p>
                </div>

                <h3>{{ $t("flatsDetails.Visit") }}</h3>
                <div class="visit">
                    <flat-pickr
                        :value="flatpickrValue"
                        :config="flatpickrConfig"
                        :placeholder="$t('flatsDetails.setDateOfVisit')"
                    />
                </div>

                <h3>{{ $t("common.Actions") }}</h3>

                <nav>
                    <ul>
                        <template v-if="flat.status !== 'user_deleted'">
                            <li ref="notationButton">
                                <template v-for="n in range(notation)">
                                    <button class="btnIcon" v-on:mouseover="handleNotationHover(n)" v-on:mouseout="handleNotationOut()" v-on:click="updateFlatNotation(n)">
                                        <i class="fa fa-star" aria-hidden="true"></i>
                                    </button>
                                </template>
                                <template v-for="n in range(5 - notation)">
                                    <button class="btnIcon" v-on:mouseover="handleNotationHover(notation + n)" v-on:mouseout="handleNotationOut()" v-on:click="updateFlatNotation(notation + n)">
                                        <i class="fa fa-star-o" aria-hidden="true"></i>
                                    </button>
                                </template>
                            </li>
                            <li>
                                <button v-on:click="updateFlatStatus('user_deleted')" class="fullButton">
                                    <i class="fa fa-trash" aria-hidden="true"></i>
                                    {{ $t("common.Remove") }}
                                </button>
                            </li>
                        </template>
                        <template v-else>
                            <li>
                                <button v-on:click="updateFlatStatus('new')" class="fullButton">
                                    <i class="fa fa-undo" aria-hidden="true"></i>
                                    {{ $t("common.Restore") }}
                                </button>
                            </li>
                        </template>
                    </ul>
                </nav>
            </div>
        </div>
    </div>
</template>

<script>
import flatPickr from 'vue-flatpickr-component'
import moment from 'moment'
import 'font-awesome-webpack'
import 'flatpickr/dist/flatpickr.css'

import FlatsMap from '../components/flatsmap.vue'
import Slider from '../components/slider.vue'

import { capitalize, range } from '../tools'

export default {
    components: {
        FlatsMap,
        Slider,
        flatPickr
    },

    created () {
        document.title = this.title  // Set title

        // Fetch data when the component is created
        this.fetchData()

        // Scrolls to top when view is displayed
        window.scrollTo(0, 0)
    },

    watch: {
        // Fetch data again when the component is updated
        '$route': 'fetchData',
        title () {
            document.title = this.title
        }
    },

    data () {
        return {
            // TODO: Flatpickr locale
            'overloadNotation': null,
            'flatpickrConfig': {
                static: true,
                altFormat: 'h:i K, M j, Y',
                altInput: true,
                enableTime: true,
                onChange: selectedDates => this.updateFlatVisitDate(selectedDates.length > 0 ? selectedDates[0] : null)
            }
        }
    },

    computed: {
        isLoading () {
            return this.$store.getters.isLoading
        },
        title () {
            return 'Flatisfy - ' + this.$route.params.id
        },
        flatMarkers () {
            return this.$store.getters.flatsMarkers(this.$router, flat => flat.id === this.$route.params.id)
        },
        flat () {
            return this.$store.getters.flat(this.$route.params.id)
        },
        'flatpickrValue' () {
            if (this.flat && this.flat.visit_date) {
                return this.flat.visit_date.local().format()
            }
            return null
        },
        timeToPlaces () {
            return this.$store.getters.timeToPlaces(this.flat.flatisfy_constraint)
        },
        notation () {
            if (this.overloadNotation) {
                return this.overloadNotation
            }
            return this.flat.notation
        },
        journeys () {
            if (Object.keys(this.flat.flatisfy_time_to).length > 0) {
                const journeys = []
                for (const place in this.flat.flatisfy_time_to) {
                    this.flat.flatisfy_time_to[place].sections.forEach(
                        section => journeys.push({
                            geojson: section.geojson,
                            options: {
                                color: section.color ? ('#' + section.color) : '#2196f3',
                                dashArray: section.color ? 'none' : '2, 10'
                            }
                        })
                    )
                }
                return journeys
            }
            return []
        },
        displayedStations () {
            if (this.flat.flatisfy_stations.length > 0) {
                const stationsNames = this.flat.flatisfy_stations.map(station => station.name)
                return stationsNames.join(', ')
            } else {
                return null
            }
        }
    },

    methods: {
        fetchData () {
            this.$store.dispatch('getFlat', { flatId: this.$route.params.id })
            this.$store.dispatch('getAllTimeToPlaces')
        },

        updateFlatNotation (notation) {
            notation = notation + 1

            if (notation === this.flat.notation) {
                this.$store.dispatch('updateFlatNotation', { flatId: this.$route.params.id, newNotation: 0 })
                this.$store.dispatch('updateFlatStatus', { flatId: this.$route.params.id, newStatus: 'new' })
            } else {
                this.$store.dispatch('updateFlatNotation', { flatId: this.$route.params.id, newNotation: notation })
                this.$store.dispatch('updateFlatStatus', { flatId: this.$route.params.id, newStatus: 'followed' })
            }
        },

        updateFlatStatus (status) {
            this.$store.dispatch('updateFlatStatus', { flatId: this.$route.params.id, newStatus: status })
        },

        updateFlatNotes () {
            const notes = this.$refs.notesTextarea.value
            this.$store.dispatch(
                'updateFlatNotes',
                { flatId: this.$route.params.id, newNotes: notes }
            )
        },

        updateFlatVisitDate (date) {
            if (date) {
                date = moment(date).utc().format()
            }
            this.$store.dispatch(
                'updateFlatVisitDate',
                { flatId: this.$route.params.id, newVisitDate: date }
            )
        },

        humanizeTimeTo (time) {
            const minutes = Math.floor(time.as('minutes'))
            return minutes + ' ' + this.$tc('common.mins', minutes)
        },

        handleNotationHover (n) {
            this.overloadNotation = n + 1
        },

        handleNotationOut () {
            this.overloadNotation = null
        },

        normalizePhoneNumber (phoneNumber) {
            phoneNumber = phoneNumber.replace(/ /g, '')
            phoneNumber = phoneNumber.replace(/\./g, '')
            return phoneNumber
        },

        capitalize: capitalize,

        range: range
    }
}
</script>

<style scoped>
.expired {
    font-weight: bold;
    text-transform: uppercase;
}

@media screen and (min-width: 768px) {
    .grid {
        display: grid;
        grid-gap: 50px;
        grid-template-columns: 75fr 25fr;
    }

    .left-panel {
        grid-column: 1;
        grid-row: 1;
    }

    .right-panel {
        grid-column: 2;
        grid-row: 1;
    }
}

.left-panel textarea {
    width: 100%;
}

.right {
    text-align: right;
}

nav ul {
    list-style-type: none;
    padding-left: 1em;
}

.contact {
    padding-left: 1em;
}

.right-panel li {
    margin-bottom: 1em;
    margin-top: 1em;
}

button {
    cursor: pointer;
    width: 75%;
    padding: 0.3em;
    font-size: 0.9em;
}

table {
    table-layout: fixed;
}

td {
    word-wrap: break-word;
    word-break: break-all;
    white-space: normal;
}

.time_to_list {
    margin: 0;
    padding-left: 0;
    list-style-position: outside;
    list-style-type: none;
}

.btnIcon {
    border: none;
    width: auto;
    background-color: transparent;
}

@media screen and (max-width: 767px) {
    .right-panel nav {
        text-align: center;
    }

    .fullButton {
        width: 100%;
    }
}
</style>
