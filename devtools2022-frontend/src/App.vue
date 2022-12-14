<template>
  <el-container>
    <el-main>
      <UserProfile
        @login-user="loginUser"
        @logout-user="logoutUser"
        :user="user"
        ref="userProfile"
      />
      <TypingTest
        :timeLeft="timeLeft"
        :testWords="testWords"
        :gameState="gameState"
        @start-test="startGame"
        @reset-test="resetGame"
        :timeLimit="timeLimit"
        :timePassed="timePassed"
        @finish-test="finishGame"
        ref="typingTest"
        @update-score="updateScore"
      />
      <AddCustomTest @add-test="addTest" v-show="user.login !== ''" />
      <CustomTests
        @run-test="runTest"
        @save-edit-test="saveEditTest"
        @delete-test="deleteTest"
        :tests="tests"
        v-show="user.login !== ''"
      />
    </el-main>
  </el-container>
</template>

<script>
import UserProfile from "./components/UserProfile.vue";
import CustomTests from "./components/CustomTests.vue";
import AddCustomTest from "./components/AddCustomTest.vue";
import TypingTest from "./components/TypingTest.vue";
import "element-plus/es/components/message/style/css";
import "element-plus/es/components/message-box/style/css";
import { ElMessage, ElMessageBox } from "element-plus";
import Service from "./Service.js";
import RandomWordsGen from "./RandomWordsGen.js";

export default {
  name: "App",
  components: {
    UserProfile,
    CustomTests,
    AddCustomTest,
    TypingTest,
  },
  data() {
    return {
      tests: [],
      timeLimit: 25,
      timePassed: 0,
      timerInterval: null,
      testWords: [],
      gameState: "waiting", //waiting, playing, finished
      user: {
        login: "",
        displayName: "",
        password: "",
        highscore: 0,
      },
    };
  },
  methods: {
    // Custom Test functionalities
    async addTest(test) {
      var data = test;
      data.owner = this.user.login;
      Service.createTest(data)
        .then(() => {
          this.getUserTests().then((response) => {
            this.tests = response;
            ElMessage({
              type: "success",
              message: "Add successful.",
            });
          });
        })
        .catch((error) => {
          ElMessage({
            type: "error",
            message: `Something went wrong. ${error}`,
          });
        });
    },
    async runTest(id) {
      Service.getTestByID(id).then((response) => {
        var words = RandomWordsGen.convertToArray(response.words);
        this.testWords = RandomWordsGen.shuffleWords(words);
        this.resetTimer();
        this.gameState = "waiting";
        this.$refs.typingTest.clearTypingTest();
      });
    },
    async saveEditTest(test) {
      Service.editTest(test)
        .then(() => {
          this.getUserTests().then((response) => {
            this.tests = response;
            ElMessage({
              type: "success",
              message: "Edit successful.",
            });
          });
        })
        .catch((error) => {
          ElMessage({
            type: "error",
            message: `Something went wrong. ${error}`,
          });
        });
    },
    async deleteTest(id) {
      ElMessageBox.confirm(
        "Are you sure you want to delete this test?",
        "Warning",
        {
          confirmButtonText: "OK",
          cancelButtonText: "Cancel",
          type: "warning",
        }
      )
        .then(() => {
          Service.deleteTest(this.user.login, id).then(() => {
            this.getUserTests().then((response) => {
              this.tests = response;
              ElMessage({
                type: "success",
                message: "Delete completed.",
              });
            });
          });
        })
        .catch((err) => {
          ElMessage({ type: "info", message: "Delete canceled." });
          console.log(err);
        });
    },
    async getUserTests() {
      Service.getTestsFromUser(this.user.login).then((response) => {
        console.log(response);
        this.tests = response;
      });
    },

    // User functionalities
    loginUser() {
      // save locally the current logged in user
      // get the tests of user logged in
      this.saveLocalUser(this.user);
      this.getUserTests();
    },

    logoutUser() {
      // reset local user
      var emptyUser = { login: "", password: "" };
      this.saveLocalUser(emptyUser);
      this.tests = [];
    },

    loadLocalUser() {
      // var emptyUser = { login: "", password: "" };
      // const data = JSON.stringify(emptyUser);
      // window.localStorage.setItem("localUser", data);

      // var user = window.localStorage.getItem("localUser");
      // console.log(user);

      var user = window.localStorage.getItem("localUser");
      if (user === null) {
        var emptyUser = { login: "", password: "" };
        const data = JSON.stringify(emptyUser);
        window.localStorage.setItem("localUser", data);
        return emptyUser;
      }

      return JSON.parse(user);
    },

    saveLocalUser(user) {
      window.localStorage.setItem(
        "localUser",
        JSON.stringify({
          login: user.login,
          password: user.password,
        })
      );
    },

    // Timer functionalities
    startTimer() {
      this.timerInterval = setInterval(() => (this.timePassed += 1), 1000);
    },
    onTimesUp() {
      clearInterval(this.timerInterval);
    },
    changeTime() {},
    resetTimer() {
      this.timePassed = 0;
      clearInterval(this.timerInterval);
    },

    // Game Loop
    startGame() {
      console.log("startGame");
      this.gameState = "playing";
      this.startTimer();
    },
    resetGame() {
      this.testWords = RandomWordsGen.getWords(200);
      this.resetTimer();
      this.gameState = "waiting";
      console.log(`resetGame ${this.gameState}`);
    },
    finishGame() {
      this.onTimesUp();
      this.gameState = "finished";
      console.log(this.gameState);
    },
    updateScore(score) {
      console.log(score);
      if (score > this.user.highscore) {
        this.user.highscore = score;
        Service.updateUser(this.user);
      }
    },
  },
  async created() {
    // try {
    //   this.tests = await Service.getTests();
    // } catch (error) {
    //   console.log(error.message);
    // }

    this.testWords = RandomWordsGen.getWords(200);
    console.log(this.testWords);
  },
  mounted() {
    //this.getUserTests();
    // this.saveLocalUser();

    var user = this.loadLocalUser();
    console.log(user);
    this.user.login = user.login;
    this.user.password = user.password;
    this.$refs.userProfile.onMount();
  },
  computed: {
    timeLeft() {
      return this.timeLimit - this.timePassed;
    },
  },

  watch: {
    timeLeft(newValue) {
      if (newValue === 0) {
        this.onTimesUp();
        this.gameState = "finished";
        console.log(this.gameState);
      }
    },
  },
};
</script>

<style>
@import url("https://fonts.googleapis.com/css?family=Roboto+Mono|Inconsolata");

#app {
  font-family: "Inconsolata", Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #aec4df;
  font-size: larger;
}
button,
input,
select,
textarea {
  font-family: inherit;
  font-size: inherit;
  line-height: inherit;
  color: inherit;
}
.el-header,
.el-footer {
  background-color: #b3b3d1;
  color: #333;
  text-align: center;
  /* line-height: 60px; */
}

.el-main {
  background-color: #dedceb;
  color: #333;
  text-align: center;
  /* line-height: 160px; */
}

body > .el-container {
  margin-bottom: 40px;
}

body {
  background-color: #c9c5e6;
}
</style>
