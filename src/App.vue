<script setup lang="ts">
import { computed, ref, watch } from 'vue'

interface User {
  id: number
  name: string
}

interface Entry {
  id: number
  price: number
  userIds: number[]
}

const users: User[] = [
  { id: 0, name: 'Flo' },
  { id: 1, name: 'Nico' },
  { id: 2, name: 'Lukas' },
  { id: 3, name: 'Juli' },
]

const nextId = ref(0)

function loadEntries() {
  const json = localStorage.getItem('entries')
  if (!json) {
    return
  }

  const entries = JSON.parse(json)
  if (!Array.isArray(entries) || entries.length === 0) {
    return
  }

  nextId.value = Math.max(...entries.map((entry: Entry) => entry.id)) + 1

  return entries
}

const entries = ref<Entry[]>(loadEntries() ?? [{ id: nextId.value++, price: 0, userIds: [] }])
const current = ref(entries.value[0])

watch(
  entries,
  (newEntries) => {
    localStorage.setItem('entries', JSON.stringify(newEntries))
  },
  { deep: true },
)

const valid = computed(() => {
  return current.value.price !== 0 && current.value.userIds.length > 0
})

function formatPrice(price: number): string {
  return price.toLocaleString('de-DE', {
    style: 'currency',
    currency: 'EUR',
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  })
}

function pushEntry(entry: Entry, clear = true) {
  entries.value.push(entry)
  current.value = entry

  // clear inputs
  if (clear) {
    current.value.price = 0
    current.value.userIds = []
  }

  // handle ui shift
  setTimeout(() => {
    containerElement.value?.scrollTo({
      top: containerElement.value.scrollHeight,
      behavior: 'smooth',
    })
  }, 0)
}

function createEntry() {
  if (!valid.value) {
    return
  }

  pushEntry({
    id: nextId.value++,
    price: current.value.price,
    userIds: current.value.userIds,
  })
}

const total = computed(() => {
  return entries.value.reduce((acc, entry) => acc + entry.price, 0)
})

const shares = computed(() => {
  return entries.value.reduce(
    (acc, entry) => {
      entry.userIds.forEach((userId) => {
        acc[userId] =
          (acc[userId] || 0) + Math.round((entry.price * 100) / entry.userIds.length) / 100
      })
      return acc
    },
    {} as Record<number, number>,
  )
})

const shareTotal = computed(() => {
  return Object.values(shares.value).reduce((acc, share) => acc + share, 0)
})

function onChange(event: Event) {
  const input = (event.target as HTMLInputElement).value
  const isNegative = input.replace(/[0,.]*/, '').startsWith('-')
  const value = parseInt(input.replace(/[^0-9]/g, '').padStart(3, '0'))
  current.value.price = (value / 100) * (isNegative ? -1 : 1)
}

function onKeydown(event: KeyboardEvent) {
  if (event.key === 'Enter') {
    createEntry()
  }
}

function onFocus() {
  setTimeout(() => {
    window.scrollTo({
      top: window.innerHeight,
      behavior: 'instant',
    })
  }, 1000)
}

function duplicateEntry() {
  pushEntry(
    {
      id: nextId.value++,
      price: current.value.price,
      userIds: [...current.value.userIds],
    },
    false,
  )
}

function deleteEntry() {
  const index = entries.value.findIndex((entry) => entry.id === current.value.id)
  if (index === -1) {
    throw new Error('Entry not found')
  }
  entries.value.splice(index, 1)
  current.value = entries.value[index] || entries.value[index - 1]
}

function addPfand() {
  const userIds = [...current.value.userIds]
  createEntry()
  current.value.price = 0.25
  current.value.userIds = userIds
  createEntry()
}

const containerElement = ref<HTMLElement | null>(null)
</script>

