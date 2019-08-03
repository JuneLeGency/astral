<template>
  <div class="absolute pin-t pin-l pin-b mt-16 w-full bg-white flex flex-col">
    <button class="btn btn-brand absolute pin-r mt-2 mr-4 z-10 focus-none" @click="saveNotes">
      保存
    </button>
    <textarea ref="editor" />
  </div>
</template>
<script>
import EasyMDE from 'easymde'
import { debounce } from 'lodash'
import 'highlight.js'
export default {
  name: 'NotesEditor',
  props: {
    notes: String,
    autosave: Boolean
  },
  data() {
    return {
      editor: null,
      currentNotes: ''
    }
  },
  watch: {
    notes(val) {
      if (this.editor) {
        if (this.editor.isPreviewActive()) {
          this.editor.togglePreview()
        }
        if (val === this.editor.value()) {
          return
        }
        this.editor.value(val)
      }
    },
    currentNotes() {
      if (this.autosave) {
        this.$nextTick(() => {
          this.debounceSaveNotes()
        })
      }
    }
  },
  created() {
    this.debounceSaveNotes = debounce(this.saveNotes, 1000)
  },
  mounted() {
    this.editor = new EasyMDE({
      element: this.$refs.editor,
      initialValue: this.notes,
      forceSync: true,
      autoDownloadFontAwesome: false,
      renderingConfig: {
        codeSyntaxHighlighting: true
      },
      spellChecker: false,
      hideIcons: ['side-by-side', 'guide'],
      showIcons: ['code'],
      status: false,
      toolbar: [
        {
          name: 'bold',
          action: EasyMDE.toggleBold,
          className: 'fas fa-bold',
          title: '粗体'
        },
        {
          name: 'italic',
          action: EasyMDE.toggleItalic,
          className: 'fas fa-italic',
          title: '斜体'
        },
        {
          name: 'heading',
          action: EasyMDE.toggleHeadingSmaller,
          className: 'fas fa-heading',
          title: '标题'
        },
        '|',
        {
          name: 'code',
          action: EasyMDE.toggleCodeBlock,
          className: 'fas fa-code',
          title: '代码'
        },
        {
          name: 'quote',
          action: EasyMDE.toggleBlockquote,
          className: 'fas fa-quote-left',
          title: '引用'
        },
        {
          name: 'unordered-list',
          action: EasyMDE.toggleUnorderedList,
          className: 'fas fa-list-ul',
          title: '无序列表'
        },
        {
          name: 'ordered-list',
          action: EasyMDE.toggleOrderedList,
          className: 'fas fa-list-ol',
          title: '有序列表'
        },
        '|',
        {
          name: 'link',
          action: EasyMDE.drawLink,
          className: 'fas fa-link',
          title: '链接'
        },
        {
          name: 'image',
          action: EasyMDE.drawImage,
          className: 'fas fa-image',
          title: '图片'
        },
        '|',
        {
          name: 'preview',
          action: EasyMDE.togglePreview,
          className: 'fas fa-eye',
          title: '预览'
        },
        {
          name: 'fullscreen',
          action: EasyMDE.toggleFullScreen,
          className: 'fas fa-arrows-alt',
          title: '全屏'
        }
      ]
    })
    this.editor.codemirror.on('change', () => {
      this.currentNotes = this.editor.value()
    })
  },
  destroyed() {
    this.editor = null
  },
  methods: {
    saveNotes() {
      this.$emit('save', this.currentNotes)
    }
  }
}
</script>
<style lang="scss">
@import '~easymde/dist/easymde.min.css';
.editor-toolbar {
  border-top-left-radius: 0;
  border-top-right-radius: 0;
  border: none;
  border-bottom: 1px solid config('colors.grey-light');
  button {
    color: config('colors.grey-dark');
  }
}
.CodeMirror {
  flex-grow: 1;
  border: none;
}
</style>
