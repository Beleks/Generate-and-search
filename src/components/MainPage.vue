<template>
  <div>
    <button class="generate-button" :disabled="!!progress" @click="generateStrings">Сгенерировать</button>
    <div class="progress-bar">
      <div class="progress-bar__fill" :style="{ width: progress + '%' }"></div>
    </div>
    <div>Сгенерировано: {{ strings.length }}</div>

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
      totalStrings: 10000000,
      searchTerm: "",
      strings: [],
      displayedStrings: [],
      currentPage: 1,
      resultsPerPage: 30,
    };
  },
  computed: {
    totalPages() {
      return Math.ceil(this.displayedStrings.length / this.resultsPerPage);
    },
    paginatedStrings() {
      const startIndex = (this.currentPage - 1) * this.resultsPerPage;
      const endIndex = startIndex + this.resultsPerPage;

      return this.strings.slice(startIndex, endIndex);
    },
    progress() {
      return (this.strings.length / this.totalStrings) * 100;
    },
  },
  methods: {
    generateStrings() {
      // Подход к реализации генерации и сохранения не лучший :(

      // Наверное, лучше бы это делать партиями, скажем, по 1000, если нам важна скорость.
      // Альтернативный вариант сохранения: Событие покидание страницы можно отловить и перед этим все записать в LS.
      // Но если мы закроем страницу, не находясь на ней, то у нас ничего не сохранится.
      // Поэтому выбрал такой вариант исходя из условий тестового

      // Возможно, к моменту проверки я все-таки доделаю вариант с использованием Web Workers, в другой ветке. Так, для эксперимента )
      localStorage.setItem("inProgress", true);
      let i = this.strings.length;

      let generationInterval;
      generationInterval = setInterval(() => {
        if (i < this.totalStrings) {
          const randomString = this.generateRandomString(100);
          this.strings.push(randomString);

          if (this.searchTerm.length) {
            this.performSearch();
          }

          // В любом случае у нас не получится сохранить 10 миллионов строк на клиенте. Даже если использовать indexDB
          try {
            localStorage.setItem("data", JSON.stringify(this.strings));
          } catch (error) {
            clearInterval(generationInterval);
            alert(`Ошибка при сохранения в LocalStorage: ${error}`);
            return;
          }

          i++;
        } else {
          localStorage.removeItem("inProgress");
          clearInterval(generationInterval);
        }
      }, 0);
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
      this.displayedStrings = this.strings.filter((result) => result.includes(this.searchTerm));
    },
    loadResults() {},
    changePage(newPage) {
      this.currentPage = newPage;
    },
  },
  mounted() {
    const savedData = localStorage.getItem("data");

    if (savedData) {
      try {
        this.strings = JSON.parse(savedData);
      } catch (error) {
        alert(`Ошибка при загрузке данных из LocalStorage: ${error}`);
      }
    }
    if (localStorage.getItem("inProgress")) {
      this.generateStrings();
    }
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
}

.progress-bar__fill {
  height: 100%;
  background-color: #e28050;
  transition: width 0.3s;
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
