<template>
  <div class="container">
    <form
      class="d-flex justify-content-center p-2"
      @submit.prevent="getFormInfo"
    >
      <input
        v-model="state.category_id"
        type="text"
        class="form-control w-25 m-2"
        placeholder="Введите ID категории"
      />
      <button type="submit" class="btn btn-primary m-2">
        Загрузить атрибуты
      </button>
    </form>

    <div
      v-if="state.is_loading"
      class="d-flex w-100 justify-content-center mt-3"
    >
      <div class="spinner-border" role="status" />
    </div>
    <form
      v-else-if="state.attributes"
      @submit.prevent="generateJSON"
      class="d-flex flex-column align-items-center mt-3"
    >
      <div class="d-flex flex-column align-items-center p-3 border w-100">
        <div
          v-for="(attr, idx) in state.attributes"
          :key="`attr-${idx}`"
          class="w-50"
        >
          <vue-select
            v-if="attr.is_collection"
            v-model="attr.selected"
            :searchable="attr.options && attr.options.length >= 20"
            :placeholder="attr.name"
            :search-placeholder="attr.name"
            :options="attr.options || []"
            close-on-select
            clear-on-select
            class="p-2 mt-2 w-100 form-control"
          />
          <input
            v-else-if="!attr.is_collection"
            v-model="attr.selected"
            type="text"
            :placeholder="attr.name"
            :required="attr.is_required"
            class="p-2 mt-2 w-100 form-control"
          />
        </div>
      </div>

      <button type="submit" class="btn btn-primary mt-3">Сохранить JSON</button>
    </form>
    <div v-else class="d-flex flex-column align-items-center mt-3">
      Нет атрибутов, введите ID категории
    </div>

    <form
      @submit.prevent="fillCardFromJSON"
      class="d-flex flex-column align-items-center mt-5"
    >
      <textarea
        v-model="state.json"
        name="json"
        id=""
        rows="10"
        class="w-100"
      />
      <button type="submit" class="btn btn-primary mt-3">
        Загрузить из JSON
      </button>
    </form>
  </div>
</template>

<script>
import axios from "axios";
import { reactive } from "@vue/runtime-core";

import VueSelect from "vue-next-select";
import "vue-next-select/dist/index.min.css";

import { createToast } from "mosha-vue-toastify";

export default {
  name: "App",
  components: {
    VueSelect,
  },
  setup() {
    const state = reactive({
      category_id: null,
      attributes: null,
      json: null,
      query: null,
      is_loading: false,
    });

    const searchOption = (options) => {
      const query = state.query?.toLowerCase();

      if (query) {
        return options.filter((option) => {
          return String(option.name).toLowerCase().includes(query);
        });
      }
    };

    const getFormInfo = async () => {
      try {
        state.is_loading = true;

        axios
          .post(
            "https://api-dev.shop-delivery.ru/int_api/ozon/v3/category/attribute",
            {
              attribute_type: "ALL",
              category_id: [+state.category_id],
              language: "DEFAULT",
            }
          )
          .then((resp) => {
            if (!resp.data.result) {
              state.is_loading = false;
              return;
            }

            state.attributes = resp.data.result[0].attributes;

            state.attributes.forEach((attr) => {
              attr.selected = null;
              if (attr.is_collection) {
                getOptions(attr);
                attr.selected = attr.name;
              }
            });

            state.is_loading = false;
          });
      } catch (e) {
        console.log(e);
        createToast(
          "Ошибка!",
          "При получении атрибутов возникла ошибка",
          "danger"
        );

        state.is_loading = false;
      }
    };

    const getOptions = async (attr) => {
      axios
        .post(
          "https://api-dev.shop-delivery.ru/int_api/ozon/v2/category/attribute/values",
          {
            attribute_id: attr.id,
            category_id: +state.category_id,
            language: "DEFAULT",
            last_value_id: 0,
            limit: 20,
            query: "",
          }
        )
        .then((resp) => {
          attr.options = resp.data.map((el) => el.value);
        });
    };

    const generateJSON = async () => {
      state.json = [];

      state.attributes.forEach((attr) => {
        if (attr.selected && attr.selected !== attr.name)
          state.json.push({ name: attr.name, value: attr.selected });
      });
      state.json = JSON.stringify(state.json);

      createToast("Вы сгенерировали JSON.");
    };

    const fillCardFromJSON = async () => {
      let obj = JSON.parse(state.json);

      if (obj) {
        obj.forEach((selected) => {
          state.attributes.forEach((attr) => {
            if (attr.name === selected.name) attr.selected = selected.value;
          });
        });

        createToast("Вы заполнили карточку из JSON.");
      }
    };

    return {
      state,
      getFormInfo,
      generateJSON,
      fillCardFromJSON,
      searchOption,
    };
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  margin: 60px 0;
}

@import "~bootstrap/dist/css/bootstrap.css";
</style>
