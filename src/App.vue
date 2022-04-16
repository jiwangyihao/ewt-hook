<template>
  <div :class="{container: true, fullscreen: fullscreen, pause:pause}">
    <iframe :src="href" height="100%" width="100%" sandbox="allow-forms allow-same-origin allow-scripts"></iframe>
    <label>
      <input v-model="inputText" id="focusHelp" type="text" @input="input" @keyup="keyup"/>
    </label>
    <div id="mouseHelp" @mouseup="mouseup" @mouseleave="wheel" @wheel="wheel"></div>
    <audio id="player" ref="player" :src="src" @ended="ended" @canplay="canplay"></audio>
  </div>
</template>

<!--suppress JSUnusedGlobalSymbols -->
<script setup>
import { useQuasar } from 'quasar'
import { ref, watch } from "vue"
import { useRouter, useRoute } from 'vue-router'

const $q = useQuasar()
const router = useRouter()
const route = useRoute()

const href = ref("https://www.ewt360.com/")
const fullscreen = ref(false)
const inputText = ref()
const src = ref()
const player = ref()
const pause = ref(true)

watch(
  () => route.params,
  async newParams => {
    if (newParams.pathMatch==="") {
      href.value = "https://www.ewt360.com/"
    } else {
      href.value = "https://web.ewt360.com/" + newParams.pathMatch.join("/")
    }
  }
)

// noinspection JSUnresolvedFunction
$q.notify.setDefaults({
  color: 'primary',
  textColor: 'white'
})

