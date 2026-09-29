<template>
  <div v-if="modals.showToken">
    <Modal v-if="!isShowing" class="show-token" @close="close">
      <h3>Einem Spieler zeigen</h3>
      <h4>Info- &amp; Fragetoken</h4>
      <ul class="cards">
        <li
          v-for="card in cards"
          :key="card.text"
          :class="[card.type, { selected: selectedCard === card.text }]"
          @click="toggleCard(card.text)"
        >
          <font-awesome-icon v-if="card.type === 'question'" icon="question" />
          {{ card.text }}
        </li>
      </ul>
      <input
        class="custom-text"
        type="text"
        v-model="customText"
        placeholder="…oder eigenen Text eingeben"
        @keyup.stop
      />
      <h4>Charaktere ({{ selectedRoles.length }}/{{ maxRoles }})</h4>
      <ul class="tokens">
        <li
          v-for="role in availableRoles"
          :key="role.id"
          :class="[role.team, { selected: isSelected(role) }]"
          @click="toggleRole(role)"
        >
          <Token :role="role" />
        </li>
      </ul>
      <h4>Wählen lassen</h4>
      <div class="chooser">
        <select v-model="chooserId" @change="applyChooser">
          <option value="">Frei wählen lassen</option>
          <optgroup v-if="scriptChoosers.length" label="Im Skript">
            <option
              v-for="chooser in scriptChoosers"
              :key="chooser.id"
              :value="chooser.id"
              >{{ chooser.role.name }}</option
            >
          </optgroup>
          <optgroup label="Weitere">
            <option
              v-for="chooser in otherChoosers"
              :key="chooser.id"
              :value="chooser.id"
              >{{ chooser.role.name }}</option
            >
          </optgroup>
        </select>
      </div>
      <div class="button-group">
        <span
          v-for="mode in pickModes"
          :key="mode.id"
          class="button"
          :class="{ townsfolk: pickMode === mode.id }"
          @click="pickMode = mode.id"
          >{{ mode.label }}</span
        >
      </div>
      <div class="button-group" v-if="isPickingRoles">
        <span
          v-for="filter in pickFilters"
          :key="filter.id"
          class="button"
          :class="{ townsfolk: pickFilter === filter.id }"
          @click="pickFilter = filter.id"
          >{{ filter.label }}</span
        >
      </div>
      <div class="button-group" v-if="isPickingPlayers">
        <span class="count-label">Spieler:</span>
        <span
          v-for="count in pickCounts"
          :key="count"
          class="button"
          :class="{ townsfolk: pickPlayerCount === count }"
          @click="pickPlayerCount = count"
          >Wähle {{ count }}</span
        >
      </div>
      <div class="button-group" v-if="isPickingRoles">
        <span class="count-label">Charaktere:</span>
        <span
          v-for="count in pickCounts"
          :key="count"
          class="button"
          :class="{ townsfolk: pickRoleCount === count }"
          @click="pickRoleCount = count"
          >Wähle {{ count }}</span
        >
      </div>
      <div class="picked" v-if="pickedRoles.length || pickedPlayers.length">
        Vom Spieler gewählt:
        <strong v-if="pickedPlayers.length">
          {{ pickedPlayers.map(player => player.name).join(", ") }}
        </strong>
        <template v-if="pickedPlayers.length && pickedRoles.length">
          ·
        </template>
        <strong v-if="pickedRoles.length">
          {{ pickedRoles.map(role => role.name).join(", ") }}
        </strong>
        <span v-if="pickedRoles.length" class="button" @click="takePicked"
          >Übernehmen</span
        >
      </div>
      <div class="note-form" v-if="plannedNotes.length">
        <input
          type="text"
          v-model="noteReason"
          list="note-reasons"
          placeholder="Grund, z.B. Wahnsinnig"
          @keyup.stop
        />
        <datalist id="note-reasons">
          <option v-for="reason in noteReasons" :key="reason" :value="reason" />
        </datalist>
        <span class="button" @click="addPickNotes">Als Notiz anlegen</span>
        <ul class="note-preview">
          <li v-for="(note, index) in plannedNotes" :key="index">
            {{ note.player.name }}:
            <strong v-if="note.role">{{ note.role.name }}</strong>
            <em>{{ reasonText }}</em>
          </li>
        </ul>
        <div v-if="isNoteAdded" class="note-added">
          Als Notiz am Dorfplatz angelegt
        </div>
      </div>
      <div class="button-group">
        <span class="button" @click="reset">Leeren</span>
        <span class="button demon" @click="startPicking">Übersicht zeigen</span>
        <span
          class="button townsfolk"
          :class="{ disabled: !canShow }"
          @click="show"
          >Zeigen</span
        >
      </div>
    </Modal>
    <transition name="modal-fade">
      <div v-if="isPicking" class="show-token-fullscreen picking">
        <div
          v-if="displayText"
          class="card small"
          :class="{ question: isQuestion }"
        >
          {{ displayText }}
        </div>
        <div class="pick-grid">
          <div v-if="isPickingPlayers" class="pick-team">
            <h3>
              {{ pickHint(pickPlayerCount, "Spieler", "Spieler") }}
              ({{ pickedPlayers.length }}/{{ pickPlayerCount }})
            </h3>
            <ul class="pick-players">
              <li
                v-for="(player, index) in players"
                :key="index"
                :class="{ picked: isPickedPlayer(player), dead: player.isDead }"
                @click="togglePickedPlayer(player)"
              >
                {{ player.name || `Platz ${index + 1}` }}
                <span
                  v-if="pickPlayerCount > 1 && isPickedPlayer(player)"
                  class="pair-badge"
                  >{{ pickedPlayers.indexOf(player) + 1 }}</span
                >
              </li>
            </ul>
          </div>
          <template v-if="isPickingRoles">
            <h3 class="pick-hint">
              {{ pickHint(pickRoleCount, "Charakter", "Charaktere") }}
              ({{ pickedRoles.length }}/{{ pickRoleCount }})
            </h3>
            <div
              v-for="group in pickGroups"
              :key="group.team"
              class="pick-team"
            >
              <h3 :class="group.team">{{ group.label }}</h3>
              <ul>
                <li
                  v-for="role in group.roles"
                  :key="role.id"
                  :class="[role.team, { picked: isPicked(role) }]"
                  @click="togglePicked(role)"
                >
                  <Token :role="role" />
                  <span
                    v-if="pickRoleCount > 1 && isPicked(role)"
                    class="pair-badge"
                    >{{ pickedRoles.indexOf(role) + 1 }}</span
                  >
                </li>
              </ul>
            </div>
          </template>
        </div>
        <div class="button-group">
          <span class="button" @click="isPicking = false">Zurück</span>
          <span
            class="button townsfolk"
            :class="{ disabled: !isPickComplete }"
            @click="confirmPick"
            >Wahl bestätigen</span
          >
        </div>
      </div>
    </transition>
    <transition name="modal-fade">
      <div v-if="isShowing" class="show-token-fullscreen" @click="hide">
        <div
          v-if="displayText"
          class="card"
          :class="{ question: isQuestion }"
        >
          <font-awesome-icon v-if="isQuestion" icon="question" class="mark" />
          {{ displayText }}
        </div>
        <ul class="big-tokens" v-if="selectedRoles.length">
          <li
            v-for="role in selectedRoles"
            :key="role.id"
            :class="[role.team, `count-${selectedRoles.length}`]"
          >
            <Token :role="role" />
          </li>
        </ul>
      </div>
    </transition>
  </div>
