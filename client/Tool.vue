<template>
  <v-card>
    <form @submit.prevent="diceroll">
      <v-text-field
        ref="commandField"
        v-model="command"
        :append-icon-cb="diceroll"
        @blur="help = false"
        @focus="help = showSystemInfo"
        label="ダイスコマンドを入力..."
        append-icon="send"
        single-line
        hide-details
        full-width
      />
    </form>
    <v-divider />
    <transition
      @before-enter="beforeEnter"
      @enter="enter"
      @before-leave="beforeLeave"
      @leave="leave"
    >
      <div v-show="help">
        <v-card-text class="sysinfo">
          <div>{{ gameName }}</div>
          <span v-html="gameInfo" />
        </v-card-text>
        <v-divider />
      </div>
    </transition>
    <v-btn
      v-for="(command, i) in shortcuts"
      :key="i"
      @click.stop="execShortcut(command)"
      depressed
      >{{ command }}</v-btn
    >
    <v-tooltip bottom>
      <v-btn slot="activator" @click.stop="edit = true" icon>
        <v-icon>add</v-icon>
      </v-btn>
      <span>ショートカット追加</span>
    </v-tooltip>
    <ShortcutDialog v-model="edit" />
    <v-snackbar v-model="snackbar" :timeout="4000" color="error" top>
      {{ errorMsg }}
      <v-btn @click="snackbar = false" dark flat>
        閉じる
      </v-btn>
    </v-snackbar>
  </v-card>
</template>

<script lang="ts">
import Vue from "vue";
import Component from "vue-class-component";
import { diceRoll, selectDiceResults } from "./dice";

import ShortcutDialog from "./ShortcutDialog.vue";

interface Dice {
  faces: number;
  value: number;
}

@Component({
  components: {
    ShortcutDialog
  }
})
export default class Tool extends Vue {
  $refs!: {
    commandField: HTMLInputElement;
  };

  data() {
    return {
      command: "",
      help: false,
      edit: false,
      snackbar: false,
      errorMsg: ""
    };
  }

  clearForm() {
    this.$data.command = "";
  }

  get gameType() {
    return this.$store.state.gameType;
  }

  get gameName() {
    return this.$store.state.gameName;
  }

  get gameInfo() {
    return this.$store.state.gameInfo.trim().replace(/\n/g, "<br />");
  }

  get shortcuts() {
    return this.$store.state.shortcuts;
  }

  get showSystemInfo() {
    return this.$store.state.settings.showSystemInfo;
  }

  execShortcut(command: string) {
    if (command.match(/[?？]/)) {
      this.$data.command = command;
      this.$refs.commandField.focus();
    } else {
      this.dicerollByText(command);
    }
  }

  diceroll() {
    this.dicerollByText(this.$data.command, true);
  }

  dicerollByText(text: string, clear: boolean = false) {
    diceRoll(this.gameType, text)
      .then(res => {
        const dices = res.dices.map((d: Dice) => {
          return { face: d.faces, value: d.value };
        });
        const log: Log = {
          gameType: this.gameType,
          command: text,
          body: res.result,
          drawables: selectDiceResults(dices),
          timestamp: new Date()
        };

        if (clear) {
          this.clearForm();
        }

        this.$store.commit("appendLogBuffer", log);
      })
      .catch(error => {
        if (error.response) {
          this.$data.errorMsg = "ダイスコマンドを実行できませんでした";
        } else {
          this.$data.errorMsg = "APIサーバーに接続できませんでした";
        }
        this.$data.snackbar = true;
      });
  }

  beforeEnter(el: HTMLElement) {
    el.style.height = "0";
  }
  enter(el: HTMLElement) {
    el.style.height = el.scrollHeight + "px";
  }
  beforeLeave(el: HTMLElement) {
    el.style.height = el.scrollHeight + "px";
  }
  leave(el: HTMLElement) {
    el.style.height = "0";
  }
}
</script>

<style lang="scss" scoped>
.v-enter-active,
.v-leave-active {
  transition: all 0.25s;
  overflow: hidden;
}
</style>
