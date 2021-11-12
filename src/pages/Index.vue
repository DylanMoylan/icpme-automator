<template>
  <q-page class="flex flex-center">
    <div class="q-ma-lg">
      <q-banner class="bg-primary text-white q-px-lg text-center text-h6">
        ICPME Certificate Registration (Automated)
      </q-banner>
      <q-stepper v-model="step" color="primary" animated>
        <q-step 
          :name="1"
          title="1. Process Spreadsheet"
          icon="add_comment"
          :done="step > 1 && !!file"
        >
          <q-input
            filled
            stack-label
            label="Enter start time"
            v-model="startTime"
            type="time"
            class="q-mb-sm"
          />
          <q-input
            filled
            stack-label
            label="Enter minimum required attendance time"
            v-model="requiredTime"
            class="q-mb-sm"
          />
          <q-file
            filled
            v-model="file"
            label="Select CSV file to import"
            class="q-mb-sm"
            accept=".csv"
          >
            <template v-slot:prepend>
              <q-icon name="attach_file" />
            </template>
          </q-file>
        </q-step>
        <q-step
          :name="2"
          :done="step > 2"
          title="Verify Results"
          icon="edit"
        >
          <div class="text-bold q-mb-md">Eligible Participants: {{ eligible && eligible.length ? `(${eligible.length - 1})` : '' }}</div>
          <table class="results-table">
            <tbody>
              <q-virtual-scroll :items="filteredCsv && filteredCsv.length ? eligible : ['No unique entries found.']" separator style="height: 30vh">
                <template v-slot="{ item, index }">
                  <tr :key="`supp${index}`">
                    <td :class="index == 0 ? 'bg-grey-4' : ''">{{ item.name }}</td>
                    <td :class="index == 0 ? 'bg-grey-4' : ''">{{ item.email }}</td>
                    <td :class="index == 0 ? 'bg-grey-4' : ''">{{ item.login }}</td>
                    <td :class="index == 0 ? 'bg-grey-4' : ''">{{ item.logout }}</td>
                    <td :class="index == 0 ? 'bg-grey-4' : ''">{{ item.viewTime }}</td>
                  </tr>
                </template>
              </q-virtual-scroll>
            </tbody>
          </table>
          <div class="row justify-end q-my-sm">
            <q-btn
              label="Copy Email List"
              no-caps
              class="bg-positive text-white"
              @click="copyEmails('eligible')"
            />
          </div>
          <div class="text-bold q-mb-md">Ineligible Participants: {{ inEligible && inEligible.length ? `(${inEligible.length - 1})` : '' }}</div>
          <table class="results-table">
            <tbody>
              <q-virtual-scroll :items="filteredCsv && filteredCsv.length ? inEligible : ['No unique entries found.']" separator style="height: 30vh">
                <template v-slot="{ item, index }">
                  <tr :key="`supp${index}`">
                    <td :class="index == 0 ? 'bg-grey-4' : ''">{{ item.name }}</td>
                    <td :class="index == 0 ? 'bg-grey-4' : ''">{{ item.email }}</td>
                    <td :class="index == 0 ? 'bg-grey-4' : (item.invalidStart ? 'bg-table-invalid' : 'bg-table-valid')">{{ item.login }}</td>
                    <td :class="index == 0 ? 'bg-grey-4' : ''">{{ item.logout }}</td>
                    <td :class="index == 0 ? 'bg-grey-4' : (item.invalidTime ? 'bg-table-invalid' : 'bg-table-valid')">{{ item.viewTime }}</td>
                  </tr>
                </template>
              </q-virtual-scroll>
            </tbody>
          </table>
          <div class="row justify-end q-my-sm">
            <q-btn
              label="Copy Email List"
              no-caps
              class="bg-positive text-white"
              @click="copyEmails('inEligible')"
            />
          </div>
        </q-step>
        <template v-slot:navigation>
          <q-stepper-navigation>
            <q-btn
              @click="proceed"
              color="primary"
              label="Continue"
              :disable="formIncomplete"
            />
            <q-btn
              v-if="step > 1"
              flat
              color="primary"
              @click="step--"
              label="Back"
              class="q-ml-sm"
            />
          </q-stepper-navigation>
        </template>
      </q-stepper>
    </div>
  </q-page>
</template>

<script>