</template>

<script>
import { mapMutations, mapState } from "vuex";
import Modal from "./Modal";
import Token from "../Token";

const teamLabels = {
  townsfolk: "Bürger",
  outsider: "Außenseiter",
  minion: "Schergen",
  demon: "Dämonen"
};

const pickFilters = [
  { id: "all", label: "Alle", teams: Object.keys(teamLabels) },
  { id: "good", label: "Gut", teams: ["townsfolk", "outsider"] },
  { id: "evil", label: "Böse", teams: ["minion", "demon"] }
];

const pickModes = [
  { id: "roles", label: "Charaktere" },
  { id: "players", label: "Spieler" },
  { id: "both", label: "Beides" }
];

const choosers = [
  { id: "gambler", players: 1, roles: 1, reason: "Glücksspieler rät" },
  {
    id: "juggler",
    players: 5,
    roles: 5,
    paired: true,
    reason: "Jongleur rät"
  },
  { id: "cerenovus", players: 1, roles: 1, filter: "good", reason: "Cerenovus" },
  { id: "pithag", players: 1, roles: 1, reason: "Verwandelt" },
  { id: "philosopher", roles: 1, filter: "good", reason: "Betrunken" },
  { id: "courtier", roles: 1, reason: "Betrunken" },
  { id: "fortuneteller", players: 2, reason: "Geprüft" },
  { id: "poisoner", players: 1, reason: "Vergiftet" },
  { id: "monk", players: 1, reason: "Beschützt" },
  { id: "butler", players: 1, reason: "Meister" },
  { id: "imp", players: 1, reason: "Ziel" },
  { id: "ravenkeeper", players: 1, reason: "Geprüft" },
  { id: "dreamer", players: 1, reason: "Geträumt" },
  { id: "seamstress", players: 2, reason: "Verglichen" },
  { id: "snakecharmer", players: 1, reason: "Geprüft" },
  { id: "exorcist", players: 1, reason: "Gewählt" },
  { id: "witch", players: 1, reason: "Verflucht" },
  { id: "devilsadvocate", players: 1, reason: "Überlebt Hinrichtung" },
  { id: "innkeeper", players: 2, reason: "Beschützt" },
  { id: "chambermaid", players: 2, reason: "Geprüft" },
  { id: "sailor", players: 1, reason: "Betrunken" },
  { id: "fearmonger", players: 1, reason: "Angst" }
];

