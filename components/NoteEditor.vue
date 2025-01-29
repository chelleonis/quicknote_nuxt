<template>
  <div class="editor-container">
    <textarea
      v-model="localContent"
      class="editor-textarea"
      placeholder="Start Typing..."
      @input="handleInput"
    />
  </div>
</template>

<script lang="ts" setup>
// components/QuickNoteEditor.vue
const props = defineProps({
  content: {
    type: String
  }
})

const emit = defineEmits(['update:content', 'change'])

const localContent = ref(props.content)

//TODO: change to sending signal after user stops typing for a bit
//debounce (i think)
watch(() => props.content, (newContent) => {
  if (newContent !== localContent.value) {
    localContent.value = newContent
  }
})

const handleInput = (event: Event) => {
  const newContent = (event.target as HTMLTextAreaElement).value
  const oldContent = props.content

  if (newContent === oldContent) return

  const diff = findDiff(oldContent, newContent)

  if (diff) {
    emit('change', diff)
    emit('update:content', newContent)
  
}
}

const findDiff = (oldStr: string, newStr: string) => {
  if (newStr.length > oldStr.length) {
    let i = 0
    //insertion
    while (i < oldStr.length && oldStr[i] == newStr[i]) i++
    return {
      type: 'insert',
      position: i,
      content: newStr.slice(i, i + (newStr.length - oldStr.length))
    }
    //deletion
  } else {
    let i = 0 
    while (i < newStr.length && oldStr[i] === newStr[i]) i++
    return {
      type: 'delete',
      position: i,
      length: oldStr.length - newStr.length
    }
  }
}


//TODO: transfer styling to tailwind.
</script>

<style>

</style>