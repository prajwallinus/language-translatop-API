# language-translatop-API

## Overview
This guide shows how to build a **scalable, feature-rich Language Translator API** and a companion **Android Studio client**. It covers architecture options (on-device and cloud), a clear REST API spec, a sample Node.js server (can use LibreTranslate / Google Translate / self-hosted models), and an Android client implemented in Kotlin using Retrofit. The design focuses on wide translation features, performance, security, and extensibility.

## Key features

* Text translation (single/multi-sentence)
* Batch translation (multiple texts at once)
* Language detection
* Auto language detection + explicit source language
* Transliteration (where supported)
* Speech-to-text (STT) and Text-to-speech (TTS) integration
* Glossary & custom dictionary support
* Formality/voice control (where supported)
* Translation memory / caching for repeated phrases
* Rate limiting, authentication (API key / JWT)
* Usage metrics & analytics endpoints

## Architecture options

1. **Cloud API (recommended for wide language coverage)**

   * Use Google Cloud Translation API, Azure Translator, or AWS Translate.
   * Pros: best accuracy, fast, managed, supports transliteration and formality in some languages.
   * Cons: cost, privacy considerations.

2. **Open-source / self-hosted (lower cost, more control)**

   * Use LibreTranslate, MarianNMT, OpenNMT, or a hosted Hugging Face Inference API.
   * Pros: privacy, customization.
   * Cons: setup complexity and compute cost for large models.

3. **On-device (mobile-first, offline)**

   * Use ML Kit or TensorFlow Lite models for limited language pairs.
   * Pros: works offline, low latency.
   * Cons: limited languages and quality.

## REST API specification (example)

Base URL: `https://api.example.com/v1`

### Authentication

* API Key: `Authorization: Bearer <API_KEY>`
* Rate limits & quotas per key

#### POST /translate

Translate a single text or batch.
Request JSON:

```json
{
  "texts": ["Hello, world!"],
  "target": "es",
  "source": "auto",     // or explicit language code like "en"
  "format": "text",     // or "html"
  "glossary_id": "optional-glossary",
  "options": {
    "formality": "formal", // optional
    "preserve_entities": true
  }
}
```

Response JSON:

```json
{
  "translations": [
    {"text": "¡Hola, mundo!", "detected_source": "en"}
  ]
}
```

#### POST /detect

Language detection.
Request JSON:

```json
{ "text": "Bonjour" }
```

Response:

```json
{ "language": "fr", "confidence": 0.98 }
```

#### POST /tts

Text-to-speech (returns URL or binary audio).
Request JSON:

```json
{ "text": "Hello", "voice": "en-US-Wavenet-D", "format": "mp3" }
```

Response:

{ "audio_url": "https://.../audio123.mp3" }


#### POST /stt

Upload audio for speech-to-text. (multipart/form-data)
Response returns detected text and language.

#### GET /languages

List supported languages and metadata (code, name, direction, transliteration support).

## Sample lightweight server (Node.js + LibreTranslate)

// server.js (Node.js + Express)
const express = require('express');
const fetch = require('node-fetch');
const rateLimit = require('express-rate-limit');
const NodeCache = require('node-cache');

const app = express();
app.use(express.json({ limit: '1mb' }));

const cache = new NodeCache({ stdTTL: 60 * 60 });

const limiter = rateLimit({ windowMs: 60*1000, max: 60 });
app.use(limiter);

// Simple API Key middleware
app.use((req, res, next) => {
  const auth = req.headers.authorization || '';
  if (!auth.startsWith('Bearer ')) return res.status(401).json({ error: 'Missing API key' });
  const key = auth.slice(7);
  // validate key (e.g., check DB)
  if (key !== process.env.DEMO_API_KEY) return res.status(403).json({ error: 'Invalid API key' });
  next();
});

app.post('/v1/translate', async (req, res) => {
  const { texts = [], target = 'en', source = 'auto', format = 'text' } = req.body;
  if (!Array.isArray(texts) || texts.length === 0) return res.status(400).json({ error: 'No texts provided' });

  const cacheKey = JSON.stringify({ texts, target, source, format });
  const cached = cache.get(cacheKey);
  if (cached) return res.json(cached);

  try {
    const response = await fetch('https://libretranslate.com/translate', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ q: texts.join('\n'), source, target, format })
    });
    const data = await response.json();
    // LibreTranslate returns single translation; split lines back for batch
    const translations = data.translatedText.split('\n').map(t => ({ text: t }));
    const result = { translations };
    cache.set(cacheKey, result);
    res.json(result);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Translation provider error' });
  }
});

app.listen(8080, () => console.log('Translator API running on :8080'));

### Dependencies (Gradle)

implementation 'com.squareup.retrofit2:retrofit:2.9.0'
implementation 'com.squareup.retrofit2:converter-moshi:2.9.0'
implementation 'com.squareup.okhttp3:logging-interceptor:4.9.3'
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.4'
implementation 'com.squareup.moshi:moshi:1.14.0'
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.6.1'
implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.6.1'
// For STT/TTS use Android's SpeechRecognizer and TextToSpeech or Google ML Kit

### Retrofit API interface

// TranslatorApi.kt
import retrofit2.http.Body
import retrofit2.http.Headers
import retrofit2.http.POST

data class TranslateRequest(
  val texts: List<String>,
  val target: String,
  val source: String = "auto",
  val format: String = "text"
)

data class TranslationResult(val translations: List<Translation>)
data class Translation(val text: String, val detected_source: String?)

interface TranslatorApi {
  @Headers("Content-Type: application/json")
  @POST("/v1/translate")
  suspend fun translate(@Body req: TranslateRequest): TranslationResult
}

### Retrofit client builder

// ApiClient.kt
import okhttp3.Interceptor
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.moshi.MoshiConverterFactory

object ApiClient {
  fun create(apiKey: String, baseUrl: String = "https://api.example.com/"): TranslatorApi {
    val logging = HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BODY }
    val auth = Interceptor { chain ->
      val req = chain.request().newBuilder()
        .addHeader("Authorization", "Bearer $apiKey")
        .build()
      chain.proceed(req)
    }

    val client = OkHttpClient.Builder()
      .addInterceptor(auth)
      .addInterceptor(logging)
      .build()

    val retrofit = Retrofit.Builder()
      .baseUrl(baseUrl)
      .client(client)
      .addConverterFactory(MoshiConverterFactory.create())
      .build()

    return retrofit.create(TranslatorApi::class.java)
  }
}