const cards = [
  { text: "Dies ist der Dämon", type: "info" },
  { text: "Dies sind deine Schergen", type: "info" },
  { text: "Diese Charaktere sind nicht im Spiel", type: "info" },
  { text: "Du bist", type: "info" },
  { text: "Dieser Spieler ist", type: "info" },
  { text: "Dieser Charakter hat dich gewählt", type: "info" },
  { text: "Die Verrückte hat diese Spieler gewählt", type: "info" },
  {
    text: "Der Cerenovus hat dich gewählt: Sei besessen davon, dieser Charakter zu sein",
    type: "info"
  },
  { text: "Wir sollten morgen reden", type: "info" },
  {
    text: "Ich habe einen Fehler gemacht, hier ist meine Korrektur",
    type: "info"
  },
  { text: "Willst du deine Fähigkeit nutzen?", type: "question" },
  { text: "Hast du heute nominiert?", type: "question" },
  { text: "Hast du heute abgestimmt?", type: "question" },
  { text: "Triff eine Wahl", type: "question" },
  { text: "Als was bluffst du?", type: "question" },
  { text: "Bitte wähle noch mal", type: "question" }
];

export default {
  components: { Token, Modal },
  data() {
    return {
      cards,
      maxRoles: 3,
      selectedCard: "",
      customText: "",
      selectedRoles: [],
      isShowing: false,
      isPicking: false,
      pickFilters,
      pickFilter: "all",
      pickCounts: [1, 2, 3, 4, 5],
      pickPlayerCount: 1,
      pickRoleCount: 1,
      pickedRoles: [],
      pickModes,
      pickMode: "roles",
      pickedPlayers: [],
      isNoteAdded: false,
      chooserId: "",
      noteReason: "",
      noteReasons: [
        ...new Set(["Gewählt", ...choosers.map(chooser => chooser.reason)])
      ]
    };
  },
  computed: {
    availableRoles() {
      return [
        ...this.roles.values(),
        ...this.otherTravelers.values(),
        ...this.fabled.values()
      ];
    },
    isPickingRoles() {
      return this.pickMode !== "players";
    },
    isPickingPlayers() {
      return this.pickMode !== "roles";
    },
    pickGroups() {
      const filter = pickFilters.find(f => f.id === this.pickFilter);
      const roles = [...this.roles.values()];
      return filter.teams
        .map(team => ({
          team,
          label: teamLabels[team],
          roles: roles.filter(role => role.team === team)
        }))
        .filter(group => group.roles.length);
    },
    chooserList() {
      const rolesById = this.$store.getters.rolesJSONbyId;
      return choosers
        .map(chooser => ({
          ...chooser,
          role: this.roles.get(chooser.id) || rolesById.get(chooser.id)
        }))
        .filter(chooser => chooser.role)
        .sort((a, b) => a.role.name.localeCompare(b.role.name));
    },
    scriptChoosers() {
      return this.chooserList.filter(chooser => this.roles.has(chooser.id));
    },
    otherChoosers() {
      return this.chooserList.filter(chooser => !this.roles.has(chooser.id));
    },
    currentChooser() {
      return this.chooserList.find(chooser => chooser.id === this.chooserId);
    },
    reasonText() {
      return this.noteReason.trim() || "Gewählt";
    },
    plannedNotes() {
      const chooserRole = this.currentChooser && this.currentChooser.role;
      if (this.pickedPlayers.length && this.pickedRoles.length) {
        if (this.currentChooser && this.currentChooser.paired) {
          return this.pickedPlayers
            .map((player, index) => ({
              player,
              role: this.pickedRoles[index]
            }))
            .filter(note => note.role);
        }
        return this.pickedPlayers.flatMap(player =>
          this.pickedRoles.map(role => ({ player, role }))
        );
      }
      if (this.isPickingRoles && this.isPickingPlayers) return [];
      if (this.pickedPlayers.length) {
        return this.pickedPlayers.map(player => ({
          player,
          role: chooserRole
        }));
      }
      return this.players
        .filter(player => this.pickedRoles.some(r => r.id === player.role.id))
        .map(player => ({ player, role: chooserRole }));
    },
    isPickComplete() {
      if (
        this.currentChooser &&
        this.currentChooser.paired &&
        this.pickMode === "both"
      ) {
        return (
          this.pickedPlayers.length > 0 &&
          this.pickedPlayers.length === this.pickedRoles.length
        );
      }
      return (
        (!this.isPickingRoles ||
          this.pickedRoles.length === this.pickRoleCount) &&
        (!this.isPickingPlayers ||
          this.pickedPlayers.length === this.pickPlayerCount)
      );
    },
    displayText() {
      return this.customText.trim() || this.selectedCard;
    },
    isQuestion() {
      if (this.customText.trim()) {
        return this.customText.trim().endsWith("?");
      }
      const card = cards.find(c => c.text === this.selectedCard);
      return !!card && card.type === "question";
    },
    canShow() {
      return !!this.displayText || this.selectedRoles.length > 0;
    },
    ...mapState(["modals", "roles", "otherTravelers", "fabled"]),
    ...mapState("players", ["players"])
  },
  watch: {
    "modals.showToken"(isOpen) {
      if (!isOpen) {
        this.isShowing = false;
        this.isPicking = false;
      }
    }
  },
  methods: {
    toggleCard(text) {
      this.customText = "";
      this.selectedCard = this.selectedCard === text ? "" : text;
    },
    isSelected(role) {
      return this.selectedRoles.some(r => r.id === role.id);
    },
    toggleRole(role) {
      if (this.isSelected(role)) {
        this.selectedRoles = this.selectedRoles.filter(r => r.id !== role.id);
      } else if (this.selectedRoles.length < this.maxRoles) {
        this.selectedRoles.push(role);
      }
    },
    show() {
      if (!this.canShow) return;
      this.isShowing = true;
    },
    pickHint(count, singular, plural) {
      return `Wähle ${count} ${count === 1 ? singular : plural}`;
    },
    confirmPick() {
      if (!this.isPickComplete) return;
      this.isPicking = false;
    },
    applyChooser() {
      const chooser = this.currentChooser;
      if (!chooser) return;
      if (chooser.players && chooser.roles) {
        this.pickMode = "both";
      } else {
        this.pickMode = chooser.players ? "players" : "roles";
      }
      this.pickPlayerCount = chooser.players || 1;
      this.pickRoleCount = chooser.roles || 1;
      this.pickFilter = chooser.filter || "all";
      this.noteReason = chooser.reason;
    },
    addPickNotes() {
      this.plannedNotes.forEach(({ player, role }) => {
        const note = role
          ? {
              role: role.id,
              name: this.reasonText,
              image: role.image,
              imageAlt: role.imageAlt
            }
          : { role: "custom", name: this.reasonText };
        const reminders = player.reminders;
        if (reminders.some(r => r.role === note.role && r.name === note.name)) {
          return;
        }
        this.$store.commit("players/update", {
          player,
          property: "reminders",
          value: [...reminders, note]
        });
      });
      this.isNoteAdded = true;
    },
    startPicking() {
      this.isNoteAdded = false;
      this.pickedRoles = [];
      this.pickedPlayers = [];
      this.isPicking = true;
    },
    isPickedPlayer(player) {
      return this.pickedPlayers.includes(player);
    },
    togglePickedPlayer(player) {
      this.pickedPlayers = this.togglePick(
        this.pickedPlayers,
        player,
        this.pickPlayerCount
      );
    },
    togglePick(list, item, limit) {
      if (list.includes(item)) return list.filter(i => i !== item);
      if (limit === 1) return [item];
      if (list.length < limit) return [...list, item];
      return list;
    },
    isPicked(role) {
      return this.pickedRoles.includes(role);
    },
    togglePicked(role) {
      this.pickedRoles = this.togglePick(
        this.pickedRoles,
        role,
        this.pickRoleCount
      );
    },
    takePicked() {
      const newRoles = this.pickedRoles.filter(role => !this.isSelected(role));
      this.selectedRoles = [...newRoles, ...this.selectedRoles].slice(
        0,
        this.maxRoles
      );
    },
    hide() {
      this.isShowing = false;
    },
    reset() {
      this.selectedCard = "";
      this.customText = "";
      this.selectedRoles = [];
      this.pickedRoles = [];
      this.pickedPlayers = [];
      this.isNoteAdded = false;
    },
    close() {
      this.isShowing = false;
      this.isPicking = false;
      this.toggleModal("showToken");
    },
    ...mapMutations(["toggleModal"])
  }
};
</script>

