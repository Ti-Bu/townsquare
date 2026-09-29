<template>
  <div class="intro">
    <img src="static/apple-icon.png" alt="" class="logo" />
    <div>
      Willkommen beim (inoffiziellen)
      <b>Virtuellen Town Square und Grimoire</b> für Blood on the Clocktower!
      Bitte füge weitere Spieler über das
      <span class="button" @click="toggleMenu">
        <font-awesome-icon icon="cog" /> Menü
      </span>
      oben rechts oder mit der Taste <b>[A]</b> hinzu. Du kannst auch einer
      Spielsitzung beitreten, indem du <b>[J]</b> drückst.<br />
      <div class="footer">
        Dieses Projekt ist kostenlos und quelloffen und ist auf
        <a href="https://github.com/bra1n/townsquare" target="_blank">GitHub</a
        > zu finden. Es steht in keiner Verbindung zum Pandemonium Institute.
        "Blood on the Clocktower" ist eine Marke von Steven Medway und dem
        Pandemonium Institute.
      </div>
    </div>
    <a
      class="redirect"
      v-if="language === 'zh-CN'"
      href="https://clocktower.gstonegames.com"
    >
      <img src="../assets/gstone.png" class="gstone" alt="" />
      你想使用中文版魔典吗？
    </a>
  </div>
</template>

<script>
import { mapMutations } from "vuex";

export default {
  data() {
    return {
      language: window.navigator.userLanguage || window.navigator.language
    };
  },
  methods: mapMutations(["toggleMenu"])
};
</script>

<style scoped lang="scss">
// Intro
.intro {
  text-align: center;
  width: 50%;
  font-size: 120%;
  position: absolute;
  padding: 10px;
  background: rgba(0, 0, 0, 0.5);
  border: 3px solid black;
  border-radius: 10px;
  z-index: 3;
  display: flex;
  justify-content: center;
  a {
    color: white;
  }
  a.redirect {
    display: block;
    text-decoration: none;
    position: absolute;
    top: 100%;
    margin-top: 2vh;
    padding: 10px;
    background: rgba(0, 0, 0, 0.5);
    border: 3px solid black;
    border-radius: 10px;

    &:hover {
      color: red;
    }
    img {
      width: 120px;
      display: block;
      margin: auto;
      margin-bottom: 1vh;
    }
  }

  img.logo {
    position: absolute;
    bottom: 100%;
    width: 25vh;
    margin-bottom: 2vh;
    max-width: 192px;
    border-radius: 50%;
    box-shadow: 0 0 10px black;
    border: 3px solid black;
  }
  .footer {
    font-size: 60%;
    opacity: 0.75;
  }
}
</style>
