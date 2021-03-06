<template>
  <div class="workspace">
    <svg width="100%" :height="height" @dragover="dragOver" @drop="drop">
      <defs>
        <filter id="focus" height="130%">
          <feGaussianBlur in="SourceAlpha" stdDeviation="4"/>
          <feOffset dx="0" dy="0" result="offsetblur"/>
          <feMerge>
            <feMergeNode/>
            <feMergeNode in="SourceGraphic"/>
          </feMerge>
        </filter>
      </defs>
      <Button :x="15" :y="15" text="S" @click="step" />
      <Button :x="15" :y="35" text="R" @click="run" />
      <Button :x="15" :y="75" text="New Session" @click="newSession" />
      <Button :x="15" :y="95" text="Open Session" @click="openSession" />
      <Link v-for="link in lines" v-bind:key="link.index"
        :x1="link.x1" :y1="link.y1" :x2="link.x2" :y2="link.y2" />
      <Artifact v-for="artifact in artifacts" :artifact="artifact"
        v-bind:key="artifact.index" @alter="alterArtifact"
        @add="addArtifact" @remove="removeArtifact" />
    </svg>
    <!-- <output><output>{{links}}</output></pre> -->
    <pre><output>{{sessionData}}</output></pre>
  </div>
</template>

<script>
import Artifact from './Artifact.vue';
import Link from './Link';
import Button from './Button';
import TestRunner from '../TestRunner';
import Model from '../Model';
import {session} from '../Session';
import {db} from '../Fire';

const keymap = {
  movements: {
    'KeyA': 'left',
    'KeyW': 'top',
    'KeyD': 'right',
  },
  help: {
    'F1': 'general',
  },
  action: {
    'KeyR': 'rename',
    'KeyS': 'follow',
    'Space': 'describe',
    'KeyC': 'connections',
    'KeyV': 'neighbors',
    'Minus': 'decrease_speech_rate',
    'Equal': 'increase_speech_rate',
    'IntlRo': 'help',
    'KeyU': 'presence',
  },
};

export default {
  name: 'Workspace',
  data: function() {
    return {
      artifacts: Model.artifacts,
      links: Model.links,
      sessionData: [],
    };
  },
  computed: {
    lines: function() {
      return this.links.map((link) => {
        const [one, another] = [Model.find(link[0]), Model.find(link[1])];
        return {
          x1: one.x, y1: one.y,
          x2: another.x, y2: another.y,
        };
      });
    },
  },
  methods: {
    openSession(e) {
      this.sessionData.splice(0);
      const name = prompt('Session name:');
      db.store.collection('sessions').where('name', '==', name).get()
          .then((snap) => {
            snap.forEach((doc) => {
              db.store.collection(`sessions/${doc.id}/interactions`)
                  .orderBy('sequence', 'asc').get().then((qs) => {
                    qs.forEach((ints) => {
                      this.sessionData.push(ints.data());
                    });
                  });
            });
          });
    },
    newSession(e) {
      if (!session.state.selected) return alert('no user');
      const userId = session.state.selected;
      Model.newSession({userId});
    },
    keyup(e) {
      // console.log(e.code);
      if (e.code in keymap.movements) {
        Model.moveFocus(keymap.movements[e.code]);
      }
      if (e.code in keymap.help) {
        Model.helpWith(keymap.help[e.code]);
      }
      if (e.code in keymap.action) {
        Model.action(keymap.action[e.code]);
      }
    },
    run() {
      TestRunner.run({count: 3, evaluate: false});
    },
    step() {
      TestRunner.run({count: 10, evaluate: true});
    },
    dragOver(e) {
      e.preventDefault();
    },
    drop(e) {
      e.preventDefault();
      const id = e.dataTransfer.getData('id');
      const x = parseInt(e.dataTransfer.getData('x'));
      const y = parseInt(e.dataTransfer.getData('y'));
      const art = Model.find(id);
      if (art) [art.x, art.y] = [art.x + (e.x - x), art.y + (e.y - y)];
    },
    alterArtifact(e) {
      const userId = session.state.selected || e.user;
      const options = {newName: e.newName};
      Model.alter(e.source.id, userId, options);
    },
    removeArtifact(e) {
      const userId = session.state.selected || e.user;
      Model.remove(e.source.id, userId);
    },
    addArtifact(e) {
      const art = e.add || {
        name: e.name || 'Nova',
        x: e.source.x + (e.source.step ?
          (++e.source.step * 120) : ((e.source.step = 1)* 120)),
        y: e.source.y + 100,
      };
      const fromId = e.source.id;
      const userId = session.state.selected || e.user;
      Model.add(art, fromId, userId);
    },
  },
  props: {
    width: Number,
    height: Number,
  },
  components: {
    Artifact, Link, Button,
  },
  created() {
    window.removeEventListener('keyup', this.keyup);
    window.addEventListener('keyup', this.keyup);
    // Model.add({name: 'Raiz', x: 100, y: 10});
    TestRunner.setup(this);
  },
};
</script>

<style scoped>
  div.workspace {
    width: 100%;
    height: 100%;
  }
  svg {
      background-color: #EEE;
  }
</style>
