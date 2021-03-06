<template>
  <div>
    <header class="live-header">
      <div class="mw6 m-auto flex items-center pl15p justify-between top__header pl-15 pr15">
        <span class="back-arrow dib">
          <a v-on:click="goBack()" class="dib">
            <img width="22" height="22" src="../assets/img/svg/times.svg">
          </a>
        </span>

        <span class="header-title live-title">Pending ({{questions.length}})</span>
      </div>
    </header>

    <main class="pl-15 mw6 m-auto live-main pl15p pr15p">
      <div v-if="records_loaded" v-for="question in questions">
        <div class="q-bubble qa-item mb10p">
          <span>{{question.attrs.question}}</span>

          <div class="qa-item__seperator"></div>

          <div>{{pick_best_scenario_answer_for_questions[question.attrs._id] && pick_best_scenario_answer_for_questions[question.attrs._id].answer}}</div>
        </div>

        <div class="pending-submit">
          <router-link
            :to="{ name: 'viewpanswers', params: { qid: question.attrs._id, top_a: pick_best_scenario_answer_for_questions[question.attrs._id]._id }}"
            v-if="pick_best_scenario_answer_for_questions[question.attrs._id]"
          >View All</router-link>
          <a v-else></a>

          <div>
            <longpress
              duration="2"
              :on-confirm="deleteQ"
              :value="question.attrs._id"
              pressing-text="Deleting in {$rcounter} secs"
              class="btn del"
              action-text="Deleting"
            >Delete</longpress>

            <button
              v-if="pick_best_scenario_answer_for_questions[question.attrs._id]"
              v-on:click="publish(question.attrs)"
              class="btn pub ml20p"
            >
              <img
                width="22"
                height="22"
                src="../assets/img/svg/loading.svg"
                alt="loading"
                v-if="posting.indexOf(question.attrs._id) >= 0"
              >
              <span v-else>Publish</span>
            </button>
          </div>
        </div>
        <div>&nbsp;</div>
      </div>
    </main>
  </div>
</template>

<script>
import Question from "../models/Question";
import Answer from "../models/Answer";
import Vote from "../models/Vote";
import { BlockstackMixin } from "../mixins/BlockstackMixin.js";
import { NavMixin } from "../mixins/NavMixin.js";
import Longpress from "vue-longpress";

import { configure } from "radiks";
import moment from "moment";

export default {
  data: function() {
    return {
      questions: [],
      answers_for_qs: [],
      all_answers: [],
      votes_for_qs: [],
      qs_with_ans_and_vote_count: [],
      records_loaded: false,
      pick_best_scenario_answer_for_questions: [],
      posting: []
    };
  },
  //votecount will be inc'ted or dec'ted when the user cast a vote..but accurate vote can be viewed only on page refresh

  mixins: [BlockstackMixin, NavMixin],

  mounted() {},

  computed: {},
  watch: {},
  methods: {
    deleteQ: async function(id) {
      this.users = await axios.delete(`${process.env.API_PATH}/question/${id}`);

      location.reload();
    },

    //callback
    redirect: function(id) {
      this.$router.push({ name: "qdetails", params: { id: id } });
    },

    async publish(attrs) {
      this.posting.push(attrs._id);
      var res = this.displayChosenOrTopAnswer(attrs);
      var qModel = await Question.findById(attrs._id);

      qModel.update({ accepted_answer: res._id, accepted_user: res.user_id });
      //  qModel.update({last_event_fired: 'question_ended'});
      await qModel.save();


   var str = JSON.stringify({
        radiksType: "Notification",
        target_user: res.user_id,
        question_id: attrs._id,
        created_by: this.current_user.username,
        type: "answer_accepted"
      });
      // await axios.post(`${process.env.API_PATH}/notification/${str}` );

      await axios.post(
        `${process.env.RADIKS_SERVER}/notification/insert/${str}`
      );



      location.reload();
    },

    displayChosenOrTopAnswer(attrs) {
      //wait for load
      // if (!this.records_loaded) return;
      if (attrs.manually_chosen_as_top) {
        return this.all_answers[attrs.manually_chosen_as_top].attrs;
      } else {
        //if it has some ans and votes
        if (this.qs_with_ans_and_vote_count[attrs._id]) {
          var q = this.qs_with_ans_and_vote_count[attrs._id];

          var n = null;
          Object.keys(q).forEach(function(o) {
            if (!n) n = q[o];
            else if (n < q[o]) {
              n = q[o];
            }
          });
          var top_votes = Object.keys(q).filter(function(o) {
            return q[o] == n;
          });

          var ans_id = top_votes[0];

          return this.all_answers[ans_id].attrs;
        } else if (
          this.answers_for_qs[attrs._id] &&
          this.answers_for_qs[attrs._id].length > 0
        ) {
          //no answers have got votes..so select an answer
          return this.answers_for_qs[attrs._id][0].attrs;
        }
      }
    },

    clearError() {
      this.submit_error = false;
    },

    async fetchRecords() {
      this.questions = await Question.fetchList({
        expiring_at: { $lt: moment().unix() },
        accepted_answer: { $exists: false },
        user_id : this.current_user.username
      });
      var q_ids = new Array();
      this.questions.forEach(q => q_ids.push(q._id));

      //  this will be converted to $in array based on query-to-mongo
      var answers = await Answer.fetchList({ question_id: q_ids.join(",") });

      answers.forEach(a => {
        this.all_answers[a.attrs._id] = a;
        if (!this.answers_for_qs[a.attrs.question_id])
          this.answers_for_qs[a.attrs.question_id] = [];
        this.answers_for_qs[a.attrs.question_id].push(a);
      });

      var votes = await Vote.fetchList(
        { question_id: q_ids.join(",") },
        { decrypt: false }
      );

      votes.forEach(v => {
        if (!this.votes_for_qs[v.attrs.question_id])
          this.votes_for_qs[v.attrs.question_id] = [];
        this.votes_for_qs[v.attrs.question_id].push(v);
        if (!this.qs_with_ans_and_vote_count[v.attrs.question_id])
          this.qs_with_ans_and_vote_count[v.attrs.question_id] = {};

        if (
          !this.qs_with_ans_and_vote_count[v.attrs.question_id][
            v.attrs.answer_id
          ]
        )
          this.qs_with_ans_and_vote_count[v.attrs.question_id][
            v.attrs.answer_id
          ] = 0;

        this.qs_with_ans_and_vote_count[v.attrs.question_id][
          v.attrs.answer_id
        ] += v.attrs.vote;
      });

      this.questions.forEach(q => {
        this.pick_best_scenario_answer_for_questions[
          q._id
        ] = this.displayChosenOrTopAnswer(q.attrs);
      });

      this.records_loaded = true;
    }
  },

  created: function() {
    configure(this.RADIKS_SERVER);


    this.fetchRecords();
  },

  components: { Longpress }
};
</script>