<template>
  <main class="flex flex-col h-full max-h-screen">
    <!-- Summary -->
    <ul class="flex flex-col gap-1 mx-4 mt-4">
      <li v-for="user in users" :key="user.id"
        class="bg-gray-800 text-white py-2 px-4 rounded flex justify-between items-center">
        <p>{{ user.name }}</p>
        <p>{{ formatPrice(shares[user.id] ?? 0) }}</p>
      </li>
      <li class="bg-gray-800 text-white py-2 px-4 rounded flex justify-between items-center">
        <p>Total</p>
        <p>{{ formatPrice(total) }} / {{ formatPrice(shareTotal) }}</p>
      </li>
    </ul>
    <!-- Entries -->
    <ul class="p-4 gap-2 flex flex-grow flex-col overflow-y-auto" ref="containerElement">
      <li v-for="entry in entries" :key="entry.id" :class="'bg-gray-800 text-white py-2 px-4 rounded-lg flex justify-end items-center border transition-colors ' +
        (current.id === entry.id ? 'border-gray-200' : 'border-transparent')
        " @click="current = entry">
        <p>{{ formatPrice(entry.price) }}</p>
      </li>
    </ul>
    <!-- Controls -->
    <div class="bg-gray-800 p-4 flex flex-col gap-4">
      <input @input="onChange" @keydown="onKeydown" @focus="onFocus" ref="inputElement" :value="current.price.toLocaleString('de-DE', {
        minimumFractionDigits: 2,
        maximumFractionDigits: 2,
      })
        " class="w-full text-4xl text-center" type="numeric" inputMode="decimal" />
      <ul class="flex gap-2 justify-center">
        <button v-for="user in users" :key="user.id" :class="'bg-gray-700 text-gray-200 py-1 px-4 rounded-full border transition-colors ' +
          (current.userIds.includes(user.id) ? 'border-gray-200' : 'border-transparent')
          " @click="
            current.userIds.includes(user.id)
              ? current.userIds.splice(current.userIds.indexOf(user.id), 1)
              : current.userIds.push(user.id)
            " @mousedown.prevent="">
          {{ user.name }}
        </button>
      </ul>
      <div class="flex items-center gap-2 justify-between">
        <button class="fill-gray-200 bg-gray-700 p-2 rounded-full flex gap-2 items-center text-sm disabled:opacity-50"
          @click="deleteEntry" @mousedown.prevent="" :disabled="entries.length <= 1">
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960" class="h-5 w-5">
            <path
              d="M280-120q-33 0-56.5-23.5T200-200v-520h-40v-80h200v-40h240v40h200v80h-40v520q0 33-23.5 56.5T680-120H280Zm400-600H280v520h400v-520ZM360-280h80v-360h-80v360Zm160 0h80v-360h-80v360ZM280-720v520-520Z" />
          </svg>
        </button>
        <button
          class="fill-gray-200 bg-gray-700 py-2 px-4 rounded-full flex gap-2 items-center text-sm disabled:opacity-50"
          @click="addPfand" @mousedown.prevent="" :disabled="!valid">
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960" class="h-5 w-5">
            <path
              d="m368-592 89-147-59-98q-12-20-34.5-20T329-837l-98 163 137 82Zm387 272-89-148 139-80 64 107q11 17 12 38t-9 39q-10 20-29.5 32T800-320h-45ZM640-40 480-200l160-160v80h190l-58 116q-11 20-30 32t-42 12h-60v80Zm-387-80q-20 0-36.5-10.5T192-158q-8-16-7.5-33.5T194-224l34-56h172v160H253Zm-99-114L89-364q-9-18-8.5-38.5T92-441l16-27-68-41 219-55 55 220-69-42-91 152Zm540-342-219-55 69-41-125-208h141q21 0 39.5 10.5T629-841l52 87 68-42-55 220Z" />
          </svg>
          Pfand
        </button>
        <button
          class="fill-gray-200 bg-gray-700 py-2 px-4 rounded-full flex gap-2 items-center text-sm disabled:opacity-50"
          @click="duplicateEntry" @mousedown.prevent="" :disabled="!valid">
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960" class="h-5 w-5">
            <path
              d="M360-240q-33 0-56.5-23.5T280-320v-480q0-33 23.5-56.5T360-880h360q33 0 56.5 23.5T800-800v480q0 33-23.5 56.5T720-240H360Zm0-80h360v-480H360v480ZM200-80q-33 0-56.5-23.5T120-160v-560h80v560h440v80H200Zm160-240v-480 480Z" />
          </svg>
          Kopie
        </button>
        <button class="fill-gray-200 bg-gray-700 p-2 rounded-full flex gap-2 items-center text-sm disabled:opacity-50"
          @click="createEntry" @mousedown.prevent="" :disabled="!valid">
          <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960" class="h-5 w-5">
            <path d="M382-240 154-468l57-57 171 171 367-367 57 57-424 424Z" />
          </svg>
        </button>
      </div>
    </div>
  </main>
</template>
