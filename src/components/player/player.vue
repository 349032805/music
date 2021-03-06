<template>
    <div class="player" v-show="playlist.length>0">
        <transition name="normal"
                    @enter="enter"
                    @after-enter="afterEnter"
                    @leave="leave"
                    @after-leave="afterLeave"
        >
            <div class="normal-player" v-show="fullScreen">
                <div class="background">
                    <img width="100%" height="100%" :src="currentSong.image">
                </div>
                <div class="top">
                    <div class="back" @click="back">
                        <i class="icon-back"></i>
                    </div>
                    <h1 class="title" v-html="currentSong.name"></h1>
                    <h2 class="subtitle" v-html="currentSong.singer"></h2>
                </div>
                <div class="middle"
                     @touchstart="touchStartMiddle"
                     @touchmove="touchMoveMiddle"
                     @touchend="touchEndMiddle"
                >
                    <div class="middle-l" ref="middleL">
                        <div class="cd-wrapper" ref="cdWrapper">
                            <div class="cd" :class="imgRotate">
                                <img class="image" :src="currentSong.image">
                            </div>
                        </div>
                        <div class="playing-lyric-wrapper">
                            <div class="playing-lyric">{{playingLyric}}</div>
                        </div>
                    </div>
                    <scroll class="middle-r" ref="lyricList" :data="currentLyric && currentLyric.lines">
                        <div class="lyric-wrapper">
                            <div v-if="currentLyric">
                                <p ref="lyricLine"
                                   class="text"
                                   v-for="(item,index) in currentLyric.lines"
                                   :class="{'current':currentLineNum===index}"

                                >{{item.txt}}</p>
                            </div>
                        </div>
                    </scroll>
                </div>
                <div class="bottom">
                    <div class="dot-wrapper">
                        <span class="dot" :class="{active : currentMiddle ==='cd'}"></span>
                        <span class="dot" :class="{active : currentMiddle ==='lyric'}"></span>
                    </div>
                    <div class="progress-wrapper">
                        <span class="time time-l">{{timeFormat(currentTime)}}</span>
                        <div class="progress-bar-wrapper">
                            <progressBar @setPercent="setPercent" :percent="percent"></progressBar>
                        </div>
                        <span class="time time-r">{{timeFormat(currentSong.duration)}}</span>
                    </div>
                    <div class="operators">
                        <div class="icon i-left" @click="changeMode">
                            <i :class="iconMode"></i>
                        </div>
                        <div class="icon i-left" @click="prevPlay" :class="disableCla">
                            <i class="icon-prev"></i>
                        </div>
                        <div class="icon i-center" :class="disableCla">
                            <i :class="toggleIcon" @click="togglePlay"></i>
                        </div>
                        <div class="icon i-right" @click="nextPlay" :class="disableCla">
                            <i class="icon-next"></i>
                        </div>
                        <div class="icon i-right">
                            <i class="icon"></i>
                        </div>
                    </div>
                </div>
            </div>
        </transition>
        <transition name="mini">
            <div class="mini-player" v-show="!fullScreen" @click="open">
                <div class="icon">
                    <img :class="imgRotate" width="40" height="40" :src="currentSong.image">
                </div>
                <div class="text">
                    <h2 class="name" v-html="currentSong.name"></h2>
                    <p class="desc" v-html="currentSong.singer"></p>
                </div>
                <div class="control">
                    <progressCircle :radius="32" :percent="percent">
                        <i class="icon-mini" :class="toggleIconMini" @click.stop="togglePlay"></i>
                    </progressCircle>
                </div>
                <div class="control">
                    <i class="icon-playlist"></i>
                </div>
            </div>
        </transition>
        <audio @ended="end" @timeupdate="updateTime" :src="currentSong.url" ref="audio" @canplay="ready"
               @error="error"></audio>
    </div>
