---
layout: post
title: Android 自定义 View 开发深入指南
categories: android
tags: [自定义View, Android, UI开发, Canvas, 绘制]
date: 2024/3/18 15:25:00
---

![title](https://image.sideproject.cn/titlex/titlex_customview.jpg)

## 引言

自定义 View 是 Android 开发中的重要技能，它允许开发者创建独特的用户界面组件，满足特定的设计需求和交互体验。通过自定义 View，我们可以突破系统控件的限制，实现复杂的动画效果、特殊的绘制需求和个性化的用户交互。本文将深入探讨 Android 自定义 View 的核心概念、开发技巧和最佳实践。

## 自定义 View 基础

### View 的绘制流程

Android 中 View 的绘制遵循三个主要步骤：

1. **Measure（测量）**：确定 View 的大小
2. **Layout（布局）**：确定 View 的位置
3. **Draw（绘制）**：将 View 绘制到屏幕上

```kotlin
class CustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private var centerX = 0f
    private var centerY = 0f
    private var radius = 0f
    
    init {
        // 初始化画笔
        paint.apply {
            color = Color.BLUE
            style = Paint.Style.FILL
            strokeWidth = 4f
        }
    }
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec)
        
        val widthMode = MeasureSpec.getMode(widthMeasureSpec)
        val widthSize = MeasureSpec.getSize(widthMeasureSpec)
        val heightMode = MeasureSpec.getMode(heightMeasureSpec)
        val heightSize = MeasureSpec.getSize(heightMeasureSpec)
        
        val width = when (widthMode) {
            MeasureSpec.EXACTLY -> widthSize
            MeasureSpec.AT_MOST -> minOf(200.dp, widthSize)
            else -> 200.dp
        }
        
        val height = when (heightMode) {
            MeasureSpec.EXACTLY -> heightSize
            MeasureSpec.AT_MOST -> minOf(200.dp, heightSize)
            else -> 200.dp
        }
        
        setMeasuredDimension(width, height)
    }
    
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        centerX = w / 2f
        centerY = h / 2f
        radius = minOf(w, h) / 2f - 20f
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 绘制圆形
        canvas.drawCircle(centerX, centerY, radius, paint)
    }
    
    // 扩展属性：dp 转 px
    private val Int.dp: Int
        get() = (this * resources.displayMetrics.density).toInt()
}
```

### 自定义属性

```xml
<!-- res/values/attrs.xml -->
<resources>
    <declare-styleable name="CircleProgressView">
        <attr name="circleColor" format="color" />
        <attr name="progressColor" format="color" />
        <attr name="strokeWidth" format="dimension" />
        <attr name="progress" format="float" />
        <attr name="maxProgress" format="float" />
        <attr name="showText" format="boolean" />
        <attr name="textSize" format="dimension" />
        <attr name="textColor" format="color" />
        <attr name="animationDuration" format="integer" />
    </declare-styleable>
    
    <declare-styleable name="WaveView">
        <attr name="waveColor" format="color" />
        <attr name="waveHeight" format="dimension" />
        <attr name="waveLength" format="dimension" />
        <attr name="waveSpeed" format="float" />
        <attr name="waveCount" format="integer" />
    </declare-styleable>
</resources>
```

```kotlin
class CircleProgressView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    // 属性变量
    private var circleColor = Color.GRAY
    private var progressColor = Color.BLUE
    private var strokeWidth = 10f
    private var progress = 0f
    private var maxProgress = 100f
    private var showText = true
    private var textSize = 48f
    private var textColor = Color.BLACK
    private var animationDuration = 1000
    
    // 绘制相关
    private val circlePaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val textPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val rectF = RectF()
    
    // 动画
    private var currentProgress = 0f
    private var progressAnimator: ValueAnimator? = null
    
    init {
        // 解析自定义属性
        context.theme.obtainStyledAttributes(
            attrs,
            R.styleable.CircleProgressView,
            0, 0
        ).apply {
            try {
                circleColor = getColor(R.styleable.CircleProgressView_circleColor, Color.GRAY)
                progressColor = getColor(R.styleable.CircleProgressView_progressColor, Color.BLUE)
                strokeWidth = getDimension(R.styleable.CircleProgressView_strokeWidth, 10f)
                progress = getFloat(R.styleable.CircleProgressView_progress, 0f)
                maxProgress = getFloat(R.styleable.CircleProgressView_maxProgress, 100f)
                showText = getBoolean(R.styleable.CircleProgressView_showText, true)
                textSize = getDimension(R.styleable.CircleProgressView_textSize, 48f)
                textColor = getColor(R.styleable.CircleProgressView_textColor, Color.BLACK)
                animationDuration = getInt(R.styleable.CircleProgressView_animationDuration, 1000)
            } finally {
                recycle()
            }
        }
        
        initPaints()
    }
    
    private fun initPaints() {
        circlePaint.apply {
            color = circleColor
            style = Paint.Style.STROKE
            strokeWidth = this@CircleProgressView.strokeWidth
            strokeCap = Paint.Cap.ROUND
        }
        
        progressPaint.apply {
            color = progressColor
            style = Paint.Style.STROKE
            strokeWidth = this@CircleProgressView.strokeWidth
            strokeCap = Paint.Cap.ROUND
        }
        
        textPaint.apply {
            color = textColor
            textSize = this@CircleProgressView.textSize
            textAlign = Paint.Align.CENTER
            typeface = Typeface.DEFAULT_BOLD
        }
    }
    
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        
        val padding = strokeWidth / 2
        rectF.set(
            padding,
            padding,
            w - padding,
            h - padding
        )
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 绘制背景圆环
        canvas.drawOval(rectF, circlePaint)
        
        // 绘制进度圆弧
        val sweepAngle = (currentProgress / maxProgress) * 360f
        canvas.drawArc(rectF, -90f, sweepAngle, false, progressPaint)
        
        // 绘制文字
        if (showText) {
            val text = "${(currentProgress / maxProgress * 100).toInt()}%"
            val centerX = width / 2f
            val centerY = height / 2f - (textPaint.descent() + textPaint.ascent()) / 2
            canvas.drawText(text, centerX, centerY, textPaint)
        }
    }
    
    // 公共方法
    fun setProgress(progress: Float, animated: Boolean = true) {
        val newProgress = progress.coerceIn(0f, maxProgress)
        
        if (animated) {
            animateProgress(newProgress)
        } else {
            this.progress = newProgress
            currentProgress = newProgress
            invalidate()
        }
    }
    
    private fun animateProgress(targetProgress: Float) {
        progressAnimator?.cancel()
        
        progressAnimator = ValueAnimator.ofFloat(currentProgress, targetProgress).apply {
            duration = animationDuration.toLong()
            interpolator = DecelerateInterpolator()
            addUpdateListener { animator ->
                currentProgress = animator.animatedValue as Float
                invalidate()
            }
            start()
        }
    }
    
    // 属性设置方法
    fun setCircleColor(color: Int) {
        circleColor = color
        circlePaint.color = color
        invalidate()
    }
    
    fun setProgressColor(color: Int) {
        progressColor = color
        progressPaint.color = color
        invalidate()
    }
    
    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        progressAnimator?.cancel()
    }
}
```

## 复杂自定义 View 实现

### 波浪效果 View

```kotlin
class WaveView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    private var waveColor = Color.BLUE
    private var waveHeight = 50f
    private var waveLength = 200f
    private var waveSpeed = 2f
    private var waveCount = 2
    
    private val wavePaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val wavePath = Path()
    private var waveOffset = 0f
    private var animator: ValueAnimator? = null
    
    init {
        context.theme.obtainStyledAttributes(
            attrs,
            R.styleable.WaveView,
            0, 0
        ).apply {
            try {
                waveColor = getColor(R.styleable.WaveView_waveColor, Color.BLUE)
                waveHeight = getDimension(R.styleable.WaveView_waveHeight, 50f)
                waveLength = getDimension(R.styleable.WaveView_waveLength, 200f)
                waveSpeed = getFloat(R.styleable.WaveView_waveSpeed, 2f)
                waveCount = getInt(R.styleable.WaveView_waveCount, 2)
            } finally {
                recycle()
            }
        }
        
        initPaint()
        startAnimation()
    }
    
    private fun initPaint() {
        wavePaint.apply {
            color = waveColor
            style = Paint.Style.FILL
            alpha = 128
        }
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        for (i in 0 until waveCount) {
            drawWave(canvas, i)
        }
    }
    
    private fun drawWave(canvas: Canvas, waveIndex: Int) {
        wavePath.reset()
        
        val waveY = height * 0.7f
        val phaseShift = waveIndex * Math.PI / 4
        
        wavePath.moveTo(-waveLength, waveY)
        
        var x = -waveLength
        while (x <= width + waveLength) {
            val y = (waveY + waveHeight * sin(
                2 * Math.PI * (x + waveOffset) / waveLength + phaseShift
            )).toFloat()
            
            wavePath.lineTo(x, y)
            x += 5f
        }
        
        wavePath.lineTo(width + waveLength, height.toFloat())
        wavePath.lineTo(-waveLength, height.toFloat())
        wavePath.close()
        
        // 设置不同波浪的透明度
        wavePaint.alpha = (128 - waveIndex * 30).coerceAtLeast(50)
        canvas.drawPath(wavePath, wavePaint)
    }
    
    private fun startAnimation() {
        animator = ValueAnimator.ofFloat(0f, waveLength).apply {
            duration = (waveLength / waveSpeed * 10).toLong()
            repeatCount = ValueAnimator.INFINITE
            interpolator = LinearInterpolator()
            addUpdateListener { animator ->
                waveOffset = animator.animatedValue as Float
                invalidate()
            }
            start()
        }
    }
    
    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        animator?.cancel()
    }
    
    override fun onAttachedToWindow() {
        super.onAttachedToWindow()
        if (animator?.isRunning != true) {
            startAnimation()
        }
    }
}
```

### 雷达扫描 View

```kotlin
class RadarView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    private val circlePaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val linePaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val sweepPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val dotPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    
    private var centerX = 0f
    private var centerY = 0f
    private var radius = 0f
    
    private var sweepAngle = 0f
    private var animator: ValueAnimator? = null
    
    // 雷达点数据
    private val radarDots = mutableListOf<RadarDot>()
    
    data class RadarDot(
        val x: Float,
        val y: Float,
        val alpha: Float = 1f,
        val size: Float = 8f
    )
    
    init {
        initPaints()
        generateRandomDots()
        startSweepAnimation()
    }
    
    private fun initPaints() {
        circlePaint.apply {
            color = Color.GREEN
            style = Paint.Style.STROKE
            strokeWidth = 2f
            alpha = 100
        }
        
        linePaint.apply {
            color = Color.GREEN
            strokeWidth = 1f
            alpha = 150
        }
        
        sweepPaint.apply {
            shader = SweepGradient(
                0f, 0f,
                intArrayOf(
                    Color.TRANSPARENT,
                    Color.argb(100, 0, 255, 0),
                    Color.argb(200, 0, 255, 0)
                ),
                floatArrayOf(0f, 0.5f, 1f)
            )
        }
        
        dotPaint.apply {
            color = Color.GREEN
            style = Paint.Style.FILL
        }
    }
    
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        
        centerX = w / 2f
        centerY = h / 2f
        radius = minOf(w, h) / 2f - 50f
        
        // 更新渐变中心点
        sweepPaint.shader = SweepGradient(
            centerX, centerY,
            intArrayOf(
                Color.TRANSPARENT,
                Color.argb(100, 0, 255, 0),
                Color.argb(200, 0, 255, 0)
            ),
            floatArrayOf(0f, 0.5f, 1f)
        )
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 绘制同心圆
        for (i in 1..4) {
            val r = radius * i / 4
            canvas.drawCircle(centerX, centerY, r, circlePaint)
        }
        
        // 绘制十字线
        canvas.drawLine(centerX - radius, centerY, centerX + radius, centerY, linePaint)
        canvas.drawLine(centerX, centerY - radius, centerX, centerY + radius, linePaint)
        
        // 绘制扫描效果
        canvas.save()
        canvas.rotate(sweepAngle, centerX, centerY)
        canvas.drawCircle(centerX, centerY, radius, sweepPaint)
        canvas.restore()
        
        // 绘制雷达点
        drawRadarDots(canvas)
    }
    
    private fun drawRadarDots(canvas: Canvas) {
        radarDots.forEach { dot ->
            dotPaint.alpha = (dot.alpha * 255).toInt()
            canvas.drawCircle(dot.x, dot.y, dot.size, dotPaint)
        }
    }
    
    private fun generateRandomDots() {
        radarDots.clear()
        
        repeat(15) {
            val angle = Math.random() * 2 * Math.PI
            val distance = Math.random() * radius * 0.8
            
            val x = (centerX + distance * cos(angle)).toFloat()
            val y = (centerY + distance * sin(angle)).toFloat()
            
            radarDots.add(
                RadarDot(
                    x = x,
                    y = y,
                    alpha = (0.3 + Math.random() * 0.7).toFloat(),
                    size = (4 + Math.random() * 8).toFloat()
                )
            )
        }
    }
    
    private fun startSweepAnimation() {
        animator = ValueAnimator.ofFloat(0f, 360f).apply {
            duration = 3000
            repeatCount = ValueAnimator.INFINITE
            interpolator = LinearInterpolator()
            addUpdateListener { animator ->
                sweepAngle = animator.animatedValue as Float
                invalidate()
            }
            start()
        }
    }
    
    fun addRadarDot(x: Float, y: Float) {
        // 将屏幕坐标转换为相对于雷达中心的坐标
        val distance = sqrt((x - centerX).pow(2) + (y - centerY).pow(2))
        if (distance <= radius) {
            radarDots.add(RadarDot(x, y))
            invalidate()
        }
    }
    
    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        animator?.cancel()
    }
}
```

## 触摸事件处理

### 可拖拽的自定义 View

```kotlin
class DraggableView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val gestureDetector: GestureDetector
    private val scaleDetector: ScaleGestureDetector
    
    // 变换矩阵
    private val matrix = Matrix()
    private val savedMatrix = Matrix()
    
    // 触摸状态
    private enum class Mode {
        NONE, DRAG, ZOOM
    }
    
    private var mode = Mode.NONE
    private var lastTouchX = 0f
    private var lastTouchY = 0f
    private var startDistance = 0f
    private var midPoint = PointF()
    
    // 绘制内容
    private var scale = 1f
    private var translateX = 0f
    private var translateY = 0f
    
    init {
        paint.apply {
            color = Color.BLUE
            style = Paint.Style.FILL
        }
        
        gestureDetector = GestureDetector(context, GestureListener())
        scaleDetector = ScaleGestureDetector(context, ScaleListener())
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        canvas.save()
        canvas.concat(matrix)
        
        // 绘制可拖拽的内容
        drawContent(canvas)
        
        canvas.restore()
    }
    
    private fun drawContent(canvas: Canvas) {
        val centerX = width / 2f
        val centerY = height / 2f
        val radius = 100f
        
        // 绘制主圆
        canvas.drawCircle(centerX, centerY, radius, paint)
        
        // 绘制装饰圆
        paint.color = Color.RED
        canvas.drawCircle(centerX - 50, centerY - 50, 30f, paint)
        canvas.drawCircle(centerX + 50, centerY - 50, 30f, paint)
        canvas.drawCircle(centerX, centerY + 50, 30f, paint)
        
        paint.color = Color.BLUE
    }
    
    override fun onTouchEvent(event: MotionEvent): Boolean {
        scaleDetector.onTouchEvent(event)
        gestureDetector.onTouchEvent(event)
        
        when (event.action and MotionEvent.ACTION_MASK) {
            MotionEvent.ACTION_DOWN -> {
                savedMatrix.set(matrix)
                lastTouchX = event.x
                lastTouchY = event.y
                mode = Mode.DRAG
            }
            
            MotionEvent.ACTION_POINTER_DOWN -> {
                startDistance = getDistance(event)
                if (startDistance > 10f) {
                    savedMatrix.set(matrix)
                    getMidPoint(midPoint, event)
                    mode = Mode.ZOOM
                }
            }
            
            MotionEvent.ACTION_MOVE -> {
                when (mode) {
                    Mode.DRAG -> {
                        matrix.set(savedMatrix)
                        val dx = event.x - lastTouchX
                        val dy = event.y - lastTouchY
                        matrix.postTranslate(dx, dy)
                    }
                    
                    Mode.ZOOM -> {
                        val newDistance = getDistance(event)
                        if (newDistance > 10f) {
                            matrix.set(savedMatrix)
                            val scale = newDistance / startDistance
                            matrix.postScale(scale, scale, midPoint.x, midPoint.y)
                        }
                    }
                    
                    else -> {}
                }
            }
            
            MotionEvent.ACTION_UP,
            MotionEvent.ACTION_POINTER_UP -> {
                mode = Mode.NONE
            }
        }
        
        invalidate()
        return true
    }
    
    private fun getDistance(event: MotionEvent): Float {
        val x = event.getX(0) - event.getX(1)
        val y = event.getY(0) - event.getY(1)
        return sqrt(x * x + y * y)
    }
    
    private fun getMidPoint(point: PointF, event: MotionEvent) {
        val x = event.getX(0) + event.getX(1)
        val y = event.getY(0) + event.getY(1)
        point.set(x / 2, y / 2)
    }
    
    private inner class GestureListener : GestureDetector.SimpleOnGestureListener() {
        
        override fun onDoubleTap(e: MotionEvent): Boolean {
            // 双击重置
            matrix.reset()
            invalidate()
            return true
        }
        
        override fun onLongPress(e: MotionEvent) {
            // 长按事件
            performHapticFeedback(HapticFeedbackConstants.LONG_PRESS)
            // 可以在这里添加长按逻辑
        }
        
        override fun onFling(
            e1: MotionEvent?,
            e2: MotionEvent,
            velocityX: Float,
            velocityY: Float
        ): Boolean {
            // 惯性滑动
            startFlingAnimation(velocityX, velocityY)
            return true
        }
    }
    
    private inner class ScaleListener : ScaleGestureDetector.SimpleOnScaleGestureListener() {
        
        override fun onScale(detector: ScaleGestureDetector): Boolean {
            val scaleFactor = detector.scaleFactor
            matrix.postScale(
                scaleFactor,
                scaleFactor,
                detector.focusX,
                detector.focusY
            )
            invalidate()
            return true
        }
    }
    
    private fun startFlingAnimation(velocityX: Float, velocityY: Float) {
        val animator = ValueAnimator.ofFloat(0f, 1f).apply {
            duration = 1000
            interpolator = DecelerateInterpolator()
            addUpdateListener { animator ->
                val fraction = animator.animatedValue as Float
                val currentVelocityX = velocityX * (1 - fraction)
                val currentVelocityY = velocityY * (1 - fraction)
                
                matrix.postTranslate(
                    currentVelocityX * 0.01f,
                    currentVelocityY * 0.01f
                )
                invalidate()
            }
        }
        animator.start()
    }
}
```

### 滑动选择器 View

```kotlin
class SliderView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    private val trackPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val thumbPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val textPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    
    private var minValue = 0f
    private var maxValue = 100f
    private var currentValue = 50f
    private var thumbRadius = 30f
    private var trackHeight = 8f
    
    private var trackRect = RectF()
    private var progressRect = RectF()
    private var thumbX = 0f
    private var thumbY = 0f
    
    private var isDragging = false
    private var onValueChangeListener: ((Float) -> Unit)? = null
    
    init {
        initPaints()
    }
    
    private fun initPaints() {
        trackPaint.apply {
            color = Color.LTGRAY
            style = Paint.Style.FILL
        }
        
        progressPaint.apply {
            color = Color.BLUE
            style = Paint.Style.FILL
        }
        
        thumbPaint.apply {
            color = Color.WHITE
            style = Paint.Style.FILL
            setShadowLayer(8f, 0f, 4f, Color.argb(50, 0, 0, 0))
        }
        
        textPaint.apply {
            color = Color.BLACK
            textSize = 32f
            textAlign = Paint.Align.CENTER
        }
        
        setLayerType(LAYER_TYPE_SOFTWARE, null) // 启用阴影
    }
    
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        
        val centerY = h / 2f
        val left = thumbRadius + 20f
        val right = w - thumbRadius - 20f
        
        trackRect.set(
            left,
            centerY - trackHeight / 2,
            right,
            centerY + trackHeight / 2
        )
        
        thumbY = centerY
        updateThumbPosition()
    }
    
    private fun updateThumbPosition() {
        val progress = (currentValue - minValue) / (maxValue - minValue)
        thumbX = trackRect.left + progress * trackRect.width()
        
        progressRect.set(
            trackRect.left,
            trackRect.top,
            thumbX,
            trackRect.bottom
        )
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 绘制轨道
        canvas.drawRoundRect(trackRect, trackHeight / 2, trackHeight / 2, trackPaint)
        
        // 绘制进度
        canvas.drawRoundRect(progressRect, trackHeight / 2, trackHeight / 2, progressPaint)
        
        // 绘制滑块
        canvas.drawCircle(thumbX, thumbY, thumbRadius, thumbPaint)
        
        // 绘制数值文本
        val text = currentValue.toInt().toString()
        canvas.drawText(text, thumbX, thumbY - thumbRadius - 20f, textPaint)
        
        // 绘制刻度
        drawScale(canvas)
    }
    
    private fun drawScale(canvas: Canvas) {
        val scaleCount = 5
        val scalePaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.GRAY
            strokeWidth = 2f
        }
        
        for (i in 0..scaleCount) {
            val x = trackRect.left + i * trackRect.width() / scaleCount
            val startY = trackRect.bottom + 10f
            val endY = startY + 15f
            
            canvas.drawLine(x, startY, x, endY, scalePaint)
            
            // 绘制刻度值
            val value = minValue + i * (maxValue - minValue) / scaleCount
            val scaleText = value.toInt().toString()
            textPaint.textSize = 24f
            canvas.drawText(scaleText, x, endY + 30f, textPaint)
            textPaint.textSize = 32f
        }
    }
    
    override fun onTouchEvent(event: MotionEvent): Boolean {
        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                val distance = sqrt(
                    (event.x - thumbX).pow(2) + (event.y - thumbY).pow(2)
                )
                
                if (distance <= thumbRadius * 1.5f) {
                    isDragging = true
                    parent.requestDisallowInterceptTouchEvent(true)
                    
                    // 添加按下动画
                    animateThumbScale(1.2f)
                    return true
                }
            }
            
            MotionEvent.ACTION_MOVE -> {
                if (isDragging) {
                    val newThumbX = event.x.coerceIn(trackRect.left, trackRect.right)
                    val progress = (newThumbX - trackRect.left) / trackRect.width()
                    val newValue = minValue + progress * (maxValue - minValue)
                    
                    setValue(newValue)
                    return true
                }
            }
            
            MotionEvent.ACTION_UP,
            MotionEvent.ACTION_CANCEL -> {
                if (isDragging) {
                    isDragging = false
                    parent.requestDisallowInterceptTouchEvent(false)
                    
                    // 添加释放动画
                    animateThumbScale(1f)
                    return true
                }
            }
        }
        
        return super.onTouchEvent(event)
    }
    
    private fun animateThumbScale(targetScale: Float) {
        val animator = ValueAnimator.ofFloat(1f, targetScale).apply {
            duration = 150
            interpolator = OvershootInterpolator()
            addUpdateListener {
                // 这里可以实现滑块缩放动画
                invalidate()
            }
        }
        animator.start()
    }
    
    fun setValue(value: Float) {
        val newValue = value.coerceIn(minValue, maxValue)
        if (newValue != currentValue) {
            currentValue = newValue
            updateThumbPosition()
            onValueChangeListener?.invoke(currentValue)
            invalidate()
        }
    }
    
    fun setValueRange(min: Float, max: Float) {
        minValue = min
        maxValue = max
        currentValue = currentValue.coerceIn(minValue, maxValue)
        updateThumbPosition()
        invalidate()
    }
    
    fun setOnValueChangeListener(listener: (Float) -> Unit) {
        onValueChangeListener = listener
    }
}
```

## ViewGroup 自定义

### 流式布局 FlowLayout

```kotlin
class FlowLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : ViewGroup(context, attrs, defStyleAttr) {
    
    private var horizontalSpacing = 10.dp
    private var verticalSpacing = 10.dp
    private val childBounds = mutableListOf<Rect>()
    
    init {
        context.theme.obtainStyledAttributes(
            attrs,
            R.styleable.FlowLayout,
            0, 0
        ).apply {
            try {
                horizontalSpacing = getDimensionPixelSize(
                    R.styleable.FlowLayout_horizontalSpacing,
                    10.dp
                )
                verticalSpacing = getDimensionPixelSize(
                    R.styleable.FlowLayout_verticalSpacing,
                    10.dp
                )
            } finally {
                recycle()
            }
        }
    }
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val widthMode = MeasureSpec.getMode(widthMeasureSpec)
        val widthSize = MeasureSpec.getSize(widthMeasureSpec)
        val heightMode = MeasureSpec.getMode(heightMeasureSpec)
        val heightSize = MeasureSpec.getSize(heightMeasureSpec)
        
        // 测量所有子 View
        measureChildren(widthMeasureSpec, heightMeasureSpec)
        
        // 计算布局
        val result = calculateLayout(widthSize)
        
        val finalWidth = when (widthMode) {
            MeasureSpec.EXACTLY -> widthSize
            else -> result.width
        }
        
        val finalHeight = when (heightMode) {
            MeasureSpec.EXACTLY -> heightSize
            else -> result.height
        }
        
        setMeasuredDimension(finalWidth, finalHeight)
    }
    
    private fun calculateLayout(maxWidth: Int): LayoutResult {
        childBounds.clear()
        
        var currentLineWidth = paddingLeft
        var currentLineHeight = 0
        var totalHeight = paddingTop
        var maxLineWidth = 0
        
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            if (child.visibility == GONE) continue
            
            val childWidth = child.measuredWidth
            val childHeight = child.measuredHeight
            
            // 检查是否需要换行
            if (currentLineWidth + childWidth + paddingRight > maxWidth && currentLineWidth > paddingLeft) {
                // 换行
                maxLineWidth = maxOf(maxLineWidth, currentLineWidth - horizontalSpacing)
                totalHeight += currentLineHeight + verticalSpacing
                currentLineWidth = paddingLeft
                currentLineHeight = 0
            }
            
            // 记录子 View 的位置
            val left = currentLineWidth
            val top = totalHeight
            val right = left + childWidth
            val bottom = top + childHeight
            
            childBounds.add(Rect(left, top, right, bottom))
            
            currentLineWidth += childWidth + horizontalSpacing
            currentLineHeight = maxOf(currentLineHeight, childHeight)
        }
        
        // 处理最后一行
        maxLineWidth = maxOf(maxLineWidth, currentLineWidth - horizontalSpacing)
        totalHeight += currentLineHeight + paddingBottom
        
        return LayoutResult(
            width = maxLineWidth + paddingRight,
            height = totalHeight
        )
    }
    
    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        var childIndex = 0
        
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            if (child.visibility == GONE) continue
            
            if (childIndex < childBounds.size) {
                val bounds = childBounds[childIndex]
                child.layout(bounds.left, bounds.top, bounds.right, bounds.bottom)
                childIndex++
            }
        }
    }
    
    override fun generateLayoutParams(attrs: AttributeSet?): LayoutParams {
        return MarginLayoutParams(context, attrs)
    }
    
    override fun generateDefaultLayoutParams(): LayoutParams {
        return MarginLayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT)
    }
    
    override fun generateLayoutParams(p: LayoutParams?): LayoutParams {
        return MarginLayoutParams(p)
    }
    
    override fun checkLayoutParams(p: LayoutParams?): Boolean {
        return p is MarginLayoutParams
    }
    
    private data class LayoutResult(
        val width: Int,
        val height: Int
    )
    
    private val Int.dp: Int
        get() = (this * resources.displayMetrics.density).toInt()
}
```

## 性能优化

### 绘制优化

```kotlin
class OptimizedCustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    // 缓存绘制对象
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val path = Path()
    private val rect = RectF()
    
    // 缓存计算结果
    private var cachedWidth = 0
    private var cachedHeight = 0
    private var cachedBitmap: Bitmap? = null
    private var cachedCanvas: Canvas? = null
    
    // 脏区域标记
    private var isDirty = true
    
    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        super.onSizeChanged(w, h, oldw, oldh)
        
        if (w != cachedWidth || h != cachedHeight) {
            cachedWidth = w
            cachedHeight = h
            
            // 重新创建缓存位图
            cachedBitmap?.recycle()
            cachedBitmap = Bitmap.createBitmap(w, h, Bitmap.Config.ARGB_8888)
            cachedCanvas = Canvas(cachedBitmap!!)
            
            isDirty = true
        }
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 使用缓存位图
        if (isDirty) {
            drawToCache()
            isDirty = false
        }
        
        cachedBitmap?.let { bitmap ->
            canvas.drawBitmap(bitmap, 0f, 0f, null)
        }
    }
    
    private fun drawToCache() {
        cachedCanvas?.let { canvas ->
            // 清除之前的内容
            canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR)
            
            // 绘制复杂内容到缓存
            drawComplexContent(canvas)
        }
    }
    
    private fun drawComplexContent(canvas: Canvas) {
        // 复杂的绘制逻辑
        paint.color = Color.BLUE
        
        // 避免在 onDraw 中创建对象
        rect.set(50f, 50f, width - 50f, height - 50f)
        canvas.drawRoundRect(rect, 20f, 20f, paint)
        
        // 使用路径缓存
        path.reset()
        path.moveTo(width / 2f, 100f)
        path.lineTo(width - 100f, height - 100f)
        path.lineTo(100f, height - 100f)
        path.close()
        
        paint.color = Color.RED
        canvas.drawPath(path, paint)
    }
    
    // 提供方法标记需要重绘
    fun markDirty() {
        isDirty = true
        invalidate()
    }
    
    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        // 释放缓存资源
        cachedBitmap?.recycle()
        cachedBitmap = null
        cachedCanvas = null
    }
}
```

### 内存优化

```kotlin
class MemoryOptimizedView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    
    companion object {
        // 使用对象池避免频繁创建对象
        private val rectPool = Pools.SimplePool<RectF>(10)
        private val paintPool = Pools.SimplePool<Paint>(5)
        
        private fun acquireRect(): RectF {
            return rectPool.acquire() ?: RectF()
        }
        
        private fun releaseRect(rect: RectF) {
            rect.setEmpty()
            rectPool.release(rect)
        }
        
        private fun acquirePaint(): Paint {
            return paintPool.acquire() ?: Paint(Paint.ANTI_ALIAS_FLAG)
        }
        
        private fun releasePaint(paint: Paint) {
            paint.reset()
            paintPool.release(paint)
        }
    }
    
    // 使用弱引用避免内存泄漏
    private var listenerRef: WeakReference<OnCustomEventListener>? = null
    
    interface OnCustomEventListener {
        fun onCustomEvent()
    }
    
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 使用对象池
        val rect = acquireRect()
        val paint = acquirePaint()
        
        try {
            rect.set(0f, 0f, width.toFloat(), height.toFloat())
            paint.color = Color.BLUE
            canvas.drawRect(rect, paint)
        } finally {
            // 确保释放对象
            releaseRect(rect)
            releasePaint(paint)
        }
    }
    
    fun setOnCustomEventListener(listener: OnCustomEventListener?) {
        listenerRef = listener?.let { WeakReference(it) }
    }
    
    private fun notifyCustomEvent() {
        listenerRef?.get()?.onCustomEvent()
    }
    
    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        // 清理引用
        listenerRef?.clear()
        listenerRef = null
    }
}
```

## 总结

自定义 View 开发是 Android 开发中的高级技能，通过本文的深入探讨，我们学习了：

1. **基础概念**：View 的绘制流程、测量、布局和绘制
2. **自定义属性**：如何定义和使用自定义属性
3. **复杂效果**：波浪动画、雷达扫描等高级视觉效果
4. **触摸处理**：手势识别、拖拽、缩放等交互功能
5. **ViewGroup**：自定义布局管理器的实现
6. **性能优化**：绘制优化、内存管理等最佳实践

掌握这些技能后，开发者可以创建出功能强大、性能优秀的自定义 UI 组件，为用户提供独特而流畅的交互体验。在实际开发中，应该根据具体需求选择合适的实现方案，并始终关注性能和用户体验。