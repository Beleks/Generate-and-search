<template>
  <div class="pagination">
    <button :disabled="current === 1" @click="changePage(current - 1)">Prev</button>
    <span
      v-for="pageNumber in pages"
      :key="pageNumber"
      @click="changePage(pageNumber)"
      :class="{ active: pageNumber === current }"
      >{{ pageNumber }}</span
    >
    <button :disabled="current === pages" @click="changePage(current + 1)">Next</button>
  </div>
</template>

<script>
export default {
  name: "PaginationTable",
  props: {
    pages: {
      type: Number,
      required: true,
    },
    current: {
      type: Number,
      required: true,
    },
  },
  methods: {
    changePage(pageNumber) {
      this.$emit("changePage", pageNumber);
    },
  },
};
</script>

<style>
.pagination button {
  cursor: pointer;
  padding: 0.2rem;
}

.pagination button:hover {
  cursor: pointer;
  color: #e28050;
}

.pagination {
  margin-top: 10px;
}

.pagination span {
  display: inline-block;
  margin: 0 5px;
  cursor: pointer;
  /* color: #000; */
}

.pagination span.active {
  font-weight: bold;
  color: #e28050;
}
</style>