</template>
<script>
    import {mapGetters, mapMutations} from 'vuex'
    import animations from 'create-keyframe-animation'
    import {prefixStyle} from 'common/js/dom'
    import progressBar from 'base/progress-bar/progress-bar'
    import progressCircle from 'base/progress-circle/progress-circle'
    import {playMode} from 'common/js/config'
    import {shuffle} from 'common/js/util'
    import {getLyric} from 'api/song'
    import Lyric from 'lyric-parser'
    import scroll from 'base/scroll/scroll'

    const transform = prefixStyle('transform')
    const transitionDuration = prefixStyle('transitionDuration')
    const TIME = 300

    export default{
        data(){
            return {
                songReady: false,
                currentTime: 0,
                currentLyric: null,
                currentLineNum: 0,
                currentMiddle: 'cd',
                playingLyric: ''
            }
        },
        mounted(){
            this.touch = {}
        },
        computed: {
            toggleIcon(){
                return this.playing ? 'icon-pause' : 'icon-play'
            },
            toggleIconMini(){
                return this.playing ? 'icon-pause-mini' : 'icon-play-mini'
            },
            imgRotate(){
                return this.playing ? 'play' : 'play pause'
            },
            disableCla(){
                return this.songReady ? '' : 'disable'
            },
            percent(){
                return this.currentTime / this.currentSong.duration
            },
            iconMode(){
                return this.mode === playMode.sequence ? 'icon-sequence' : this.mode === playMode.loop ? 'icon-loop' : 'icon-random'
            },
            ...mapGetters([
                'fullScreen',
                'playlist',
                'currentSong',
                'playing',
                'currentIndex',
                'mode',
                'sequenceList'
            ])
        },
        methods: {
            back(){
                this.setFullScreen(false)
            },
            open(){
                this.setFullScreen(true)
            },
            enter(el, done){
                let {x, y, scale} = this._getPosAndScale()

                let animation = {
                    0: {
                        transform: `translate3d(${x}px,${y}px,0) scale(${scale})`
                    },
                    60: {
                        transform: `translate3d(0,0,0) scale(1.1)`
                    },
                    100: {
                        transform: `translate3d(0,0,0) scale(1)`
                    }
                }
                animations.registerAnimation({
                    name: 'move',
                    animation,
                    presets: {
                        duration: 400,
                        easing: 'linear'
                    }
                })
                animations.runAnimation(this.$refs.cdWrapper, 'move', done)
            },
            afterEnter(){
                animations.unregisterAnimation('move')
                this.$refs.cdWrapper.style['animation'] = ''
            },
            leave(el, done){
                this.$refs.cdWrapper.style['transition'] = 'all 0.4s'
                const {x, y, scale} = this._getPosAndScale()
                this.$refs.cdWrapper.style[transform] = `translate3d(${x}px,${y}px,0) scale(${scale})`
                this.$refs.cdWrapper.addEventListener('transitionend', done)
            },
            afterLeave(){
                this.$refs.cdWrapper.style['transition'] = ''
                this.$refs.cdWrapper.style[transform] = ''
            },
            _getPosAndScale(){
                const targetWidth = 40
                const paddingLeft = 40
                const paddingBottom = 30
                const paddingTop = 80
                const width = window.innerWidth * 0.8
                const scale = targetWidth / width
                const x = -(window.innerWidth / 2 - paddingLeft)
                const y = window.innerHeight - paddingTop - width / 2 - paddingBottom
                return {x, y, scale}
            },
            togglePlay(){
                if (!this.songReady) {
                    return
                }
                this.setPlayState(!this.playing)
                if (this.currentLyric) {
                    this.currentLyric.togglePlay()
                }
            },
            prevPlay(){
                this.play(false)
            },
            nextPlay(){
                this.play(true)
            },
            play(flag){
                let i
                if (this.songReady) {
                    if (this.playlist.length===1){
                        this.loop()
                    }else{
                        if (flag) {
                            i = this.currentIndex + 1
                            if (i > this.playlist.length - 1) {
                                i = 0
                            }
                        } else {
                            i = this.currentIndex - 1
                            if (i < 0) {
                                i = this.playlist.length - 1
                            }
                        }
                        this.setCurrentIndex(i)
                        this.songReady = false
                        this.setPlayState(true)
                    }

                }
            },
            end(){
                if (this.mode === playMode.loop) {
                    this.loop()
                } else {
                    this.nextPlay()
                }
            },
            loop(){
                this.$refs.audio.currentTime = 0
                this.$refs.audio.play()
                if (this.currentLyric) {
                    this.currentLyric.seek(0)
                }
            },
            ready(){
                this.songReady = true
            },
            error(){
                this.songReady = true
                alert('歌曲获取出错请换一首歌曲播放')
            },
            updateTime(e){
                let time = e.target.currentTime | 0
                this.currentTime = time
            },
            timeFormat(num){
                let s = num % 60 | 0
                if (s < 10) {
                    s = "0" + s
                }
                let m = num / 60 | 0
                return `${m}:${s}`
            },
            setPercent(percent){
                const time = percent * this.currentSong.duration
                this.$refs.audio.currentTime = time
                if (!this.playing) {
                    this.togglePlay()
                }
                if (this.currentLyric) {
                    this.currentLyric.seek(time * 1000)
                }
            },
            changeMode(){
                const mode = (this.mode + 1) % 3
                this.setPlayMode(mode)
                let list = null
                if (mode === playMode.random) {
                    list = shuffle(this.sequenceList)
                } else {
                    list = this.sequenceList
                }
                this.resetCurrentIndex(list)
                this.setPlayList(list)
            },
            resetCurrentIndex(list){
                let index = list.findIndex((item) => {
                    return item.id === this.currentSong.id
                })
                this.setCurrentIndex(index)
            },
            getLyric(){
                this.currentSong.getLyric().then((lyric) => {
                    this.currentLyric = new Lyric(lyric, this.handleLyric)
                    if (this.playing) {
                        this.currentLyric.play()
                    }
                }).catch(() => {
                    this.currentLyric = null
                    this.playingLyric = ''
                    this.currentLineNum = 0
                })
            },
            handleLyric({lineNum, txt}){
                this.currentLineNum = lineNum
                if (lineNum > 5) {
                    let lyricEl = this.$refs.lyricLine[lineNum - 5]
                    this.$refs.lyricList.scrollToElement(lyricEl, 1000)
                } else {
                    this.$refs.lyricList.scrollTo(0, 0, 1000)
                }
                this.playingLyric = txt
            },
            touchStartMiddle(e){
                const touch = e.touches[0]
                this.touch.initSwitch = true
                this.touch.initMove = false
                //按下的位置
                this.touch.pressX = touch.pageX
                this.touch.pressY = touch.pageY
            },
            touchMoveMiddle(e){
                if (!this.touch.initSwitch) {
                    return
                }
                this.touch.initMove = true
                const touch = e.touches[0]
                //偏移量
                const deltaX = touch.pageX - this.touch.pressX
                const deltaY = touch.pageY - this.touch.pressY

                //y>x return
                if (Math.abs(deltaY) > Math.abs(deltaX)) {
                    return
                }
                const left = this.currentMiddle === 'cd' ? 0 : -window.innerWidth
                const offsetWidth = Math.min(0, Math.max(-window.innerWidth, left + deltaX))
                //比例
                this.touch.absWidth = Math.abs(offsetWidth / window.innerWidth)
                //透明度
                const opacity = 1 - this.touch.absWidth

                this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px,0,0)`
                this.$refs.middleL.style['opacity'] = opacity
                this.$refs.lyricList.$el.style[transitionDuration] = `${TIME}ms`

            },
            touchEndMiddle(e){
                if (!this.touch.initMove){
                    return
                }
                let offsetWidth
                let opacity
                //当前为CD页
                if (this.currentMiddle === 'cd') {
                    //滑动超过百分之10变化
                    if (this.touch.absWidth > 0.1) {
                        offsetWidth = -window.innerWidth
                        this.currentMiddle = 'lyric'
                        opacity = 0
                    } else {
                        offsetWidth = 0
                        opacity = 1
                    }
                } else {
                    if (this.touch.absWidth < 0.9) {
                        offsetWidth = 0
                        this.currentMiddle = 'cd'
                        opacity = 1
                    } else {
                        offsetWidth = -window.innerWidth
                        opacity = 0
                    }
                }
                this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px,0,0)`
                this.$refs.middleL.style['opacity'] = opacity
                this.$refs.lyricList.$el.style[transitionDuration] = `${TIME}ms`
                this.touch.initiated = false
            },
            ...mapMutations({
                setFullScreen: 'SET_FULL_SCREEN',
                setPlayState: 'SET_PLAYING_STATE',
                setCurrentIndex: 'SET_CURRENT_INDEX',
                setPlayMode: 'SET_PLAY_MODE',
                setPlayList: 'SET_PLAYLIST'
            })
        },
        watch: {
            playing(newPlaying){
                let play = this.$refs.audio
                this.$nextTick(() => {
                    newPlaying ? play.play() : play.pause()
                })

            },
            currentSong(newSong, oldSong){
                if (newSong.id === oldSong.id) {
                    return
                }
                if (this.currentLyric) {
                    this.currentLyric.stop()
                }
                setTimeout(() => {
                    this.$refs.audio.play()
                    this.getLyric()
                },1000)
            }
        },
        components: {
            progressBar,
            progressCircle,
            scroll
        }
    }


