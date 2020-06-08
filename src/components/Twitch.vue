<template>
  <div class="twitch">
    <div id="player-view">
      <vue-twitch-player
        ref="player"
        width="960"
        height="540"
        quality="auto"
        :volume="volume"
        :video="video"
        @ready="ready"
      ></vue-twitch-player>
      <div id="caption">{{ caption }}</div>
    </div>
    <hr />
    <div>
      <input
        type="checkbox"
        id="preview"
        name="preview"
        v-model="previewMode"
      />
      <label for="preview">プレビューモード</label> |
      <button @click="beginning()">最初から再生</button> |
      <button @click="insertCurrentTime()">ここから30秒を挿入</button>
    </div>
    <textarea v-model="script" id="script" ref="script"></textarea>
    <div>再生時間 : {{ time }}</div>
  </div>
</template>

<script>
import dayjs from "dayjs";
import utc from "dayjs/plugin/utc";
import VueTwitchPlayer from "vue-twitch-player";
import demoText from "../demoText.js";

dayjs.extend(utc);

let captionTimer = null;

export default {
  name: "Twitch",

  props: {
    video: String
  },

  data() {
    return {
      currentTime: 0,
      volume: 1,
      previewMode: false,
      caption: "",
      script: demoText,
      timeline: []
    };
  },

  computed: {
    time() {
      const time = this.timeline.reduce((acc, val) => {
        if (val.type === "video") return acc + (val.out - val.in);
        else return acc;
      }, 0);
      const format = dayjs.unix(time).utc();
      return format.format("HH:mm:ss");
    }
  },

  watch: {
    script() {
      this.parseScript();
    },

    previewMode() {
      this.captionClear();
    },

    currentTime(time) {
      if (!this.previewMode) return;

      // 範囲内？
      const inRange = this.timeline.find(
        range => range.type === "video" && range.in <= time && time < range.out
      );

      // キャプション表示？
      if (inRange) {
        const caption = this.timeline.find(
          range => range.type === "caption" && range.in === time
        );

        if (caption) {
          this.captionClear();
          this.volume = 0.02;
          this.caption = caption.msg;
          setTimeout(this.captionClear, 6000);
        }

        // スキップ対象範囲？
      } else {
        const skipped = this.timeline.find(range => {
          if (range.type === "video" && time < range.in) {
            this.$refs.player.seek(range.in);
            return true;
          }
        });

        if (!skipped) {
          // this.$refs.player.seek(this.timeline[0].in);
          this.captionClear();
          this.$refs.player.pause();
          this.$refs.player.seek(0);
        }
      }
    }
  },

  methods: {
    parseScript() {
      if (!this.script) return;
      const lines = this.script.split("\n");

      this.timeline = [];

      lines.forEach(line => {
        // 行頭が//ならコメント
        if (/^ *\/\//.test(line)) {
          return true;

          // 行頭が#なら動画範囲
        } else if (/^ *#/.test(line)) {
          const match = line
            .replace(/ /g, "")
            .match(/(?<in>[\d:]+) *- *(?<out>[\d:]+)/i);

          if (match && match.groups.in && match.groups.out) {
            const range = [match.groups.in, match.groups.out].map(time =>
              this.parseTimeInSeconds(time)
            );

            this.timeline.push({
              type: "video",
              in: range[0],
              out: range[1]
            });
          }

          // キャプション
        } else {
          if (line.trim() !== "") {
            let last = this.timeline[this.timeline.length - 1];
            let inTime = last ? last.in : 0;

            // in 指示
            if (/^ *\(/.test(line)) {
              const match = line.match(/\((?<in>[\d:]+)\)/);

              if (match && match.groups.in) {
                inTime = this.parseTimeInSeconds(match.groups.in);
                line = line.replace(match[0], "").trim();
                last = null;
              }
            }

            if (last && last.type === "caption") {
              this.timeline[this.timeline.length - 1].msg +=
                (last.msg === "" ? "" : "\n") + line;
            } else {
              this.timeline.push({
                type: "caption",
                msg: line,
                in: inTime
              });
            }
          }
        }
      });
    },

    parseTimeInSeconds(time) {
      time = time.match(/:/g).length === 1 ? `00:${time}` : time;
      return Date.parse(`01 Jan 1970 ${time} GMT`) / 1000;
    },

    getCurrentTime() {
      this.currentTime = parseInt(this.$refs.player.getCurrentTime(), 10);
    },

    beginning() {
      this.captionClear();
      this.$refs.player.seek(0);
      this.$refs.player.play();
    },

    captionClear() {
      this.volume = 1;
      this.caption = "";
      clearTimeout(captionTimer);
    },

    insertCurrentTime() {
      const time = dayjs.unix(this.currentTime).utc();

      const i = time.format("HH:mm:ss");
      const o = time.add(30, "s").format("HH:mm:ss");

      this.script = `${this.script}\n\n# ${i}-${o}\n`;
      this.$refs.script.focus();
    },

    ready() {
      this.$refs.player.seek(0);
      this.$refs.player.pause();
      this.parseScript();
      setInterval(this.getCurrentTime, 300);
    }
  },

  components: {
    VueTwitchPlayer
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}

#player-view {
  line-height: 0;
}

#caption {
  width: 960px;
  height: 100px;
  margin: auto;
  padding: 0 12px;
  box-sizing: border-box;
  background: #111;
  color: #eee;
  font-size: 23px;
  line-height: 1.25;
  white-space: pre-wrap;
  display: flex;
  align-items: center;
  justify-content: center;
}

#script {
  width: 960px;
  height: 400px;
}

textarea {
  margin-top: 10px;
}
</style>
