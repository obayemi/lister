<template>
<div>
  <form v-on:submit.prevent='addNewItem'>
    <input
      v-model='newText'
      id='new-text'
    >
  </form>
  <draggable
    :list="items"
    :disabled="!enabled"
    v-bind="dragOptions"
    tag="ul"
    class="list-group"
    ghost-class="ghost"
    @change="saveTodos"
    @start="dragging = true"
    @end="dragging = false"
  >
    <transition-group type='transition' :name="!dragging ? 'flip-list' : null">
      <todo-item
        v-for="element in items"
        :label="element.text"
        :position="element.position"
        :key="element.position"
        v-show="showCompleted || !element.text.startsWith('x ')"
        @update="onElementUpdate(element, $event)"
        @totop="onToTop($event)"
      ></todo-item>
    </transition-group>
  </draggable>
  <a href="#" @click.prevent="onToggleCompleted(event)" class="toggle-completed-link">{{toggleCompletedText}}</a>
</div>
</template>

<script>
import draggable from 'vuedraggable'
import { getContent, updateContent } from '@/store/api'
import TodoItem from '@/components/TodoItem'

export default {
  name: 'SyncList',
  components: {
    draggable,
    TodoItem
  },
  data: function () {
    return {
      folder: '',
      filename: '',
      enabled: true,
      newText: '',
      items: [],
      dragging: false,
      nextId: 4,
      showCompleted: true,
      toggleCompletedText: 'Hide completed'
    }
  },
  created () {
    const path = this.$route.path.split('/')
    this.folder = path[1]
    this.filename = path[2]
    if (!this.folder || !this.filename) {
      return
    }
    this.loadList()
  },
  methods: {
    addNewItem () {
      if (!this.newText) return
      this.items.unshift({
        position: this.nextId++,
        text: this.newText
      })
      this.newText = ''
      this.saveTodos()
    },
    loadList () {
      getContent(this.folder, this.filename).then(response => {
        this.items = response.data.split('\n').filter(line => line.trim()).map((text, index) => {
          return { text, position: index }
        })
      })
    },
    saveTodos () {
      updateContent(this.folder, this.filename, this.items.map(item => item.text).join('\n'))
    },
    // Return index of the first completed task
    getLastCompletedTaskIndex (position) {
      let completedTaskIndex = this.items.length + 1
      for (let i = 0; i < this.items.length; i++) {
        if (this.items[i].text.startsWith('x ') && this.items[i].position !== position && i <= completedTaskIndex) {
          completedTaskIndex = i
        }
      }
      return completedTaskIndex
    },
    onElementUpdate (element, newText) {
      element.text = newText
      const completedTaskIndex = this.getLastCompletedTaskIndex(element.position)
      const items = this.items
      for (let i = 0; i < this.items.length; i++) {
        if (this.items[i].position === element.position) {
          if (newText.startsWith('x ')) {
            items.splice(i, 1)
            items.splice(completedTaskIndex - 1, 0, element)
          } else {
            items.splice(i, 1, element)
          }
        }
      }
      this.items = items
      this.saveTodos()
    },
    onToggleCompleted () {
      this.showCompleted = !this.showCompleted
      if (this.showCompleted) {
        this.toggleCompletedText = 'Hide completed'
      } else {
        this.toggleCompletedText = 'Show completed'
      }
    },
    onToTop (position) {
      for (let i = 0; i < this.items.length; i++) {
        if (this.items[i].position === position) {
          const movedItem = this.items[i]
          this.items.splice(i, 1)
          this.items.unshift(movedItem)
        }
      }
    }
  },
  computed: {
    dragOptions () {
      return {
        animation: 200,
        group: 'description',
        disabled: false,
        ghostClass: 'ghost'
      }
    }
  }
}
</script>

<style scoped>

form {
  margin: 10px;
}
input {
  min-width: 95vw;
  padding: 12px;
  border: none;
  background-color: rgba(187, 187, 187, 0.151);
  border-radius: 4px;
  position: relative;
  font-size: 1.2em;
  color:rgb(121, 121, 121);
  font-weight: 500;
}
ul {
  padding: 0;
  margin: 0;
}
.list-group-item {
  list-style-type: none;
  padding: 10px;
  margin: 8px;
  background-color: rgb(245, 244, 244);
  border-radius: 4px;
  text-align: left;
  box-shadow: 1px 1px 2px rgba(0,0,0,0.3);
}
.flip-list-move {
  transition: transform 0.5s;
}
.no-move {
  transition: transform 0s;
}
.ghost {
  opacity: 0.5;
  background: #c8ebfb;
}
.toggle-completed-link {
  font-size: .9em;
  text-decoration: none;
  margin: 10px;
  color: rgb(121, 121, 121);
  border-bottom: 1px dotted rgb(121, 121, 121);
}
</style>
