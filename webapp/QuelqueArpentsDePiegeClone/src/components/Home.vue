<template>
  <transition name="fade">
    <div class="home" v-if="!secondplayer">
      <div class="inner">
        <h1>Quelque Arpents de Pièges Clone</h1>
        <p>Avez-vous les connaissances requises pour survoler ces questions ?</p>
        <p>Inviter un ami avec ce lien: {{url}}.</p>
      </div>
    </div>
    <div class="play" v-if="secondplayer">
      <div>
        <div class="container hamilton--header--text">
          <h1>Quelque Arpents De Pièges</h1>

          <div class="columns hamilton--inner">
            <div class="column is-half left">
              <p class="title">User 1</p>
              <p class="subtitle">Total Score: {{playerdata.one.score}}</p>
            </div>
            <div v-if="secondplayer" class="column is-half right">
              <p class="title">User 2</p>
              <p class="subtitle">Total Score: {{playerdata.two.score}}</p>
            </div>
          </div>

          <div class="hamilton--lyrics--text">
            <p>{{question.question}}
            </p>
            <div class="hamilton--answers">
              <a v-bind:class="{ 'wronganswer': hasAnswered && !isCorrect, 'correctanswer': hasAnswered && isCorrect}" @click="checkAnswer(item)" v-for="(item, index) in options">{{item}}</a>
            </div>
          </div>
        </div>
      </div>
    </div>
  </transition>
</template>

