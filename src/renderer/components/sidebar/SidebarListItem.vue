<template>
  <div
    v-click-outside="onClickOutside"
    class="sidebar-list-item"
    :class="{
      'sidebar-list-item--selected': isSelected,
      'sidebar-list-item--context': context
    }"
    v-on="$listeners"
    @click="onClick"
    @contextmenu="onContext"
  >
    <div @click="$emit('click:toggle')">
      <AppIcon
        v-if="children"
        class="sidebar-list-item__child-icon"
        :class="{'is-open': open}"
        name="chevron-right"
      />
    </div>
    <AppIcon :name="icon" />
    <input
      ref="input"
      v-model="folderName"
      class="sidebar-list-item__input"
      :class="{
        'is-editable': editable
      }"
      type="text"
      :disabled="!editable"
      @keydown.enter="onClickOutside"
    >
    <slot />
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import context from '@@/lib/context'
import dialog from '@@/lib/dialog'

export default {
  name: 'SidebarListItem',

  props: {
    model: {
      type: Object,
      default: () => {}
    },
    id: {
      type: String,
      default: ''
    },
    title: {
      type: String,
      default: ''
    },
    icon: {
      type: String,
      default: 'folder'
    },
    children: {
      type: Boolean,
      default: false
    },
    open: {
      type: Boolean,
      default: false
    }
  },

  data () {
    return {
      context: false,
      editable: false,
      updatedFolderName: null
    }
  },

  computed: {
    ...mapGetters('folders', ['selectedId', 'editableId']),
    isSelected () {
      if (!this.selectedId) return null
      return this.selectedId === this.id
    },
    folderName: {
      get () {
        return this.title
      },
      set (e) {
        this.updatedFolderName = e
      }
    },
    isEditable () {
      return this.editableId === this.id
    }
  },

  created () {
    if (this.isEditable) {
      this.setEditable()
    }
  },

  methods: {
    onClick () {
      this.$store.dispatch('folders/setSelectedFolder', this.id)
    },
    onContext () {
      const libraryItems = ['inBox', 'favorites', 'allSnippets', 'trash']
      const isLibrary = libraryItems.includes(this.id)

      if (this.id === 'trash') {
        this.trashContext()
      }

      if (isLibrary) return

      this.folderContext()
    },
    folderContext () {
      this.context = true
      const menu = context.popup([
        {
          label: 'Rename',
          click: () => this.setEditable()
        },
        { type: 'separator' },
        {
          label: 'Delete',
          click: () => {
            const buttonId = dialog.showMessageBoxSync({
              message: `Are you sure you want to delete "${this.title}" folder?`,
              detail: 'All snippets in this folder will be moved to trash.',
              buttons: ['Delete', 'Cancel'],
              defaultId: 0,
              cancelId: 1
            })
            if (buttonId === 0) {
              console.log('delete folder')
              this.$store.dispatch('folders/deleteFolder', this.id)
            }
          }
        }
      ])
      menu.addListener('menu-will-close', () => {
        this.context = false
      })
    },
    trashContext () {
      this.context = true
      const menu = context.popup([
        {
          label: 'Empty Trash',
          click: () => {
            const buttonId = dialog.showMessageBoxSync({
              message: 'Are you sure you want to permanently delete all snippets in Trash?',
              detail: 'You cannot undo this action.',
              buttons: ['Delete', 'Cancel'],
              defaultId: 0,
              cancelId: 1
            })
            if (buttonId === 0) {
              console.log('delete trash')
              this.$store.dispatch('snippets/emptyTrash')
            }
          }
        }
      ])
      menu.addListener('menu-will-close', () => {
        this.context = false
      })
    },
    setEditable () {
      this.editable = true
      this.$emit('edit', true)
      this.$nextTick(() => {
        this.$refs.input.focus()
        this.$refs.input.select()
      })
    },
    onClickOutside () {
      if (this.editable) {
        this.$emit('edit', false)
        this.editable = false
        this.$store.commit('folders/SET_EDITABLE', null)
      }
      if (this.updatedFolderName) {
        const id = this.id
        const payload = this.updatedFolderName
        this.$store.dispatch('folders/updateFolderName', { id, payload })
      }
    }
  }
}
</script>

<style lang="scss">
.sidebar-list-item {
  display: flex;
  align-items: center;
  padding: 4px calc(var(--spacing-sm) + 2px);
  cursor: default;
  user-select: none;
  position: relative;
  &__input {
    width: 100%;
    border: 1px solid transparent;
    background-color: transparent;
    outline: none;
    &[disabled] {
      color: var(--color-text);
    }
    &.is-editable {
      border: 1px solid var(--color-primary);
      background-color: #fff;
    }
  }
  svg {
    width: 16px;
    height: 16px;
    position: relative;
    top: 1px;
    margin-right: var(--spacing-xs);
    stroke: var(--color-contrast-medium);
  }
  &:last-child {
    margin-bottom: 0;
  }
  &--selected {
    background-color: var(--color-contrast-low);
  }
  &--context {
    position: relative;
    &::after {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      border: 2px solid var(--color-primary);
      border-radius: 4px;
    }
  }
  &__child-icon {
    position: absolute;
    top: 6px;
    left: 2px;
    &.is-open {
      svg {
        transform: rotate(90deg);
      }
    }
    svg {
      transition: all .1s;
      width: 14px;
      height: 14px;
    }
  }
}
</style>