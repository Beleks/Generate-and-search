<template>
  <div>
    <button class="generate-button" :disabled="!!progress" @click="generateStrings">Сгенерировать</button>
    <div class="progress-bar">
      <div class="progress-bar-saving__fill" :style="{ width: progressSaving + '%' }"></div>
      <div class="progress-bar__fill" :style="{ width: progress + '%' }"></div>
    </div>
    <!-- TODO: Значения загружаются не сразу. Желательно показывать текст, что идёт загрузка -->
    <div>Сгенерировано: {{ strings.length }} Сохранено: {{ dataSavingLength }}</div>

    <input class="search-input" type="text" @input="performSearch" v-model="searchTerm" placeholder="Поиск..." />

    <div>
      <div class="table" v-if="displayedStrings.length > 0">
        <div class="string" v-for="result in paginatedStrings" :key="result">
          {{ result }}
        </div>
      </div>
      <div v-else>
        <div class="empty-text">Нет результатов</div>
      </div>
    </div>

    <div v-if="displayedStrings.length > 0">
      <!-- Проблемс: Когда количество страниц будет большим, это будет очень неудобно. TODO: Исправить, как будет время -->
      <PaginationTable :pages="totalPages" :current="currentPage" @changePage="changePage" />
    </div>
  </div>
</template>

<script>
import PaginationTable from "./PaginationTable.vue";

export default {
  components: {
    PaginationTable,
  },
  data() {
    return {
      totalStrings: 100_000,
      batchSize: 1000,
      dataSavingLength: 0,
      searchTerm: "",
      strings: [],
      displayedStrings: [],
      currentPage: 1,
      resultsPerPage: 30,
      dbConfig: {
        name: "testDB",
        objName: "strings",
      },
    };
  },
  watch: {
    dataSavingLength() {
      if (this.dataSavingLength === this.totalStrings) {
        localStorage.removeItem("inProgress");
      }
    },
  },
  computed: {
    totalPages() {
      return Math.ceil(this.displayedStrings.length / this.resultsPerPage);
    },
    paginatedStrings() {
      const startIndex = (this.currentPage - 1) * this.resultsPerPage;
      const endIndex = startIndex + this.resultsPerPage;

      return this.displayedStrings.slice(startIndex, endIndex);
    },
    progress() {
      return (this.strings.length / this.totalStrings) * 100;
    },
    progressSaving() {
      return (this.dataSavingLength / this.totalStrings) * 100;
    },
  },
  methods: {
    generateStrings() {
      let i = this.strings.length;

      localStorage.setItem("inProgress", true);

      const generationInterval = setInterval(() => {
        const batchData = this.generateBatchData(this.batchSize);
        this.strings = this.strings.concat(batchData);

        i += this.batchSize;
        this.saveDataToIndexedDB(batchData);

        if (i >= this.totalStrings) {
          clearInterval(generationInterval);
        }
      }, 0);
    },
    generateBatchData(batchSize) {
      const batchData = [];
      for (let i = 0; i < batchSize; i++) {
        batchData.push(this.generateRandomString(100));
      }
      return batchData;
    },
    generateRandomString(length) {
      const characters = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
      let result = "";

      for (let i = 0; i < length; i++) {
        const randomIndex = Math.floor(Math.random() * characters.length);
        result += characters.charAt(randomIndex);
      }

      return result;
    },
    performSearch() {
      if (this.searchTerm.length) {
        // TODO: debounce тоже неплохо бы добавить
        this.displayedStrings = this.strings.filter((result) => result.includes(this.searchTerm));
      } else {
        this.displayedStrings = [];
      }
    },
    saveDataToIndexedDB(strings) {
      const request = window.indexedDB.open(this.dbConfig.name, 1);

      request.onerror = function (event) {
        console.error("Ошибка открытия базы данных", event.target.error);
      };

      request.onupgradeneeded = function (event) {
        const db = event.target.result;
        if (!db.objectStoreNames.contains(this.dbConfig.objName)) {
          db.createObjectStore(this.dbConfig.objName, { keyPath: "id", autoIncrement: true });
        }
      }.bind(this);

      request.onsuccess = function (event) {
        const db = event.target.result;
        const transaction = db.transaction(this.dbConfig.objName, "readwrite");
        const objectStore = transaction.objectStore(this.dbConfig.objName);

        let addDataRequest;

        for (const string of strings) {
          addDataRequest = objectStore.add({ value: string });
        }

        addDataRequest.onsuccess = function () {
          this.dataSavingLength += this.batchSize;
          console.log("Данные успешно добавлены в IndexedDB");
        }.bind(this);

        addDataRequest.onerror = function (event) {
          console.error("Ошибка при добавлении данных", event.target.error);
        };

        transaction.oncomplete = function () {
          db.close();
        };

        transaction.onerror = function (event) {
          console.error("Ошибка", event.target.error);
        };
      }.bind(this);
    },
    loadDataFromIndexedDB() {
      const request = window.indexedDB.open(this.dbConfig.name, 1);

      request.onerror = function (event) {
        console.error("Ошибка открытия базы данных", event.target.error);
      };

      request.onupgradeneeded = function (event) {
        const db = event.target.result;
        if (!db.objectStoreNames.contains(this.dbConfig.objName)) {
          db.createObjectStore(this.dbConfig.objName, { keyPath: "id", autoIncrement: true });
        }
      }.bind(this);

      request.onsuccess = function (event) {
        const db = event.target.result;

        const transaction = db.transaction(this.dbConfig.objName, "readonly");
        const objectStore = transaction.objectStore(this.dbConfig.objName);

        const getDataRequest = objectStore.getAll();

        getDataRequest.onsuccess = function (event) {
          if (event.target.result && event.target.result.length > 0) {
            this.strings = event.target.result.map((item) => item.value);
            this.dataSavingLength = this.strings.length;
            console.log("Данные успешно загружены из IndexedDB");
          } else {
            console.log("Данные в IndexedDB не найдены");
          }
          if (localStorage.getItem("inProgress")) {
            this.generateStrings();
          }
        }.bind(this);

        getDataRequest.onerror = function (event) {
          console.error("Ошибка при чтении данных", event.target.error);
        };

        db.close();
      }.bind(this);
    },
    changePage(newPage) {
      this.currentPage = newPage;
    },
  },
  mounted() {
    this.loadDataFromIndexedDB();
  },
};
</script>

