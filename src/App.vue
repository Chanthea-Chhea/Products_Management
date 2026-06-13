<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue'

type Category = {
  id: number
  name: string
  desc: string
  is_active: boolean
  created_at?: string
  updated_at?: string
}

type CategoryPayload = {
  name: string
  desc: string
  is_active: boolean
}

type CategoryResponse = {
  success: boolean
  message: string
  data?: Category | Category[]
  errors?: Record<string, string[]>
}

const apiBaseUrl = import.meta.env.VITE_API_BASE_URL || '/api'

const categories = ref<Category[]>([])
const isLoading = ref(false)
const isSaving = ref(false)
const deletingId = ref<number | null>(null)
const editingId = ref<number | null>(null)
const search = ref('')
const message = ref('')
const error = ref('')

const form = reactive<CategoryPayload>({
  name: '',
  desc: '',
  is_active: true,
})

const activeCount = computed(() => categories.value.filter((category) => category.is_active).length)
const inactiveCount = computed(() => categories.value.length - activeCount.value)
const isEditing = computed(() => editingId.value !== null)

const filteredCategories = computed(() => {
  const keyword = search.value.trim().toLowerCase()

  if (!keyword) {
    return categories.value
  }

  return categories.value.filter((category) => {
    return (
      category.name.toLowerCase().includes(keyword) ||
      category.desc.toLowerCase().includes(keyword)
    )
  })
})

async function request<T>(path: string, options: RequestInit = {}): Promise<T> {
  const response = await fetch(`${apiBaseUrl}${path}`, {
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
      ...options.headers,
    },
    ...options,
  })

  const payload = (await response.json().catch(() => null)) as CategoryResponse | null

  if (!response.ok) {
    const validationMessage = payload?.errors
      ? Object.values(payload.errors).flat().join(' ')
      : payload?.message

    throw new Error(validationMessage || 'Something went wrong. Please try again.')
  }

  return payload as T
}

async function loadCategories() {
  isLoading.value = true
  error.value = ''

  try {
    const response = await request<CategoryResponse>('/categories')
    categories.value = Array.isArray(response.data) ? response.data : []
  } catch (err) {
    error.value = getErrorMessage(err)
  } finally {
    isLoading.value = false
  }
}

async function saveCategory() {
  if (!form.name.trim() || !form.desc.trim()) {
    error.value = 'Please enter both category name and description.'
    return
  }

  isSaving.value = true
  error.value = ''
  message.value = ''

  const payload: CategoryPayload = {
    name: form.name.trim(),
    desc: form.desc.trim(),
    is_active: form.is_active,
  }

  try {
    const response = isEditing.value
      ? await request<CategoryResponse>(`/categories/${editingId.value}`, {
          method: 'PUT',
          body: JSON.stringify(payload),
        })
      : await request<CategoryResponse>('/categories', {
          method: 'POST',
          body: JSON.stringify(payload),
        })

    message.value = response.message
    resetForm()
    await loadCategories()
  } catch (err) {
    error.value = getErrorMessage(err)
  } finally {
    isSaving.value = false
  }
}

function editCategory(category: Category) {
  editingId.value = category.id
  form.name = category.name
  form.desc = category.desc
  form.is_active = category.is_active
  message.value = ''
  error.value = ''
}

function resetForm() {
  editingId.value = null
  form.name = ''
  form.desc = ''
  form.is_active = true
}

async function deleteCategory(category: Category) {
  const confirmed = window.confirm(`Delete "${category.name}"? This cannot be undone.`)

  if (!confirmed) {
    return
  }

  deletingId.value = category.id
  error.value = ''
  message.value = ''

  try {
    const response = await request<CategoryResponse>(`/categories/${category.id}`, {
      method: 'DELETE',
    })

    message.value = response.message

    if (editingId.value === category.id) {
      resetForm()
    }

    await loadCategories()
  } catch (err) {
    error.value = getErrorMessage(err)
  } finally {
    deletingId.value = null
  }
}

