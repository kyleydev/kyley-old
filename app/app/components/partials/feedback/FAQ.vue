<template>
  <section id="ffb-faq">
    <!-- Spinner -->
    <div v-if="!state.faq" class="h4 text-center">
      <i class="fa fa-spinner fa-spin fa-2x fa-fw" aria-hidden="true"></i>
    </div>

    <!-- Answers -->
    <div v-else
      :class="`${/^all$/.test(state.toggle) ? 'all' : ''} ${search !== '' ? 'filter' : 'nofilter'}`">

      <div v-if="siteIn.txt"
        class="subtitle"
        v-html="`${$t('feedback.faq')} <b>${$root.txt[$root.site]}</b>`">
      </div>
      <div v-else
        class="subtitle"
        v-html="$t('feedback.mainFaq')">
      </div>

      <div v-for="(item, index) in faq"
        :key="index"
        :class="`list-group-item ${state.toggle === item.id ? 'active' : ''} ${item.class}`">
        <a v-show="state.toggle === item.id"
          href="#!"
          class="pull-right close"
          aria-hidden="true"
          onclick="return false"
          @click="state.toggle = 'all'">
          ×
        </a>
        <a
          :id="item.id"
          :href="`${$root.link.contact}/faq/#${item.id}`"
          class="toggle"
          onclick="return false"
          @click="toggleAnswer(item.id)">
        </a>
        <h3 class="list-group-item-heading"
          @click="state.toggle = 'all'"
          v-html="item.question">
        </h3>
        <div class="list-group-item-text" v-html="item.answer">
        </div>
        <p class="text-center">
          <a v-show="state.toggle === item.id"
            href="#!"
            class="btn btn-xs btn-default"
            onclick="return false"
            @click="state.toggle = 'all'"
            v-text="$t('txt.close')">
          </a>
        </p>
      </div>
      <p v-if="!state.mainFaq && state.faq && state.toggle === 'all'">
        <button class="btn btn-default btn-xs btn-block"
          @click="addMainFaq()"
          v-html="$t('feedback.more')">
        </button>
      </p>
    </div>

    <!-- No answer ? -->
    <div v-show="state.toggle === 'all'" class="no-answer">
      <p v-show="state.faq" v-html="$t('feedback.noanswer')"></p>
      <div class="row">
        <div class="col-xs-6">
          <a :href="$root.link.colibri"
            class="btn btn-default btn-block">
            <span class="fa-stack fa-2x" aria-hidden="true">
              <i class="fa fa-circle fa-stack-2x fc_o5"></i>
              <i class="fa fa-stack-1x fa-comment fa-inverse"></i>
            </span>
            <br>
            <span v-html="$t('feedback.forum')"></span>
          </a>
        </div>
        <div class="col-xs-6">
          <a :href="`${$root.link.contact}/#${$root.lname}`"
            class="btn btn-default btn-block"
            onclick="return false;"
            @click="showContact()">
            <span class="fa-stack fa-2x" aria-hidden="true">
              <i class="fa fa-circle fa-stack-2x fc_o5"></i>
              <i class="fa fa-stack-1x fa-envelope fa-inverse"></i>
            </span>
            <br>
            <span v-html="`${$t('feedback.contact')} ${$root.color.soft}`"></span>
          </a>
        </div>
      </div>
    </div>
  </section>
</template>

