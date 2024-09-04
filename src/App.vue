<!-- App.vue -->
<template>
  <div class="container">
    <div v-if="!isSignedIn" class="d-flex justify-content-center align-items-center vh-100">
      <div class="card p-4">
        <h2 class="card-title text-center">{{ $t('login.title') }}</h2>
        <form @submit.prevent="handleSignIn">
          <div class="mb-3">
            <label for="username" class="form-label">{{ $t('login.username') }}</label>
            <input type="text" v-model="username" class="form-control" required/>
          </div>
          <div class="mb-3">
            <label for="password" class="form-label">{{ $t('login.password') }}</label>
            <input type="password" v-model="password" class="form-control" required/>
          </div>
          <button type="submit" class="btn btn-primary w-100">{{ $t('login.enter') }}</button>
        </form>
        <p v-if="error" class="text-danger mt-3">{{ error }}</p>
      </div>
    </div>
    <div v-else class="vh-100 counter align-content-center">
      <h1 class="responsive-font">{{ counter }}</h1>
      <p class="user-name">{{ username }}</p>
      <button @click="incrementCounter" class="btn btn-lg btn-success me-3">{{ $t('info.increment') }}</button>
      <button @click="signOut" class="btn btn-lg btn-danger ms-3">{{ $t('info.exit') }}</button>
    </div>
  </div>
</template>

<script lang="ts">
import {defineComponent, ref, onMounted} from 'vue';
import initSqlJs, {Database} from 'sql.js';
import CryptoJS from 'crypto-js';

const DATABASE_NAME: string = "mrp_database";

export default defineComponent({
  name: 'App',
  data() {
    return {
      username: ref(''),
      password: ref(''),
      isSignedIn: ref(false),
      counter: ref(0),
      error: '',
      db: null as Database | null
    };
  },
  mounted() {
    this.initDb();
  },
  methods: {
    async initDb() {
      const SQL = await initSqlJs();
      const savedDb = localStorage.getItem(DATABASE_NAME);
      if (savedDb) {
        const uint8Array = new Uint8Array(JSON.parse(savedDb));
        this.db = new SQL.Database(uint8Array);
      } else {
        this.db = new SQL.Database();
        this.db.run("CREATE TABLE IF NOT EXISTS account (username TEXT, password TEXT, counter INTEGER)");
      }
    },
    handleSignIn() {
      const encryptedPassword = CryptoJS.SHA512(this.password).toString();
      const userCount = this.db.exec(`SELECT count(*)
                                      FROM account
                                      WHERE username = '${this.username}'`)[0].values[0][0];

      if (userCount === 0) {
        this.db.run(`INSERT INTO account (username, password, counter)
                     VALUES ('${this.username}', '${encryptedPassword}', 0)`);
        this.counter = 0;
      } else {
        const user = this.db.exec(`SELECT username, password, counter
                                   FROM account
                                   WHERE username = '${this.username}'`);
        const storedPassword = user[0].values[0][1];
        if (storedPassword === encryptedPassword) {
          this.counter = user[0].values[0][2];
        } else {
          this.error = this.$t('error.invalidPassword');
          return;
        }
      }
      this.isSignedIn = true;
      this.saveDb();
    },
    incrementCounter() {
      this.counter++;
      this.db.run(`UPDATE account
                   SET counter = ${this.counter}
                   WHERE username = '${this.username}'`);
      this.saveDb();
    },
    signOut() {
      this.isSignedIn = false;
      this.username = '';
      this.password = '';
      this.error = '';
    },
    saveDb() {
      const data = this.db.export();
      const buffer = Array.from(data);
      localStorage.setItem(DATABASE_NAME, JSON.stringify(buffer));
    }
  }
});
</script>

<style>
.container {
  max-width: 600px;
  margin: 0 auto;
  padding-top: 50px;
}

.sign-in-form, .counter {
  text-align: center;
}

.responsive-font {
  font-size: 20vw;
}

.user-name {
  font-size: 5vw;
  font-style: italic;
}

</style>
