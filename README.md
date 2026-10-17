# AudioKnyha
name: Build APK
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.9"
      - name: Create project
        run: |
          set -e
          mkdir -p proj/app/src/main/java/com/audioknyha
          cd proj
          cat > settings.gradle.kts <<'EOF'
          pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "AudioKnyha"
          include(":app")
          EOF
          cat > build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF
          cat > gradle.properties <<'EOF'
          android.useAndroidX=true
          org.gradle.jvmargs=-Xmx2g
          EOF
          cat > app/build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.audioknyha"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.audioknyha"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          dependencies {
              implementation("androidx.core:core-ktx:1.13.1")
              implementation("androidx.media:media:1.7.0")
              implementation("com.google.mlkit:translate:17.0.3")
              implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
              implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.8.1")
          }
          EOF
          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.INTERNET" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.WAKE_LOCK" />
              <queries>
                  <intent><action android:name="android.intent.action.TTS_SERVICE" /></intent>
              </queries>
              <application
                  android:label="АудіоКнига"
                  android:allowBackup="true"
                  android:theme="@android:style/Theme.DeviceDefault.Light.NoActionBar">
                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:launchMode="singleTop"
                      android:configChanges="orientation|screenSize|keyboardHidden">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
                  <service
                      android:name=".ReaderService"
                      android:exported="false"
                      android:foregroundServiceType="mediaPlayback" />
              </application>
          </manifest>
          EOF
          cat > app/src/main/java/com/audioknyha/ReaderService.kt <<'EOF'
          package com.audioknyha

          import android.app.Notification
          import android.app.NotificationChannel
          import android.app.NotificationManager
          import android.app.PendingIntent
          import android.app.Service
          import android.content.Intent
          import android.content.SharedPreferences
          import android.content.pm.ServiceInfo
          import android.media.AudioAttributes
          import android.media.AudioFocusRequest
          import android.media.AudioManager
          import android.os.Binder
          import android.os.Handler
          import android.os.IBinder
          import android.os.Looper
          import android.os.PowerManager
          import android.speech.tts.TextToSpeech
          import android.speech.tts.UtteranceProgressListener
          import android.support.v4.media.session.MediaSessionCompat
          import android.support.v4.media.session.PlaybackStateCompat
          import androidx.core.app.NotificationCompat
          import androidx.core.app.ServiceCompat
          import androidx.media.app.NotificationCompat.MediaStyle
          import com.google.mlkit.common.model.DownloadConditions
          import com.google.mlkit.nl.translate.TranslateLanguage
          import com.google.mlkit.nl.translate.Translation
          import com.google.mlkit.nl.translate.TranslatorOptions
          import kotlinx.coroutines.*
          import kotlinx.coroutines.tasks.await
          import java.io.File
          import java.util.Locale
          import kotlin.coroutines.resume

          class ReaderService : Service() {

              inner class LocalBinder : Binder() {
                  val svc: ReaderService get() = this@ReaderService
              }

              private val binder = LocalBinder()
              override fun onBind(intent: Intent?): IBinder = binder

              companion object {
                  val UK: Locale = Locale.forLanguageTag("uk-UA")
                  private val SPLIT = Regex("""[^.!?…]+(?:[.!?…]+["”’')\]]*|$)\s*""")
                  fun split(text: String): List<String> {
                      val out = ArrayList<String>()
                      for (p in text.split(Regex("""\n\s*\n"""))) {
                          val q = p.replace(Regex("""\s+"""), " ").trim()
                          if (q.isEmpty()) continue
                          for (m in SPLIT.findAll(q)) {
                              val s = m.value.trim()
                              if (s.length > 1) out.add(s)
                          }
                      }
                      return out
                  }
              }

              private val main = Handler(Looper.getMainLooper())
              private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
              private var job: Job? = null
              private var sleepJob: Job? = null
              private lateinit var tts: TextToSpeech
              private var ttsReady = false
              private var ukOk = true
              private val waiters = HashMap<String, CancellableContinuation<Unit>>()
              private lateinit var prefs: SharedPreferences
              private lateinit var session: MediaSessionCompat
              private lateinit var am: AudioManager
              private var focus: AudioFocusRequest? = null
              private var wake: PowerManager.WakeLock? = null
              private var posted = false

              var sentences: List<String> = emptyList()
                  private set
              var title = ""
                  private set
              var idx = 0
                  private set
              var playing = false
                  private set
              var ukText = ""
                  private set
              var status = ""
                  private set
              var onChange: (() -> Unit)? = null

              var mode = "eu"
              var rateEn = 1f
              var rateUk = 1f
              var gapMs = 300L

              private val cache = HashMap<Int, String>()
              private val translator = Translation.getClient(
                  TranslatorOptions.Builder()
                      .setSourceLanguage(TranslateLanguage.ENGLISH)
                      .setTargetLanguage(TranslateLanguage.UKRAINIAN)
                      .build()
              )
              private var modelReady = false

              override fun onCreate() {
                  super.onCreate()
                  prefs = getSharedPreferences("p", MODE_PRIVATE)
                  am = getSystemService(AUDIO_SERVICE) as AudioManager
                  mode = prefs.getString("mode", "eu") ?: "eu"
                  rateEn = prefs.getFloat("rateEn", 1f)
                  rateUk = prefs.getFloat("rateUk", 1f)
                  gapMs = prefs.getLong("gap", 300L)
                  getSystemService(NotificationManager::class.java)
                      .createNotificationChannel(NotificationChannel("read", "Читання", NotificationManager.IMPORTANCE_LOW))
                  session = MediaSessionCompat(this, "AudioKnyha").apply {
                      setCallback(object : MediaSessionCompat.Callback() {
                          override fun onPlay() { this@ReaderService.play() }
                          override fun onPause() { this@ReaderService.pause() }
                          override fun onSkipToNext() { this@ReaderService.jump(idx + 1) }
                          override fun onSkipToPrevious() { this@ReaderService.jump(idx - 1) }
                      })
                      isActive = true
                  }
                  tts = TextToSpeech(this) { st ->
                      ttsReady = st == TextToSpeech.SUCCESS
                      if (ttsReady) {
                          tts.setAudioAttributes(
                              AudioAttributes.Builder()
                                  .setUsage(AudioAttributes.USAGE_MEDIA)
                                  .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH).build()
                          )
                          tts.setOnUtteranceProgressListener(object : UtteranceProgressListener() {
                              override fun onStart(id: String?) {}
                              override fun onDone(id: String?) { finishUtt(id) }
                              @Deprecated("Deprecated in Java")
                              override fun onError(id: String?) { finishUtt(id) }
                              override fun onError(id: String?, code: Int) { finishUtt(id) }
                          })
                          ukOk = tts.isLanguageAvailable(UK) >= TextToSpeech.LANG_AVAILABLE
                          if (!ukOk) status = "Немає українського голосу TTS. Відкрийте «Голос Android» і встановіть українську."
                          changed()
                      }
                  }
                  restore()
                  scope.launch { ensureModel(); changed() }
              }

              // ---------- Книга ----------
              fun setBook(text: String, list: List<String>, name: String) {
                  pause()
                  sentences = list; title = name; idx = 0; ukText = ""; cache.clear()
                  File(filesDir, "book.txt").writeText(text)
                  prefs.edit().putString("title", name).putInt("idx", 0).apply()
                  changed()
              }

              private fun restore() {
                  val f = File(filesDir, "book.txt")
                  if (f.exists()) {
                      sentences = split(f.readText())
                      title = prefs.getString("title", "") ?: ""
                      idx = prefs.getInt("idx", 0).coerceIn(0, (sentences.size - 1).coerceAtLeast(0))
                  }
              }

              fun saveSettings() {
                  prefs.edit().putString("mode", mode).putFloat("rateEn", rateEn)
                      .putFloat("rateUk", rateUk).putLong("gap", gapMs).apply()
              }

              // ---------- Переклад (офлайн, ML Kit) ----------
              private suspend fun ensureModel(): Boolean {
                  if (modelReady) return true
                  status = "Завантаження мовної моделі (потрібен інтернет один раз)…"; changed()
                  return try {
                      translator.downloadModelIfNeeded(DownloadConditions.Builder().build()).await()
                      modelReady = true
                      status = "Переклад працює офлайн ✔"
                      true
                  } catch (e: CancellationException) {
                      throw e
                  } catch (e: Exception) {
                      status = "Модель перекладу ще не завантажена. Підключіться до інтернету."
                      false
                  }
              }

              private suspend fun tr(i: Int): String? {
                  cache[i]?.let { return it }
                  if (!modelReady || i !in sentences.indices) return null
                  return try {
                      translator.translate(sentences[i]).await().also { cache[i] = it }
                  } catch (e: CancellationException) {
                      throw e
                  } catch (e: Exception) {
                      null
                  }
              }

              // ---------- Озвучення ----------
              private suspend fun speak(text: String, loc: Locale, rate: Float) =
                  suspendCancellableCoroutine<Unit> { c ->
                      val id = System.nanoTime().toString()
                      waiters[id] = c
                      c.invokeOnCancellation { waiters.remove(id); tts.stop() }
                      tts.setLanguage(loc)
                      tts.setSpeechRate(rate)
                      val r = tts.speak(text, TextToSpeech.QUEUE_FLUSH, null, id)
                      if (r != TextToSpeech.SUCCESS) { waiters.remove(id); c.resume(Unit) }
                  }

              private fun finishUtt(id: String?) {
                  main.post { waiters.remove(id)?.let { if (it.isActive) it.resume(Unit) } }
              }

              private suspend fun say(text: String, loc: Locale, rate: Float) {
                  if (!ttsReady || (loc == UK && !ukOk)) { delay((text.length * 70L).coerceAtMost(8000L)); return }
                  speak(text, loc, rate)
              }

              // ---------- Керування ----------
              fun play() {
                  if (sentences.isEmpty() || playing) return
                  playing = true; posted = true
                  requestFocus()
                  wake = (getSystemService(POWER_SERVICE) as PowerManager)
                      .newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "audioknyha:read").apply { acquire(6 * 60 * 60 * 1000L) }
                  ServiceCompat.startForeground(this, 1, buildNotification(), ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK)
                  startLoop()
                  changed()
              }

              fun pause() {
                  job?.cancel(); job = null
                  if (ttsReady) tts.stop()
                  val was = playing
                  playing = false
                  wake?.let { if (it.isHeld) it.release() }; wake = null
                  focus?.let { am.abandonAudioFocusRequest(it) }; focus = null
                  if (was) ServiceCompat.stopForeground(this, ServiceCompat.STOP_FOREGROUND_DETACH)
                  changed()
              }

              fun jump(i: Int) {
                  if (sentences.isEmpty()) return
                  idx = i.coerceIn(0, sentences.size - 1); ukText = ""
                  prefs.edit().putInt("idx", idx).apply()
                  if (playing) { job?.cancel(); if (ttsReady) tts.stop(); startLoop() }
                  changed()
              }

              fun setSleep(minutes: Int) {
                  sleepJob?.cancel()
                  if (minutes > 0) sleepJob = scope.launch { delay(minutes * 60_000L); pause() }
              }

              private fun startLoop() {
                  job?.cancel()
                  job = scope.launch {
                      ensureModel()
                      while (isActive && idx < sentences.size) {
                          ukText = ""; changed()
                          prefs.edit().putInt("idx", idx).apply()
                          for (k in 1..3) { val n = idx + k; launch { tr(n) } }
                          for (s in mode) {
                              if (s == 'e') {
                                  say(sentences[idx], Locale.US, rateEn)
                              } else {
                                  val t = tr(idx)
                                  if (t != null) {
                                      ukText = t; changed(); say(t, UK, rateUk)
                                  } else if (mode == "u") {
                                      ukText = "(переклад недоступний — читаю оригінал)"; changed()
                                      say(sentences[idx], Locale.US, rateEn)
                                  }
                              }
                          }
                          delay(gapMs)
                          idx++
                      }
                      if (idx >= sentences.size) { idx = (sentences.size - 1).coerceAtLeast(0); pause() }
                  }
              }

              private fun requestFocus() {
                  val req = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN)
                      .setAudioAttributes(AudioAttributes.Builder().setUsage(AudioAttributes.USAGE_MEDIA)
                          .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH).build())
                      .setOnAudioFocusChangeListener { ch ->
                          if (ch == AudioManager.AUDIOFOCUS_LOSS || ch == AudioManager.AUDIOFOCUS_LOSS_TRANSIENT) main.post { pause() }
                      }.build()
                  focus = req
                  am.requestAudioFocus(req)
              }

              // ---------- Сповіщення та сесія ----------
              private fun act(a: String, icon: Int, label: String) = NotificationCompat.Action(
                  icon, label,
                  PendingIntent.getService(
                      this, a.hashCode(), Intent(this, ReaderService::class.java).setAction(a),
                      PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
                  )
              )

              private fun buildNotification(): Notification {
                  val open = PendingIntent.getActivity(this, 0, Intent(this, MainActivity::class.java), PendingIntent.FLAG_IMMUTABLE)
                  return NotificationCompat.Builder(this, "read")
                      .setSmallIcon(android.R.drawable.ic_media_play)
                      .setContentTitle(title.ifEmpty { "АудіоКнига" })
                      .setContentText(sentences.getOrNull(idx) ?: "")
                      .setContentIntent(open)
                      .setOngoing(playing)
                      .setOnlyAlertOnce(true)
                      .setVisibility(NotificationCompat.VISIBILITY_PUBLIC)
                      .addAction(act("prev", android.R.drawable.ic_media_previous, "Назад"))
                      .addAction(act("toggle",
                          if (playing) android.R.drawable.ic_media_pause else android.R.drawable.ic_media_play,
                          if (playing) "Пауза" else "Читати"))
                      .addAction(act("next", android.R.drawable.ic_media_next, "Далі"))
                      .setStyle(MediaStyle().setMediaSession(session.sessionToken).setShowActionsInCompactView(0, 1, 2))
                      .build()
              }

              private fun changed() {
                  session.setPlaybackState(
                      PlaybackStateCompat.Builder()
                          .setActions(PlaybackStateCompat.ACTION_PLAY or PlaybackStateCompat.ACTION_PAUSE or
                              PlaybackStateCompat.ACTION_PLAY_PAUSE or PlaybackStateCompat.ACTION_SKIP_TO_NEXT or
                              PlaybackStateCompat.ACTION_SKIP_TO_PREVIOUS)
                          .setState(if (playing) PlaybackStateCompat.STATE_PLAYING else PlaybackStateCompat.STATE_PAUSED,
                              PlaybackStateCompat.PLAYBACK_POSITION_UNKNOWN, 1f)
                          .build()
                  )
                  if (posted) getSystemService(NotificationManager::class.java).notify(1, buildNotification())
                  onChange?.invoke()
              }

              override fun onStartCommand(i: Intent?, flags: Int, startId: Int): Int {
                  when (i?.action) {
                      "toggle" -> if (playing) pause() else play()
                      "next" -> jump(idx + 1)
                      "prev" -> jump(idx - 1)
                  }
                  return START_NOT_STICKY
              }

              override fun onDestroy() {
                  scope.cancel()
                  wake?.let { if (it.isHeld) it.release() }
                  if (ttsReady) tts.shutdown()
                  translator.close()
                  session.release()
                  super.onDestroy()
              }
          }
          EOF
          cat > app/src/main/java/com/audioknyha/MainActivity.kt <<'EOF'
          package com.audioknyha

          import android.Manifest
          import android.app.Activity
          import android.content.ComponentName
          import android.content.Intent
          import android.content.ServiceConnection
          import android.content.pm.PackageManager
          import android.graphics.Color
          import android.os.Build
          import android.os.Bundle
          import android.os.IBinder
          import android.provider.OpenableColumns
          import android.text.Html
          import android.view.View
          import android.widget.*
          import java.util.zip.ZipInputStream

          class MainActivity : Activity() {
              private var svc: ReaderService? = null
              private lateinit var enV: TextView
              private lateinit var ukV: TextView
              private lateinit var infoV: TextView
              private lateinit var statV: TextView
              private lateinit var playB: Button
              private lateinit var seek: SeekBar
              private var seeking = false

              private val conn = object : ServiceConnection {
                  override fun onServiceConnected(n: ComponentName?, b: IBinder?) {
                      svc = (b as ReaderService.LocalBinder).svc
                      svc?.onChange = { render() }
                      render()
                  }
                  override fun onServiceDisconnected(n: ComponentName?) { svc = null }
              }

              private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()

              override fun onCreate(s: Bundle?) {
                  super.onCreate(s)
                  buildUi()
                  if (Build.VERSION.SDK_INT >= 33 &&
                      checkSelfPermission(Manifest.permission.POST_NOTIFICATIONS) != PackageManager.PERMISSION_GRANTED)
                      requestPermissions(arrayOf(Manifest.permission.POST_NOTIFICATIONS), 1)
                  val i = Intent(this, ReaderService::class.java)
                  startService(i)
                  bindService(i, conn, BIND_AUTO_CREATE)
              }

              private fun btn(t: String, f: () -> Unit) = Button(this).apply { text = t; setOnClickListener { f() } }

              private fun row(vararg v: View) = LinearLayout(this).apply {
                  orientation = LinearLayout.HORIZONTAL
                  v.forEach { addView(it, LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1f)) }
              }

              private fun slider(label: String, max: Int, init: Int, fmt: (Int) -> String, on: (Int) -> Unit): View {
                  val box = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setPadding(0, dp(8), 0, 0) }
                  val tv = TextView(this).apply { text = label + fmt(init) }
                  val sb = SeekBar(this).apply {
                      this.max = max; progress = init
                      setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
                          override fun onProgressChanged(s: SeekBar?, p: Int, u: Boolean) { tv.text = label + fmt(p); if (u) on(p) }
                          override fun onStartTrackingTouch(s: SeekBar?) {}
                          override fun onStopTrackingTouch(s: SeekBar?) {}
                      })
                  }
                  box.addView(tv); box.addView(sb)
                  return box
              }

              private fun spinner(items: List<String>, sel: Int, on: (Int) -> Unit) = Spinner(this).apply {
                  adapter = ArrayAdapter(this@MainActivity, android.R.layout.simple_spinner_dropdown_item, items)
                  setSelection(sel)
                  onItemSelectedListener = object : AdapterView.OnItemSelectedListener {
                      override fun onItemSelected(p: AdapterView<*>?, v: View?, pos: Int, id: Long) { on(pos) }
                      override fun onNothingSelected(p: AdapterView<*>?) {}
                  }
              }

              private fun buildUi() {
                  val prefs = getSharedPreferences("p", MODE_PRIVATE)
                  val root = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setPadding(dp(14), dp(24), dp(14), dp(14)) }
                  setContentView(ScrollView(this).apply { addView(root) })

                  root.addView(TextView(this).apply { text = "📖 АудіоКнига EN → UK"; textSize = 20f })
                  root.addView(row(btn("Відкрити книгу") { openBook() }, btn("Голос Android") { ttsSettings() }))
                  infoV = TextView(this).apply { setTextColor(Color.GRAY) }; root.addView(infoV)
                  enV = TextView(this).apply { textSize = 19f; setPadding(0, dp(12), 0, 0) }; root.addView(enV)
                  ukV = TextView(this).apply { textSize = 19f; setTextColor(Color.rgb(180, 87, 43)); setPadding(0, dp(8), 0, dp(8)) }; root.addView(ukV)
                  playB = btn("▶ Читати") { svc?.let { if (it.playing) it.pause() else it.play() } }
                  root.addView(row(btn("⏮") { svc?.let { it.jump(it.idx - 1) } }, playB, btn("⏭") { svc?.let { it.jump(it.idx + 1) } }))
                  seek = SeekBar(this).apply {
                      setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
                          override fun onProgressChanged(s: SeekBar?, p: Int, u: Boolean) {}
                          override fun onStartTrackingTouch(s: SeekBar?) { seeking = true }
                          override fun onStopTrackingTouch(s: SeekBar?) { seeking = false; svc?.jump(progress) }
                      })
                  }
                  root.addView(seek)

                  val modes = listOf("eu", "u", "e", "ue")
                  root.addView(TextView(this).apply { text = "Режим читання"; setPadding(0, dp(12), 0, 0) })
                  root.addView(spinner(
                      listOf("Англійською, потім українською", "Лише українською", "Лише англійською", "Українською, потім англійською"),
                      modes.indexOf(prefs.getString("mode", "eu"))
                  ) { svc?.mode = modes[it]; svc?.saveSettings() })

                  root.addView(slider("Швидкість англ.: ", 150, ((prefs.getFloat("rateEn", 1f) - 0.5f) * 100).toInt(),
                      { "%.2f".format(0.5f + it / 100f) }) { svc?.rateEn = 0.5f + it / 100f; svc?.saveSettings() })
                  root.addView(slider("Швидкість укр.: ", 150, ((prefs.getFloat("rateUk", 1f) - 0.5f) * 100).toInt(),
                      { "%.2f".format(0.5f + it / 100f) }) { svc?.rateUk = 0.5f + it / 100f; svc?.saveSettings() })
                  root.addView(slider("Пауза між реченнями: ", 30, (prefs.getLong("gap", 300L) / 100).toInt(),
                      { "%.1f с".format(it / 10f) }) { svc?.gapMs = it * 100L; svc?.saveSettings() })

                  val mins = listOf(0, 15, 30, 60)
                  root.addView(TextView(this).apply { text = "Таймер сну"; setPadding(0, dp(12), 0, 0) })
                  root.addView(spinner(listOf("Вимкнено", "15 хв", "30 хв", "60 хв"), 0) { svc?.setSleep(mins[it]) })

                  statV = TextView(this).apply { setTextColor(Color.GRAY); setPadding(0, dp(12), 0, 0) }; root.addView(statV)
              }

              private fun render() {
                  val s = svc ?: return
                  enV.text = s.sentences.getOrNull(s.idx) ?: "Відкрийте книгу (.txt, .epub або .html)"
                  ukV.text = s.ukText
                  infoV.text = if (s.sentences.isEmpty()) "Книгу не завантажено." else "«${s.title}» — ${s.idx + 1} / ${s.sentences.size}"
                  playB.text = if (s.playing) "⏸ Пауза" else "▶ Читати"
                  statV.text = s.status
                  if (!seeking) { seek.max = (s.sentences.size - 1).coerceAtLeast(0); seek.progress = s.idx }
              }

              private fun ttsSettings() {
                  try { startActivity(Intent("com.android.settings.TTS_SETTINGS")) } catch (e: Exception) {
                      Toast.makeText(this, "Налаштування → Система → Мова → Синтез мовлення", Toast.LENGTH_LONG).show()
                  }
              }

              private fun openBook() {
                  val i = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
                      addCategory(Intent.CATEGORY_OPENABLE); type = "*/*"
                      putExtra(Intent.EXTRA_MIME_TYPES, arrayOf("text/plain", "application/epub+zip", "text/html"))
                  }
                  @Suppress("DEPRECATION") startActivityForResult(i, 7)
              }

              @Suppress("DEPRECATION")
              override fun onActivityResult(req: Int, res: Int, data: Intent?) {
                  super.onActivityResult(req, res, data)
                  val uri = data?.data ?: return
                  if (req != 7 || res != RESULT_OK) return
                  Thread {
                      try {
                          var name = "book"
                          contentResolver.query(uri, null, null, null, null)?.use { c ->
                              if (c.moveToFirst()) { val k = c.getColumnIndex(OpenableColumns.DISPLAY_NAME); if (k >= 0) name = c.getString(k) }
                          }
                          val stream = contentResolver.openInputStream(uri) ?: return@Thread
                          val text = stream.use { ins ->
                              when {
                                  name.endsWith(".epub", true) -> epubText(ins)
                                  name.endsWith(".html", true) || name.endsWith(".htm", true) ->
                                      Html.fromHtml(String(ins.readBytes(), Charsets.UTF_8), Html.FROM_HTML_MODE_COMPACT).toString()
                                  else -> String(ins.readBytes(), Charsets.UTF_8)
                              }
                          }
                          val list = ReaderService.split(text)
                          runOnUiThread { svc?.setBook(text, list, name.substringBeforeLast('.')) }
                      } catch (e: Exception) {
                          runOnUiThread { Toast.makeText(this, "Не вдалося відкрити файл", Toast.LENGTH_LONG).show() }
                      }
                  }.start()
              }

              private fun epubText(ins: java.io.InputStream): String {
                  val parts = sortedMapOf<String, String>()
                  ZipInputStream(ins).use { z ->
                      var e = z.nextEntry
                      while (e != null) {
                          if (Regex(""".*\.(x?html?)$""", RegexOption.IGNORE_CASE).matches(e.name))
                              parts[e.name] = String(z.readBytes(), Charsets.UTF_8)
                          e = z.nextEntry
                      }
                  }
                  return parts.values.joinToString("\n\n") { Html.fromHtml(it, Html.FROM_HTML_MODE_COMPACT).toString() }
              }

              override fun onDestroy() {
                  svc?.onChange = null
                  val playing = svc?.playing == true
                  unbindService(conn)
                  if (isFinishing && !playing) stopService(Intent(this, ReaderService::class.java))
                  super.onDestroy()
              }
          }
          EOF
      - name: Build APK
        working-directory: proj
        run: gradle assembleDebug --no-daemon
      - name: Publish APK
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          cp proj/app/build/outputs/apk/debug/app-debug.apk AudioKnyha.apk
          gh release create "v${{ github.run_number }}" AudioKnyha.apk --repo "${{ github.repository }}" --title "АудіоКнига, збірка ${{ github.run_number }}" --notes "Завантажте файл AudioKnyha.apk"
