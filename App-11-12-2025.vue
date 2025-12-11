<script setup lang="ts">
import { ref, onMounted } from 'vue'

// Types
interface Contact {
  id: number
  name: string
  email: string
  phone: string
}

interface NewContact {
  name: string
  email: string
  phone: string
}

// Etats
const contacts = ref<Contact[]>([])
const loading = ref(false)
const error = ref<string | null>(null)

// Formulaire d'ajout
const newContact = ref<NewContact>({
  name: '',
  email: '',
  phone: ''
})
const adding = ref(false)

// Edition
const editingId = ref<number | null>(null)
const editForm = ref<NewContact>({
  name: '',
  email: '',
  phone: ''
})
const saving = ref(false)

// Suppression
const deletingId = ref<number | null>(null)

const API_URL = 'https://jsonplaceholder.typicode.com/users'

// ============ READ ============
async function fetchContacts() {
  loading.value = true
  error.value = null

  try {
    const response = await fetch(API_URL)
    if (!response.ok) throw new Error('Erreur de chargement')

    const data = await response.json()
    // Limiter a 5 pour la demo
    contacts.value = data.slice(0, 5)
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Erreur'
  } finally {
    loading.value = false
  }
}

// ============ CREATE ============
async function addContact() {
  if (!newContact.value.name || !newContact.value.email) return

  adding.value = true

  try {
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(newContact.value)
    })

    if (!response.ok) throw new Error('Erreur creation')

    const created = await response.json()

    // Ajouter localement avec un ID unique
    contacts.value.unshift({
      ...created,
      id: Date.now() // ID unique local
    })

    // Reset formulaire
    newContact.value = { name: '', email: '', phone: '' }
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Erreur'
  } finally {
    adding.value = false
  }
}

// ============ UPDATE ============
function startEdit(contact: Contact) {
  editingId.value = contact.id
  editForm.value = {
    name: contact.name,
    email: contact.email,
    phone: contact.phone
  }
}

function cancelEdit() {
  editingId.value = null
  editForm.value = { name: '', email: '', phone: '' }
}

async function saveEdit(id: number) {
  saving.value = true

  try {
    const response = await fetch(`${API_URL}/${id}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(editForm.value)
    })

    if (!response.ok) throw new Error('Erreur modification')

    // Mettre a jour localement
    const index = contacts.value.findIndex(c => c.id === id)
    if (index !== -1) {
      contacts.value[index] = {
        ...contacts.value[index],
        ...editForm.value
      }
    }

    cancelEdit()
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Erreur'
  } finally {
    saving.value = false
  }
}

// ============ DELETE ============
async function deleteContact(id: number) {
  if (!confirm('Supprimer ce contact ?')) return

  deletingId.value = id

  try {
    const response = await fetch(`${API_URL}/${id}`, {
      method: 'DELETE'
    })

    if (!response.ok) throw new Error('Erreur suppression')

    // Supprimer localement
    contacts.value = contacts.value.filter(c => c.id !== id)
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Erreur'
  } finally {
    deletingId.value = null
  }
}

onMounted(fetchContacts)
</script>

<template>
  <div class="contact-manager">
    <h1>Gestion des Contacts</h1>

    <!-- Formulaire d'ajout -->
    <form @submit.prevent="addContact" class="add-form">
      <h2>Nouveau Contact</h2>
      <div class="form-row">
        <input
          v-model.trim="newContact.name"
          placeholder="Nom *"
          required
        >
        <input
          v-model.trim="newContact.email"
          type="email"
          placeholder="Email *"
          required
        >
        <input
          v-model.trim="newContact.phone"
          placeholder="Telephone"
        >
        <button type="submit" :disabled="adding">
          {{ adding ? 'Ajout...' : 'Ajouter' }}
        </button>
      </div>
    </form>

    <!-- Message d'erreur -->
    <div v-if="error" class="error-msg">
      {{ error }}
      <button @click="error = null">X</button>
    </div>

    <!-- Chargement -->
    <div v-if="loading" class="loading">
      Chargement des contacts...
    </div>

    <!-- Liste des contacts -->
    <div v-else class="contacts-list">
      <div
        v-for="contact in contacts"
        :key="contact.id"
        class="contact-card"
      >
        <!-- Mode edition -->
        <template v-if="editingId === contact.id">
          <div class="edit-form">
            <input
              v-model.trim="editForm.name"
              placeholder="Nom"
            >
            <input
              v-model.trim="editForm.email"
              type="email"
              placeholder="Email"
            >
            <input
              v-model.trim="editForm.phone"
              placeholder="Telephone"
            >
          </div>
          <div class="actions">
            <button
              class="btn-save"
              @click="saveEdit(contact.id)"
              :disabled="saving"
            >
              {{ saving ? 'Sauvegarde...' : 'Sauvegarder' }}
            </button>
            <button
              class="btn-cancel"
              @click="cancelEdit"
            >
              Annuler
            </button>
          </div>
        </template>

        <!-- Mode affichage -->
        <template v-else>
          <div class="contact-info">
            <h3>{{ contact.name }}</h3>
            <p>{{ contact.email }}</p>
            <p>{{ contact.phone }}</p>
          </div>
          <div class="actions">
            <button
              class="btn-edit"
              @click="startEdit(contact)"
            >
              Modifier
            </button>
            <button
              class="btn-delete"
              @click="deleteContact(contact.id)"
              :disabled="deletingId === contact.id"
            >
              {{ deletingId === contact.id ? '...' : 'Supprimer' }}
            </button>
          </div>
        </template>
      </div>

      <p v-if="contacts.length === 0" class="empty">
        Aucun contact
      </p>
    </div>
  </div>
</template>

<style scoped>
.contact-manager {
  max-width: 700px;
  margin: 0 auto;
  padding: 20px;
}

.add-form {
  background: #f8f9fa;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.add-form h2 {
  margin: 0 0 15px;
  font-size: 18px;
}

.form-row {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.form-row input {
  flex: 1;
  min-width: 150px;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.form-row button {
  padding: 10px 20px;
  background: #42B883;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.error-msg {
  background: #f8d7da;
  color: #721c24;
  padding: 10px 15px;
  border-radius: 4px;
  margin-bottom: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.error-msg button {
  background: none;
  border: none;
  font-size: 18px;
  cursor: pointer;
}

.loading {
  text-align: center;
  padding: 40px;
  color: #666;
}

.contacts-list {
  display: grid;
  gap: 15px;
}

.contact-card {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 15px;
}

.contact-info h3 {
  margin: 0 0 5px;
  color: #333;
}

.contact-info p {
  margin: 3px 0;
  color: #666;
  font-size: 14px;
}

.edit-form {
  display: flex;
  gap: 10px;
  flex: 1;
}

.edit-form input {
  flex: 1;
  padding: 8px;
  border: 1px solid #42B883;
  border-radius: 4px;
}

.actions {
  display: flex;
  gap: 10px;
}

.actions button {
  padding: 8px 15px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.btn-edit {
  background: #007bff;
  color: white;
}

.btn-delete {
  background: #dc3545;
  color: white;
}

.btn-save {
  background: #42B883;
  color: white;
}

.btn-cancel {
  background: #6c757d;
  color: white;
}

.actions button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.empty {
  text-align: center;
  color: #666;
  padding: 40px;
}
</style>
