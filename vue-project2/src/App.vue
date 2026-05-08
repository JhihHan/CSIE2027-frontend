<script setup lang="ts">
import {ref} from 'vue'
import Checkbox from './components/ui/checkbox/Checkbox.vue'

const newTask = ref('')
const todos = ref([
  {id:1,text: 'Learn Vue',completed:false},
  {id:2,text: 'Install Shadcn UI',completed:false}
])

const addTodo=()=>{
  if (newTask.value.trim()==='')return
  todos.value.push({
    id:Date.now(),
    text:newTask.value,
    completed:false
  })
  newTask.value=''
}
</script>


<template>
  <div>
    <h1>My TODO list</h1>
    <div>
      <input v-model="newTask" placeholder="Write down todos" />
      <button @click="addTodo">Add TODO</button>
    </div>
    <ul>
      <li v-for="todo in todos":key="todo.id"> <!-- v-for = for loop -->
        <Checkbox
          :id="todo.id"
          :checkbox="todo.completed"
          @update.checkbox="(val:any) => todo.completed=val"
        />
        <input type="checkbox" v-model="todo.completed" />
        <span :class="{'done':todo.completed}">{{todo.text}}</span>
      </li>
    </ul>
  </div>
</template>

<style scoped>
.done{
    text-decoration: line-through;
    color: gray;
}
</style>
