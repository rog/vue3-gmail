<template>
  <button
    @click="selectScreen('inbox')"
    :disabled="selectedScreen === 'inbox'"
  >Inbox</button>
  <button
    @click="selectScreen('archived')"
    :disabled="selectedScreen === 'archived'"
  >Archived</button>

  <BulkActionBar :emails="filteredEmails" :selectedScreen="selectedScreen" />
  <table class="mail-table">
    <tbody>
      <tr v-for="email in filteredEmails"
          :key="email.id"
          :class="['clickable', email.read ? 'read' : '']"
      >
        <td>
          <input type="checkbox"
            @click="emailSelection.toggle(email)"
            :checked="emailSelection.emails.has(email)"
          />
        </td>
        <td @click="openEmail(email)">{{email.from}}</td>
        <td @click="openEmail(email)">
          <p><strong>{{email.subject}}</strong> - {{email.body}}</p>
        </td>
        <td class="date">{{format(new Date(email.sentAt), 'MMM do yyyy')}}</td>
        <td><button @click="archiveEmail(email)">{{selectedScreen === 'inbox' ? 'Archive' : 'Inbox'}}</button></td>
      </tr>
    </tbody>
  </table>
  <ModalView v-if="openedEmail" @closeModal="openedEmail = null">
    <MailView :email="openedEmail" @changeEmail="changeEmail" />
  </ModalView>
</template>

<script>
  import { reactive, ref } from "vue";
  import { format } from 'date-fns';
  import axios from 'axios';

  import ModalView from '@/components/ModalView.vue';
  import MailView from '@/components/MailView.vue';
  import BulkActionBar from '@/components/BulkActionBar.vue';

  import useEmailSelection from '../composables/use-email-selection'

  export default {
    async setup(){
      let {data: emails} = await axios.get('http://localhost:3000/emails')

      return {
        emailSelection: useEmailSelection(),
        format,
        emails: ref(emails),
        openedEmail: ref(null),
        selectedScreen: ref('inbox')
      }
    },
    components: {
      MailView,
      ModalView,
      BulkActionBar
    },
    computed: {
      sortedEmails() {
        return this.emails.sort((e1, e2) => {
          return e1.sentAt < e2.sentAt ? 1 : -1
        })
      },
      filteredEmails() {
        if (this.selectedScreen === 'inbox') {
          return this.sortedEmails.filter(e => !e.archived)
        }
        if (this.selectedScreen === 'archived') {
          return this.sortedEmails.filter(e => e.archived)
        }
      }
    },
    methods: {
      selectScreen (newScreen) {
        this.selectedScreen = newScreen
        this.emailSelection.clear()
      },
      openEmail(email) {
        this.openedEmail = email
        if(email) {
          email.read = true
          this.updateEmail(email)
        }
      },
      archiveEmail(email) {
        email.archived = !email.archived
        this.updateEmail(email)
      },
      updateEmail(email) {
        axios.put(`http://localhost:3000/emails/${email.id}`, email)
      },
      changeEmail({toggleRead, toggleArchive, save, closeModal, changeIndex}) {
        let email = this.openedEmail
        if (toggleRead) { email.read = !email.read }
        if (toggleArchive) { email.archived = !email.archived }
        if (save) { this.updateEmail(email) }
        if (closeModal) { this.openedEmail = null }
        if (changeIndex) {
          let emails = this.unarchivedEmails
          let currentIndex = emails.indexOf(this.openedEmail)
          let newEmail = emails[currentIndex + changeIndex]
          this.openEmail(newEmail)
        }
      }
    }
  }
</script>

<style scoped>
</style>