function formatDate(value?: string) {
  if (!value) {
    return 'Recently'
  }

  return new Intl.DateTimeFormat('en', {
    dateStyle: 'medium',
    timeStyle: 'short',
  }).format(new Date(value))
}

function getErrorMessage(err: unknown) {
  return err instanceof Error ? err.message : 'Something went wrong. Please try again.'
}

onMounted(loadCategories)
</script>

<template>
  <main class="app-shell">
    <section class="page-heading">
      <div>
        <p class="eyebrow">Products Management</p>
        <h1>Category CRUD</h1>
      </div>

      <button class="ghost-button" type="button" :disabled="isLoading" @click="loadCategories">
        Refresh
      </button>
    </section>

    <section class="summary-grid" aria-label="Category summary">
      <div class="summary-item">
        <span>Total</span>
        <strong>{{ categories.length }}</strong>
      </div>
      <div class="summary-item">
        <span>Active</span>
        <strong>{{ activeCount }}</strong>
      </div>
      <div class="summary-item">
        <span>Inactive</span>
        <strong>{{ inactiveCount }}</strong>
      </div>
    </section>

    <section class="workspace">
      <form class="category-form" @submit.prevent="saveCategory">
        <div class="form-header">
          <div>
            <p class="eyebrow">{{ isEditing ? 'Update category' : 'New category' }}</p>
            <h2>{{ isEditing ? 'Edit category' : 'Create category' }}</h2>
          </div>
        </div>

        <label>
          <span>Name</span>
          <input v-model="form.name" type="text" placeholder="Example: Electronics" />
        </label>

        <label>
          <span>Description</span>
          <textarea v-model="form.desc" rows="5" placeholder="Short description"></textarea>
        </label>

        <label class="switch-row">
          <span>Status</span>
          <input v-model="form.is_active" type="checkbox" />
          <strong>{{ form.is_active ? 'Active' : 'Inactive' }}</strong>
        </label>

        <div class="form-actions">
          <button class="primary-button" type="submit" :disabled="isSaving">
            {{ isSaving ? 'Saving...' : isEditing ? 'Update' : 'Create' }}
          </button>
          <button v-if="isEditing" class="ghost-button" type="button" @click="resetForm">
            Cancel
          </button>
        </div>
      </form>

      <section class="category-panel">
        <div class="panel-header">
          <div>
            <p class="eyebrow">Laravel API</p>
            <h2>Categories</h2>
          </div>

          <input v-model="search" class="search-input" type="search" placeholder="Search categories" />
        </div>

        <p v-if="message" class="notice success">{{ message }}</p>
        <p v-if="error" class="notice error">{{ error }}</p>

        <div v-if="isLoading" class="empty-state">Loading categories...</div>

        <div v-else-if="filteredCategories.length === 0" class="empty-state">
          {{ categories.length === 0 ? 'No categories yet. Create the first one.' : 'No matching categories.' }}
        </div>

        <div v-else class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Name</th>
                <th>Description</th>
                <th>Status</th>
                <th>Updated</th>
                <th class="actions-column">Actions</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="category in filteredCategories" :key="category.id">
                <td>
                  <strong>{{ category.name }}</strong>
                </td>
                <td>{{ category.desc }}</td>
                <td>
                  <span class="status-pill" :class="{ inactive: !category.is_active }">
                    {{ category.is_active ? 'Active' : 'Inactive' }}
                  </span>
                </td>
                <td>{{ formatDate(category.updated_at || category.created_at) }}</td>
                <td class="actions-cell">
                  <button class="table-button" type="button" @click="editCategory(category)">Edit</button>
                  <button
                    class="table-button danger"
                    type="button"
                    :disabled="deletingId === category.id"
                    @click="deleteCategory(category)"
                  >
                    {{ deletingId === category.id ? 'Deleting...' : 'Delete' }}
                  </button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </section>
  </main>
</template>
