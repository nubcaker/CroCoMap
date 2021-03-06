<template>
    <div :class="[{'md-layout-item': inGrid}, size]">
        <md-card ref="card">
            <md-card-actions class="md-alignment-left">
                <div class="md-title">{{title}} Annotation</div>
                <md-icon class="info-icon">info
                    <md-tooltip md-direction="left">{{title}} Annotation Snapshot</md-tooltip>
                </md-icon>
            </md-card-actions>
            <md-card-content class="card-content">
                <div v-if="image !== ''" class="img-container">
                    <img class="img-img" :src="image" alt="Image failed to load."/>
                </div>
                <md-empty-state
                        v-if="image === ''"
                        md-icon="track_changes"
                        :md-label="'No ' + title + ' Annotation'">
                </md-empty-state>
            </md-card-content>
            <md-card-actions v-if="image !== ''" class="actions">
                <md-button class="vote-button md-raised" @click="verifyPrompt = true">VOTE</md-button>
            </md-card-actions>
        </md-card>
        <md-dialog :md-active="verifyPrompt">
            <md-dialog-title class="dialog-title dialog-title-custom">Confirm Your Choice</md-dialog-title>
            <md-dialog-content class="dialog-content dialog-content-custom">
                <md-empty-state
                        md-icon="check_circle_outline"
                        md-description="Are you sure you wish to vote for this annotation? You may only pick one.">
                    <span class="button-span">
                        <md-button class="vote-button md-raised" @click="verify()">Confirm</md-button>
                        <md-button class="omit-button md-raised" @click="verifyPrompt = false">CANCEL</md-button>
                    </span>
                </md-empty-state>
            </md-dialog-content>
        </md-dialog>
    </div>
</template>
<script>
    import {mapActions, mapGetters} from 'vuex';

    export default {
        name: 'VerifyCard',
        props: {
            title: String, // Title of the card.
            size: String, // Size of card in gutter.
            inGrid: Boolean, // Is this card in gutter?
            image: String, // B64 encoded image of the snapshot to show.
            index: Number // Index of the snapshot in the annotation object.
        },
        data: function() {
          return {
              verifyPrompt: false, // Prompt confirmation for voting for snapshot?
          }
        },
        computed: {},
        methods: {
            ...mapActions(['updateVerifyVotes']),
            ...mapGetters(['getVerifyVotes']),
            verify: function() { // Confirm the vote.
                this.updateVerifyVotes(this.image); // Vote image string, could be changed to image index if you like.
                this.verifyPrompt = false; // Stop showing the confirmation prompt.
            }
        }
    };
</script>
<style scoped>
    .card-content {
        height: calc(100% - 100px) !important;
    }

    .img-container {
        text-align: center;
        height: 100%;
        max-height: 200px;
        overflow: hidden;
    }

    .img-img {
        height: 100%;
    }

    .actions {
        justify-content: center !important;
    }
</style>