<template>
  <div>
    <div v-if="editor" class="rich-text-option-container">
      <button
        class="rich-text-button"
        :class="boldButtonDynamicClasses"
        title="Bold"
        @click="handleBold"
      >
        <img src="/bold.svg" />
      </button>
      <button
        class="rich-text-button"
        :class="italicButtonDynamicClasses"
        title="Italic"
        @click="handleItalic"
      >
        <img src="/italic.svg" />
      </button>
      <button
        class="rich-text-button"
        :class="bulletListDynamicClasses"
        title="Bullet List"
        @click="handleBulletList"
      >
        <img src="/fa-list.svg" />
      </button>
      <button
        class="rich-text-button"
        :class="numberListDynamicClasses"
        title="Number List"
        @click="handleNumberList"
      >
        <img src="/fa-ordered-list.svg" />
      </button>
      <DialogRoot v-model:open="addLinkDialog">
        <DialogTrigger as-child>
          <button
            class="rich-text-button"
            :class="addLinkDynamicClasses"
            title="Add Link"
            @click="handleAddLinkButtonClick"
          >
            <img src="/fa-link.svg" />
          </button>
        </DialogTrigger>
        <DialogPortal>
          <DialogOverlay class="dialog-overlay" />
          <DialogContent class="dialog-content">
            <DialogTitle>Add Link</DialogTitle>
            <input
              v-model="dialogLinkInitialValue"
              placeholder="Link address"
              class="dialog-input"
            />
            <button class="dialog-button">Confirm</button>
          </DialogContent>
        </DialogPortal>
      </DialogRoot>
      <button
        class="rich-text-button"
        title="Remove Link"
        :disabled="!editor.isActive('link')"
        @click="handleRemoveLink"
      >
        <img src="/fa-link-broken.svg" />
      </button>
    </div>

    <div
      class="message-editor-outer-container"
      :class="outerContainerDynamicClasses"
    >
      <EditorContent
        v-if="editor"
        :editor="editor"
        class="message-editor-container"
      />
      <div
        v-if="label"
        class="message-editor-label"
        :class="labelDynamicClasses"
      >
        {{ label }}
      </div>
    </div>

    <div class="bottom-container" :class="bottomContainerDynamicClasses">
      <p v-if="hasError" class="error-message">{{ errorMessage }}</p>

      <div v-if="maxLength != null" class="message-editor-counter-container">
        <p class="message-editor-counter">
          character count: {{ characterCount }},
        </p>
        <p class="message-editor-counter">
          output count: {{ messageCount }} / {{ maxLength }}
        </p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { Editor, EditorContent } from '@tiptap/vue-3';
import { Document } from '@tiptap/extension-document';
import { Paragraph } from '@tiptap/extension-paragraph';
import { Text } from '@tiptap/extension-text';
import { HardBreak } from '@tiptap/extension-hard-break';
import { ListItem } from '@tiptap/extension-list-item';
import { OrderedList } from '@tiptap/extension-ordered-list';
import { BulletList } from '@tiptap/extension-bullet-list';
import { Link } from '@tiptap/extension-link';
import { Bold } from '@tiptap/extension-bold';
import { Italic } from '@tiptap/extension-italic';
import { CharacterCount } from '@tiptap/extension-character-count';
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue';
import {
  DialogClose,
  DialogContent,
  DialogDescription,
  DialogOverlay,
  DialogPortal,
  DialogRoot,
  DialogTitle,
  DialogTrigger,
} from 'radix-vue';

const MOBILE_DEEP_LINK_PROTOCOL = `pf-dcu`;
const VALID_PROTOCOLS = ['https', 'http', MOBILE_DEEP_LINK_PROTOCOL];
const TIPTAP_EMPTY_VALUE = '<p></p>';
const MESSAGE_COUNTER_TOOLTIP =
  'This field generates HTML text which is ultimately displayed for the member. The number of characters must be capped to prevent display issues our platforms. We display both the number of character seen in this field as well as the number of characters in the final HTML output.';

///// props/emits /////
const props = defineProps({
  modelValue: {
    type: String,
    required: true,
  },
  maxLength: {
    type: Number,
    default: null,
  },
  label: {
    type: String,
    default: null,
  },
  hasError: {
    type: Boolean,
    default: false,
  },
  errorMessage: {
    type: String,
    default: 'The message is invalid',
  },
  disable: {
    type: Boolean,
    default: false,
  },
});

const emit = defineEmits(['update:modelValue', 'input-raw']);

///// refs/provide-inject/variables /////
const editor = ref(null);
const addLinkDialog = ref(false);
const dialogLinkInitialValue = ref(null);
const dialogOpenLinkInNewTabValue = ref(null);

///// computed /////
/**
 * Interface into the message value
 */