<style>
.generate-button {
  padding: 0.5rem 1rem;
  width: 200px;
  background-color: #e28050;
  text-align: center;
  cursor: pointer;
  border-radius: 5px;
}

button:disabled.generate-button {
  background-color: gray;
  cursor: not-allowed;
}

.search-input {
  padding: 0.5rem;
  width: 1000px;
  border-bottom: 1px solid white;
  margin-bottom: 0.5rem;
}

input.search-input:focus {
  border-color: #e28050;
}

.progress-bar {
  margin-top: 0.5rem;
  margin-bottom: 0.5rem;
  width: 100%;
  height: 10px;
  background-color: #f2f2f2;
  border-radius: 5px;
  overflow: hidden;
  position: relative;
}

.progress-bar__fill {
  position: absolute;
  height: 100%;
  background-color: #e28150ab;
  transition: width 0.3s;
  z-index: 5;
}

.progress-bar-saving__fill {
  position: absolute;
  height: 100%;
  background-color: #e28150;
  transition: width 0.3s;
  z-index: 10;
}

.table {
  border: 1px solid gray;
  border-radius: 5px;
}

.empty-text {
  text-align: center;
  padding: 1rem;
}

.string {
  display: flex;
  padding: 0.2rem 0.2rem;
  border-bottom: 1px solid gray;
}
.string:last-child {
  border-bottom: none;
}
</style>