<style scoped lang="scss">
@import "../../vars.scss";

h4 {
  margin: 10px 0 5px;
  font-size: 110%;
}

ul.cards {
  li {
    margin: 4px;
    padding: 4px 12px;
    border-radius: 10px;
    border: 2px solid #555;
    background: rgba(255, 255, 255, 0.05);
    cursor: pointer;
    font-family: "Papyrus", serif;
    font-weight: bold;
    line-height: 140%;
    &.question {
      border-color: $townsfolk;
    }
    &:hover {
      color: red;
    }
    &.selected {
      border-color: $fabled;
      background: rgba($fabled, 0.2);
    }
  }
}

.custom-text {
  display: block;
  width: 60%;
  margin: 8px auto;
  padding: 5px 10px;
  border-radius: 10px;
  border: 2px solid #555;
  background: rgba(0, 0, 0, 0.5);
  color: white;
  font-size: 90%;
}

ul.tokens {
  max-height: 40vh;
  overflow-y: auto;
  align-content: flex-start;
  padding: 10px 0;
  li {
    border-radius: 50%;
    width: 5vw;
    margin: 0.5%;
    opacity: 0.6;
    transition: transform 250ms ease, opacity 250ms ease;
    &:hover,
    &.selected {
      opacity: 1;
    }
    &.selected {
      transform: scale(1.1);
      box-shadow: 0 0 15px $fabled, 0 0 15px $fabled;
    }
  }
}