const localValue = computed({
  get() {
    return props.modelValue;
  },
  set(value) {
    if (props.disable) {
      return;
    }

    // check for tiptap's empty html value and override with an empty string
    // this allows the preview section and web to not have to handle this case
    if (value === '<p></p>') {
      value = '';
    }

    emit('update:modelValue', value);
    emit('input-raw', editor.value.getText());
  },
});

/**
 * Returns dynamic class names for the bold button
 *
 * @returns {Object}
 */
const boldButtonDynamicClasses = computed(() => {
  return {
    'is-active': editor.value.isActive('bold'),
  };
});

/**
 * Returns dynamic class names for the italic button
 *
 * @returns {Object}
 */
const italicButtonDynamicClasses = computed(() => {
  return {
    'is-active': editor.value.isActive('italic'),
  };
});

/**
 * Returns dynamic class names for the bullet list button
 *
 * @returns {Object}
 */
const bulletListDynamicClasses = computed(() => {
  return {
    'is-active': editor.value.isActive('bulletList'),
  };
});

/**
 * Returns dynamic class names for the number list button
 *
 * @returns {Object}
 */
const numberListDynamicClasses = computed(() => {
  return {
    'is-active': editor.value.isActive('orderedList'),
  };
});

/**
 * Returns dynamic class names for the add link button
 *
 * @returns {Object}
 */
const addLinkDynamicClasses = computed(() => {
  return {
    'is-active': editor.value.isActive('link'),
  };
});

/**
 * Returns dynamic class names for the outer container element
 *
 * Used to update child elements based on certain states, such as error and disable
 *
 * @returns {Object}
 */
const outerContainerDynamicClasses = computed(() => {
  return {
    'message-editor-outer-container-error': props.hasError,
    'message-editor-outer-container-disabled': props.disable,
  };
});

/**
 * Returns dynamic class names for the label element
 *
 * @returns {Object}
 */
const labelDynamicClasses = computed(() => {
  return {
    'message-editor-label-populated':
      props.modelValue?.length > 0 && props.modelValue !== TIPTAP_EMPTY_VALUE,
  };
});

/**
 * Returns dynamic class names for the bottom container element
 *
 * Used to update child elements based on certain states, such as error and disable
 *
 * @returns {Object}
 */
const bottomContainerDynamicClasses = computed(() => {
  return {
    'bottom-container-error': props.hasError,
  };
});

/**
 * Returns the count of the html message
 *
 * @returns {Number}
 */
const messageCount = computed(() => {
  return Math.max(localValue.value?.length ?? 0, 0);
});

/**
 * Returns the count of the message
 *
 * @returns {Number}
 */
const characterCount = computed(() => {
  return Math.max(editor.value?.getText()?.length ?? 0, 0);
});

///// methods /////

/**
 * Opens the add link dialog and prepares any prefill
 */
const handleAddLinkButtonClick = () => {
  console.log('init');
  dialogLinkInitialValue.value =
    editor.value.getAttributes('link').href ?? null;
  dialogOpenLinkInNewTabValue.value =
    editor.value.getAttributes('link').target === '_blank';
  addLinkDialog.value = true;
};

/**
 * Adds a link with the given params to the currently selected text
 *
 * @param {String} linkAddress - the url to add as the link
 * @param {Boolean} openLinkInNewTab - determines if the link should open in a new tab or in the same tab
 */
const handleAddLink = () => {
  //console.log('add link');
  //const linkOptions = {
  //  href: dialogLinkInitialValue.value,
  //
  //  // links open in the same tab by default
  //  target: null,
  //};

  //console.log(linkOptions);

  if (openLinkInNewTab) {
    linkOptions.target = undefined;
  } else {
    // tiptap, when prefilling existing content during initialization, will inject target="_blank"
    // if there is no value for target. "_self" is the same as not setting target, so by adding it here
    // tiptap will not inject "_blank"
    linkOptions.target = '_self';
  }

  // update link
  editor.value
    .chain()
    .focus()
    .extendMarkRange('link')
    .setLink(linkOptions)
    .run();

  if (
    editor.value?.state?.selection?.$head?.pos &&
    isNaN(editor.value?.state?.selection?.$head?.pos) === false
  ) {
    editor.value.commands.setTextSelection(
      editor.value.state.selection.$head.pos + 1
    );
  }

  // reset local values used for dialog prefill
  dialogLinkInitialValue.value = null;
  dialogOpenLinkInNewTabValue.value = null;
};

/**
 * Toggles bold styles for the currently selected text
 */
const handleBold = () => {
  editor.value.chain().focus().toggleBold().run();
};

/**
 * Toggles italic styles for the currently selected text
 */
const handleItalic = () => {
  editor.value.chain().focus().toggleItalic().run();
};

