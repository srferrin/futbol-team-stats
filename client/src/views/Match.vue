<template>
  <div>
    <v-card>
      <v-container>
      <v-card offset-sm3>

      <v-alert
        v-model="alert"
        dismissible
        color="warning"
        icon="priority_high">
        {{ alertMessage }}
      </v-alert>

      <v-card-title>
        <v-spacer></v-spacer>
        <div v-if="is_editor">
          <v-btn
            :to="{
              name: 'MatchEdit',
              params: {
                  id: match.id
              }
            }"
            color="primary">
            Edit
          </v-btn>
          <v-btn @click="deleteMatch()" color="error">Delete</v-btn>

          <v-btn v-if="match.match_stats"
            :to="{
              name: 'MatchStatsEdit',
              params: {
                  id: match.match_stats
              }
            }"
            color="primary">Edit Extra Stats</v-btn>
          <v-btn v-else @click="createMatchStats()">Add Extra Stats</v-btn>
        </div>
      </v-card-title>

      <v-card-text>
        <h4>
          Date : {{ match.date_time | formatDate }} <br/>
          Competition : {{ match.competition_name }} <br/>
          Opponent : {{ match.opponent_name }} <br/>
          Duration : {{ match.duration }} mins <br/>
          <div v-if="match.note">
            Note: {{ match.note }}
          </div>
        </h4>
      </v-card-text>

        <v-card-text>
          <v-data-table
              :headers="matchTableHeaders"
              :items="matchTableRows"
              hide-actions
              item-key="stat">
            <template slot="items" slot-scope="props">
              <tr>
                <td v-if="props.item.team_stat instanceof Array" class="text-xs-right">
                  <div v-for="goal in props.item.team_stat" v-bind:key="goal">
                    {{ goal }} <br/>
                  </div>
                </td>

                <td v-else class="text-xs-right">
                  {{ props.item.team_stat }}
                </td>

                <td class="text-xs-center">{{ props.item.stat }}</td>

                <td v-if="props.item.opponent_stat instanceof Array" class="text-xs-left">
                  <div v-for="goal in props.item.opponent_stat" v-bind:key="goal">
                    {{ goal }} <br/>
                  </div>
                </td>

                <td v-else class="text-xs-left">
                  {{ props.item.opponent_stat }}
                </td>
              </tr>
            </template>
          </v-data-table>
        </v-card-text>
      </v-card>
      </v-container>

      <v-container>
        <v-card>
          <v-card-title>
            <h2>Player match stats</h2>

            <v-spacer></v-spacer>

            <v-btn v-if="is_editor"
                   :to="{
                     name: 'PlayerMatchEdit',
                     params: {id: match.id}
                   }"
                   color="primary">
              Edit Player Stats
            </v-btn>
          </v-card-title>
          <v-card-text>
            <player-matches-table :player_matches="player_matches"
                                  :match="match"
                                  :showPlayer="true"
                                  :showProgress="showProgress">
            </player-matches-table>
          </v-card-text>
        </v-card>
      </v-container>

      <v-container>
        <v-card>
          <v-card-title>
            <h2>Shots</h2>
          <v-spacer></v-spacer>
            <v-btn v-if="is_editor"
                   :to="{
                     name: 'MatchShotsEdit',
                     params: {id: match.id}
                   }"
                   color="primary">
              Edit Shots
            </v-btn>
          </v-card-title>

          <shots-graph :shots="shots"
                       :onClickShot=onClickShot>
          </shots-graph>
        </v-card>
        <v-btn @click="backToCompetition()" color="error">Back To Competition</v-btn>
      </v-container>
    </v-card>
  </div>
</template>

<script>
import API from '@/lib/API'
import PlayerMatchesTable from '@/components/PlayerMatchesTable'
import ShotsGraph from '@/components/ShotsGraph'

export default {
  components: {
    PlayerMatchesTable, ShotsGraph
  },
  data () {
    return {
      alert: false,
      alertMessage: '',
      match: {
        id: -1,
        match_stats: -1,
        date_time: ''
      },
      player_matches: [],
      shots: [],
      matchTableHeaders: [
        { text: 'Team', value: 'team_stat', sortable: false, align: 'right' },
        { text: '', value: 'stat', sortable: false, align: 'center' },
        { text: 'Opponent', value: 'opponent_stat', sortable: false, align: 'left' }
      ],
      showProgress: true
    }
  },
  computed: {
    is_editor () {
      return this.$store.state.user.is_editor
    },
    matchTableRows: function () {
      let rows = [{
        team_stat: this.match.num_goals,
        stat: 'Goals',
        opponent_stat: this.match.num_goals_against
      },
      {
        team_stat: this.match.num_shots + ' (' + this.match.shot_on_target_pct + '%)',
        stat: 'Shots (on target %)',
        opponent_stat: this.match.num_shots_against + ' (' + this.match.opponent_shot_on_target_pct + '%)'
      },
      {
        team_stat: this.match.num_corners,
        stat: 'Corners',
        opponent_stat: this.match.num_opponent_corners
      },
      {
        team_stat: this.match.num_yellow_cards,
        stat: 'Yellow Cards',
        opponent_stat: this.match.num_opponent_yellow_cards
      },
      {
        team_stat: this.match.num_red_cards,
        stat: 'Red Cards',
        opponent_stat: this.match.num_opponent_red_cards
      },
      {
        team_stat: this.match.goals_timeline,
        stat: 'Goals Timeline',
        opponent_stat: this.match.goals_against_timeline
      }]

      return rows
    }
  },
  mounted () {
    const { id } = this.$route.params
    this.load(id)
  },
  methods: {
    load (id) {
      API.getMatch(id)
        .then((match) => {
          this.match = match
        })
        .catch((err) => {
          if (err.response.status === 401) {
            this.$router.push({name: 'login'})
          }
        })
      API.getPlayerMatchesByMatch(id)
        .then((player_matches) => {
          this.player_matches = player_matches,
          this.showProgress = false
        })
      API.getShotsByMatch(id)
        .then((shots) => {
          this.shots = shots
        })
    },
    onClickShot (shot) {
      // todo: vuetify hover https://vuetifyjs.com/en/components/hover
      alert('shot details\nplayer: ' + shot.player_label)
    },
    createMatchStats () {
      const { id } = this.$route.params
      this.match_stats = { match: id }
      API.createMatchStats(this.match_stats)
        .then(this.load(id))
    },
    deleteMatch () {
      API.deleteMatch(this.match.id)
        .then(() => {
          this.$router.push({
            name: 'Competition',
            params: { id: this.match.competition }}
          )
        })
        .catch((err) => {
          this.alert = true
          this.alertMessage = err.response.data.error
        })
    },
    backToCompetition () {
      this.$router.push({
        name: 'Competition',
        params: { id: this.match.competition }}
      )
    }
  }
}
</script>