.count-label {
  margin-right: 8px;
  font-size: 90%;
}

.count-label + .button {
  border-top-left-radius: 15px;
  border-bottom-left-radius: 15px;
}

.chooser {
  text-align: center;
  select {
    margin: 3px;
    padding: 4px 8px;
    border-radius: 10px;
    border: 2px solid #555;
    background: rgba(0, 0, 0, 0.5);
    color: white;
    font-size: 90%;
    option,
    optgroup {
      color: black;
    }
  }
}

.pair-badge {
  position: absolute;
  top: -0.8vh;
  right: -0.8vh;
  width: 3.5vh;
  height: 3.5vh;
  line-height: 3.5vh;
  border-radius: 50%;
  background: $fabled;
  color: black;
  font-size: 2.2vh;
  font-weight: bold;
  text-align: center;
  z-index: 20;
}

.note-form {
  text-align: center;
  margin: 5px 0;
  select,
  input {
    margin: 3px;
    padding: 4px 8px;
    border-radius: 10px;
    border: 2px solid #555;
    background: rgba(0, 0, 0, 0.5);
    color: white;
    font-size: 90%;
    option {
      color: black;
    }
  }
  .button {
    display: inline-block;
    margin-left: 6px;
  }
  .note-preview {
    display: block;
    font-size: 85%;
    li {
      display: block;
      margin: 2px 0;
    }
    em {
      margin-left: 6px;
      opacity: 0.8;
    }
  }
}