/**
 * Toggles bullet list styles for the currently selected text
 */
const handleBulletList = () => {
  editor.value.chain().focus().toggleBulletList().run();
};

/**
 * Toggles number list styles for the currently selected text
 */
const handleNumberList = () => {
  editor.value.chain().focus().toggleOrderedList().run();
};

/**
 * Removes any link from the currently selected text
 */
const handleRemoveLink = () => {
  editor.value.chain().focus().unsetLink().run();
};

/**
 * Creates the rich text editor
 */
const initializeEditor = () => {
  editor.value = new Editor({
    content: localValue.value,
    autofocus: false,
    editable: !props.disable,
    injectCSS: false,
    parseOptions: {
      preserveWhitespace: 'full',
    },
    extensions: [
      Document,
      Paragraph,
      Text,
      Bold,
      Italic,
      HardBreak,
      ListItem,
      OrderedList,
      BulletList,
      CharacterCount.configure({
        limit: props.maxLength ?? undefined,
      }),
      Link.configure({
        openOnClick: false,
        protocols: VALID_PROTOCOLS,
      }),
    ],
    onUpdate: () => {
      localValue.value = editor.value.getHTML();
    },
  });
};

///// watchers /////
watch(props.disable, (newValue) => {
  editor.value.setEditable(!newValue);
});

///// lifecycle /////
onMounted(() => {
  initializeEditor();
});
onBeforeUnmount(() => {
  editor.value?.destroy();
});
</script>

<style lang="scss" scoped>
.rich-text-option-container {
  display: flex;
  gap: 2px;
  margin-top: 5px;
}

.rich-text-button {
  width: 30px;
  height: auto;
  display: grid;
  place-content: center;

  &.is-active {
    background-color: #d0d0d7;
    border: 1px solid black;
    border-radius: 5px;
  }
  &.is-active i {
    font-weight: bold;
  }
}

.message-editor-outer-container {
  position: relative;
  margin-top: 5px;
}

.message-editor-container {
  border: 1px solid #c2c2c2;
}

.message-editor-label {
  position: absolute;
  top: 8px;
  left: 10px;
  color: #797979;
  font-size: 14px;

  transition: font-size 260ms cubic-bezier(0.4, 0, 0.2, 1);
}

.message-editor-outer-container:focus-within {
  .message-editor-label {
    color: black;
  }

  .message-editor-container {
    border: 2px solid black;
  }
}

.message-editor-label-populated {
  font-size: 11px;
  top: 5px;
}

.bottom-container {
  display: flex;
  flex-flow: row wrap;
  justify-content: space-between;
}

.bottom-container-error {
  color: var(--q-negative, #c10015);

  .message-editor-counter {
    color: var(--q-negative, #c10015);
  }
}

.error-message,
.message-editor-counter-container {
  padding: 0px 12px;
  margin: 0px;
  display: flex;
  flex-flow: row wrap;
  align-items: center;
  gap: 1ch;
}

.error-message {
  flex: 11;
  font-size: 12px;
}

.message-editor-counter-container {
  font-size: 11px;
  flex: 1;
  justify-content: flex-end;
}

.message-editor-counter {
  color: rgba(0, 0, 0, 0.54);
  text-align: end;
  margin: 0px;
}

.message-editor-outer-container-disabled {
  cursor: not-allowed;
}

.message-counter-tooltip {
  max-width: 400px;
}

.message-editor-outer-container-error {
  border: 2px solid var(--q-negative, #c10015);

  .message-editor-label {
    color: var(--q-negative, #c10015);
  }

  .message-editor-container {
    border: none;
  }
}

.message-editor-outer-container-error:focus-within {
  .message-editor-label {
    color: var(--q-negative, #c10015);
  }

  .message-editor-container {
    border: none;
  }
}

// TIPTAP STYLE OVERRIDES
:deep(.message-editor-container) {
  .ProseMirror {
    border: none;
    min-height: 100px;
    padding-top: 10px;

    &:focus {
      outline: none;
    }

    p {
      margin: 0px;
      margin-left: 10px;
    }
    > p:first-child {
      margin-top: 20px;
    }

    li p {
      margin: 0px;
    }

    a {
      color: var(--q-primary, black);
      // there is no cursor pointer here because this link is not a real anchor
    }

    & * {
      white-space: pre-wrap;
      word-wrap: break-word;
    }
  }
}

:deep(.message-editor-outer-container-error) {
  .ProseMirror {
    color: black;
  }
}

.dialog-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
}
.dialog-content {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: white;
  padding: 20px;
  border-radius: 8px;
}
.dialog-input {
  width: 100%;
  padding: 8px;
  margin-top: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
}
.dialog-button {
  margin-top: 10px;
  padding: 8px 12px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
</style>
