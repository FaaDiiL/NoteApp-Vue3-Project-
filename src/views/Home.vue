<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { useSignOut, useUserId } from '@nhost/vue'
import { useMutation, useQuery } from '@vue/apollo-composable'
import { gql } from '@apollo/client/core'

const router = useRouter()
const { signOut } = useSignOut()
const userId = useUserId()

const newNote = ref({
  title: '',
  content: '',
})

const logout = () => {
  signOut()

  router.push('/login')
}

const {
  loading: notesLoading,
  result: notesResult,
  refetch: notesRefetch,
} = useQuery(
  gql`
    query ($userId: String!) {
      notes(order_by: { created: desc }, where: { user_id: { _eq: $userId } }) {
        id
        title
        content
        created
      }
    }
  `,
  {
    userId: userId.value,
  }
)

const { mutate: createNote, onDone: createNoteDone } = useMutation(gql`
  mutation ($userId: String!, $title: String!, $content: String!) {
    insert_notes_one(
      object: { content: $content, title: $title, user_id: $userId }
    ) {
      id
    }
  }
`)

const handleCreateNote = () => {
  if (!newNote.value.title || !newNote.value.content) {
    return alert('Please fill in all fields')
  }

  createNote({
    userId: userId.value,
    title: newNote.value.title,
    content: newNote.value.content,
  })
}

createNoteDone(() => {
  notesRefetch()
  newNote.value = {
    title: '',
    content: '',
  }
})

const { mutate: deleteNote, onDone: deleteDone } = useMutation(gql`
  mutation ($id: Int!) {
    delete_notes(where: { id: { _eq: $id } }) {
      affected_rows
    }
  }
`)

deleteDone(() => {
  notesRefetch()
})

const convertToHTML = (content) => {
  return content.replace(/\n/g, '<br />')
}

const convertToDate = (date) => {
  return new Date(date).toLocaleString('sv-SE')
}
</script>

<template>
  <main>
    <div class="flex justify-between mb-8 item-center">
      <h1 class="text-3xl font-bold">My Note App</h1>
      <button
        @click="logout"
        class="text-red-500 cursor-pointer hover:underline"
      >
        Logout
      </button>
    </div>

    <form @submit.prevent="handleCreateNote" class="mb-8">
      <label class="block mb-4">
        <span class="block mb-2 text-sm uppercase">Title</span>
        <input
          type="text"
          v-model="newNote.title"
          placeholder="What's the title"
          class="block w-full px-4 py-2 text-slate-800"
        />
      </label>
      <label class="block mb-4">
        <span class="block mb-2 text-sm uppercase">Content</span>
        <textarea
          type="text"
          v-model="newNote.content"
          placeholder="Write your content..."
          class="block w-full px-4 py-2 text-slate-800"
        ></textarea>
      </label>

      <input
        type="submit"
        value="Create note"
        class="text-green-500 cursor-pointer hover:underline"
      />
    </form>

    <div v-if="!notesLoading">
      <div
        v-for="note in notesResult.notes"
        :key="note.id"
        class="relative p-6 mb-6 bg-white rounded-lg text-slate-700"
      >
        <button
          class="absolute text-red-500 top-6 right-6"
          @click="() => deleteNote({ id: note.id })"
        >
          Delete
        </button>
        <h3 class="mb-4 text-2xl font-bold">{{ note.title }}</h3>
        <p
          class="mb-4 text-lg text-gray-500"
          v-html="convertToHTML(note.content)"
        ></p>
        <div class="text-sm italic text-gray-500">
          {{ convertToDate(note.created) }}
        </div>
      </div>
    </div>
  </main>
</template>