.note-added {
  font-size: 80%;
  color: $fabled;
}

.picked {
  text-align: center;
  margin: 5px 0;
  .button {
    display: inline-block;
    margin-left: 10px;
  }
}

.show-token-fullscreen.picking {
  justify-content: flex-start;
  padding: 2vh 2vw;
  cursor: default;
  overflow-y: auto;

  .card.small {
    font-size: 4vh;
    margin-bottom: 2vh;
  }

  .pick-hint {
    font-size: 3.5vh;
    margin-bottom: 1vh;
  }

  .pick-grid {
    width: 100%;
    flex: 1;
  }

  .pick-team {
    margin-bottom: 1.5vh;
    h3 {
      font-size: 4vh;
      &.townsfolk {
        color: $townsfolk;
      }
      &.outsider {
        color: $outsider;
      }
      &.minion {
        color: $minion;
      }
      &.demon {
        color: $demon;
      }
    }
    ul {
      display: flex;
      flex-wrap: wrap;
      justify-content: center;
    }
    li {
      width: 13vh;
      margin: 0.8vh;
      border-radius: 50%;
      cursor: pointer;
      transition: transform 200ms ease;
      &:hover {
        transform: scale(1.1);
      }
      &.picked {
        transform: scale(1.2);
        z-index: 10;
        box-shadow: 0 0 25px $fabled, 0 0 25px $fabled;
      }
    }
    ul.pick-players li {
      width: auto;
      min-width: 18vh;
      padding: 1vh 2vh;
      border-radius: 2vh;
      border: 0.4vh solid #555;
      background: rgba(255, 255, 255, 0.08);
      font-size: 3vh;
      font-weight: bold;
      text-align: center;
      &.dead {
        opacity: 0.6;
        text-decoration: line-through;
      }
      &.picked {
        border-color: $fabled;
        background: rgba($fabled, 0.25);
        transform: scale(1.08);
      }
    }
  }
}

.show-token-fullscreen {
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 200;
  background: rgba(0, 0, 0, 0.95);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  cursor: pointer;

  .card {
    max-width: 90vw;
    margin-bottom: 4vh;
    padding: 2vh 5vw;
    border-radius: 3vh;
    border: 0.6vh solid #7a1b1b;
    background: linear-gradient(#f3e3c0, #d8c08e);
    color: #2b1a0a;
    font-family: "Papyrus", serif;
    font-weight: bold;
    font-size: 7vh;
    line-height: 120%;
    text-align: center;
    box-shadow: 0 0 40px rgba(0, 0, 0, 0.8);
    &.question {
      border-color: $townsfolk;
    }
    .mark {
      margin-right: 2vh;
      color: $townsfolk;
    }
  }

  ul.big-tokens {
    display: flex;
    justify-content: center;
    align-items: center;
    li {
      margin: 0 2vw;
      border-radius: 50%;
      width: 55vh;
      &.count-2 {
        width: 42vh;
      }
      &.count-3 {
        width: 32vh;
      }
      &.townsfolk {
        box-shadow: 0 0 30px $townsfolk;
      }
      &.outsider {
        box-shadow: 0 0 30px $outsider;
      }
      &.minion {
        box-shadow: 0 0 30px $minion;
      }
      &.demon {
        box-shadow: 0 0 30px $demon;
      }
      &.traveler {
        box-shadow: 0 0 30px $traveler;
      }
      &.fabled {
        box-shadow: 0 0 30px $fabled;
      }
      ::v-deep .ability {
        display: none;
      }
    }
  }
}
</style>