//网易云相关
let netMusic = {
  currentNode: {
    src: 'default',
    lastNode: {},
    nextNode: {}
  },
  playList: {
    name: "",
    trackCount: 0,
    trackIds: []
  },
  playListId: "",
  defaultPlayListId: "2378294571",
  playListInput: false,
  playMode: "byOrder",
  songNum: 0,
  isPlaying: false,
  isPaused: true,
  volume: 50,
  orderListCache: [],
  getPlayList: function () {
    $q.localStorage.set("playMode", this.playMode)
    $q.localStorage.set("playListId", this.playListId)
    fetch(`https://ncm.jwyihao.top/playlist/detail?id=${this.playListId}`)
      .then(response => response.json())
      .then(data => {
        console.log(data)
        if (data.code === 200) {
          this.playList = data["playlist"]
          $q.notify(`已切换歌单“${this.playList.name}”共${this.playList.trackCount}首`)
          this.generatePlayOrder()
        } else {
          $q.notify("请求失败，请更换歌单或再试一次")
        }
      })
  },
  playLastSong: function () {
    if (this.currentNode.lastNode.src==="default") {
      if (!this.currentNode.nextNode.hasOwnProperty("src")) {
        this.generatePlayOrder(function () {
          netMusic.playNextSong()
        })
      }
      $q.notify("已经到第一首了")
    } else {
      this.currentNode=this.currentNode.lastNode
      src.value=this.currentNode.src
      this.isPlaying=true
      if (!this.isPaused) {
        setTimeout(function () {
          netMusic.play()
        }, 0)
      }
    }
  },
  playNextSong: function () {
    if (!this.currentNode.nextNode.hasOwnProperty("src")) {
      this.generatePlayOrder(function () {
        netMusic.playNextSong()
      })
    } else {
      this.currentNode=this.currentNode.nextNode
      src.value=this.currentNode.src
      this.isPlaying=true
      if (!this.isPaused) {
        setTimeout(function () {
          netMusic.play()
        }, 0)
      }
      if (!this.currentNode.nextNode.hasOwnProperty("src")) {
        this.generatePlayOrder()
      }
    }
  },
  play: function () {
    this.isPaused=false
    pause.value=false
    if (this.isPlaying) {
      player.value.play()
        .catch((reason) => {
          console.log(reason)
          this.isPaused=true
          pause.value=true
          if (reason==="DOMException: Failed to load because no supported source was found.") {
            console.log(player.value.error)
            if (player.value.error.code===4&&player.value.error.message==="") {
              netMusic.refreshSongUrl()
            }
          }
        })
    } else {
      this.playNextSong()
    }
  },
  pause: function () {
    this.isPaused=true
    pause.value=true
    player.value.pause()
  },
  calculateVolume: function () {
    let result = (2.48736840486504 - -1.63737224763414E-08) / [1 + (this.volume/110.421911643009)**-4.00458707238465] + -1.63737224763414E-08
    if (result > 1) return 1
    else if (result < 0) return 0
    else return result
  },
  changeVolume: function () {
    if (this.volume > 100) this.volume = 100
    else if (this.volume < 0) this.volume = 0
    player.value.volume = this.calculateVolume()
  },
  generatePlayOrder: function (callback = ()=>{}) {
    if (this.playList.trackIds===[]) {
      $q.notify("请先加载歌单")
      return false
    } else {
      $q.notify("正在生成播放列表")
    }
    let orderList
    if (this.orderListCache.length<=0) {
      orderList = this.playList.trackIds.slice()
      if (this.playMode==="random") {
        //打乱播放列表
        let i = orderList.length;
        while (--i) {
          let j = Math.floor(Math.random() * i);
          [orderList[j], orderList[i]] = [orderList[i], orderList[j]];
        }
      }
    } else {
      orderList = this.orderListCache.slice()
    }
    function addNode(lastNode, orderList, num) {
      let node = {
        trackId: orderList.shift(),
        src: "",
        lastNode: lastNode,
        nextNode: {}
      }
      //console.log(netMusic.getSongUrl(node.trackId.id))
      netMusic.getSongUrl(node.trackId.id)
        .then((data) => {
          if (data===undefined) {
            throw new Error(`ID${node.trackId.id}请求失败`)
          }
          lastNode.nextNode = node
          node.src=data
          if (num > 10) {
            netMusic.orderListCache = orderList
            callback()
            $q.notify("播放列表缓冲完毕")
            console.log(netMusic)
          } else if (orderList.length<=0) {
            //
          } else {
            addNode(node, orderList, num+1)
          }
        })
        .catch((reason) => {
          console.log(reason)
          addNode(lastNode, orderList, num+1)
        })
    }
    addNode(this.currentNode, orderList, 1)
  },
  getSongUrl: function (trackId, retry = 1) {
    return fetch(`https://ncm.jwyihao.top/song/url?id=${trackId}`, { credentials: 'include' })
      .then(response => response.json())
      .then(data => {
        if (data.code === 200 && data.data[0].code === 200) {
          $q.localStorage.set(`song${trackId}`, data.data[0].url.replace("http","https"))
          return data.data[0].url.replace("http", "https")
        } else {
          if (retry>=3) {
            $q.notify(`ID${trackId}请求失败`)
          } else {
            this.getSongUrl(trackId, retry+1)
          }
        }
      })
  },
  refreshSongUrl: function () {
    $q.notify("正在刷新播放列表")
    function singleRefresh(node) {
      return netMusic.getSongUrl((node.trackId.id))
        .then((data) => {
          if (data===undefined) {
            throw new Error(`ID${node.trackId.id}请求失败`)
          }
          node.src=data
          return true
        })
        .catch((e) => {
          node.lastNode.nextNode=node.nextNode
          node.nextNode.lastNode=node.lastNode
          $q.notify(e+"，已从播放列表中移除")
        })
    }
    function upRefresh(node) {
      if (node.lastNode.src==="default") return false
      singleRefresh(node.lastNode)
        .then(() => {
          upRefresh(node.lastNode)
        })
    }
    function downRefresh(node) {
      if (node.nextNode.hasOwnProperty("src")) return false
      singleRefresh(node.nextNode)
        .then(() => {
          downRefresh(node.nextNode)
        })
    }
    singleRefresh(netMusic.currentNode)
    upRefresh(netMusic.currentNode.lastNode)
    downRefresh(netMusic.currentNode.nextNode)
  }
}

if ($q.localStorage.has("playMode")) {
  $q.notify("正在加载配置")
  netMusic.playMode=$q.localStorage.getItem("playMode")
}

if ($q.localStorage.has("playListId")) {
  $q.notify("正在加载缓存的歌单")
  netMusic.playListId=$q.localStorage.getItem("playListId")
  netMusic.getPlayList()
}

//基本功能

