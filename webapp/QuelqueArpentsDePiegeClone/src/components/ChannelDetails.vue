<script>
  import Pusher from 'pusher-js'
  import pusherSecrets from '../../secrets/PusherSecrets.js'
  
  const pusher = new Pusher(pusherSecrets.key, {
    cluster: pusherSecrets.cluster,
    encrypted: true,
    authEndpoint: 'http://localhost:5000/pusher/auth'
  })

  export default ({
    getPresenceID () {
      let getQueryString = (field, url) => {
        let href = url ? url : window.location.href
        let reg = new RegExp('[?&]' + field + '=([^&#]*)', 'i')
        let string = reg.exec(href)
        return string ? string[1] : null
      }
      let id = getQueryString('id')
      id = 'presence-' + id
      return id
    },
    subscribeToPusher () {
      let presenceid = this.getPresenceID()
      let channel = pusher.subscribe(presenceid)
      return channel
    }
  })
</script>
