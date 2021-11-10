<template>
  <q-page class="flex flex-center">
    <div>
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
            type="time"
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
          <!-- <q-btn
            class="bg-positive text-white"
            label="Go"
            @click="readCSV"
            :disable="!file"
          /> -->
        </q-step>
        <q-step
          :name="2"
          :done="step > 2"
          title="Verify Results"
          icon="edit"
        >
          <div class="text-bold q-mb-md">Eligible Participants:</div>
          <table class="results-table">
            <tbody>
              <q-virtual-scroll :items="filteredCsv && filteredCsv.length ? filteredCsv : ['No unique entries found.']" separator style="height: 30vh">
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
          <div class="row justify-end q-mt-sm">
            <q-btn
              label="Copy Email List"
              no-caps
              class="bg-positive text-white"
              @click="copyEmails"
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
      startTime: '',
      requiredTime: ''
    }
  },
  methods: {
    copyEmails() {
      const emails = this.filteredCsv.filter(item => !/email/i.test(item.email)).map(item => item.email).join('\n')
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
          .replace(/,+/gm, ',')
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
    }
  },
  computed: {
    filteredCsv() {
      /**
       * Filter the csv results based on user time viewing:
       * - user must have $requiredTime spent while the event was live.
       * For each user:
       *  if their log in time <= event start, their (log out time - event start) must be >= requiredTime
       *  if their log in time > event start, their (logout time - login time) must >= requiredTime 
       */
      if(this.csv) {
        return this.csv.map((item, index) => {
          let details = item.split(',')
          return {
            name: `${details[2]} ${details[1]}`,
            email: details[3],
            login: index == 0 ? details[7] : details[9],
            logout: index == 0 ? details[8] : details[10],
            viewTime: index == 0 ? details[9] : details[11]
          }
        })
      } else {
        return []
      }
    },
    formIncomplete() {
      if(this.step == 1) {
        // return !this.file || !this.startTime.length || !this.requiredTime.length
        return !this.file
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
</style>