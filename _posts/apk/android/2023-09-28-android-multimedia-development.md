---
layout: post
title: Android 多媒体开发深入指南
categories: android
tags: [多媒体, Android, 音频, 视频, 相机]
date: 2023/9/28 11:15:00
---

![title](https://image.sideproject.cn/titlex/titlex_multimedia.jpg)

## 引言

多媒体功能是现代 Android 应用的重要组成部分，涵盖了音频播放、视频处理、相机拍摄、图像编辑等多个方面。随着移动设备硬件性能的不断提升和用户对多媒体体验要求的日益增长，掌握 Android 多媒体开发技术变得越来越重要。Android 平台提供了丰富的多媒体 API，从基础的 MediaPlayer 到高级的 Camera2 API，从简单的音频录制到复杂的视频编辑，开发者可以利用这些工具创建出功能强大、体验优秀的多媒体应用。本文将深入探讨 Android 多媒体开发的各个方面，帮助开发者掌握核心技术和最佳实践。

## 音频开发

### MediaPlayer 音频播放

```kotlin
class AudioPlayerManager {
    
    private var mediaPlayer: MediaPlayer? = null
    private var isPrepared = false
    private var currentPosition = 0
    
    // 播放状态监听器
    interface PlaybackListener {
        fun onPrepared(duration: Int)
        fun onStarted()
        fun onPaused()
        fun onStopped()
        fun onCompleted()
        fun onError(error: String)
        fun onProgressUpdate(currentPosition: Int, duration: Int)
    }
    
    private var playbackListener: PlaybackListener? = null
    private var progressUpdateHandler: Handler? = null
    private var progressUpdateRunnable: Runnable? = null
    
    fun setPlaybackListener(listener: PlaybackListener) {
        this.playbackListener = listener
    }
    
    // 准备播放
    fun prepareAudio(audioUrl: String) {
        try {
            release()
            
            mediaPlayer = MediaPlayer().apply {
                setDataSource(audioUrl)
                setOnPreparedListener { player ->
                    isPrepared = true
                    val duration = player.duration
                    playbackListener?.onPrepared(duration)
                }
                
                setOnCompletionListener {
                    stopProgressUpdates()
                    playbackListener?.onCompleted()
                }
                
                setOnErrorListener { _, what, extra ->
                    playbackListener?.onError("MediaPlayer error: what=$what, extra=$extra")
                    true
                }
                
                prepareAsync()
            }
            
        } catch (e: Exception) {
            playbackListener?.onError("Failed to prepare audio: ${e.message}")
        }
    }
    
    // 开始播放
    fun play() {
        if (isPrepared && mediaPlayer?.isPlaying == false) {
            mediaPlayer?.start()
            startProgressUpdates()
            playbackListener?.onStarted()
        }
    }
    
    // 暂停播放
    fun pause() {
        if (mediaPlayer?.isPlaying == true) {
            mediaPlayer?.pause()
            currentPosition = mediaPlayer?.currentPosition ?: 0
            stopProgressUpdates()
            playbackListener?.onPaused()
        }
    }
    
    // 停止播放
    fun stop() {
        mediaPlayer?.let { player ->
            if (player.isPlaying) {
                player.stop()
            }
            player.reset()
            isPrepared = false
            currentPosition = 0
            stopProgressUpdates()
            playbackListener?.onStopped()
        }
    }
    
    // 跳转到指定位置
    fun seekTo(position: Int) {
        if (isPrepared) {
            mediaPlayer?.seekTo(position)
            currentPosition = position
        }
    }
    
    // 设置音量
    fun setVolume(leftVolume: Float, rightVolume: Float) {
        mediaPlayer?.setVolume(leftVolume, rightVolume)
    }
    
    // 获取当前播放位置
    fun getCurrentPosition(): Int {
        return mediaPlayer?.currentPosition ?: currentPosition
    }
    
    // 获取总时长
    fun getDuration(): Int {
        return if (isPrepared) mediaPlayer?.duration ?: 0 else 0
    }
    
    // 是否正在播放
    fun isPlaying(): Boolean {
        return mediaPlayer?.isPlaying ?: false
    }
    
    // 开始进度更新
    private fun startProgressUpdates() {
        progressUpdateHandler = Handler(Looper.getMainLooper())
        progressUpdateRunnable = object : Runnable {
            override fun run() {
                if (mediaPlayer?.isPlaying == true) {
                    val currentPos = mediaPlayer?.currentPosition ?: 0
                    val duration = mediaPlayer?.duration ?: 0
                    playbackListener?.onProgressUpdate(currentPos, duration)
                    progressUpdateHandler?.postDelayed(this, 1000)
                }
            }
        }
        progressUpdateRunnable?.let {
            progressUpdateHandler?.post(it)
        }
    }
    
    // 停止进度更新
    private fun stopProgressUpdates() {
        progressUpdateRunnable?.let {
            progressUpdateHandler?.removeCallbacks(it)
        }
        progressUpdateHandler = null
        progressUpdateRunnable = null
    }
    
    // 释放资源
    fun release() {
        stopProgressUpdates()
        mediaPlayer?.let { player ->
            if (player.isPlaying) {
                player.stop()
            }
            player.release()
        }
        mediaPlayer = null
        isPrepared = false
        currentPosition = 0
    }
}

// 音频播放列表管理
class PlaylistManager {
    
    data class AudioTrack(
        val id: String,
        val title: String,
        val artist: String,
        val url: String,
        val duration: Int,
        val albumArt: String? = null
    )
    
    private val playlist = mutableListOf<AudioTrack>()
    private var currentIndex = 0
    private val audioPlayer = AudioPlayerManager()
    
    enum class PlayMode {
        SEQUENTIAL, REPEAT_ALL, REPEAT_ONE, SHUFFLE
    }
    
    private var playMode = PlayMode.SEQUENTIAL
    private var shuffleIndices = mutableListOf<Int>()
    private var shuffleIndex = 0
    
    interface PlaylistListener {
        fun onTrackChanged(track: AudioTrack, index: Int)
        fun onPlaylistUpdated(playlist: List<AudioTrack>)
        fun onPlayModeChanged(mode: PlayMode)
    }
    
    private var playlistListener: PlaylistListener? = null
    
    init {
        audioPlayer.setPlaybackListener(object : AudioPlayerManager.PlaybackListener {
            override fun onPrepared(duration: Int) {}
            override fun onStarted() {}
            override fun onPaused() {}
            override fun onStopped() {}
            
            override fun onCompleted() {
                when (playMode) {
                    PlayMode.REPEAT_ONE -> {
                        audioPlayer.seekTo(0)
                        audioPlayer.play()
                    }
                    PlayMode.REPEAT_ALL -> {
                        if (hasNext()) {
                            next()
                        } else {
                            first()
                        }
                    }
                    PlayMode.SHUFFLE -> {
                        nextShuffle()
                    }
                    PlayMode.SEQUENTIAL -> {
                        if (hasNext()) {
                            next()
                        }
                    }
                }
            }
            
            override fun onError(error: String) {}
            override fun onProgressUpdate(currentPosition: Int, duration: Int) {}
        })
    }
    
    // 添加音频到播放列表
    fun addTrack(track: AudioTrack) {
        playlist.add(track)
        updateShuffleIndices()
        playlistListener?.onPlaylistUpdated(playlist)
    }
    
    fun addTracks(tracks: List<AudioTrack>) {
        playlist.addAll(tracks)
        updateShuffleIndices()
        playlistListener?.onPlaylistUpdated(playlist)
    }
    
    // 移除音频
    fun removeTrack(index: Int) {
        if (index in 0 until playlist.size) {
            playlist.removeAt(index)
            if (currentIndex >= index && currentIndex > 0) {
                currentIndex--
            }
            updateShuffleIndices()
            playlistListener?.onPlaylistUpdated(playlist)
        }
    }
    
    // 清空播放列表
    fun clearPlaylist() {
        playlist.clear()
        currentIndex = 0
        shuffleIndices.clear()
        shuffleIndex = 0
        audioPlayer.stop()
        playlistListener?.onPlaylistUpdated(playlist)
    }
    
    // 播放指定音频
    fun playTrack(index: Int) {
        if (index in 0 until playlist.size) {
            currentIndex = index
            val track = playlist[currentIndex]
            audioPlayer.prepareAudio(track.url)
            playlistListener?.onTrackChanged(track, currentIndex)
        }
    }
    
    // 播放当前音频
    fun play() {
        if (playlist.isNotEmpty()) {
            if (currentIndex >= playlist.size) {
                currentIndex = 0
            }
            val track = playlist[currentIndex]
            audioPlayer.prepareAudio(track.url)
            playlistListener?.onTrackChanged(track, currentIndex)
        }
    }
    
    // 暂停
    fun pause() {
        audioPlayer.pause()
    }
    
    // 停止
    fun stop() {
        audioPlayer.stop()
    }
    
    // 下一首
    fun next() {
        when (playMode) {
            PlayMode.SHUFFLE -> nextShuffle()
            else -> {
                if (hasNext()) {
                    currentIndex++
                    playTrack(currentIndex)
                } else if (playMode == PlayMode.REPEAT_ALL) {
                    first()
                }
            }
        }
    }
    
    // 上一首
    fun previous() {
        when (playMode) {
            PlayMode.SHUFFLE -> previousShuffle()
            else -> {
                if (hasPrevious()) {
                    currentIndex--
                    playTrack(currentIndex)
                } else if (playMode == PlayMode.REPEAT_ALL) {
                    last()
                }
            }
        }
    }
    
    // 设置播放模式
    fun setPlayMode(mode: PlayMode) {
        playMode = mode
        if (mode == PlayMode.SHUFFLE) {
            updateShuffleIndices()
        }
        playlistListener?.onPlayModeChanged(mode)
    }
    
    private fun hasNext(): Boolean = currentIndex < playlist.size - 1
    private fun hasPrevious(): Boolean = currentIndex > 0
    
    private fun first() {
        currentIndex = 0
        playTrack(currentIndex)
    }
    
    private fun last() {
        currentIndex = playlist.size - 1
        playTrack(currentIndex)
    }
    
    private fun updateShuffleIndices() {
        shuffleIndices.clear()
        shuffleIndices.addAll(playlist.indices)
        shuffleIndices.shuffle()
        shuffleIndex = shuffleIndices.indexOf(currentIndex)
        if (shuffleIndex == -1) shuffleIndex = 0
    }
    
    private fun nextShuffle() {
        if (shuffleIndices.isNotEmpty()) {
            shuffleIndex = (shuffleIndex + 1) % shuffleIndices.size
            currentIndex = shuffleIndices[shuffleIndex]
            playTrack(currentIndex)
        }
    }
    
    private fun previousShuffle() {
        if (shuffleIndices.isNotEmpty()) {
            shuffleIndex = if (shuffleIndex > 0) shuffleIndex - 1 else shuffleIndices.size - 1
            currentIndex = shuffleIndices[shuffleIndex]
            playTrack(currentIndex)
        }
    }
    
    fun getCurrentTrack(): AudioTrack? {
        return if (currentIndex in 0 until playlist.size) playlist[currentIndex] else null
    }
    
    fun getPlaylist(): List<AudioTrack> = playlist.toList()
    
    fun getCurrentIndex(): Int = currentIndex
    
    fun getPlayMode(): PlayMode = playMode
    
    fun release() {
        audioPlayer.release()
    }
}
```

### AudioRecord 音频录制

```kotlin
class AudioRecorderManager {
    
    private var audioRecord: AudioRecord? = null
    private var isRecording = false
    private var recordingThread: Thread? = null
    private var outputFile: File? = null
    
    // 录音参数
    private val sampleRate = 44100
    private val channelConfig = AudioFormat.CHANNEL_IN_MONO
    private val audioFormat = AudioFormat.ENCODING_PCM_16BIT
    private val bufferSize = AudioRecord.getMinBufferSize(sampleRate, channelConfig, audioFormat)
    
    interface RecordingListener {
        fun onRecordingStarted()
        fun onRecordingStopped(file: File)
        fun onRecordingError(error: String)
        fun onAmplitudeUpdate(amplitude: Int)
    }
    
    private var recordingListener: RecordingListener? = null
    
    fun setRecordingListener(listener: RecordingListener) {
        this.recordingListener = listener
    }
    
    // 开始录音
    fun startRecording(outputFile: File): Boolean {
        if (isRecording) {
            return false
        }
        
        try {
            this.outputFile = outputFile
            
            audioRecord = AudioRecord(
                MediaRecorder.AudioSource.MIC,
                sampleRate,
                channelConfig,
                audioFormat,
                bufferSize
            )
            
            if (audioRecord?.state != AudioRecord.STATE_INITIALIZED) {
                recordingListener?.onRecordingError("AudioRecord initialization failed")
                return false
            }
            
            audioRecord?.startRecording()
            isRecording = true
            
            recordingThread = Thread {
                writeAudioDataToFile()
            }
            recordingThread?.start()
            
            recordingListener?.onRecordingStarted()
            return true
            
        } catch (e: Exception) {
            recordingListener?.onRecordingError("Failed to start recording: ${e.message}")
            return false
        }
    }
    
    // 停止录音
    fun stopRecording() {
        if (!isRecording) {
            return
        }
        
        isRecording = false
        
        audioRecord?.let { recorder ->
            try {
                recorder.stop()
                recorder.release()
            } catch (e: Exception) {
                recordingListener?.onRecordingError("Error stopping recording: ${e.message}")
            }
        }
        
        audioRecord = null
        
        recordingThread?.join()
        recordingThread = null
        
        outputFile?.let { file ->
            if (file.exists() && file.length() > 0) {
                recordingListener?.onRecordingStopped(file)
            } else {
                recordingListener?.onRecordingError("Recording file is empty or doesn't exist")
            }
        }
    }
    
    // 写入音频数据到文件
    private fun writeAudioDataToFile() {
        val audioData = ByteArray(bufferSize)
        var fileOutputStream: FileOutputStream? = null
        
        try {
            fileOutputStream = FileOutputStream(outputFile)
            
            while (isRecording) {
                val bytesRead = audioRecord?.read(audioData, 0, bufferSize) ?: 0
                
                if (bytesRead > 0) {
                    fileOutputStream.write(audioData, 0, bytesRead)
                    
                    // 计算音量振幅
                    val amplitude = calculateAmplitude(audioData, bytesRead)
                    recordingListener?.onAmplitudeUpdate(amplitude)
                }
            }
            
        } catch (e: Exception) {
            recordingListener?.onRecordingError("Error writing audio data: ${e.message}")
        } finally {
            try {
                fileOutputStream?.close()
            } catch (e: Exception) {
                // Ignore
            }
        }
    }
    
    // 计算音频振幅
    private fun calculateAmplitude(audioData: ByteArray, bytesRead: Int): Int {
        var sum = 0.0
        for (i in 0 until bytesRead step 2) {
            val sample = (audioData[i].toInt() or (audioData[i + 1].toInt() shl 8)).toShort()
            sum += sample * sample
        }
        val rms = sqrt(sum / (bytesRead / 2))
        return (rms * 100).toInt() // 转换为 0-100 的范围
    }
    
    // 转换 PCM 为 WAV 格式
    fun convertPcmToWav(pcmFile: File, wavFile: File): Boolean {
        return try {
            val pcmInputStream = FileInputStream(pcmFile)
            val wavOutputStream = FileOutputStream(wavFile)
            
            val totalAudioLen = pcmInputStream.available()
            val totalDataLen = totalAudioLen + 36
            val channels = 1
            val byteRate = sampleRate * channels * 2
            
            // 写入 WAV 文件头
            writeWavHeader(wavOutputStream, totalAudioLen, totalDataLen, sampleRate, channels, byteRate)
            
            // 复制 PCM 数据
            val buffer = ByteArray(4096)
            var bytesRead: Int
            while (pcmInputStream.read(buffer).also { bytesRead = it } != -1) {
                wavOutputStream.write(buffer, 0, bytesRead)
            }
            
            pcmInputStream.close()
            wavOutputStream.close()
            
            true
        } catch (e: Exception) {
            recordingListener?.onRecordingError("Error converting PCM to WAV: ${e.message}")
            false
        }
    }
    
    private fun writeWavHeader(
        out: FileOutputStream,
        totalAudioLen: Int,
        totalDataLen: Int,
        sampleRate: Int,
        channels: Int,
        byteRate: Int
    ) {
        val header = ByteArray(44)
        
        header[0] = 'R'.code.toByte()
        header[1] = 'I'.code.toByte()
        header[2] = 'F'.code.toByte()
        header[3] = 'F'.code.toByte()
        header[4] = (totalDataLen and 0xff).toByte()
        header[5] = (totalDataLen shr 8 and 0xff).toByte()
        header[6] = (totalDataLen shr 16 and 0xff).toByte()
        header[7] = (totalDataLen shr 24 and 0xff).toByte()
        header[8] = 'W'.code.toByte()
        header[9] = 'A'.code.toByte()
        header[10] = 'V'.code.toByte()
        header[11] = 'E'.code.toByte()
        header[12] = 'f'.code.toByte()
        header[13] = 'm'.code.toByte()
        header[14] = 't'.code.toByte()
        header[15] = ' '.code.toByte()
        header[16] = 16
        header[17] = 0
        header[18] = 0
        header[19] = 0
        header[20] = 1
        header[21] = 0
        header[22] = channels.toByte()
        header[23] = 0
        header[24] = (sampleRate and 0xff).toByte()
        header[25] = (sampleRate shr 8 and 0xff).toByte()
        header[26] = (sampleRate shr 16 and 0xff).toByte()
        header[27] = (sampleRate shr 24 and 0xff).toByte()
        header[28] = (byteRate and 0xff).toByte()
        header[29] = (byteRate shr 8 and 0xff).toByte()
        header[30] = (byteRate shr 16 and 0xff).toByte()
        header[31] = (byteRate shr 24 and 0xff).toByte()
        header[32] = (channels * 2).toByte()
        header[33] = 0
        header[34] = 16
        header[35] = 0
        header[36] = 'd'.code.toByte()
        header[37] = 'a'.code.toByte()
        header[38] = 't'.code.toByte()
        header[39] = 'a'.code.toByte()
        header[40] = (totalAudioLen and 0xff).toByte()
        header[41] = (totalAudioLen shr 8 and 0xff).toByte()
        header[42] = (totalAudioLen shr 16 and 0xff).toByte()
        header[43] = (totalAudioLen shr 24 and 0xff).toByte()
        
        out.write(header, 0, 44)
    }
    
    fun isRecording(): Boolean = isRecording
    
    fun release() {
        stopRecording()
    }
}
```

## 视频开发

### VideoView 视频播放

```kotlin
class VideoPlayerManager(private val context: Context) {
    
    private var videoView: VideoView? = null
    private var mediaController: MediaController? = null
    private var currentVideoUrl: String? = null
    private var currentPosition = 0
    
    interface VideoPlayerListener {
        fun onPrepared(duration: Int)
        fun onStarted()
        fun onPaused()
        fun onCompleted()
        fun onError(error: String)
        fun onProgressUpdate(currentPosition: Int, duration: Int)
    }
    
    private var playerListener: VideoPlayerListener? = null
    private var progressUpdateHandler: Handler? = null
    private var progressUpdateRunnable: Runnable? = null
    
    fun setVideoPlayerListener(listener: VideoPlayerListener) {
        this.playerListener = listener
    }
    
    // 初始化 VideoView
    fun initVideoView(videoView: VideoView) {
        this.videoView = videoView
        
        // 创建媒体控制器
        mediaController = MediaController(context)
        videoView.setMediaController(mediaController)
        
        // 设置监听器
        videoView.setOnPreparedListener { mediaPlayer ->
            val duration = mediaPlayer.duration
            playerListener?.onPrepared(duration)
            
            // 恢复播放位置
            if (currentPosition > 0) {
                videoView.seekTo(currentPosition)
            }
        }
        
        videoView.setOnCompletionListener {
            stopProgressUpdates()
            playerListener?.onCompleted()
        }
        
        videoView.setOnErrorListener { _, what, extra ->
            playerListener?.onError("Video error: what=$what, extra=$extra")
            true
        }
        
        videoView.setOnInfoListener { _, what, extra ->
            when (what) {
                MediaPlayer.MEDIA_INFO_VIDEO_RENDERING_START -> {
                    startProgressUpdates()
                    playerListener?.onStarted()
                }
                MediaPlayer.MEDIA_INFO_BUFFERING_START -> {
                    // 缓冲开始
                }
                MediaPlayer.MEDIA_INFO_BUFFERING_END -> {
                    // 缓冲结束
                }
            }
            true
        }
    }
    
    // 播放视频
    fun playVideo(videoUrl: String) {
        currentVideoUrl = videoUrl
        videoView?.let { view ->
            try {
                val uri = Uri.parse(videoUrl)
                view.setVideoURI(uri)
                view.start()
            } catch (e: Exception) {
                playerListener?.onError("Failed to play video: ${e.message}")
            }
        }
    }
    
    // 暂停播放
    fun pause() {
        videoView?.let { view ->
            if (view.isPlaying) {
                view.pause()
                currentPosition = view.currentPosition
                stopProgressUpdates()
                playerListener?.onPaused()
            }
        }
    }
    
    // 恢复播放
    fun resume() {
        videoView?.let { view ->
            if (!view.isPlaying) {
                view.start()
                startProgressUpdates()
                playerListener?.onStarted()
            }
        }
    }
    
    // 停止播放
    fun stop() {
        videoView?.let { view ->
            view.stopPlayback()
            currentPosition = 0
            stopProgressUpdates()
        }
    }
    
    // 跳转到指定位置
    fun seekTo(position: Int) {
        videoView?.seekTo(position)
        currentPosition = position
    }
    
    // 获取当前播放位置
    fun getCurrentPosition(): Int {
        return videoView?.currentPosition ?: currentPosition
    }
    
    // 获取视频总时长
    fun getDuration(): Int {
        return videoView?.duration ?: 0
    }
    
    // 是否正在播放
    fun isPlaying(): Boolean {
        return videoView?.isPlaying ?: false
    }
    
    // 设置视频缩放模式
    fun setVideoScaleType(scaleType: VideoScaleType) {
        videoView?.let { view ->
            when (scaleType) {
                VideoScaleType.FIT_CENTER -> {
                    view.layoutParams = view.layoutParams.apply {
                        width = ViewGroup.LayoutParams.MATCH_PARENT
                        height = ViewGroup.LayoutParams.WRAP_CONTENT
                    }
                }
                VideoScaleType.CENTER_CROP -> {
                    view.layoutParams = view.layoutParams.apply {
                        width = ViewGroup.LayoutParams.MATCH_PARENT
                        height = ViewGroup.LayoutParams.MATCH_PARENT
                    }
                }
                VideoScaleType.FIT_XY -> {
                    view.layoutParams = view.layoutParams.apply {
                        width = ViewGroup.LayoutParams.MATCH_PARENT
                        height = ViewGroup.LayoutParams.MATCH_PARENT
                    }
                }
            }
        }
    }
    
    enum class VideoScaleType {
        FIT_CENTER, CENTER_CROP, FIT_XY
    }
    
    // 开始进度更新
    private fun startProgressUpdates() {
        progressUpdateHandler = Handler(Looper.getMainLooper())
        progressUpdateRunnable = object : Runnable {
            override fun run() {
                videoView?.let { view ->
                    if (view.isPlaying) {
                        val currentPos = view.currentPosition
                        val duration = view.duration
                        playerListener?.onProgressUpdate(currentPos, duration)
                        progressUpdateHandler?.postDelayed(this, 1000)
                    }
                }
            }
        }
        progressUpdateRunnable?.let {
            progressUpdateHandler?.post(it)
        }
    }
    
    // 停止进度更新
    private fun stopProgressUpdates() {
        progressUpdateRunnable?.let {
            progressUpdateHandler?.removeCallbacks(it)
        }
        progressUpdateHandler = null
        progressUpdateRunnable = null
    }
    
    // 保存播放状态
    fun savePlaybackState(): Bundle {
        return Bundle().apply {
            putString("video_url", currentVideoUrl)
            putInt("current_position", getCurrentPosition())
        }
    }
    
    // 恢复播放状态
    fun restorePlaybackState(savedState: Bundle) {
        val videoUrl = savedState.getString("video_url")
        currentPosition = savedState.getInt("current_position", 0)
        
        if (!videoUrl.isNullOrEmpty()) {
            playVideo(videoUrl)
        }
    }
    
    fun release() {
        stopProgressUpdates()
        stop()
        videoView = null
        mediaController = null
    }
}

// ExoPlayer 高级视频播放器
class ExoPlayerManager(private val context: Context) {
    
    private var exoPlayer: ExoPlayer? = null
    private var playerView: PlayerView? = null
    
    interface ExoPlayerListener {
        fun onPlayerReady()
        fun onPlayerError(error: String)
        fun onPlaybackStateChanged(playbackState: Int)
        fun onIsPlayingChanged(isPlaying: Boolean)
    }
    
    private var playerListener: ExoPlayerListener? = null
    
    fun setExoPlayerListener(listener: ExoPlayerListener) {
        this.playerListener = listener
    }
    
    // 初始化播放器
    fun initPlayer(playerView: PlayerView) {
        this.playerView = playerView
        
        exoPlayer = ExoPlayer.Builder(context)
            .build()
            .also { player ->
                playerView.player = player
                
                player.addListener(object : Player.Listener {
                    override fun onPlaybackStateChanged(playbackState: Int) {
                        when (playbackState) {
                            Player.STATE_READY -> {
                                playerListener?.onPlayerReady()
                            }
                            Player.STATE_ENDED -> {
                                // 播放结束
                            }
                        }
                        playerListener?.onPlaybackStateChanged(playbackState)
                    }
                    
                    override fun onIsPlayingChanged(isPlaying: Boolean) {
                        playerListener?.onIsPlayingChanged(isPlaying)
                    }
                    
                    override fun onPlayerError(error: PlaybackException) {
                        playerListener?.onPlayerError(error.message ?: "Unknown error")
                    }
                })
            }
    }
    
    // 播放视频
    fun playVideo(videoUrl: String) {
        exoPlayer?.let { player ->
            val mediaItem = MediaItem.fromUri(videoUrl)
            player.setMediaItem(mediaItem)
            player.prepare()
            player.play()
        }
    }
    
    // 播放播放列表
    fun playPlaylist(videoUrls: List<String>) {
        exoPlayer?.let { player ->
            val mediaItems = videoUrls.map { MediaItem.fromUri(it) }
            player.setMediaItems(mediaItems)
            player.prepare()
            player.play()
        }
    }
    
    // 播放 HLS 流
    fun playHlsStream(hlsUrl: String) {
        exoPlayer?.let { player ->
            val dataSourceFactory = DefaultHttpDataSource.Factory()
            val hlsMediaSourceFactory = HlsMediaSource.Factory(dataSourceFactory)
            val mediaSource = hlsMediaSourceFactory.createMediaSource(MediaItem.fromUri(hlsUrl))
            
            player.setMediaSource(mediaSource)
            player.prepare()
            player.play()
        }
    }
    
    // 播放 DASH 流
    fun playDashStream(dashUrl: String) {
        exoPlayer?.let { player ->
            val dataSourceFactory = DefaultHttpDataSource.Factory()
            val dashMediaSourceFactory = DashMediaSource.Factory(dataSourceFactory)
            val mediaSource = dashMediaSourceFactory.createMediaSource(MediaItem.fromUri(dashUrl))
            
            player.setMediaSource(mediaSource)
            player.prepare()
            player.play()
        }
    }
    
    // 控制播放
    fun play() {
        exoPlayer?.play()
    }
    
    fun pause() {
        exoPlayer?.pause()
    }
    
    fun stop() {
        exoPlayer?.stop()
    }
    
    fun seekTo(position: Long) {
        exoPlayer?.seekTo(position)
    }
    
    // 获取播放信息
    fun getCurrentPosition(): Long {
        return exoPlayer?.currentPosition ?: 0
    }
    
    fun getDuration(): Long {
        return exoPlayer?.duration ?: 0
    }
    
    fun isPlaying(): Boolean {
        return exoPlayer?.isPlaying ?: false
    }
    
    // 设置播放速度
    fun setPlaybackSpeed(speed: Float) {
        exoPlayer?.setPlaybackSpeed(speed)
    }
    
    // 设置音量
    fun setVolume(volume: Float) {
        exoPlayer?.volume = volume
    }
    
    // 释放资源
    fun release() {
        exoPlayer?.release()
        exoPlayer = null
        playerView?.player = null
        playerView = null
    }
}
```

## 相机开发

### Camera2 API 实现

```kotlin
class Camera2Manager(private val context: Context) {
    
    private var cameraManager: CameraManager = context.getSystemService(Context.CAMERA_SERVICE) as CameraManager
    private var cameraDevice: CameraDevice? = null
    private var captureSession: CameraCaptureSession? = null
    private var imageReader: ImageReader? = null
    private var backgroundThread: HandlerThread? = null
    private var backgroundHandler: Handler? = null
    
    private var previewSize: Size? = null
    private var cameraId: String? = null
    private var sensorOrientation: Int = 0
    
    interface CameraListener {
        fun onCameraOpened()
        fun onCameraClosed()
        fun onCameraError(error: String)
        fun onPhotoTaken(imageFile: File)
        fun onPreviewStarted()
    }
    
    private var cameraListener: CameraListener? = null
    
    fun setCameraListener(listener: CameraListener) {
        this.cameraListener = listener
    }
    
    // 开启后台线程
    private fun startBackgroundThread() {
        backgroundThread = HandlerThread("CameraBackground").also { it.start() }
        backgroundHandler = Handler(backgroundThread?.looper!!)
    }
    
    // 停止后台线程
    private fun stopBackgroundThread() {
        backgroundThread?.quitSafely()
        try {
            backgroundThread?.join()
            backgroundThread = null
            backgroundHandler = null
        } catch (e: InterruptedException) {
            Log.e("Camera2Manager", "Error stopping background thread", e)
        }
    }
    
    // 获取最佳预览尺寸
    private fun chooseOptimalSize(
        choices: Array<Size>,
        textureViewWidth: Int,
        textureViewHeight: Int,
        maxWidth: Int,
        maxHeight: Int,
        aspectRatio: Size
    ): Size {
        val bigEnough = mutableListOf<Size>()
        val notBigEnough = mutableListOf<Size>()
        val w = aspectRatio.width
        val h = aspectRatio.height
        
        for (option in choices) {
            if (option.width <= maxWidth && option.height <= maxHeight &&
                option.height == option.width * h / w
            ) {
                if (option.width >= textureViewWidth && option.height >= textureViewHeight) {
                    bigEnough.add(option)
                } else {
                    notBigEnough.add(option)
                }
            }
        }
        
        return when {
            bigEnough.isNotEmpty() -> Collections.min(bigEnough, CompareSizesByArea())
            notBigEnough.isNotEmpty() -> Collections.max(notBigEnough, CompareSizesByArea())
            else -> {
                Log.e("Camera2Manager", "Couldn't find any suitable preview size")
                choices[0]
            }
        }
    }
    
    private class CompareSizesByArea : Comparator<Size> {
        override fun compare(lhs: Size, rhs: Size): Int {
            return Long.signum(lhs.width.toLong() * lhs.height - rhs.width.toLong() * rhs.height)
        }
    }
    
    // 打开相机
    @SuppressLint("MissingPermission")
    fun openCamera(textureView: TextureView, width: Int, height: Int) {
        startBackgroundThread()
        
        try {
            // 选择后置摄像头
            for (id in cameraManager.cameraIdList) {
                val characteristics = cameraManager.getCameraCharacteristics(id)
                val facing = characteristics.get(CameraCharacteristics.LENS_FACING)
                if (facing != null && facing == CameraCharacteristics.LENS_FACING_BACK) {
                    cameraId = id
                    break
                }
            }
            
            if (cameraId == null) {
                cameraListener?.onCameraError("No back camera found")
                return
            }
            
            val characteristics = cameraManager.getCameraCharacteristics(cameraId!!)
            val map = characteristics.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP)!!
            
            // 获取传感器方向
            sensorOrientation = characteristics.get(CameraCharacteristics.SENSOR_ORIENTATION)!!
            
            // 选择最佳预览尺寸
            previewSize = chooseOptimalSize(
                map.getOutputSizes(SurfaceTexture::class.java),
                width, height, width, height, Size(width, height)
            )
            
            // 设置 TextureView 的缓冲区大小
            textureView.surfaceTextureListener = object : TextureView.SurfaceTextureListener {
                override fun onSurfaceTextureAvailable(surface: SurfaceTexture, width: Int, height: Int) {
                    openCameraDevice()
                }
                
                override fun onSurfaceTextureSizeChanged(surface: SurfaceTexture, width: Int, height: Int) {}
                override fun onSurfaceTextureDestroyed(surface: SurfaceTexture): Boolean = true
                override fun onSurfaceTextureUpdated(surface: SurfaceTexture) {}
            }
            
            // 如果 TextureView 已经可用，直接打开相机
            if (textureView.isAvailable) {
                openCameraDevice()
            }
            
        } catch (e: CameraAccessException) {
            cameraListener?.onCameraError("Camera access exception: ${e.message}")
        }
    }
    
    // 打开相机设备
    @SuppressLint("MissingPermission")
    private fun openCameraDevice() {
        try {
            cameraManager.openCamera(cameraId!!, object : CameraDevice.StateCallback() {
                override fun onOpened(camera: CameraDevice) {
                    cameraDevice = camera
                    createCameraPreviewSession()
                    cameraListener?.onCameraOpened()
                }
                
                override fun onDisconnected(camera: CameraDevice) {
                    camera.close()
                    cameraDevice = null
                    cameraListener?.onCameraClosed()
                }
                
                override fun onError(camera: CameraDevice, error: Int) {
                    camera.close()
                    cameraDevice = null
                    cameraListener?.onCameraError("Camera device error: $error")
                }
            }, backgroundHandler)
            
        } catch (e: CameraAccessException) {
            cameraListener?.onCameraError("Failed to open camera: ${e.message}")
        }
    }
    
    // 创建预览会话
    private fun createCameraPreviewSession() {
        try {
            val texture = textureView.surfaceTexture!!
            texture.setDefaultBufferSize(previewSize!!.width, previewSize!!.height)
            
            val surface = Surface(texture)
            
            val previewRequestBuilder = cameraDevice!!.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW)
            previewRequestBuilder.addTarget(surface)
            
            cameraDevice!!.createCaptureSession(
                listOf(surface),
                object : CameraCaptureSession.StateCallback() {
                    override fun onConfigured(session: CameraCaptureSession) {
                        if (cameraDevice == null) return
                        
                        captureSession = session
                        try {
                            previewRequestBuilder.set(
                                CaptureRequest.CONTROL_AF_MODE,
                                CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE
                            )
                            previewRequestBuilder.set(
                                CaptureRequest.CONTROL_AE_MODE,
                                CaptureRequest.CONTROL_AE_MODE_ON_AUTO_FLASH
                            )
                            
                            val previewRequest = previewRequestBuilder.build()
                            captureSession!!.setRepeatingRequest(
                                previewRequest,
                                null,
                                backgroundHandler
                            )
                            
                            cameraListener?.onPreviewStarted()
                            
                        } catch (e: CameraAccessException) {
                            cameraListener?.onCameraError("Failed to start preview: ${e.message}")
                        }
                    }
                    
                    override fun onConfigureFailed(session: CameraCaptureSession) {
                        cameraListener?.onCameraError("Failed to configure camera session")
                    }
                },
                null
            )
            
        } catch (e: CameraAccessException) {
            cameraListener?.onCameraError("Failed to create preview session: ${e.message}")
        }
    }
    
    // 拍照
    fun takePicture(outputFile: File) {
        if (cameraDevice == null) {
            cameraListener?.onCameraError("Camera not opened")
            return
        }
        
        try {
            val characteristics = cameraManager.getCameraCharacteristics(cameraId!!)
            val jpegSizes = characteristics.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP)!!
                .getOutputSizes(ImageFormat.JPEG)
            
            val width = jpegSizes[0].width
            val height = jpegSizes[0].height
            
            imageReader = ImageReader.newInstance(width, height, ImageFormat.JPEG, 1)
            
            val readerListener = object : ImageReader.OnImageAvailableListener {
                override fun onImageAvailable(reader: ImageReader) {
                    val image = reader.acquireLatestImage()
                    val buffer = image.planes[0].buffer
                    val bytes = ByteArray(buffer.remaining())
                    buffer.get(bytes)
                    
                    try {
                        FileOutputStream(outputFile).use { output ->
                            output.write(bytes)
                        }
                        cameraListener?.onPhotoTaken(outputFile)
                    } catch (e: IOException) {
                        cameraListener?.onCameraError("Failed to save image: ${e.message}")
                    } finally {
                        image.close()
                    }
                }
            }
            
            imageReader!!.setOnImageAvailableListener(readerListener, backgroundHandler)
            
            val captureBuilder = cameraDevice!!.createCaptureRequest(CameraDevice.TEMPLATE_STILL_CAPTURE)
            captureBuilder.addTarget(imageReader!!.surface)
            
            // 设置自动对焦和自动曝光
            captureBuilder.set(CaptureRequest.CONTROL_AF_MODE, CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE)
            captureBuilder.set(CaptureRequest.CONTROL_AE_MODE, CaptureRequest.CONTROL_AE_MODE_ON_AUTO_FLASH)
            
            // 设置图片方向
            val rotation = (context as Activity).windowManager.defaultDisplay.rotation
            captureBuilder.set(CaptureRequest.JPEG_ORIENTATION, getOrientation(rotation))
            
            val captureCallback = object : CameraCaptureSession.CaptureCallback() {
                override fun onCaptureCompleted(
                    session: CameraCaptureSession,
                    request: CaptureRequest,
                    result: TotalCaptureResult
                ) {
                    // 拍照完成，重新开始预览
                    createCameraPreviewSession()
                }
            }
            
            captureSession!!.capture(captureBuilder.build(), captureCallback, backgroundHandler)
            
        } catch (e: CameraAccessException) {
            cameraListener?.onCameraError("Failed to take picture: ${e.message}")
        }
    }
    
    // 获取图片方向
    private fun getOrientation(rotation: Int): Int {
        return (ORIENTATIONS.get(rotation) + sensorOrientation + 270) % 360
    }
    
    companion object {
        private val ORIENTATIONS = SparseIntArray().apply {
            append(Surface.ROTATION_0, 90)
            append(Surface.ROTATION_90, 0)
            append(Surface.ROTATION_180, 270)
            append(Surface.ROTATION_270, 180)
        }
    }
    
    // 关闭相机
    fun closeCamera() {
        captureSession?.close()
        captureSession = null
        
        cameraDevice?.close()
        cameraDevice = null
        
        imageReader?.close()
        imageReader = null
        
        stopBackgroundThread()
        
        cameraListener?.onCameraClosed()
    }
}
```

## 图像处理

### Bitmap 操作

```kotlin
class ImageProcessor {
    
    companion object {
        
        // 缩放图片
        fun scaleBitmap(bitmap: Bitmap, targetWidth: Int, targetHeight: Int): Bitmap {
            return Bitmap.createScaledBitmap(bitmap, targetWidth, targetHeight, true)
        }
        
        // 按比例缩放图片
        fun scaleBitmapByRatio(bitmap: Bitmap, ratio: Float): Bitmap {
            val width = (bitmap.width * ratio).toInt()
            val height = (bitmap.height * ratio).toInt()
            return Bitmap.createScaledBitmap(bitmap, width, height, true)
        }
        
        // 裁剪图片
        fun cropBitmap(bitmap: Bitmap, x: Int, y: Int, width: Int, height: Int): Bitmap {
            return Bitmap.createBitmap(bitmap, x, y, width, height)
        }
        
        // 旋转图片
        fun rotateBitmap(bitmap: Bitmap, degrees: Float): Bitmap {
            val matrix = Matrix().apply {
                postRotate(degrees)
            }
            return Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true)
        }
        
        // 翻转图片
        fun flipBitmap(bitmap: Bitmap, horizontal: Boolean = true): Bitmap {
            val matrix = Matrix().apply {
                if (horizontal) {
                    preScale(-1.0f, 1.0f)
                } else {
                    preScale(1.0f, -1.0f)
                }
            }
            return Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true)
        }
        
        // 圆角图片
        fun roundCornerBitmap(bitmap: Bitmap, radius: Float): Bitmap {
            val output = Bitmap.createBitmap(bitmap.width, bitmap.height, Bitmap.Config.ARGB_8888)
            val canvas = Canvas(output)
            
            val paint = Paint().apply {
                isAntiAlias = true
                color = Color.WHITE
            }
            
            val rect = Rect(0, 0, bitmap.width, bitmap.height)
            val rectF = RectF(rect)
            
            canvas.drawRoundRect(rectF, radius, radius, paint)
            
            paint.xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_IN)
            canvas.drawBitmap(bitmap, rect, rect, paint)
            
            return output
        }
        
        // 圆形图片
        fun circularBitmap(bitmap: Bitmap): Bitmap {
            val size = minOf(bitmap.width, bitmap.height)
            val output = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888)
            val canvas = Canvas(output)
            
            val paint = Paint().apply {
                isAntiAlias = true
                color = Color.WHITE
            }
            
            val rect = Rect(0, 0, size, size)
            val radius = size / 2f
            
            canvas.drawCircle(radius, radius, radius, paint)
            
            paint.xfermode = PorterDuffXfermode(PorterDuff.Mode.SRC_IN)
            
            val srcRect = if (bitmap.width > bitmap.height) {
                Rect((bitmap.width - bitmap.height) / 2, 0, 
                     (bitmap.width + bitmap.height) / 2, bitmap.height)
            } else {
                Rect(0, (bitmap.height - bitmap.width) / 2, 
                     bitmap.width, (bitmap.height + bitmap.width) / 2)
            }
            
            canvas.drawBitmap(bitmap, srcRect, rect, paint)
            
            return output
        }
        
        // 添加水印
        fun addWatermark(bitmap: Bitmap, watermarkText: String, textSize: Float = 50f): Bitmap {
            val output = bitmap.copy(bitmap.config, true)
            val canvas = Canvas(output)
            
            val paint = Paint().apply {
                color = Color.WHITE
                this.textSize = textSize
                isAntiAlias = true
                setShadowLayer(2f, 2f, 2f, Color.BLACK)
            }
            
            val textBounds = Rect()
            paint.getTextBounds(watermarkText, 0, watermarkText.length, textBounds)
            
            val x = bitmap.width - textBounds.width() - 20f
            val y = bitmap.height - 20f
            
            canvas.drawText(watermarkText, x, y, paint)
            
            return output
        }
        
        // 调整亮度
        fun adjustBrightness(bitmap: Bitmap, brightness: Float): Bitmap {
            val colorMatrix = ColorMatrix().apply {
                set(floatArrayOf(
                    1f, 0f, 0f, 0f, brightness,
                    0f, 1f, 0f, 0f, brightness,
                    0f, 0f, 1f, 0f, brightness,
                    0f, 0f, 0f, 1f, 0f
                ))
            }
            
            return applyColorMatrix(bitmap, colorMatrix)
        }
        
        // 调整对比度
        fun adjustContrast(bitmap: Bitmap, contrast: Float): Bitmap {
            val colorMatrix = ColorMatrix().apply {
                set(floatArrayOf(
                    contrast, 0f, 0f, 0f, 0f,
                    0f, contrast, 0f, 0f, 0f,
                    0f, 0f, contrast, 0f, 0f,
                    0f, 0f, 0f, 1f, 0f
                ))
            }
            
            return applyColorMatrix(bitmap, colorMatrix)
        }
        
        // 调整饱和度
        fun adjustSaturation(bitmap: Bitmap, saturation: Float): Bitmap {
            val colorMatrix = ColorMatrix().apply {
                setSaturation(saturation)
            }
            
            return applyColorMatrix(bitmap, colorMatrix)
        }
        
        // 灰度化
        fun toGrayscale(bitmap: Bitmap): Bitmap {
            val colorMatrix = ColorMatrix().apply {
                setSaturation(0f)
            }
            
            return applyColorMatrix(bitmap, colorMatrix)
        }
        
        // 应用颜色矩阵
        private fun applyColorMatrix(bitmap: Bitmap, colorMatrix: ColorMatrix): Bitmap {
            val output = Bitmap.createBitmap(bitmap.width, bitmap.height, bitmap.config)
            val canvas = Canvas(output)
            
            val paint = Paint().apply {
                colorFilter = ColorMatrixColorFilter(colorMatrix)
            }
            
            canvas.drawBitmap(bitmap, 0f, 0f, paint)
            
            return output
        }
        
        // 模糊效果
        fun blurBitmap(context: Context, bitmap: Bitmap, radius: Float): Bitmap {
            val output = Bitmap.createBitmap(bitmap.width, bitmap.height, bitmap.config)
            
            val renderScript = RenderScript.create(context)
            val blurScript = ScriptIntrinsicBlur.create(renderScript, Element.U8_4(renderScript))
            
            val inputAllocation = Allocation.createFromBitmap(renderScript, bitmap)
            val outputAllocation = Allocation.createFromBitmap(renderScript, output)
            
            blurScript.setRadius(radius)
            blurScript.setInput(inputAllocation)
            blurScript.forEach(outputAllocation)
            
            outputAllocation.copyTo(output)
            
            renderScript.destroy()
            
            return output
        }
        
        // 压缩图片
        fun compressBitmap(bitmap: Bitmap, quality: Int = 80): ByteArray {
            val outputStream = ByteArrayOutputStream()
            bitmap.compress(Bitmap.CompressFormat.JPEG, quality, outputStream)
            return outputStream.toByteArray()
        }
        
        // 从文件加载图片并压缩
        fun loadAndCompressBitmap(filePath: String, maxWidth: Int, maxHeight: Int): Bitmap? {
            val options = BitmapFactory.Options().apply {
                inJustDecodeBounds = true
            }
            
            BitmapFactory.decodeFile(filePath, options)
            
            options.inSampleSize = calculateInSampleSize(options, maxWidth, maxHeight)
            options.inJustDecodeBounds = false
            
            return BitmapFactory.decodeFile(filePath, options)
        }
        
        // 计算采样率
        private fun calculateInSampleSize(
            options: BitmapFactory.Options,
            reqWidth: Int,
            reqHeight: Int
        ): Int {
            val height = options.outHeight
            val width = options.outWidth
            var inSampleSize = 1
            
            if (height > reqHeight || width > reqWidth) {
                val halfHeight = height / 2
                val halfWidth = width / 2
                
                while (halfHeight / inSampleSize >= reqHeight && halfWidth / inSampleSize >= reqWidth) {
                    inSampleSize *= 2
                }
            }
            
            return inSampleSize
        }
    }
}
```

## 总结

Android 多媒体开发涵盖了音频、视频、相机和图像处理等多个重要领域。通过本文的深入探讨，我们学习了：

1. **音频开发**：MediaPlayer 播放、AudioRecord 录制、播放列表管理
2. **视频开发**：VideoView 基础播放、ExoPlayer 高级播放器
3. **相机开发**：Camera2 API 的完整实现和最佳实践
4. **图像处理**：Bitmap 操作、滤镜效果、图片压缩优化

掌握这些技术后，开发者可以创建功能丰富、性能优秀的多媒体应用。

在实际开发中，还需要注意以下几个关键点：

**性能优化**：多媒体操作通常比较耗费资源，需要合理管理内存、优化图片加载、使用后台线程处理耗时操作。

**权限管理**：相机、麦克风、存储等功能需要动态申请权限，确保用户体验的同时保护隐私。

**兼容性处理**：不同设备的硬件能力差异较大，需要做好适配和降级处理。

**用户体验**：提供直观的操作界面、及时的反馈信息、流畅的交互体验。

通过系统学习和实践这些多媒体开发技术，开发者能够构建出专业级的多媒体应用，为用户提供优质的视听体验。