</script>
<style scoped lang="stylus" rel="stylesheet/stylus">
    @import "~common/stylus/variable"
    @import "~common/stylus/mixin"

    .player
        .normal-player
            position: fixed
            left: 0
            right: 0
            top: 0
            bottom: 0
            z-index: 150
            background: $color-background
            .background
                position: absolute
                left: 0
                top: 0
                width: 100%
                height: 100%
                z-index: -1
                opacity: 0.6
                filter: blur(20px)
            .top
                position: relative
                margin-bottom: 25px
                .back
                    position absolute
                    top: 0
                    left: 6px
                    z-index: 50
                    .icon-back
                        display: block
                        padding: 9px
                        font-size: $font-size-large-x
                        color: $color-theme
                        transform: rotate(-90deg)
                .title
                    width: 70%
                    margin: 0 auto
                    line-height: 40px
                    text-align: center
                    no-wrap()
                    font-size: $font-size-large
                    color: $color-text
                .subtitle
                    line-height: 20px
                    text-align: center
                    font-size: $font-size-medium
                    color: $color-text
            .middle
                position: fixed
                width: 100%
                top: 80px
                bottom: 170px
                white-space: nowrap
                font-size: 0
                .middle-l
                    display: inline-block
                    vertical-align: top
                    position: relative
                    width: 100%
                    height: 0
                    padding-top: 80%
                    .cd-wrapper
                        position: absolute
                        left: 10%
                        top: 0
                        width: 80%
                        height: 100%
                        .cd
                            width: 100%
                            height: 100%
                            box-sizing: border-box
                            border: 10px solid rgba(255, 255, 255, 0.1)
                            border-radius: 50%
                            &.play
                                animation: rotate 20s linear infinite
                            &.pause
                                animation-play-state: paused
                            .image
                                position: absolute
                                left: 0
                                top: 0
                                width: 100%
                                height: 100%
                                border-radius: 50%

                    .playing-lyric-wrapper
                        width: 80%
                        margin: 30px auto 0 auto
                        overflow: hidden
                        text-align: center
                        .playing-lyric
                            height: 20px
                            line-height: 20px
                            font-size: $font-size-medium
                            color: $color-text-l
                .middle-r
                    display: inline-block
                    vertical-align: top
                    width: 100%
                    height: 100%
                    overflow: hidden
                    .lyric-wrapper
                        width: 80%
                        margin: 0 auto
                        overflow: hidden
                        text-align: center
                        .text
                            line-height: 32px
                            color: $color-text-l
                            font-size: $font-size-medium
                            &.current
                                color: $color-text
            .bottom
                position: absolute
                bottom: 50px
                width: 100%
                .dot-wrapper
                    text-align: center
                    font-size: 0
                    .dot
                        display: inline-block
                        vertical-align: middle
                        margin: 0 4px
                        width: 8px
                        height: 8px
                        border-radius: 50%
                        background: $color-text-l
                        &.active
                            width: 20px
                            border-radius: 5px
                            background: $color-text-ll
                .progress-wrapper
                    display: flex
                    align-items: center
                    width: 80%
                    margin: 0px auto
                    padding: 10px 0
                    .time
                        color: $color-text
                        font-size: $font-size-small
                        flex: 0 0 30px
                        line-height: 30px
                        width: 30px
                        &.time-l
                            text-align: left
                        &.time-r
                            text-align: right
                    .progress-bar-wrapper
                        flex: 1
                .operators
                    display: flex
                    align-items: center
                    .icon
                        flex: 1
                        color: $color-theme
                        &.disable
                            color: $color-theme-d
                        i
                            font-size: 30px
                    .i-left
                        text-align: right
                    .i-center
                        padding: 0 20px
                        text-align: center
                        i
                            font-size: 40px
                    .i-right
                        text-align: left
                    .icon-favorite
                        color: $color-sub-theme
            &.normal-enter-active, &.normal-leave-active
                transition: all 0.4s
                .top, .bottom
                    transition: all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32)
            &.normal-enter, &.normal-leave-to
                opacity: 0
                .top
                    transform: translate3d(0, -100px, 0)
                .bottom
                    transform: translate3d(0, 100px, 0)
        .mini-player
            display: flex
            align-items: center
            position: fixed
            left: 0
            bottom: 0
            z-index: 180
            width: 100%
            height: 60px
            background: $color-highlight-background
            &.mini-enter-active, &.mini-leave-active
                transition: all 0.4s
            &.mini-enter, &.mini-leave-to
                opacity: 0
            .icon
                flex: 0 0 40px
                width: 40px
                padding: 0 10px 0 20px
                img
                    border-radius: 50%
                    &.play
                        animation: rotate 10s linear infinite
                    &.pause
                        animation-play-state: paused
            .text
                display: flex
                flex-direction: column
                justify-content: center
                flex: 1
                line-height: 20px
                overflow: hidden
                .name
                    margin-bottom: 2px
                    no-wrap()
                    font-size: $font-size-medium
                    color: $color-text
                .desc
                    no-wrap()
                    font-size: $font-size-small
                    color: $color-text-d
            .control
                flex: 0 0 30px
                width: 30px
                padding: 0 10px
                .icon-play-mini, .icon-pause-mini, .icon-playlist
                    font-size: 30px
                    color: $color-theme-d
                .icon-mini
                    font-size: 32px
                    position: absolute
                    left: 0
                    top: 0

    @keyframes rotate
        0%
            transform: rotate(0)
        100%
            transform: rotate(360deg)
</style>