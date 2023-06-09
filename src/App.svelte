<script>
  import { onMount } from 'svelte'
  import { sortBy } from 'lodash'
  import PouchDB from 'pouchdb-browser'

  import TodoItem from './todo-item.svelte'

  // Set up local PouchDB and continuous replication ke remote CouchDB
  let db = new PouchDB('db')
  const replication = PouchDB.sync('db', 'http://localhost:5984/chapter-todo', {
    live: true,
    retry: true,
    auth:{

      username: 'admin',
      password: 'Loginaja123'
    }
  }).on('change', async function (info) {
    await updateTodos()
  }).on('error', function (err) {
    console.log('Replication error:', err)
  })

  
  //default vars diisi kosong dulu
  let newTodoText = ''
  let sortByWhat = 'createdAt'
  let filterByWhat = ''
  let isLoading = true
  // All the todos directly from the PouchDB. Sorting and filtering comes later
  let todos = []
  $: sortedAndFilteredTodos = sortBy(todos, [sortByWhat]).filter((todo) => {
    const [filterKey, filterValue] = filterByWhat.split(':')
    // Only filter if there’s a proper filter set
    return filterKey && filterValue ? todo[filterKey].toString() === filterValue : true
  })

  // Helper untuk reloading semua todos  dari  local PouchDB. in local dan zero latency.
  async function updateTodos() {
    const allDocs = await db.allDocs({
      include_docs: true
    })
    todos = allDocs.rows.map(row => row.doc)
    isLoading = false
  }

  // Event handlers for adding, updating and removing todos
  async function add(event) {
    const newTodo = {
      text: newTodoText,
      complete: false,
      createdAt: new Date().toISOString()
    }
    const addition = await db.post(newTodo)
    if (addition.ok) {
      await updateTodos()
    }
    newTodoText = ''
  }

  async function updateStatus(event) {
    const { todo } = event.detail
    const update = await db.put(todo)
    if (update.ok) {
      await updateTodos()
    }
  }

  async function removeItem(event) {
    const { todo: todoToRemove } = event.detail
    const removal = await db.remove(todoToRemove)
    if (removal.ok) {
      // For removal, we can just update the local state instead of reloading everything from PouchDB,
      // since we no longer care about the doc’s revision.
      todos = todos.filter((todo) => {
        return todo._id !== todoToRemove._id
        })
    }
  }

  // Load todos on first run
  onMount(async () => {
    await updateTodos()
  })
</script>

<style>
  ul {
    list-style: none;
    padding: 0;
  }
  button {
    margin-left: 0.75em;
  }
  input[type='text'] {
    width: 440px;
  }
</style>

{#if isLoading}
  <h1>
    Loading your todos…
  </h1>
{:else}
  {#if todos.length === 0}
    <h1>
      Zero Todos! 
    </h1>
  {:else}
    <h1>
      Showing {sortedAndFilteredTodos.length} of {todos.length} todos
    </h1>
  {/if}
{/if}

<div>
  <label>Sort by:</label>
  <select bind:value={sortByWhat}>
    <option value='createdAt'>Time</option>
    <option value='text'>Todo text</option>
    <option value='complete'>Completion</option>
  </select>
</div>
<div>
  <label>Filter:</label>
  <select bind:value={filterByWhat}>
    <option value=''>Show all todos</option>
    <option value='complete:true'>Show completed todos</option>
    <option value='complete:false'>Show open todos</option>
  </select>
</div>

<ul>
  {#each sortedAndFilteredTodos as todo (todo._id)}
    <TodoItem todo={todo} on:remove={removeItem} on:update={updateStatus}/>
  {/each}
</ul>

<form on:submit|preventDefault={add}>
  <input type='text' bind:value={newTodoText}>
  <button type='submit'>➕ Add new task</button>
</form>
