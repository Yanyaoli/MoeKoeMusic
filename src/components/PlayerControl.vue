<template>
    <div class="player-container">
        <div class="progress-bar">
            <div class="progress" :style="{ width: progressWidth + '%' }"></div>
        </div>
        <div class="player-bar">
            <div class="album-art" @click="toggleLyrics">
                <img v-if="currentSong.img" :src="currentSong.img" alt="Album Art" />
                <i v-else class="fas fa-music"></i>
            </div>
            <div class="song-info" @click="toggleLyrics">
                <div class="song-title">{{ currentSong?.name || "MoeKoeMusic" }}</div>
                <div class="artist">{{ currentSong?.author || "MoeJue" }}</div>
            </div>
            <div class="controls">
                <button class="control-btn" @click="playSongFromQueue('previous')">
                    <i class="fas fa-step-backward"></i>
                </button>
                <button class="control-btn" @click="togglePlayPause">
                    <i :class="playing ? 'fas fa-pause' : 'fas fa-play'"></i>
                </button>
                <button class="control-btn" @click="playSongFromQueue('next')">
                    <i class="fas fa-step-forward"></i>
                </button>
            </div>
            <div class="extra-controls">
                <button class="extra-btn" @click="togglePlaybackMode">
                    <i v-if="currentPlaybackModeIndex != '2'" :class="currentPlaybackMode.icon" :title="currentPlaybackMode.title"></i>
                    <span v-else class="loop-icon" :title="currentPlaybackMode.title">
                        <i class="fas fa-repeat"></i>
                        <sup>1</sup>
                    </span>
                </button>
                <button class="extra-btn" @click="toggleQueue"><i class="fas fa-list"></i></button>
                <!-- 音量控制 -->
                <div class="volume-control" @wheel="handleVolumeScroll">
                    <i :class="isMuted ? 'fas fa-volume-mute' : 'fas fa-volume-up'" @click="toggleMute"></i>
                    <div class="volume-slider" @mousedown="onDragStart">
                        <div class="volume-progress" :style="{ width: volume + '%' }"></div>
                        <input type="range" min="0" max="100" v-model="volume" @input="changeVolume" />
                    </div>
                </div>
            </div>
        </div>


        <!-- 播放队列 -->
        <transition name="fade">
            <div v-if="showQueue" class="queue-popup">
                <div class="queue-header">
                    <h3>
                        <span>{{ $t('bo-fang-lie-biao') }}</span> ({{ musicQueueStore.queue.length }})
                        <i class="fas fa-trash-alt close-store" @click="musicQueueStore.clearQueue()" title="close"></i>
                    </h3>
                </div>

                <RecycleScroller :items="musicQueueStore.queue" :item-size="50" key-field="id" :buffer="500"
                    :items-limit="2000" :prerender="Math.min(10, musicQueueStore.queue.length)" ref="queueScroller"
                    class="queue-list">
                    <template #default="{ item, index }">
                        <li class="queue-item" :class="{ 'playing': currentSong.hash == item.hash }" :key="item.id">
                            <div class="queue-song-info">
                                <span class="queue-song-title">{{ item.name }}</span>
                                <span class="queue-artist">{{ $formatMilliseconds(item.timeLength) }}</span>
                            </div>
                            <div class="queue-actions">
                                <button v-if="currentSong.hash == item.hash"
                                    class="queue-play-btn fas fa-music"></button>
                                <template v-else>
                                    <button class="queue-play-btn" @click="addSongToQueue(
                                        item.hash,
                                        item.name,
                                        item.img,
                                        item.author
                                    )"><i class="fas fa-play"></i></button>
                                    <i class="fas fa-times close-store" @click="musicQueueStore.queue.splice(index, 1);$event.target.closest('li').classList.add('marked-as-deleted');"></i>
                                </template>
                            </div>
                        </li>
                    </template>
                </RecycleScroller>
            </div>
        </transition>
    </div>

    <!-- 全屏歌词界面 -->
    <transition name="slide-up">
        <div v-if="showLyrics" class="lyrics-bg"
            :style="(lyricsBackground == 'on' ? ({ backgroundImage: `url(${currentSong?.img || 'https://random.MoeJue.cn/randbg.php'})` }) : ({ background: 'var(--background-color)' }))">
            <div class="lyrics-screen">
                <div class="close-btn">
                    <i class="fas fa-chevron-down" @click="toggleLyrics"></i>
                </div>

                <div class="left-section">
                    <div class="album-art-large">
                        <img v-if="easterEggImage" :src="easterEggImage.src" :class="easterEggClass" alt="Easter Egg" />
                        <img :src="currentSong?.img || 'https://random.MoeJue.cn/randbg.php'" alt="Album Art" />
                    </div>
                    <div class="song-details">
                        <div class="song-title">{{ currentSong?.name }}</div>
                        <div class="artist">{{ currentSong?.author }}</div>
                    </div>

                    <!-- 播放进度条 -->
                    <div class="progress-bar-container">
                        <span class="current-time">{{ formattedCurrentTime }}</span>
                        <div class="progress-bar">
                            <div class="progress" :style="{ width: progressWidth + '%' }"></div>
                        </div>
                        <span class="duration">{{ formattedDuration }}</span>
                    </div>

                    <div class="player-controls">
                        <button class="control-btn" @click="playSongFromQueue('previous')">
                            <i class="fas fa-step-backward"></i>
                        </button>
                        <button class="control-btn" @click="togglePlayPause">
                            <i :class="playing ? 'fas fa-pause' : 'fas fa-play'"></i>
                        </button>
                        <button class="control-btn" @click="playSongFromQueue('next')">
                            <i class="fas fa-step-forward"></i>
                        </button>
                        <button class="control-btn" @click="togglePlaybackMode">
                            <i v-if="currentPlaybackMode.mode !== 'loop_one'" :class="currentPlaybackMode.icon" :title="currentPlaybackMode.title"></i>
                            <span v-else class="loop-icon" :title="currentPlaybackMode.title">
                                <i class="fas fa-repeat"></i>
                                <sup>1</sup>
                            </span>
                        </button>
                    </div>
                </div>
                <div id="lyrics-container">
                    <div v-if="lyricsData.length > 0" id="lyrics"
                        :style="{ transform: `translateY(${scrollAmount}px)` }">
                        <div v-for="(lineData, lineIndex) in lyricsData" :key="lineIndex" class="line">
                            <span v-for="(charData, charIndex) in lineData.characters" :key="charIndex" class="char"
                                :class="{ highlight: charData.highlighted }">
                                {{ charData.char }}
                            </span>
                        </div>
                    </div>
                    <div v-else class="no-lyrics">{{ SongTips }}</div>
                </div>
            </div>
        </div>
    </transition>