<script>
  import ChannelDetails from '@/components/ChannelDetails'
  import axios from 'axios'

  export default {
    name: 'home',
    data () {
      return {
        token: null,
        presenceid: null,
        questions: null,
        hasAnswered: false,
        isCorrect: true,
        question: null,
        options: null,
        correctanswer: null,
        count: null,
        players: 1,
        secondplayer: false,
        playerdata: {
          one: {
            id: null,
            score: 0,
            userid: null
          },
          two: {
            id: null,
            score: 0,
            userid: null
          }
        },
        userid: null,
        url: null
      }
    },
    async created () {
      // get game token
      await axios
      .get('https://opentdb.com/api_token.php?command=request')
      .then(response => (this.token = response.data.token))

      this.fetchData()
    },
    methods: {
      fetchData () {
        this.presenceid = this.getUniqueId()
        if (!this.checkPresenceID()) {
          var separator = (window.location.href.indexOf('?') === -1) ? '?' : '&'
          window.location.href = window.location.href + separator + this.presenceid
        }
        this.url = window.location.href
        this.getNewQuestion()
        let channel = ChannelDetails.subscribeToPusher()
        channel.bind('pusher:member_added', members => {
          this.players += 1
          this.secondplayer = true
        })
        channel.bind('pusher:subscription_succeeded', members => {
          if (members.count === 1 && !this.playerdata.one.id) {
            this.playerdata.one.id = members.myID
            this.playerdata.one.userid = 1
            this.userid = 1
          } else if (members.count === 2) {
            this.secondplayer = true
            this.playerdata.two.id = members.myID
            this.playerdata.two.userid = 2
            this.userid = 2
          }
        })
        channel.bind('pusher:member_removed', member => {
          this.players -= 1
          if (member.count === 1) {
            this.secondplayer = false
          }
        })
        channel.bind('client-send', (data) => {
          if (this.userid === 1) {
            this.playerdata.two.score = data.data.two.score
          } else if (this.userid === 2) {
            this.playerdata.one.score = data.data.one.score
          }
        })
      },
      getUniqueId () {
        return 'id=' + Math.random().toString(36).substr(2, 8)
      },
      checkPresenceID () {
        let getQueryString = (field, url) => {
          let href = url ? url : window.location.href
          let reg = new RegExp('[?&]' + field + '=([^&#]*)', 'i')
          let string = reg.exec(href)
          return string ? string[1] : null
        }
        let id = getQueryString('id')
        return id
      },
      checkAnswer (item) {
        let channel = ChannelDetails.subscribeToPusher()
        this.hasAnswered = true
        console.log(item)
        console.log(this.correctanswer)
        if (item === this.correctanswer) {
          this.isCorrect = true
          if (this.userid === 1) {
            this.playerdata.one.score += 10
          } else if (this.userid === 2) {
            this.playerdata.two.score += 10
          }
        } else {
          if (this.userid === 1) {
            this.playerdata.one.score = Math.max(0, this.playerdata.one.score -= 10)
          } else if (this.userid === 2) {
            this.playerdata.two.score = Math.max(0, this.playerdata.two.score -= 10)
          }
        }
        channel.trigger('client-send', {data: this.playerdata})
        this.count = 3
        let countdown = setInterval(() => {
          this.count -= 1
          if (this.count === 0) {
            clearInterval(countdown)
            this.getNewQuestion()
          }
        }, 1000)
      },
      async getNewQuestion () {
        let question
        await axios
        .get(`https://opentdb.com/api.php?amount=1&token=${this.token}`)
        .then(response => (question = response.data.results[0]))

        let correctAnswer = question.correct_answer
        let incorrectAnswers = question.incorrect_answers
        incorrectAnswers.push(correctAnswer)
        let options = this.shuffle(incorrectAnswers)

        this.question = question
        this.options = options
        this.correctanswer = question.correct_answer
        this.hasAnswered = false
        this.isCorrect = false
      },
      // Knuth Shuffle -> https://github.com/coolaj86/knuth-shuffle
      shuffle (array) {
        console.log(array)
        var currentIndex = array.length
        let temporaryValue
        let randomIndex

        // While there remain elements to shuffle...
        while (currentIndex !== 0) {
          // Pick a remaining element...
          randomIndex = Math.floor(Math.random() * currentIndex)
          console.log(currentIndex)
          currentIndex -= 1

          // And swap it with the current element.
          temporaryValue = array[currentIndex]
          array[currentIndex] = array[randomIndex]
          array[randomIndex] = temporaryValue
        }

        return array
      }
    }
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .home {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100vh;
  }
  h1 {
    font-size: 3rem;
    font-weight: bold;
  }
  p {
    font-size: 1.5rem;
    margin: 0 0 20px 0;
  }
  .play--button {
    background-color: white;
    color: #7fd4d3;
    font-weight: bold;
    border-radius: 20px;
    letter-spacing: 1px;
    padding: 20px;
    transition: all .3s ease;
    text-shadow: 0 1px 3px rgba(36,180,126,.4);
    text-transform: uppercase;
    box-shadow: 0 4px 6px rgba(50,50,93,.11), 0 1px 3px rgba(0,0,0,.08);
  }
  .play--button:hover {
    background-color: white;
    color: #7fd4d3;
    transform: translateY(-1px);
    box-shadow: 0 7px 14px rgba(50,50,93,.1), 0 3px 6px rgba(0,0,0,.08);
  }
  .fade-enter-active, .fade-leave-active {
    transition: opacity .5s
  }
  .fade-enter, .fade-leave-to {
    opacity: 0
  }
  a {
    color: #fff;
  }
  p {
    color: #fff;
  }
  h1 {
    font-size: 3rem;
    font-weight: bold;
    text-align: center;
  }
  .fade-enter-active, .fade-leave-active {
    transition: opacity .5s
  }
  .fade-enter, .fade-leave-to /* .fade-leave-active in <2.1.8 */ {
    opacity: 0
  }
  .play--button {
    background-color: white;
    color: #7fd4d3;
    font-weight: bold;
    border-radius: 20px;
    letter-spacing: 1px;
    padding: 20px;
    transition: all .3s ease;
    text-shadow: 0 1px 3px rgba(36,180,126,.4);
    text-transform: uppercase;
    box-shadow: 0 4px 6px rgba(50,50,93,.11), 0 1px 3px rgba(0,0,0,.08);
    position: absolute;
    top: 20px;
    right: 20px;
    z-index: 5;
  }
  .play--button:hover {
    background-color: white;
    color: #7fd4d3;
    transform: translateY(-1px);
    box-shadow: 0 7px 14px rgba(50,50,93,.1), 0 3px 6px rgba(0,0,0,.08);
  }
  .hamilton--header--text {
    margin-top: 50px;
  }
  .hamilton--inner {
    margin-top: 20px;
  }
  .hamilton--inner .left{
    text-align: left;
  }
  .hamilton--inner .right{
    text-align: right;
  }
  .title {
    font-weight: bold;
  }
  .hamilton--lyrics--text {
    width: 600px;
    margin: 0 auto;
  }
  .hamilton--lyrics--text p {
    font-weight: bold;
  }
  .hamilton--answers a{
    display: block;
    border: 3px solid white;
    border-radius: 50px;
    margin: 20px auto;
    width: 500px;
    padding: 10px;
  }
  .wronganswer {
    background-color: #ec6969;
    border: none !important;
    opacity: 0.4;
    transition: background-color 0.5s ease;
  }
  .correctanswer {
    background-color: #00c4a7;
    border: none !important;
    transition: background-color 0.5s ease;
  }
</style>
