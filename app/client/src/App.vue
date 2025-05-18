<script setup>
import { ref, onMounted } from "vue";

const newTodo = ref("");
const todos = ref([]);
const error = ref("");

// Fetch todos from backend when component mounts
onMounted(async () => {
  try {
    const response = await fetch("/api/todos");
    if (!response.ok) throw new Error("Failed to fetch todos");
    todos.value = await response.json();
  } catch (err) {
    error.value = err.message;
  }
});

const addTodo = async () => {
  if (!newTodo.value.trim()) return;

  try {
    const response = await fetch("/api/todos", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        text: newTodo.value,
        completed: false,
      }),
    });

    if (!response.ok) throw new Error("Failed to add todo");

    const addedTodo = await response.json();
    todos.value.push(addedTodo);
    newTodo.value = "";
    error.value = "";
  } catch (err) {
    error.value = err.message;
  }
};

const removeTodo = async (id, index) => {
  try {
    const response = await fetch(`/api/todos/${id}`, {
      method: "DELETE",
    });

    if (!response.ok) throw new Error("Failed to delete todo");

    todos.value.splice(index, 1);
  } catch (err) {
    error.value = err.message;
  }
};

const toggleTodo = async (todo) => {
  try {
    const response = await fetch(`/api/todos/${todo._id}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        completed: !todo.completed,
      }),
    });

    if (!response.ok) throw new Error("Failed to update todo");

    todo.completed = !todo.completed;
  } catch (err) {
    error.value = err.message;
  }
};
</script>

<template>
  <div class="todo-container">
    <h2>My Todo List</h2>
    <div class="todo-input">
      <input
        v-model="newTodo"
        @keyup.enter="addTodo"
        placeholder="Add a new task..."
      />
      <button @click="addTodo">Add</button>
    </div>
    <p v-if="error" class="error-message">{{ error }}</p>
    <ul class="todo-list">
      <li v-for="(todo, index) in todos" :key="todo._id">
        <input
          type="checkbox"
          :checked="todo.completed"
          @change="toggleTodo(todo)"
        />
        <span :class="{ completed: todo.completed }">{{ todo.text }}</span>
        <button @click="removeTodo(todo._id, index)">×</button>
      </li>
    </ul>
  </div>
</template>

<style scoped>
.todo-container {
  max-width: 500px;
  width: 90%;
  margin: 0 auto;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 20px;
  background: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

h2 {
  text-align: center;
  color: #333;
}

.todo-input {
  display: flex;
  margin-bottom: 20px;
}

.todo-input input {
  flex-grow: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px 0 0 4px;
}

.todo-input button {
  padding: 10px 15px;
  background: #42b983;
  color: white;
  border: none;
  border-radius: 0 4px 4px 0;
  cursor: pointer;
}

.todo-list {
  list-style: none;
  color: #183b4e;
  padding: 0;
}

.todo-list li {
  display: flex;
  align-items: center;
  padding: 10px;
  background: white;
  margin-bottom: 8px;
  border-radius: 4px;
}

.todo-list li input[type="checkbox"] {
  margin-right: 10px;
}

.todo-list li span {
  flex-grow: 1;
}

.todo-list li span.completed {
  text-decoration: line-through;
  color: black;
}

.todo-list li button {
  background: #ff6b6b;
  color: black;
  border: none;
  border-radius: 4px;
  padding: 5px 10px;
  cursor: pointer;
}

.error-message {
  color: #ff4444;
  text-align: center;
  margin-bottom: 10px;
}
</style>
