# MJPEG_INGO_DOC# MJPEG player

### Отображение кадров

Для отображения mjpeg видео в реальном времени, нужно написать свой плеер, который будет читать данные из
http-соединения и рендерить их на поверхности.
Можно просмотреть библиотеки, которые уже реализуют эту функциональность,
например, [mjpeg-view-android-kotlin](https://github.com/faizkhan12/mjpeg-view-android-kotlin)
или [ipcam-view](https://github.com/niqdev/ipcam-view).

### Воспроизведение аудио

Камера может отображать звук, следуя из документации, стрим данных сочетает очередность кадра и аудио.

> "Флаг получения видео - и аудиоданных канала в рамках одного соединения.
> Возможные значения: `on` – включить получение звука, `off` – выключить.
> При `sound=on` сервер возвращает звуковые кадры в формате `G.711U`, чередуя их в потоке данных с видеокадрами.
> Опциональный параметр"

Для трансляции аудио с такого стрима, нужно разделить звуковые и видеокадры, которые чередуются в потоке данных, и
декодировать звук из формата `G.711U` в `PCM`.
Затем мы можем воспроизводить `PCM-данные` с помощью `AudioTrack` или другого аудио-плеера.

Для декодирования `G.711U` в `PCM` можно использовать
библиотеку [android-g711-codec](https://github.com/theeasiestway/android-g711-codec), которая предоставляет простой
интерфейс для кодирования и декодирования `G.711A` и `G.711U`.
Или можно посмотреть на [этот ответ](https://stackoverflow.com/questions/3251782/decode-g711pcm-u-law) на Stack
Overflow, где объясняется алгоритм декодирования `G.711`

### Извлечение аудио и медиа байтов

Чередование аудио и видео кадров в одном потоке данных зависит от формата и кодека, которые используются для передачи
медиа.

В случае с `mjpeg` и `G.711U`, можно предположить, что каждый видеокадр начинается с **маркера JPEG** (`0xFFD8`) и
заканчивается **маркером конца JPEG** (`0xFFD9`).
Аудиокадры имеют фиксированный размер, например, `160 байт`, и идут после видеокадра.

Таким образом, можно читать данные из потока по байтам и определять, когда начинается и заканчивается видеокадр,
а также сколько байтов нужно пропустить, чтобы добраться до следующего видеокадра.

#### Пример получения байтов из потока данных

```Kotlin
// Предполагаем, что у нас есть InputStream is, который читает данные из MJPEG-потока
// Предполагаем, что у нас есть SurfaceView surfaceView, на котором мы хотим отобразить видео
// Предполагаем, что у нас есть AudioTrack audioTrack, который воспроизводит PCM-данные

// Предполагаем, что размер аудиокадра равен 160 байтам
val audioFrameSize = 160

// Предполагаем, что мы знаем, сколько кадров мы хотим прочитать
val frameCount = 100

// Создаем буфер для хранения байтов
val buffer = ByteArray(1024)

// Создаем переменные для хранения текущего состояния
var isVideoFrame = false // флаг, указывающий, читаем ли мы видеокадр или аудиокадр
var videoFrameLength = 0 // длина текущего видеокадра в байтах
var audioFrameLength = 0 // длина текущего аудиокадра в байтах
var bytesRead = 0 // количество прочитанных байтов из потока
var framesRead = 0 // количество прочитанных кадров из потока

// Создаем объект для декодирования G.711U в PCM
val g711Decoder = G711UCodec()

// Создаем объект для отображения MJPEG-видео на поверхности
val mjpegView = MjpegView(surfaceView.context)

// Добавляем виджет на поверхность
surfaceView.addView(mjpegView)

// Создаем поток для работы с видео
val videoThread = Thread {

  // Создаем выходной поток для видео
  val osVideo = ByteArrayOutputStream()
  
  // Читаем данные из потока, пока не достигнем конца потока или не прочитаем нужное количество кадров
  while (bytesRead != -1 && framesRead < frameCount) {
  
    // Читаем байты из потока в буфер
    bytesRead = is.read(buffer)
    
    // Проходим по буферу по байтам
    for (i in 0 until bytesRead) {
    
      // Если мы читаем видеокадр
      if (isVideoFrame) {
      
        // Пишем текущий байт в выходной поток видео
        osVideo.write(buffer[i])
        
        // Увеличиваем длину видеокадра на 1
        videoFrameLength++
        
        // Проверяем, не закончился ли видеокадр
        if (buffer[i] == 0xD9.toByte() && i > 0 && buffer[i-1] == 0xFF.toByte()) {
        
          // Видеокадр закончился, переключаем флаг
          isVideoFrame = false
          
          // Сбрасываем длину видеокадра
          videoFrameLength = 0
          
          // Увеличиваем количество прочитанных кадров на 1
          framesRead++
          
          // Отображаем видеокадр на поверхности с помощью виджета
          mjpegView.setSource(osVideo.toByteArray())
          
          // Очищаем выходной поток видео
          osVideo.reset()
        }
      } else {
      
        // Мы читаем аудиокадр
        // Увеличиваем длину аудиокадра на 1
        audioFrameLength++
        
        // Проверяем, не закончился ли аудиокадр
        if (audioFrameLength == audioFrameSize) {
        
          // Аудиокадр закончился, переключаем флаг
          isVideoFrame = true
          
          // Сбрасываем длину аудиокадра
          audioFrameLength = 0
        }
      }
      
      // Проверяем, не начался ли новый видеокадр
      if (buffer[i] == 0xD8.toByte() && i > 0 && buffer[i-1] == 0xFF.toByte()) {
      
        // Новый видеокадр начался, переключаем флаг
        isVideoFrame = true
      }
    }
  }
  
  // Закрываем выходной поток видео
  osVideo.close()
}

// Создаем поток для работы с аудио
val audioThread = Thread {

  // Создаем выходной поток для аудио
  val osAudio = ByteArrayOutputStream()
  
  // Читаем данные из потока, пока не достигнем конца потока или не прочитаем нужное количество кадров
  while (bytesRead != -1 && framesRead < frameCount) {
  
    // Читаем байты из потока в буфер
    bytesRead = is.read(buffer)
    
    // Проходим по буферу по байтам
    for (i in 0 until bytesRead) {
    
      // Если мы читаем видеокадр
      if (isVideoFrame) {
      
        // Увеличиваем длину видеокадра на 1
        videoFrameLength++
        
        // Проверяем, не закончился ли видеокадр
        if (buffer[i] == 0xD9.toByte() && i > 0 && buffer[i-1] == 0xFF.toByte()) {
        
          // Видеокадр закончился, переключаем флаг
          isVideoFrame = false
          
          // Сбрасываем длину видеокадра
          videoFrameLength = 0
          
          // Увеличиваем количество прочитанных кадров на 1
          framesRead++
        }
      } else {
      
        // Мы читаем аудиокадр
        // Пишем текущий байт в выходной поток аудио
        osAudio.write(buffer[i])
        
        // Увеличиваем длину аудиокадра на 1
        audioFrameLength++
        
        // Проверяем, не закончился ли аудиокадр
        if (audioFrameLength == audioFrameSize) {
        
          // Аудиокадр закончился, переключаем флаг
          isVideoFrame = true
          
          // Декодируем аудиокадр из G.711U в PCM
          val pcmFrame = g711Decoder.decode(osAudio.toByteArray(), 0, audioFrameSize)
          
          // Воспроизводим PCM-данные с помощью AudioTrack
          audioTrack.write(pcmFrame, 0, pcmFrame.size)
          
          // Очищаем выходной поток аудио
          osAudio.reset()
        }
      }
      
      // Проверяем, не начался ли новый видеокадр
      if (buffer[i] == 0xD8.toByte() && i > 0 && buffer[i-1] == 0xFF.toByte()) {
      
        // Новый видеокадр начался, переключаем флаг
        isVideoFrame = true
      }
    }
  }
  
  // Закрываем выходной поток аудио
  osAudio.close()
}

// Запускаем потоки
videoThread.start()
audioThread.start()

// Ждем, пока потоки завершатся
videoThread.join()
audioThread.join()

// Закрываем входной поток
is.close()
```


### Полезные ссылки

- [Android-Simple-MjpegViewer](https://github.com/dydwo92/Android-Simple-MjpegViewer)
- [mjpeg-view-android-kotlin](https://github.com/faizkhan12/mjpeg-view-android-kotlin)
- [ipcam-view(MjpegViewDefault)](https://github.com/niqdev/ipcam-view/blob/master/mjpeg-view/src/main/java/com/github/niqdev/mjpeg/MjpegViewDefault.java)