function keyup (event) {
  console.log(event);
  const actions = {
    Enter: function () {
      //网页全屏
      if (document.fullscreenElement) {
        document.exitFullscreen()
        fullscreen.value=false
      } else {
        document.body.requestFullscreen()
        fullscreen.value=true
      }
    },
    KeyD: function () {
      //默认歌单
      $q.notify("正在加载默认歌单")
      netMusic.playListId = netMusic.defaultPlayListId
      netMusic.getPlayList()
    },
    KeyG: function () {
      //歌单输入
      if (netMusic.playListInput) {
        netMusic.playListInput = false
        netMusic.playListId = inputText.value.match(/\d+/gi)[0]
        netMusic.getPlayList()
      } else {
        netMusic.playListInput = true
        $q.notify({
          message: "请输入纯数字的歌单ID，按G完成输入",
        })
      }
    },
    KeyH: function () {
      router.push("/weekendElitePc/#/")
    },
    KeyO: function () {
      $q.notify("播放模式：顺序播放")
      netMusic.playMode = "byOrder"
    },
    KeyR: function () {
      $q.notify("播放模式：随机播放")
      netMusic.playMode = "random"
    },
    BracketRight: function () {
      netMusic.volume+=5
      netMusic.changeVolume()
    },
    BracketLeft: function () {
      netMusic.volume-=5
      netMusic.changeVolume()
    },
    Equal: function () {
      netMusic.volume++
      netMusic.changeVolume()
    },
    Minus: function () {
      netMusic.volume--
      netMusic.changeVolume()
    },
    Period: function (){
      netMusic.playNextSong()
    },
    Comma: function () {
      netMusic.playLastSong()
    }
  }
  if (actions.hasOwnProperty(event.code)) {
    actions[event.code]()
  } else {
    netMusic.pause()
  }
}

function mouseup (event) {
  if (event.button===1) {
    //鼠标中键被按下
    netMusic.play()
  } else {
    //左键或右键
    netMusic.pause()
  }
}

function wheel () {
  netMusic.pause()
}

window.onload=function () {
  if (route.params.pathMatch==="") {
    href.value = "https://www.ewt360.com/"
    $q.notify({
      message: '欢迎使用，请在登陆账号后点击这里进入周培优',
      timeout: 0,
      actions: [
        {
          label: 'OK',
          color: 'white',
          handler: () => {
            router.push("/weekendElitePc/#/")
            $q.notify({
              message: '要打开任何链接，请右键复制链接地址后按Ctrl + V',
              timeout: 0,
              actions: [{ icon: 'close', color: 'white' }]
            })
            setInterval(function() {
              document.querySelector("#focusHelp").focus()
            }, 100)
          }
        }
      ]
    })
  } else {
    href.value = "https://web.ewt360.com/" + route.params.pathMatch.join("/")
    setInterval(function() {
      document.querySelector("#focusHelp").focus()
    }, 100)
  }
  netMusic.changeVolume()
}


function input () {
  console.log(inputText.value)
  if (inputText.value.match(/^http(s)?:\/\//gi)) {
    //document.querySelector("iframe").src=document.querySelector("#focusHelp").value
    router.push("/" + inputText.value.replace(/^http(s)?:\/\/.+?\//gi, ""))
    if (inputText.value.match("course-play")) {
      $q.notify({
        message: "要全屏播放，请按Enter",
        timeout: 0,
        actions: [{ icon: 'close', color: 'white' }]
      })
    }
    inputText.value=""
  } else if (netMusic.playListInput && inputText.value.match(/\d+[Gg]?/gi)[0]===inputText.value.toString()) {
    //歌单输入模式
  }
  else {
    inputText.value=""
  }
}

function canplay(event) {
  console.log(event)
}

function ended () {
  netMusic.playNextSong()
}
</script>

<style lang="scss">
html, body, #q-app, .container {
  height: 100%;
  margin: 0;
  overflow: hidden;
}

.q-notification__message.col {
  line-height: 20px;
}

iframe {
  border: 0;
}

input {
  width: 0;
  height: 0;
  margin: 0;
  padding: 0;
  border: none;
}

#mouseHelp {
  width: 20vw;
  height: 20vh;
  position: fixed;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  background-color: transparent;
  border: 2px solid black;
  border-radius: 4vmin;
  opacity: 0;
  visibility: hidden;
  transition: {
    property: width, height, opacity, visibility;
    duration: .5s, .5s, .5s, 0s;
    delay: 0s, 0s, 0s, .5s;
  };
}

.fullscreen {
  iframe {
    position: absolute;
    width: calc(100% + 17px);
  }

  #mouseHelp {
    visibility: visible;
    transition-delay: 0s;

    &:hover {
      opacity: .05;
    }
  }
}

.pause {
  #mouseHelp {
    width: 8vmin;
    height: 8vmin;
  }
}
</style>
