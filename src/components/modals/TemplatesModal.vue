<template>
  <div class="modal__inner-1 modal__inner-1--templates" role="dialog" aria-label="Manage templates">
    <div class="modal__inner-2">
      <div class="form-entry">
        <label class="form-entry__label" for="template">Template</label>
        <div class="form-entry__field">
          <input v-if="isEditing" id="template" type="text" class="textfield" v-focus @blur="submitEdit()" @keyup.enter="submitEdit()" @keyup.esc.stop="submitEdit(true)" v-model="editingName">
          <select v-else id="template" v-model="selectedId" class="textfield">
            <option v-for="(template, id) in templates" :key="id" :value="id">
              {{ template.name }}
            </option>
          </select>
        </div>
        <div class="form-entry__actions flex flex--row flex--end">
          <button class="form-entry__button button" @click="create">
            <icon-file-plus></icon-file-plus>
          </button>
          <button class="form-entry__button button" @click="copy">
            <icon-file-multiple></icon-file-multiple>
          </button>
          <button v-if="!isReadOnly" class="form-entry__button button" @click="isEditing = true">
            <icon-pen></icon-pen>
          </button>
          <button v-if="!isReadOnly" class="form-entry__button button" @click="remove">
            <icon-delete></icon-delete>
          </button>
        </div>
      </div>
      <div class="form-entry">
        <label class="form-entry__label">Value</label>
        <div class="form-entry__field" v-for="(template, id) in templates" :key="id" v-if="id === selectedId">
          <code-editor lang="handlebars" :value="template.value" :disabled="isReadOnly" @changed="template.value = $event"></code-editor>
        </div>
      </div>
      <div v-if="!isReadOnly">
        <a href="javascript:void(0)" v-if="!showHelpers" @click="showHelpers = true">Add helpers ▾</a>
        <div class="form-entry" v-else>
          <br>
          <label class="form-entry__label">Helpers</label>
          <div class="form-entry__field" v-for="(template, id) in templates" :key="id" v-if="id === selectedId">
            <code-editor lang="javascript" :value="template.helpers" @changed="template.helpers = $event"></code-editor>
          </div>
        </div>
      </div>
      <div class="modal__button-bar">
        <button class="button" @click="config.reject()">Cancel</button>
        <button class="button" @click="resolve()">Ok</button>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import utils from '../../services/utils';
import CodeEditor from '../CodeEditor';
import emptyTemplateValue from '../../data/emptyTemplateValue.html';
import emptyTemplateHelpers from '!raw-loader!../../data/emptyTemplateHelpers.js'; // eslint-disable-line

const collator = new Intl.Collator(undefined, { sensitivity: 'base' });

function fillEmptyFields(template) {
  if (template.value === '\n') {
    template.value = emptyTemplateValue;
  }
  if (template.helpers === '\n') {
    template.helpers = emptyTemplateHelpers;
  }
}

export default {
  components: {
    CodeEditor,
  },
  data: () => ({
    selectedId: '',
    templates: {},
    showHelpers: false,
    isEditing: false,
    editingName: '',
  }),
  computed: {
    ...mapGetters('modal', [
      'config',
    ]),
    isReadOnly() {
      return this.templates[this.selectedId].isAdditional;
    },
  },
  created() {
    this.$watch(
      () => this.$store.getters['data/allTemplates'],
      (allTemplates) => {
        const templates = {};
        // Sort templates by name
        Object.keys(allTemplates)
          .sort((id1, id2) => collator.compare(allTemplates[id1].name, allTemplates[id2].name))
          .forEach((id) => {
            const template = utils.deepCopy(allTemplates[id]);
            fillEmptyFields(template);
            templates[id] = template;
          });
        this.templates = templates;
        this.selectedId = this.config.selectedId;
        if (!templates[this.selectedId]) {
          this.selectedId = Object.keys(templates)[0];
        }
        this.isEditing = false;
      }, { immediate: true });
    this.$watch('selectedId', (selectedId) => {
      const template = this.templates[selectedId];
      this.showHelpers = template.helpers !== emptyTemplateHelpers;
      this.editingName = template.name;
    }, { immediate: true });
  },
  methods: {
    create() {
      const template = {
        name: 'New template',
        value: '\n',
        helpers: '\n',
      };
      fillEmptyFields(template);
      this.selectedId = utils.uid();
      this.templates[this.selectedId] = template;
      this.isEditing = true;
    },
    copy() {
      const template = utils.deepCopy(this.templates[this.selectedId]);
      template.name += ' copy';
      delete template.isAdditional;
      this.selectedId = utils.uid();
      this.templates[this.selectedId] = template;
      this.isEditing = true;
    },
    remove() {
      delete this.templates[this.selectedId];
      this.selectedId = Object.keys(this.templates)[0];
    },
    submitEdit(cancel) {
      const template = this.templates[this.selectedId];
      if (!cancel && this.editingName) {
        template.name = utils.sanitizeName(this.editingName);
      } else {
        this.editingName = template.name;
      }
      setTimeout(() => { // For the form-entry to get the blur event
        this.isEditing = false;
      }, 1);
    },
    resolve() {
      this.config.resolve({
        templates: this.templates,
        selectedId: this.selectedId,
      });
    },
  },
};
</script>

<style lang="scss">
.modal__inner-1--templates {
  max-width: 680px;
}
</style>
