<template>
    <div class="verify-page">
        <div v-if="!done">
            <div class="md-layout md-gutter">
                <streetview-card :position="this.snapshots[voteIndex]['position']" :pov="this.snapshots[voteIndex]['pov']" :inGrid="true" size="md-size-33"></streetview-card>
                <verify-card title="First" :image="this.snapshots[voteIndex]['First']" :index="voteIndex" :inGrid="true"
                             size="md-size-33"></verify-card>
                <verify-card title="Second" :image="this.snapshots[voteIndex]['Second']" :index="voteIndex" :inGrid="true"
                             size="md-size-33"></verify-card>
            </div>
            <div class="md-layout md-gutter">
                <verify-card title="Third" :image="this.snapshots[voteIndex]['Third']" :index="voteIndex" :inGrid="true"
                             size="md-size-33"></verify-card>
                <verify-card title="Fourth" :image="this.snapshots[voteIndex]['Fourth']" :index="voteIndex" :inGrid="true"
                             size="md-size-33"></verify-card>
                <verify-card title="Fifth" :image="this.snapshots[voteIndex]['Fifth']" :index="voteIndex" :inGrid="true"
                             size="md-size-33"></verify-card>
            </div>
            <md-speed-dial class="verify-dial" md-direction="top">
                <md-speed-dial-target class="annotate-button vote-button">
                    <md-icon class="md-morph-initial">menu_open</md-icon>
                    <md-icon class="md-morph-final">sentiment_satisfied_alt</md-icon>
                </md-speed-dial-target>
                <md-speed-dial-content class="">
                    <md-button class="md-icon-button" @click="showHelp">
                        <md-tooltip md-direction="right">Show Help</md-tooltip>
                        <md-icon>help</md-icon>
                    </md-button>
                </md-speed-dial-content>
            </md-speed-dial>
        </div>
        <md-empty-state
                v-if="done"
                class="verify-empty"
                md-icon="verified"
                md-label="All set!"
                md-description="You have completed your verify tasks for now. Thank you!">
        </md-empty-state>
        <md-dialog :md-active="uncertainPrompt">
            <md-dialog-title class="dialog-title dialog-title-custom">I don't know.</md-dialog-title>
            <md-dialog-content class="dialog-content dialog-content-custom">
                <md-empty-state
                        md-icon="help_outline"
                        md-description="Only use this options if you really do not know whether this object is potentially risky.">
                    <span class="button-span">
                        <md-button class="vote-button md-raised" @click="uncertain()">Confirm</md-button>
                        <md-button class="omit-button md-raised" @click="uncertainPrompt = false">CANCEL</md-button>
                    </span>
                </md-empty-state>
            </md-dialog-content>
        </md-dialog>
        <md-dialog :md-active="omitPrompt">
            <md-dialog-title class="dialog-title dialog-title-custom">Is this object not risky?</md-dialog-title>
            <md-dialog-content class="dialog-content dialog-content-custom">
                <md-empty-state
                        md-icon="remove_circle_outline"
                        md-description="If you mark this object as not risky, you will not be able to vote on the other snapshots.">
                    <span class="button-span">
                        <md-button class="vote-button md-raised" @click="omit()">Confirm</md-button>
                        <md-button class="omit-button md-raised" @click="omitPrompt = false">CANCEL</md-button>
                    </span>
                </md-empty-state>
            </md-dialog-content>
        </md-dialog>
        <md-snackbar class="vote-snackbar" md-position="center" :md-duration="Infinity" :md-active="!done" md-persistent>
            <span style="width: 100%; text-align: center;">
                <md-button class="unct-button md-raised" @click="uncertainPrompt = true">UNCERTAIN</md-button>
                <md-button class="omit-button md-raised" @click="omitPrompt = true">NO RISK</md-button>
            </span>
        </md-snackbar>
    </div>
</template>
<script>
    import VerifyCard from "../elements/VerifyCard";
    import {mapActions, mapGetters, mapMutations} from 'vuex'
    import StreetviewCard from "../elements/StreetviewCard";

    export default {
        name: 'Fix',
        components: {
            StreetviewCard,
            VerifyCard
        },
        data: function () {
            return {
                voteIndex: 0, // Index of annotation working on.
                done: false, // Completed task?
                uncertainPrompt: false, // Show prompt to confirm whether worker is uncertain.
                omitPrompt: false, // Show prompt to confirm whether worker finds object non-risky.
            }
        },
        computed: {
            snapshots: function () { // Get the annotations to verify.
                return this.getVerifySnapshots();
            },
            votes: function () { // Get the votes the worker has already casted (also gets on updatign votes).
                return this.getVerifyVotes();
            }
        },
        watch: {
            snapshots: function () { // Set done = true when there are no annotations to verify.
                this.done = this.snapshots.length === 0;
            },
            votes: { // On change of casted votes:
                immediate: true, // Do immediately (also on first instance of votes).
                handler: function () { // Handler method.
                    if (this.snapshots.length === 0) { // If no snapshots, set done to true.
                        this.done = true;
                    } else if (this.snapshots.length > this.votes.length) { // Increase voteIndex if not yet done.
                        this.voteIndex++;
                    } else { // Automatically submit when done.
                        this.submit();
                    }
                }
            }
        },
        mounted: function () { // On first loading page.
            this.loadVerifySnapshots();
        },
        methods: {
            ...mapMutations(['setResults', 'setShowTutorial']),
            ...mapActions(['updateVerifyVotes', 'loadVerifySnapshots', 'storeFile']),
            ...mapGetters(['getVerifySnapshots', 'getVerifyVotes']),
            uncertain: function () { // Method for voting uncertain.
                this.updateVerifyVotes("0");
                this.uncertainPrompt = false;
            },
            omit: function () { // Method for voting non-risky.
                this.updateVerifyVotes("-1");
                this.omitPrompt = false;
            },
            submit: function() { // Method to submit the votes.
                this.done = true;

                var results = []; // Convert to results array to write.
                for (var i = 0; i < this.snapshots.length; i++) {
                    var snapshot = this.snapshots[i];
                    var result = {
                        position: snapshot.position, // Position where snapshot is taken from
                        location: snapshot.location, // Location of the marker
                        pov: snapshot.pov, // POV of street view of snapshot.
                        truth: this.votes[i] // non-risky -> -1; uncertain -> 0; vote -> b64 encoded image.
                    };

                    results.push(result);
                }

                this.storeFile(['results.json', results]) // Write results to file.
            },
            showHelp: function() { // Show the tutorial again.
                this.setShowTutorial(true);
            }
        }
    };
</script>
<style scoped>
    .verify-page {
        width: 100%;
        padding: 16px;
        height: calc(100vh - 48px);
        overflow-y: scroll;
    }

    .md-icon {
        font-family: 'Material Icons Outlined';
    }

    .md-layout {
        margin-bottom: 16px;
    }

    .vote-snackbar {
        width: 33%;
    }

    .button-span {
        width: 100%;
        justify-content: center;
    }

    .md-snackbar-content .md-button:first-child {
        margin-left: -4px !important;
        /*margin-right: 4px !important;*/
    }

    .vote-button, .unct-button, .omit-button {
        color: white !important;
    }
</style>
<style>
    .verify-dial {
        position: absolute;
        left: 10px;
        bottom: 10px;
        z-index: 999;
    }

    .dialog-title-custom {
        text-align: center;
        margin-bottom: 0;
    }

    .dialog-content-custom {
        padding-bottom: 0;
    }

    .button-span .vote-button, .button-span .unct-button, .button-span .omit-button {
        margin-left: 4px !important;
        margin-right: 4px !important;
    }
</style>