export default {
  data() {
    return {
      step: 1,
      file: null,
      csv: null,
      startTime: '12:00',
      requiredTime: '0:50'
    }
  },
  methods: {
    copyEmails(type) {
      const emails = this[type].filter(item => !/email/i.test(item.email)).map(item => item.email).join('\n')
      const text = document.createElement('textarea')
      document.body.appendChild(text)
      text.value = emails
      text.select()
      document.execCommand('copy')
      document.body.removeChild(text)
    },
    proceed() {
      if(this.step == 1) {
        this.readCSV()
      }
      this.step++
    },
    readCSV() {
      const reader = new FileReader()
      reader.onload = (e) => {
        this.csv = e.target.result
          .replace(/,+\n/gm, ',')
          .replace(/^,\s?(\n|$)/gm, '')
          .replace(/^"([^"]+)"/gm, "$1")
          .replace(/\s+,/gm, ',')
          .replace(/,?[^,]+$/gm, ',')
          .split(/,\n/)
          .map(item => item.trim())
        // this.csv = _.uniq(this.csv)
        this.csv[this.csv.length - 1] = this.csv[this.csv.length - 1].replace(/,$/, '')
      }
      reader.readAsText(this.file)
    },
    toMilli(timeString) {
      let time = timeString.split(':'),
        minutes = parseInt(time[1]),
        hours = parseInt(time[0])
      return (minutes * 60000) + (hours * (60000 * 60))
    },
    dateToMilli(timeString) {
      return new Date(`11/11/2021 ${timeString}`).getTime()
    }
  },
  computed: {
    filteredCsv() {
      if(this.csv) {
        return this.csv.map(item => {
          let details = item.split(',')
          let first = this.csv[0].split(',')
          let firstName = first.indexOf('First Name'),
            lastName = first.indexOf('Last Name'),
            email = first.indexOf('Email'),
            login = first.indexOf('Login'),
            logout = first.indexOf('Logout'),
            viewTime = first.indexOf('Time Viewing')
          return {
            name: `${details[lastName]} ${details[firstName]}`,
            email: details[email],
            login: details[login],
            logout: details[logout],
            viewTime: details[viewTime]
          }
        })
      } else {
        return []
      }
    },
    requiredTimeToMilli() {
      return this.toMilli(this.requiredTime)
    },
    eventStartToMilli() {
      return this.dateToMilli(this.startTime)
    },
    eligible() {
      /**
       * Filter the csv results based on user time viewing:
       * - user must have $requiredTime spent while the event was live.
       * For each user:
       *  if their log in time <= event start, their (log out time - event start) must be >= requiredTime
       *  if their log in time > their log in time must event start, their (logout time - login time) must >= requiredTime 
       */
      return this.filteredCsv.filter((item, index) => {
        let startTime = this.dateToMilli(item.login)
        let endTime = this.dateToMilli(item.logout)
        let requiredTime = this.requiredTimeToMilli
        let viewTime = this.toMilli(item.viewTime)
        return index == 0 || (
          viewTime >= requiredTime &&
          startTime <= Math.abs(this.eventStartToMilli + (10 * 60000)) &&
          (
            startTime <= this.eventStartToMilli ? Math.abs(endTime - this.eventStartToMilli) >= requiredTime :
            Math.abs(endTime - this.eventStartToMilli) >= requiredTime
          )
        )
      })
    },
    inEligible() {
      return this.filteredCsv.filter((item, index) => {
        return index == 0 || this.eligible.findIndex(user => item.email == user.email) < 0
      }).map(item => {
        return Object.assign({
          invalidTime: !(this.toMilli(item.viewTime) >= this.toMilli(this.requiredTime)),
          invalidStart: this.dateToMilli(item.login) > Math.abs(this.eventStartToMilli + (10 * 60000))
        }, item)
      })
    },
    formIncomplete() {
      if(this.step == 1) {
        // return !this.file || !this.startTime.length || !this.requiredTime.length
        return !this.file || !this.requiredTime.length
      }
    }
  }
}
</script>

<style>
.supp-instructions li {
  list-style: revert;
}
.supp-import-results:nth-child(odd) {
  background-color: rgb(241, 241, 241);
}
.results-table th {
  background-color:grey;
  padding: 5px;
}
.results-table td {
  border: 1px solid;
  padding: 5px;
}
.bg-table-invalid {
  background-color: #dd7171;
}
.bg-table-valid {
  background-color: #b1ffb1;
}
</style>