<script>
export default {
  props: {
    open: {
      type: Boolean,
      required: true,
    },
    search: {
      type: String,
    }
  },
  data() {
    return {
      general: !(Object.keys(this.$root.txt).indexOf(this.$root.site) > -1),
      mainFaq: [],
      faq: [],
      siteIn: {
        txt: this.$root.txt[this.$root.site] !== undefined,
        status: this.$root.status[this.$root.site] !== undefined,
      },
      state: {
        mainFaq: false,
        faq: false,
        toggle: 'all',
        status: false,

      },
    }
  },
  watch: {
    open: function (newValue) {
      if (newValue) {
        this.loadFaq();
      }
    },
    search: function (newValue) {
      if (newValue !== '') {
        this.searchUpdate();
      }
    }
  },
  methods: {
    loadFaq() {
      if (!this.state.faq && this.$parent.section === 'faq') {
        fetch('https://contact.framasoft.org/fr/faq/index.html')
          .then((res) => {
            return res.text();
          })
          .then((data) => {
            const parser = new DOMParser();
            let html = null;

            /* Main Faq */
            html = parser.parseFromString(data, "text/html").querySelector('#autre');
            if (html) {
              for (let i = 1; i < html.children.length; i += 1) { // .list-group-item only (first node is h2)
                this.mainFaq[i - 1] = {
                  id: html.children[i].querySelector('h3 a').id,
                  question: html.children[i].querySelector('h3 span').innerHTML,
                  answer: html.children[i].querySelector('.list-group-item-text').innerHTML,
                  class: '',
                }
              }
            }

            /* Site Faq */
            if (!this.general) {
              html = parser.parseFromString(data, "text/html").querySelector(`#${this.text(this.$root.txt[this.$root.site], 'latin sanitize')}`);
              if (html) {
                for (let i = 1; i < html.children.length; i += 1) { // .list-group-item only (first node is h2)
                  this.faq[i - 1] = {
                    id: html.children[i].querySelector('h3 a').id,
                    question: html.children[i].querySelector('h3 span').innerHTML,
                    answer: html.children[i].querySelector('.list-group-item-text').innerHTML,
                    class: '',
                  }
                }
              }
            }
            if (this.faq.length === 0) {
              this.faq = this.mainFaq;
              this.state.mainFaq = true;
            }
            this.state.faq = true;

            this.checkStatus();
            this.loadScheduledIncidents();
          }).catch(function (err) {
            console.error(err); // eslint-disable-line
          });
      }
    },
    checkStatus() {
      /* Status */
      if (!this.state.status && this.siteIn.status) {
        fetch(`https://status.framasoft.org/api/v1/components?id=${this.$root.status[this.$root.site]}`).then((res) => {
            return res.json();
          })
          .then((data) => {
            if (data.data[0].status > 1) { // There is a incident
              const cssClass = data.data[0].status < 4 ? 'bg-warning' : 'bg-danger';
              this.loadLastIncident(this.$root.status[this.$root.site], cssClass);
            }
            this.state.status = true;
          }).catch(function (err) {
            console.error(err); // eslint-disable-line
          });
      }
    },
    loadLastIncident(id, cssClass) {
      fetch(`https://status.framasoft.org/api/v1/incidents?component_id=${id}&sort=id&order=desc`)
        .then((res) => {
          return res.json();
        })
        .then((data) => {
          if (data.data.length > 0) {
            this.faq.unshift({
              id: 'status_0',
              question: data.data[0].name,
              answer: `<p>${data.data[0].message.replace('\r\n', '<br>')}</p>`,
              class: cssClass,
            });
          }
        }).catch(function (err) {
          console.error(err); // eslint-disable-line
        });
    },
    loadScheduledIncidents() {
      fetch(`https://status.framasoft.org/api/v1/incidents?component_id=0&sort=id&order=desc`)
        .then((res) => {
          return res.json();
        })
        .then((data) => {
          for (let i = 0; i < data.data.length; i += 1) {
            if (this.is.before(data.data[i].scheduled_at.replace(/\-/g, '/').substr(0, 10))
              && this.siteIn.txt) {
              const reg = new RegExp(`(${this.text(this.$root.txt[this.$root.site], 'latin').toLowerCase()})`, 'g');
              const content = this.text(`${data.data[i].message} ${data.data[i].name}`, 'latin').toLowerCase();
              console.log(reg, content); // eslint-disable-line
              if (reg.test(content)) {
                this.faq.unshift({
                  id: `status_${i + 1}`,
                  question: data.data[i].name,
                  answer: `<p>${data.data[i].message.replace('\r\n', '<br>')}</p>`,
                  class: 'bg-info',
                });
              }
            }
          }
        }).catch(function (err) {
          console.error(err); // eslint-disable-line
        });
    },
    toggleAnswer(id) {
      this.state.toggle = this.state.toggle === 'all' ? id : 'all';
      this.$parent.$refs['ffb-top'].scrollIntoView();
    },
    showContact() {
      this.$parent.section = 'contact-faq';
      this.$parent.$refs['ffb-top'].scrollIntoView();
    },
    addMainFaq() {
      this.faq = this.faq.concat(this.mainFaq);
      this.state.mainFaq = true;
    },
    searchUpdate() {
      if (!this.state.mainFaq) { /* search = more results */
        this.addMainFaq();
      }
      this.state.toggle = 'all';

      for (let i = 1; i < this.faq.length; i += 1) {
        const words = this.text(this.search, 'latin').toLowerCase().replace(', ', ',').split(',');
        const reg = new RegExp(`(${words.join('|')})`, 'g');
        const content = this.text(`${this.faq[i].question} ${this.faq[i].answer}`, 'latin').toLowerCase();
        if (reg.test(content)) {
          this.faq[i].class = `${this.faq[i].class.replace(/search/g, '')} search`;
        } else {
          this.faq[i].class = this.faq[i].class.replace(/search/g, '');
        }
      }
    }
  }
}
</script>