</template>

<script setup>
import { ref, onMounted, computed, onUnmounted, nextTick } from 'vue';
import { RecycleScroller } from 'vue3-virtual-scroller';
import 'vue3-virtual-scroller/dist/vue3-virtual-scroller.css'; 
import { get } from '../utils/request'; 
import { useMusicQueueStore } from '../stores/musicQueue'; 
import { useI18n } from 'vue-i18n';
const { t } = useI18n();
const showLyrics = ref(false); // 是否显示歌词
const isDragging = ref(false);
const showQueue = ref(false);
const currentSong = ref({ name: '', author: '', img: '', url: '', hash: '' }); // 当前播放的音乐信息
const playing = ref(false); // 是否正在播放
const isMuted = ref(false); // 是否静音
const volume = ref(66); // 音量初始值为 100
const audio = new Audio(); // 创建音频对象
const musicQueueStore = useMusicQueueStore(); // 使用 Pinia store
let sliderElement = null; // 用来保存音量滑块的 DOM 引用
const progressWidth = ref(0);
const lyricsData = ref([]);
const scrollAmount = ref(226);
const currentTime = ref(0);
const SongTips = ref(t('zan-wu-ge-ci'));
const lyricsBackground = ref('on');
let currentLineIndex = 0;
const queueScroller = ref(null);
const timeoutId = ref(null);
const playbackModes = ref([
    { icon: 'fas fa-random', title: t('sui-ji-bo-fang') },
    { icon: 'fas fa-refresh', title: t('lie-biao-xun-huan') },
    { icon: '', title: t('dan-qu-xun-huan') } 
]);
const currentPlaybackModeIndex = ref(0);
const currentPlaybackMode = computed(() => playbackModes.value[currentPlaybackModeIndex.value]);
// 切换随机/顺序/单曲播放
const togglePlaybackMode = () => {
    currentPlaybackModeIndex.value = (currentPlaybackModeIndex.value + 1) % playbackModes.value.length;
    audio.loop = currentPlaybackModeIndex.value == 2;
    localStorage.setItem('player_playback_mode', currentPlaybackModeIndex.value);
};
onMounted(() => {
    const savedVolume = localStorage.getItem('player_volume');
    if (savedVolume !== null) volume.value = parseFloat(savedVolume);
    isMuted.value = volume.value === 0;
    audio.volume = volume.value / 100;
    const current_song = localStorage.getItem('current_song');
    if (current_song) currentSong.value = JSON.parse(current_song);
    currentPlaybackModeIndex.value = localStorage.getItem('player_playback_mode') || 1;
    audio.loop = currentPlaybackModeIndex.value == 2;

    if (isElectron()) {
        window.electron.ipcRenderer.on('play-previous-track', playSongFromQueue('previous'));
        window.electron.ipcRenderer.on('play-next-track', playSongFromQueue('next'));
    }

    if (localStorage.getItem('settings')) {
        lyricsBackground.value = JSON.parse(localStorage.getItem('settings'))['lyricsBackground']
    }
});
const formattedCurrentTime = computed(() => formatTime(currentTime.value));
const formattedDuration = computed(() => formatTime(currentSong.value?.timeLength || 0));
const formatTime = (seconds) => {
    const minutes = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${minutes}:${secs.toString().padStart(2, '0')}`;
};
const easterEggImages = [
    { src: './assets/images/miku.png', class: 'miku' },
    { src: './assets/images/miku2.png', class: 'miku2' },
    { src: './assets/images/miku3.png', class: 'miku3' }
];
const easterEggImage = computed(() => {
    const author = currentSong.value?.author || '';
    if (author.includes('初音') || author.includes('Miku')) {
        const randomIndex = Math.floor(Math.random() * easterEggImages.length);
        return easterEggImages[randomIndex];
    }
    return null;
});
const easterEggClass = computed(() => easterEggImage.value?.class || '');
// 播放音乐
const playSong = (song) => {
    currentSong.value = structuredClone(song);
    lyricsData.value = [];
    audio.src = song.url;
    audio.play();
    playing.value = true;
    localStorage.setItem('current_song', JSON.stringify(currentSong.value));
    getLyrics(currentSong.value.hash)
    if (canRequestVip()) {
        // 自动领取VIP
        get('/youth/vip')
    }
};

// 从队列中播放歌曲
const playSongFromQueue = (direction) => {
    if (musicQueueStore.queue.length === 0) {
        window.$modal.alert(t('ni-huan-mei-you-tian-jia-ge-quo-kuai-qu-tian-jia-ba'));
        return;
    }
    const currentIndex = musicQueueStore.queue.findIndex(song => song.hash === currentSong.value.hash);
    let targetIndex;

    if (currentIndex == -1) {
        targetIndex = 0;
    } else if (currentPlaybackModeIndex.value == 0) {
        targetIndex = Math.floor(Math.random() * musicQueueStore.queue.length);
    } else {
        targetIndex = direction === 'previous' 
            ? (currentIndex === 0 ? musicQueueStore.queue.length - 1 : currentIndex - 1) 
            : (currentIndex + 1) % musicQueueStore.queue.length;
    }
    addSongToQueue(
        musicQueueStore.queue[targetIndex].hash,
        musicQueueStore.queue[targetIndex].name,
        musicQueueStore.queue[targetIndex].img,
        musicQueueStore.queue[targetIndex].author
    );
};

// 切换播放/暂停
const togglePlayPause = () => {
    if (!currentSong.value.hash) {
        playSongFromQueue('next');
        return;
    } else if (!audio.src) {
        addSongToQueue(
            currentSong.value.hash,
            currentSong.value.name,
            currentSong.value.img,
            currentSong.value.author
        );
        return;
    }

    if (playing.value) {
        audio.pause();
        playing.value = false;
    } else {
        audio.play();
        playing.value = true;
    }
};

// 切换静音
const toggleMute = () => {
    isMuted.value = !isMuted.value;
    audio.muted = isMuted.value;
    if (isMuted.value) {
        volume.value = 0;
    } else {
        volume.value = audio.volume * 100;
    }
    localStorage.setItem('player_volume', volume.value);
};
// 更新音量的点击处理函数
const setVolumeOnClick = (event) => {
    const slider = event.target.closest('.volume-slider');
    if (slider) {
        const sliderWidth = slider.offsetWidth;
        const offsetX = event.offsetX;
        volume.value = Math.round((offsetX / sliderWidth) * 100);
        changeVolume(); 
    }
};
// 开始拖拽时触发，初始化拖拽并实时更新音量
const onDragStart = (event) => {
    sliderElement = event.target.closest('.volume-slider');
    if (sliderElement) {
        isDragging.value = true;
        setVolumeOnClick(event); 
        document.addEventListener('mousemove', onDrag);
        document.addEventListener('mouseup', onDragEnd);
    }
};
// 拖拽过程中的音量更新
const onDrag = (event) => {
    if (isDragging.value && sliderElement) {
        const sliderWidth = sliderElement.offsetWidth;
        const rect = sliderElement.getBoundingClientRect();
        const offsetX = event.clientX - rect.left; 
        const newVolume = Math.max(0, Math.min(100, Math.round((offsetX / sliderWidth) * 100))); 
        volume.value = newVolume;
        changeVolume();
    }
};
const onDragEnd = () => {
    isDragging.value = false;
    sliderElement = null; 
    document.removeEventListener('mousemove', onDrag); 
    document.removeEventListener('mouseup', onDragEnd);
};
// 音量调节
const changeVolume = () => {
    audio.volume = volume.value / 100;
    localStorage.setItem('player_volume', volume.value);
    isMuted.value = volume.value === 0;
};

// 获取歌单全部歌曲
const getPlaylistAllSongs = async (id) => {
    try {
        let allSongs = [];
        for (let page = 1; page <= 2; page++) {
            const url = `/playlist/track/all?id=${id}&pagesize=250&page=${page}`;
            const response = await get(url);
            if (response.status !== 1) {
                window.$modal.alert(t('huo-qu-ge-dan-shi-bai'));
                return;
            }
            if (Object.keys(response.data.info).length === 0) {
                break;
            }
            allSongs = allSongs.concat(response.data.info);
            if(response.data.info.length < 250){
                break;
            }
        }
        addPlaylistToQueue(allSongs);
    } catch (error) {
        console.error(error);
        window.$modal.alert(t('huo-qu-ge-dan-shi-bai'));
    }
}
// 添加歌单到播放列表
const addPlaylistToQueue = async (info) => {
    musicQueueStore.clearQueue();
    const songs = info.map((song, index) => {
        return {
            id: index + 1,
            hash: song.hash,
            name: song.name,
            img: song.cover.replace("{size}", 480),
            author: song.name,
            timeLength: song.timelen
        };
    });
    musicQueueStore.queue = songs;
    addSongToQueue(songs[0].hash, songs[0].name, songs[0].img, songs[0].author);
};

// 添加歌曲到队列并播放的方法
const addSongToQueue = async (hash, name, img, author) => {
    try {
        clearTimeout(timeoutId.value);
        currentSong.value.author = author;
        currentSong.value.name = name;
        currentSong.value.img = img;
        currentSong.value.hash = hash;
        const url = `/song/url?hash=${hash}&free_part=1`;
        const response = await get(url);
        if (response.status !== 1) {
            currentSong.value.author = currentSong.value.name = t('huo-qu-yin-le-shi-bai');
            if(response.status == 3){
                currentSong.value.name = t('gai-ge-qu-zan-wu-ban-quan')
            }
            if (musicQueueStore.queue.length === 0) return;
            currentSong.value.author = t('3-miao-hou-zi-dong-qie-huan-xia-yi-shou');
            timeoutId.value = setTimeout(() => {
                playSongFromQueue('next');
            }, 3000);
            return;
        }
        const existingSongIndex = musicQueueStore.queue.findIndex(song => song.hash === hash);

        if (existingSongIndex === -1) {
            const song = {
                id: musicQueueStore.queue.length + 1,
                hash: hash,
                name: name,
                img: img,
                author: author,
                timeLength: response.timeLength,
                url: response.url[0]
            };
            musicQueueStore.addSong(song);
            playSong(song);
        } else {
            const updatedQueue = [...musicQueueStore.queue];
            const newSong = {
                id: musicQueueStore.queue[existingSongIndex].id,
                hash: hash,
                name: name,
                img: img,
                author: author,
                timeLength: response.timeLength,
                url: response.url[0]
            };
            updatedQueue[existingSongIndex] = newSong;
            musicQueueStore.setQueue(updatedQueue);
            playSong(newSong);
        }
    } catch (error) {
        currentSong.value.author = currentSong.value.name = t('huo-qu-yin-le-di-zhi-shi-bai');
        if (musicQueueStore.queue.length === 0) return;
        currentSong.value.author = t('3-miao-hou-zi-dong-qie-huan-xia-yi-shou');
        timeoutId.value = setTimeout(() => {
            playSongFromQueue('next');
        }, 3000);
    }
};

// 音乐播放结束后自动播放下一首
audio.addEventListener('ended', () => {
    if (currentPlaybackModeIndex.value == 2) return;
    playSongFromQueue('next');
});
audio.addEventListener('pause', () => {
    playing.value = false;
});
audio.addEventListener('play', () => {
    playing.value = true;
});
const toggleQueue = async () => {
    showQueue.value = !showQueue.value;
    if (showQueue.value) {
        await nextTick();
        setTimeout(() => {
            const currentIndex = musicQueueStore.queue.findIndex(song => song.hash === currentSong.value.hash);
            if (currentIndex !== -1 && queueScroller.value) {
                const middleOffset = Math.floor(queueScroller.value.$el.clientHeight / 100);
                const scrollToIndex = Math.max(0, currentIndex - middleOffset);
                queueScroller.value.scrollToItem(scrollToIndex);
            }
        }, 100);
    }
};

const toggleLyrics = async () => {
    showLyrics.value = !showLyrics.value;
    SongTips.value = t('huo-qu-ge-ci-zhong');
    if (!currentSong.value) return;
    try {
        if (lyricsData.value.length != 0) return;
        getLyrics(currentSong.value.hash)
    } catch (error) {
        SongTips.value = t('huo-qu-ge-ci-shi-bai');
    }
};

// 请求歌词
const getLyrics = async (hash) => {
    if (!showLyrics.value) return;
    const lyricSearchResponse = await get(`/search/lyric?hash=${hash}`);
    if (lyricSearchResponse.status !== 200 || lyricSearchResponse.candidates.length === 0) {
        SongTips.value = t('zan-wu-ge-ci');
        return;
    }
    const lyricResponse = await get(`/lyric?id=${lyricSearchResponse.candidates[0].id}&accesskey=${lyricSearchResponse.candidates[0].accesskey}&decode=true`);
    if (lyricResponse.status !== 200) {
        SongTips.value = t('huo-qu-ge-ci-shi-bai');
        return;
    }
    parseLyrics(lyricResponse.decodeContent);
    centerFirstLine();
}

const parseLyrics = (text) => {
    const lines = text.split('\n');
    const parsedLyrics = lines
        .map((line) => {
            const match = line.match(/^\[(\d+),(\d+)\](.*)/);
            if (match) {
                const time = parseInt(match[1]);
                const duration = parseInt(match[2]);
                const lyric = match[3].replace(/<.*?>/g, '');
                const characters = lyric.split('').map((char, index) => ({
                    char,
                    startTime: time + (index * duration) / lyric.length,
                    endTime: time + ((index + 1) * duration) / lyric.length,
                    highlighted: false,
                }));
                return { characters };
            }
            return null;
        })
        .filter((line) => line);
    lyricsData.value = parsedLyrics;
};
const centerFirstLine = () => {
    const lyricsContainer = document.getElementById('lyrics-container');
    if (!lyricsContainer) return;
    const containerHeight = lyricsContainer.offsetHeight;
    const lyricsElement = document.getElementById('lyrics');
    if (!lyricsElement) return;
    const lyricsHeight = lyricsElement.offsetHeight;
    scrollAmount.value = (containerHeight - lyricsHeight) / 2;
};
const throttle = (func, delay) => {
    let lastTime = 0;
    return function (...args) {
        const now = Date.now();
        if (now - lastTime >= delay) {
            lastTime = now;
            func.apply(this, args);
        }
    };
};
// 监听方法
const highlightCurrentChar = (currentTime) => {
    lyricsData.value.forEach((lineData, index) => {
        let isLineHighlighted = false;
        lineData.characters.forEach((charData) => {
            if (currentTime * 1000 >= charData.startTime && !charData.highlighted) {
                charData.highlighted = true;
                isLineHighlighted = true;
            }
        });

        if (isLineHighlighted && currentLineIndex !== index) {
            currentLineIndex = index;
            const lyricsContainer = document.getElementById('lyrics-container');
            if (!lyricsContainer) return;
            const containerHeight = lyricsContainer.offsetHeight;
            const lineElement = document.querySelectorAll('.line')[index];
            if (lineElement) {
                const lineHeight = lineElement.offsetHeight;
                scrollAmount.value = -lineElement.offsetTop + (containerHeight / 2) - (lineHeight / 2); // 计算中心位置
            }
        }
    });
};
// 节流处理
const throttledHighlight = throttle(() => {
    currentTime.value = audio.currentTime; // 实时更新当前时间
    progressWidth.value = (currentTime.value / audio.duration) * 100; // 更新进度条
    if (audio && showLyrics.value && lyricsData.value) {
        highlightCurrentChar(audio.currentTime);
    }
}, 200);
// 启动监听
audio.addEventListener('timeupdate', throttledHighlight);


defineExpose({
    addSongToQueue,
    getPlaylistAllSongs
});

onMounted(() => {
    document.addEventListener('click', handleClickOutside);
});

onUnmounted(() => {
    document.removeEventListener('click', handleClickOutside);
    if (isElectron()) {
        window.electron.ipcRenderer.removeAllListeners('play-previous-track');
        window.electron.ipcRenderer.removeAllListeners('play-next-track');
    }
});
const isElectron = () => {
    return typeof window !== 'undefined' && typeof window.electron !== 'undefined';
};
const handleClickOutside = (event) => {
    const queuePopup = document.querySelector('.queue-popup');
    if (queuePopup && !queuePopup.contains(event.target) && !event.target.closest('.extra-btn')) {
        showQueue.value = false;
    }
};
const handleVolumeScroll = (event) => {
    event.preventDefault();
    const delta = Math.sign(event.deltaY) * -1; 
    volume.value = Math.min(Math.max(volume.value + delta * 10, 0), 100);
    changeVolume();
};
const canRequestVip = () => {
    const lastRequestTime = localStorage.getItem('lastVipRequestTime');
    if (lastRequestTime) {
        const now = new Date().getTime();
        const elapsedTime = now - parseInt(lastRequestTime);
        const threeHours = 3 * 60 * 60 * 1000;
        if (elapsedTime < threeHours) {
            return false;
        }
    }
    localStorage.setItem('lastVipRequestTime', new Date().getTime().toString());
    return true;
}
</script>


<style scoped>
#lyrics-container {
    height: 75vh;
    overflow: hidden;
    display: flex;
    justify-content: center;
    padding: 20px;
    border-radius: 8px;
    margin-top: 80px;
    position: relative;
    width: 60%;
    text-align: center;
}

#lyrics {
    font-size: 24px;
    line-height: 1.8;
    white-space: pre-wrap;
    color: #666;
    transition: transform 0.6s ease;
}

.line {
    margin-bottom: 12px;
}

.char {
    display: inline-block;
    color: transparent;
    background: linear-gradient(to right, #FF69B4 50%, #666 50%);
    background-size: 200% 100%;
    background-position: 100%;
    -webkit-background-clip: text;
    background-clip: text;
    transition: background-position 0.6s ease;
}

.highlight {
    background-position: 0%;
}


.player-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100%;
    position: fixed;
    bottom: 0px;
    background: #FFF;
    box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
    border-radius: 10px;
    background: rgba(255, 255, 255, .7);
    -webkit-backdrop-filter: blur(10px);
    backdrop-filter: blur(10px);
}

.player-bar {
    display: flex;
    align-items: center;
    padding: 10px;
    width: 100%;
    max-width: 800px;
}

.album-art {
    width: 60px;
    height: 60px;
    border-radius: 5px;
    margin-right: 15px;
    background-color: #ddd;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
}

.album-art img {
    width: 64px;
    height: 64px;
    border-radius: 5px;
}

.album-art i {
    font-size: 24px;
    color: #666;
}

.song-info {
    flex-grow: 0;
    cursor: pointer;
    margin-right: auto;
}

.song-title {
    font-weight: bold;
    margin-bottom: 5px;
    width: 200px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.fa-volume-up {
    width: 10px;
}

.artist {
    font-size: 0.9em;
    color: #666;
    width: 200px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.controls {
    display: flex;
    align-items: center;
    justify-content: center;
    flex-grow: 1;
}

.control-btn {
    background: none;
    border: none;
    font-size: 24px;
    cursor: pointer;
    padding: 0 10px;
    color: #333;
}


.progress-bar {
    width: 100%;
    height: 6px;
    background-color: #ddd;
    position: relative;
    border-radius: 5px;
    overflow: hidden;
}

.progress {
    width: 30%;
    height: 100%;
    background-color: var(--primary-color);
    position: absolute;
    transition: width 0.2s ease;
}

.extra-controls {
    display: flex;
    align-items: center;
    margin-left: auto;
    gap: 10px;
}

.extra-btn {
    background: none;
    border: none;
    font-size: 16px;
    cursor: pointer;
    padding: 0 8px;
    color: #666;
}


.volume-control {
    display: flex;
    align-items: center;
    gap: 10px;
}

.volume-control i {
    font-size: 18px;
    color: #666;
}

.volume-slider {
    width: 100px;
    height: 6px;
    background-color: #ddd;
    border-radius: 3px;
    position: relative;
    overflow: hidden;
    cursor: pointer;
}

.volume-progress {
    height: 100%;
    background-color: var(--primary-color);
    position: absolute;
    left: 0;
    top: 0;
    border-radius: 3px;
}

/* 全屏歌词界面样式 */
.lyrics-bg {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    top: 0;
    z-index: 99;
    background-repeat: no-repeat;
    background-position: center;
    background-size: cover;
}

.lyrics-screen {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    top: 0;
    backdrop-filter: blur(18px);
    color: white;
    display: flex;
    justify-content: space-between;
    padding: 20px;
}

.close-btn {
    position: absolute;
    top: 30px;
    right: 100px;
    font-size: 24px;
    cursor: pointer;
    color: white;
    z-index: 99;
    text-align: right;
    width: 100%;
}

.left-section {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 40%;
    margin-top: 135px;
}

.album-art-large img {
    width: 400px;
    height: 400px;
    border-radius: 10px;
}

.song-details {
    text-align: center;
    margin-top: 20px;
}

.player-controls {
    display: flex;
    gap: 15px;
    margin-top: 20px;
}



@keyframes scroll-lyrics {
    0% {
        transform: translateY(100%);
    }

    100% {
        transform: translateY(-100%);
    }
}

.slide-up-enter-active,
.slide-up-leave-active {
    transition: transform 0.3s ease;
}

.slide-up-enter-from,
.slide-up-leave-to {
    transform: translateY(100%);
}



.progress-bar-container {
    display: flex;
    align-items: center;
    width: 100%;
    margin: 10px 0;
}

.current-time,
.duration {
    color: white;
    font-size: 12px;
}

.progress-bar {
    flex-grow: 1;
    height: 2px;
    background-color: #555;
    border-radius: 5px;
    margin: 0 10px;
    position: relative;
}

.progress {
    width: 50%;
    height: 100%;
    background-color: #ff69b4;
    border-radius: 5px;
}

.player-controls {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 15px;
    margin-top: 10px;
}


/* 播放队列样式 */
.queue-popup {
    position: fixed;
    right: 20px;
    bottom: 100px;
    width: 300px;
    max-height: 400px;
    background: #fff;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
    border-radius: 10px;
    padding: 10px;
    z-index: 100;
    overflow-y: auto;
}

.queue-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
    position: sticky;
    top: 0px;
    z-index: 1;
    border-radius: 5px;
    padding: 5px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);

}

.queue-header h3 {
    margin: 0;
    font-size: 1.2em;
    color: var(--text-color);
}


.queue-list {
    list-style: none;
    padding: 0;
    margin: 0;
    flex: 1;
    overflow-y: auto;
    height: 350px;
    scroll-behavior: smooth;
}

.queue-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid #ddd;
}

.queue-song-info {
    display: flex;
    flex-direction: column;
}

.queue-song-title {
    font-weight: bold;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    width: 250px;
}

.queue-artist {
    font-size: 0.9em;
    color: #666;
}

.queue-actions {
    display: flex;
    align-items: center;
}

.queue-play-btn {
    background: none;
    border: none;
    font-size: 16px;
    color: var(--primary-color);
    cursor: pointer;
}

.fade-enter-active,
.fade-leave-active {
    transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
    opacity: 0;
}

.volume-control {
    display: flex;
    align-items: center;
    position: relative;
}

.volume-slider {
    width: 100px;
    margin-left: 10px;
    position: relative;
}

.volume-slider input[type="range"] {
    width: 100%;
    -webkit-appearance: none;
    appearance: none;
    height: 5px;
    background: transparent;
    position: relative;
    z-index: 1;
    pointer-events: none;
    /* 禁止对 input 的点击事件 */
}

.volume-slider input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 12px;
    height: 12px;
    border-radius: 50%;
    background: #4899f8;
    cursor: pointer;
    pointer-events: auto;
    /* 允许 thumb 的拖动事件 */
}

.volume-slider input[type="range"]::-moz-range-thumb {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    background: #4899f8;
    cursor: pointer;
}

.album-art-large .miku {
    position: absolute;
    height: 419px;
    width: 401px;
}

.album-art-large .miku2 {
    position: absolute;
    height: 452px;
    width: 404px;
}

.album-art-large .miku3 {
    position: absolute;
    height: 563px;
}

.no-lyrics {
    color: var(--primary-color);
    margin: auto;
    font-size: 2em;
}

.close-store {
    margin-left: 8px;
    cursor: pointer;
    font-size: 14px;
}

.queue-item.playing .queue-song-title {
    color: var(--primary-color);
    font-weight: bold;
}

.queue-item.playing .queue-artist {
    color: var(--primary-color);
}

.queue-item.marked-as-deleted {
    text-decoration: line-through;
    color: #888;
    transition: all 0.5s ease;
    opacity: 0.5;
    pointer-events: none;
}
.loop-icon {
    position: relative;
    display: inline-block;
}

.loop-icon sup {
    position: absolute;
    font-size: 0.6em;
}
</style>