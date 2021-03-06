<template>
    <div class="halite-visualizer">
        <div class="halite-visualizer-controls">
            <div class="btn-group" role="group">
                <button class="btn btn-default playbutton" v-if="playing" v-on:click="pause" title="Pause"><i class="fa fa-pause" aria-hidden="true"></i></button>
                <button class="btn btn-default playbutton" v-else v-on:click="play" title="Play"><i class="fa fa-play" aria-hidden="true"></i> </button>
                <button class="btn btn-default" v-if="!recording" v-on:click="captureVideo" title="Record video of game"><i class="fa fa-video-camera" aria-hidden="true"></i> </button>
                <span v-else class="recording-indicator">● REC</span>
            </div>

            Frame: <input type="text" :value="frame + '.' + Math.floor(time * 100)" />
        </div>

        <input type="range" min="0" :max="replay.frames.length - 1"
               class="halite-frame-scrubber"
               :value="frame"
               v-on:input="scrub" />

        <div class="row">
            <div class="halite-visualizer-canvas col-md-8">
                <div ref="visualizer_container"></div>
                <dl>
                    <dt>Map Size</dt>
                    <dd>{{ replay.width }}x{{ replay.height }}</dd>

                    <dt>Map Parameters</dt>
                    <dd>{{ replay.map_generator }}, seed: {{ replay.seed }}</dd>
                </dl>
                <small>Replay version {{ replay.version }}</small>
            </div>
            <div class="halite-visualizer-info col-md-6">
                <div v-for="(player_name, index) in player_names">
                    <h3 :style="'color: ' + colors[index]">
                        <a :style="'color: ' + colors[index]"
                           :href="player_name.user_link || '#'"
                           target="_blank">
                            <img v-if="player_name.profile_image_link"
                                 :src="player_name.profile_image_link"
                                 class="img-circle">
                            {{ player_name.bot_name }}
                        </a>
                    </h3>
                    <dl>
                        <template v-if="frame == replay.frames.length - 1">
                            <dt>Rank</dt>
                            <dd>{{ player_name.rank }}</dd>
                        </template>

                        <dt>Ship Count</dt>
                        <dd>{{ statistics[index].ships }} (total produced: {{ stats.frames[frame].players[index].totalShips }})</dd>

                        <dt>Planet Count</dt>
                        <dd>{{ statistics[index].planets }}</dd>

                        <dt>Total Attacks Made</dt>
                        <dd>{{ stats.frames[frame].players[index].totalAttacks }} (total targets: {{ stats.frames[frame].players[index].totalTargets }})</dd>

                        <dt>Health</dt>
                        <dd>{{ stats.frames[frame].players[index].totalHealths }}</dd>

                        <dt>Total Damages</dt>
                        <dd>{{ stats.frames[frame].players[index].totalDamages }}</dd>

                        <dt>Production Rates (per turn)</dt>
                        <dd>{{ stats.frames[frame].players[index].currentProductions }}</dd>

                        <dt>Attack (per turn)</dt>
                        <dd>{{ stats.frames[frame].players[index].currentAttacks }}</dd>
                    </dl>
                </div>
                <div v-if="selected.kind === 'planet' && selected_planet !== null">
                    <h3>Selected Planet</h3>
                    <dl>
                        <dt>ID</dt>
                        <dd>{{ selected.id }}</dd>

                        <dt>Location</dt>
                        <dd>({{ selected_planet.base.x }}, {{ selected_planet.base.y }}) radius={{ selected_planet.base.r }}</dd>

                        <dt>Health</dt>
                        <dd>{{ selected_planet.state.health }}/{{ selected_planet.base.health }}</dd>

                        <dt>Docking Spots</dt>
                        <dd>{{ selected_planet.base.docking_spots - selected_planet.state.docked_ships.length }}/{{ selected_planet.base.docking_spots }}</dd>

                        <dt>Remaining Production</dt>
                        <dd>{{ selected_planet.state.remaining_production }}/{{ selected_planet.base.production }}</dd>

                        <dt>Owner</dt>
                        <dd v-if="selected_planet.state.owner !== null">{{ replay.player_names[selected_planet.state.owner] }}</dd>
                        <dd v-else>(indepdendent)</dd>
                    </dl>
                </div>
                <div v-if="selected.kind === 'ship' && selected_ship !== null">
                    <h3>Selected Ship</h3>
                    <dl>
                        <dt>ID</dt>
                        <dd>{{ replay.player_names[selected.owner] }}&mdash;{{ selected.id }}</dd>

                        <dt>Location</dt>
                        <dd>({{ selected_ship.x.toFixed(4) }}, {{ selected_ship.y.toFixed(4) }})</dd>

                        <dt>Velocity</dt>
                        <dd>({{ selected_ship.vel_x.toFixed(4) }}, {{ selected_ship.vel_y.toFixed(4) }})</dd>

                        <dt>Health</dt>
                        <dd>{{ selected_ship.health }}</dd>
                    </dl>
                </div>
                <div v-if="selected.kind === 'point'">
                    <h3>Selected Coordinate</h3>
                    <dl>
                        <dt>Location</dt>
                        <dd>({{ selected.x.toFixed(4) }}, {{ selected.y.toFixed(4) }})</dd>
                    </dl>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
 import {HaliteVisualizer} from "./visualizer";
 import {PLAYER_COLORS} from "./assets";

 export default {
     name: "halite-hud",
     props: ["replay", "game", "makeUserLink", "getUserProfileImage"],
     data: function() {
         return {
             colors: PLAYER_COLORS.map((color) => {
                 color = color.toString(16);
                 while (color.length < 6) {
                     color = "0" + color;
                 }
                 return '#' + color;
             }),
             frame: 0,
             time: 0,
             playing: false,
             selected: {
                 kind: null,
                 owner: null,
                 id: null,
                 x: null,
                 y: null,
             },
             stats: null,
             player_names: null,
             encoding: false,
             recording: false,
             start_frame: 0,
         };
     },
     mounted: function() {
         console.log(this.replay);
         const visualizer = new HaliteVisualizer(this.replay);
         this.stats = visualizer.stats;

         visualizer.attach(this.$refs.visualizer_container);
         visualizer.onUpdate = () => {
             this.frame = visualizer.frame;
             this.time = visualizer.time;
         };
         visualizer.onSelect = (kind, args) => {
             this.selected.kind = kind;
             this.selected.id = args.id;
             this.selected.owner = args.owner;
             this.selected.x = args.x;
             this.selected.y = args.y;
             visualizer.onUpdate();
             this.$forceUpdate();
         };
         visualizer.onPlay = () => {
             this.playing = true;
         };
         visualizer.onPause = () => {
             this.playing = false;
         };
         visualizer.play();
         this.play = () => {
             if (this.encoding) return;
             if (visualizer.frame === visualizer.replay.frames.length - 1 && visualizer.time >= 1.0) {
                 // Restart
                 visualizer.frame = 0;
                 visualizer.time = 0.0;
             }
             visualizer.play();
         };
         this.pause = () => {
             if (this.encoding) return;
             visualizer.pause();
         };
         this.scrub = (e) => {
             visualizer.scrub(e.target.value, 0);
         };
         this.getVisualizer = () => visualizer;
         this.captureVideo = () => {
             this.recording = true;
             visualizer.encodeVideo().then((blob) => {
                console.log(blob);
                window.open(URL.createObjectURL(blob));
                this.recording = false;
             });
         };

         let player_names = [];
         if (this.game) {
             let ids = Object.keys(this.game.players);
             ids.sort((player1, player2) => {
                 let idx1 = this.game.players[player1].player_index;
                 let idx2 = this.game.players[player2].player_index;

                 if (idx1 < idx2) return -1;
                 else if (idx1 == idx2) return 0;
                 else return 1;
             });
             for (let player_id of ids) {
                 const player_record = this.game.players[player_id];
                 const idx = player_record.player_index;
                 let player_data = {
                     bot_name: this.replay.player_names[idx],
                     profile_image_link: "",
                     user_id: player_id,
                     user_link: this.makeUserLink(player_id),
                     rank: player_record.rank,
                 };
                 player_names.push(player_data);

                 this.getUserProfileImage(player_id).then((url) => {
                     player_data.profile_image_link = url;
                 });
             }
         }
         else {
             for (let idx = 0; idx < this.replay.player_names.length; idx++) {
                 player_names.push({
                     bot_name: this.replay.player_names[idx],
                     username: "",
                     user_id: null,
                     rank: this.replay.stats[idx].rank,
                 });
             }
         }
         this.player_names = player_names;
     },
     methods: {
         // Stubs - see mounted()
         play: function() {
         },
         pause: function() {
         },
         scrub: function(e) {
         },
         getVisualizer: function() {},
         captureVideo: function() {},
     },
     computed: {
         statistics: function() {
             let count = {};
             for (let i = 0; i < this.replay.num_players; i++) {
                 count[i] = {
                     ships: 0,
                     planets: 0,
                 };
             }

             let frame = this.replay.frames[this.frame];
             for (let owner of Object.keys(frame.ships)) {
                 count[owner].ships += Object.values(frame.ships[owner]).length;
             }

             for (let planet of Object.values(frame.planets)) {
                 if (planet.owner !== null) {
                     count[planet.owner].planets++;
                 }
             }

             return count;
         },
         selected_planet: function() {
             if (this.selected.kind === "planet") {
                 let frame = this.replay.frames[this.frame];
                 let state = frame.planets[this.selected.id];
                 if (state) {
                     return {
                         base: this.replay.planets[this.selected.id],
                         state: state,
                     };
                 }
             }
             return null;
         },
         selected_ship: function() {
            if (this.selected.kind === "ship") {
                let frame = this.replay.frames[this.frame];
                let state = frame.ships[this.selected.owner][this.selected.id];

                if (state) {
                    return state;
                }
            }

            return null;
         },
     },
 };
</script>

<style lang="scss" scoped>
    .halite-visualizer {
    }

    .halite-visualizer-controls {
        input {
            color: #000;
        }
    }

    .halite-visualizer-canvas {
        width: 768px;
        float: left;
    }

    .halite-visualizer-info {
        float: left;
        width: 300px;
        padding-left: 40px;

        h3 {
            margin-top: 0.15em;
        }

        img {
            max-width: 40px;
            max-height: 40px;
        }
    }

    .halite-frame-scrubber {
        width: 768px;
    }

    .playbutton {
        margin-bottom: 5px;
    }

    .frameinput {
        vertical-align: top;
        line-height: 2em;
    }

    .infosummary {
        float: right
    }

    .recording-indicator {
        color: #F00;
        animation: blinker 2s linear infinite alternate;
    }

    @keyframes blinker {
        0% {
            opacity: 1.0;
        }

        50% {
            opacity: 0.0;
        }
    }

    dl {
        margin: 0;
    }
</style>
