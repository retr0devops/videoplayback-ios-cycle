# Цикл воспроизведения видеофайла на iOS с AVFoundation и VideoToolbox

В этом техническом обзоре рассматривается полный цикл низкоуровневого воспроизведения видео на iOS с использованием фреймворков AVFoundation и VideoToolbox. Мы пройдем последовательно все этапы: от чтения медиаданных из файла до декодирования видео/аудио и синхронного вывода. Будут описаны ключевые классы и структуры данных, API, а также приведены фрагменты кода для иллюстрации.

## 1. Чтение видеофайла с помощью AVAsset и AVAssetReader

Первый шаг – загрузить видеофайл и подготовить механизм для чтения его содержимого. В AVFoundation основной класс для представления медиа-файла – `AVAsset`. Он инкапсулирует информацию о ресурсах (видео, аудио, метаданные и т.д.) и предоставляет доступ к его дорожкам. Для получения `AVAsset` обычно используют URL файла:

```swift
let url = URL(fileURLWithPath: "path/to/video.mp4")
let asset = AVAsset(url: url)
```

Загрузив ассет, мы не можем напрямую извлечь из него сырые выборки данных (семплы). Для пошагового чтения медиа-выборок используется класс **`AVAssetReader`**. Он связывается с `AVAsset` и позволяет последовательно читать закодированные сэмплы с дорожек. Создание reader’а происходит так:

```swift
guard let reader = try? AVAssetReader(asset: asset) else {
    fatalError("Failed to create AVAssetReader")
}
```

После инициализации `AVAssetReader` находится в состоянии подготовки. Прежде чем начать чтение (`startReading()`), ему нужно указать, *что* именно читать – т.е. добавить соответствующие outputs (выходы) для видеодорожки и аудиодорожки (о них далее). Только после этого reader можно запустить. Если ассет содержит сложные композиции (несколько сегментов, эффекты), `AVAssetReader` позаботится о склеивании согласно таймлайну.

**Примечание:** `AVAssetReader` читает контент с той же скоростью, с которой вы запрашиваете данные. Он **не выполняет автоматического воспроизведения** – управление скоростью и синхронизацией лежит на нас. Reader можно настроить на чтение всего файла или определенного отрезка (свойство `timeRange`). Также у него есть свойство `status` для контроля состояния (в частности, `.reading`, `.completed`, `.failed` и т.д.).

## 2. Извлечение видео- и аудио треков из ассета

Медиафайл (например, `.mp4`) обычно содержит одну видеодорожку и одну аудиодорожку. В `AVAsset` дорожки представлены объектами класса `AVAssetTrack`. Чтобы получить треки, нужно запросить их у ассета по типу медиа:

```swift
guard let videoTrack = asset.tracks(withMediaType: .video).first,
      let audioTrack = asset.tracks(withMediaType: .audio).first else {
    fatalError("Не найдены необходимыe дорожки")
}
```

Метод `tracks(withMediaType:)` возвращает массив дорожек указанного типа. Здесь мы берем первую видеодорожку и первую аудиодорожку (предполагается, что они есть). Каждый `AVAssetTrack` содержит параметры кодека, формата, длительности и пр. Например, у видео-трека можно получить размер кадра (`naturalSize`), частоту кадров (`nominalFrameRate`), у аудио – частоту дискретизации, число каналов и пр. через формат описания.

Имея ссылки на `AVAssetTrack` видео и аудио, переходим к настройке их чтения через `AVAssetReader`.

## 3. Настройка AVAssetReaderTrackOutput для видео- и аудио-потоков

**`AVAssetReaderTrackOutput`** – это подкласс `AVAssetReaderOutput`, который служит «выходом» данных с определенной дорожки. Именно через него `AVAssetReader` будет выдавать нам выборки (`CMSampleBuffer`) с данной дорожки. При создании `AVAssetReaderTrackOutput` важно указать параметры вывода (output settings), определяющие требуемый формат данных на выходе.

Для **видеотрека** мы можем выбрать один из двух подходов:

- **Выдавать декодированные пиксели** (RAW-формат) – тогда `AVAssetReader` сам будет декодировать видео в указанный пиксельный формат.
- **Выдавать сжатые (оригинальные) фреймы** – тогда декодирование мы будем выполнять самостоятельно через VideoToolbox.

В нашем сценарии мы хотим использовать низкоуровневый декодер VideoToolbox, поэтому оптимально настроить output на выдачу **закодированных** сэмплов, как они хранятся в файле. Для этого в `outputSettings` нужно передать `nil`. Согласно документации Apple, если `outputSettings == nil`, то дорожка будет читаться в исходном формате без декодирования ([sdks/iPhoneOS9.3.sdk/System/Library/Frameworks/AVFoundation.framework/Headers/AVAssetReaderOutput.h at master · theos/sdks · GitHub](https://github.com/theos/sdks/blob/master/iPhoneOS9.3.sdk/System/Library/Frameworks/AVFoundation.framework/Headers/AVAssetReaderOutput.h#:~:text=A%20value%20of%20nil%20for,used%20with%20the%20specified%20track)). Иными словами, `AVAssetReader` вернет CMSampleBuffer, содержащий сжатые данные кадра (например, H.264-битстрим) ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=In%20order%20to%20get%20raw,dictionary%20specifying%20a%20pixel%20format)). 

Пример настройки output для видео:

```swift
// Выход для видеодорожки (компрессированный формат)
let videoOutput = AVAssetReaderTrackOutput(track: videoTrack, outputSettings: nil)
// Отключаем копирование данных для эффективности
videoOutput.alwaysCopiesSampleData = false  // чтобы избежать лишних копий ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=with%20ProRes%204444%20content,absolutely%20must%20not%20modify%20them))
if reader.canAdd(videoOutput) {
    reader.add(videoOutput)
}
```

Здесь мы указали `outputSettings: nil`, поэтому получим сэмплы в исходном кодеке (например, H.264). Также мы установили `alwaysCopiesSampleData = false`, что отключает копирование буфера при выдаче. Это позволяет сэкономить память и время: CMSampleBuffer может ссылаться на внутренние буферы, не дублируя их ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=with%20ProRes%204444%20content,absolutely%20must%20not%20modify%20them)). **Важно:** при выключенном копировании нельзя модифицировать полученные данные, иначе возникнет непредсказуемое поведение, так как буфер может быть разделяемым.

Для **аудиотрека** мы обычно хотим получить несжатое PCM-аудио, удобное для вывода на аудиоустройство. Поэтому `outputSettings` для аудио должны задать линейный PCM формат. Apple указывает, что для аудио **обязательным** ключом является `AVFormatIDKey` со значением `kAudioFormatLinearPCM` ([AVAssetReaderTrackOutput | Apple Developer Documentation](https://developer.apple.com/documentation/avfoundation/avassetreadertrackoutput#:~:text=AVAssetReaderTrackOutput%20,For%20video%20output%20settings%2C)). Кроме него, полезно задать частоту дискретизации и число каналов, соответствующие исходному треку. Эти параметры можно узнать из форматного описания аудио (типа `CMAudioFormatDescription`). Например:

```swift
// Получаем параметры аудио (пример для иллюстрации)
let audioFormatDesc = audioTrack.formatDescriptions.first as! CMAudioFormatDescription
let asbd = CMAudioFormatDescriptionGetStreamBasicDescription(audioFormatDesc)!.pointee
let sampleRate = asbd.mSampleRate         // Частота дискретизации, например 44100
let channels = Int(asbd.mChannelsPerFrame) // Число каналов (моно=1, стерео=2 и т.д.)

// Настройка вывода аудио в Linear PCM (16-bit little-endian)
let audioSettings: [String: Any] = [
    AVFormatIDKey: kAudioFormatLinearPCM,
    AVSampleRateKey: sampleRate,
    AVNumberOfChannelsKey: channels,
    AVLinearPCMBitDepthKey: 16,
    AVLinearPCMIsFloatKey: false,
    AVLinearPCMIsBigEndianKey: false
]
let audioOutput = AVAssetReaderTrackOutput(track: audioTrack, outputSettings: audioSettings)
if reader.canAdd(audioOutput) {
    reader.add(audioOutput)
}
```

В этом примере мы настроили вывод аудио на 16-битное несжатое PCM с исходной частотой и числом каналов. Если дорожка сжатая (например, AAC), `AVAssetReader` сам прозрачно декодирует ее в PCM при чтении с такими настройками. 

После добавления всех необходимых output-ов вызываем `reader.startReading()`. С этого момента `AVAssetReader` начнет поставлять данные. Получать сэмплы мы будем вызовом `copyNextSampleBuffer()` на каждом output. Чтение идет асинхронно от воспроизведения – мы сами решаем, когда вызывать copyNextSampleBuffer (обычно в цикле, либо по таймеру).

## 4. Декодирование видеокадров с помощью VideoToolbox (VTDecompressionSession)

На этапе 3 мы настроили видео-выход без декодирования (`outputSettings = nil`), поэтому получаем сырые сжатые кадры. Такие кадры – объекты типа `CMSampleBuffer` – содержат внутри сжатые данные (например, набор NALUnit-ов H.264) и описание формата (`CMFormatDescription`). Теперь необходимо декодировать эти CMSampleBuffer в растровые изображения (пиксельные буферы). За это отвечает **VideoToolbox**, предоставляющий API для работы с аппаратными кодеками.

Основная сущность для декодирования видео – **`VTDecompressionSession`** (сессия декодирования). Она представляет собой сессию работы видеодекодера. На iOS фактически любые H.264/H.265 декодируются аппаратно, но VideoToolbox скрывает детали, предоставляя единый интерфейс. Общий порядок использования `VTDecompressionSession` следующий ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=The%20steps%20to%20use%20a,First%2C%20create%20a%20VTDecompressionSession)) ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=Once%20the%20VTDecompressionSession%20is%20created%2C,be%20set%20in%20the%20decodeFlags)):

1. **Создание сессии** – указываем формат видеопотока и желаемые параметры вывода.
2. **Настройка сессии (опционально)** – например, требования к использованию аппаратного декодера.
3. **Отправка кадров на декодирование** – по одному CMSampleBuffer за раз.
4. **Получение декодированных кадров** – через callback или блок-обработчик, куда приходят готовые `CVPixelBuffer` (кадры) или сообщения об ошибке.

Перед созданием сессии нам понадобится формат описания видео – объект `CMVideoFormatDescription`. Его можно получить из одного из CMSampleBuffer видеодорожки:

```swift
let formatDesc = CMSampleBufferGetFormatDescription(sampleBuffer)!
```

Но проще еще на этапе настройки взять первый формат из `videoTrack.formatDescriptions` (они имеют тип `CMVideoFormatDescription`). Формат описания включает кодек (например, H.264) и параметры типа ширина/высота, параметрические сетки (SPS/PPS для H.264), которые необходимы декодеру. VideoToolbox требует этот объект при инициализации сессии, чтобы знать, с каким форматом он будет работать ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=VTDecompressionSession%20creation,its%20default%20hardware%20decoder%20selection)).

Кроме того, надо определить параметры **выводимых пиксельных буферов**. По умолчанию декодер может выдавать кадры в формате, удобном для себя (например, NV12 – YUV 4:2:0 семипланар). Однако мы можем запросить конкретный формат или свойства через словарь **destination image buffer attributes**. Например, если планируем использовать Metal для отображения, имеет смысл запросить буферы, совместимые с Metal. Основные ключи атрибутов:
- `kCVPixelBufferPixelFormatTypeKey` – тип пиксельного формата (например, `kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange` для NV12 или `kCVPixelFormatType_32BGRA` для BGRA).
- `kCVPixelBufferWidthKey`/`HeightKey` (необязательно) – размер кадра (обычно декодер сам знает из `formatDesc`).
- `kCVPixelBufferMetalCompatibilityKey` (или OpenGL) – Boolean, требовать совместимости с Metal (или OpenGL), чтобы полученные `CVPixelBuffer` можно было без копий привязать к текстуре GPU.

Настроив `formatDesc` и словарь атрибутов буфера, можно создать сессию:

```swift
var decompressionSession: VTDecompressionSession?
var callbackRecord = VTDecompressionOutputCallbackRecord(
    decompressionOutputCallback: myOutputCallback, 
    decompressionOutputRefCon: UnsafeMutableRawPointer(Unmanaged.passUnretained(self).toOpaque())
)
// Вариант: можно использовать OutputHandler API вместо callback

let status = VTDecompressionSessionCreate(
    allocator: nil,
    formatDescription: formatDesc,
    decoderSpecification: nil,  // можно указать требование hardware decoder
    imageBufferAttributes: destPixelBufferAttributes as CFDictionary?,
    outputCallback: &callbackRecord,
    decompressionSessionOut: &decompressionSession
)
guard status == noErr else { /* обработка ошибки создания сессии */ }
```

Здесь `decoderSpecification` мы передаем `nil`, позволяя VideoToolbox самому выбрать декодер (обычно аппаратный) ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=VTDecompressionSession%20creation,its%20default%20hardware%20decoder%20selection)). При необходимости можно задать, чтобы сессия **обязательно** использовала аппаратное ускорение (ключ `kVTVideoDecoderSpecification_RequireHardwareAcceleratedVideoDecoder` = `true`) либо отключить его (`EnableHardwareAcceleratedVideoDecoder = false`) ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=with%20no%20opt,specification%20option%20set%20to%20true)). 

Мы передали структуру `VTDecompressionOutputCallbackRecord`, ссылающуюся на нашу функцию обратного вызова `myOutputCallback`. Эта функция будет вызываться VideoToolbox’ом для каждого декодированного кадра. Сигнатура callback’а определена типом `VTDecompressionOutputCallback` – он получает указатель контекста, сам буфер `CVImageBuffer` (обычно `CVPixelBuffer`), а также ряд параметров, включая presentation timestamp (PTS) этого кадра.

> **Примечание:** Вместо низкоуровневого callback можно использовать удобный API **DecodeFrameWithOutputHandler** (доступен начиная с iOS 8). В этом случае при декодировании кадра передается блок, который будет вызван с результатом декодирования ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=The%20block,results%20of%20the%20decode%20operation)). Это избавляет от необходимости писать C-функцию, особенно удобно в Swift. Однако принцип работы тот же – внутри сессии кадры декодируются асинхронно.

Настроив сессию, запускаем цикл декодирования. Например, в отдельном потоке/очереди читаем сэмплы видео:

```swift
while reader.status == .reading {
    if let sampleBuffer = videoOutput.copyNextSampleBuffer() {
        // Отправляем кадр на декодирование
        let flags: VTDecodeFrameFlags = [._EnableAsynchronousDecompression]  // асинхронно для эффективности ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=Once%20the%20VTDecompressionSession%20is%20created%2C,be%20set%20in%20the%20decodeFlags))
        VTDecompressionSessionDecodeFrame(decompressionSession!,
                                          sampleBuffer: sampleBuffer,
                                          flags: flags,
                                          infoFlagsOut: nil) { (status, flags, imageBuffer, presentationTimeStamp, duration) in
            // Output handler (Swift): здесь получаем декодированный frame
            if status == noErr, let pixelBuffer = imageBuffer {
                // Сохранить или отобразить pixelBuffer
            }
        }
    } else {
        break // больше нет сэмплов (reader завершил чтение)
    }
}
```

В этом псевдокоде для каждого полученного `sampleBuffer` мы вызываем `VTDecompressionSessionDecodeFrame`. Если мы использовали callback, то обработка кадра происходит внутри `myOutputCallback`. Если использовали outputHandler (как показано), то в замыкании получаем `pixelBuffer`. Вызов декодирования можно делать с флагом `kVTDecodeFrame_EnableAsynchronousDecompression` ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=Once%20the%20VTDecompressionSession%20is%20created%2C,be%20set%20in%20the%20decodeFlags)), чтобы позволить VideoToolbox декодировать кадры асинхронно (параллельно, увеличивая производительность на многоядерных системах). В результате декодер может выдавать кадры немного позже вызова decode – это нормально, сессия сама управляет буферизацией.

VideoToolbox гарантирует, что вызовы outputCallback/block будут **сериализованными** – т.е. никогда не будет двух одновременных выдач декодированных кадров, даже если декодирование идет асинхронно ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=As%20long%20as%20the%20VTDecompressionSession,back%20pressure%20through%20the%20decoder)). Это упрощает потоковую безопасность при обработке выходных изображений. Тем не менее, не рекомендуется выполнять тяжелую работу внутри callback-а – лучше копировать/передать `CVPixelBuffer` дальше, а обработку (например, отображение) делать вне его, чтобы не задерживать очередь декодера ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=As%20long%20as%20the%20VTDecompressionSession,back%20pressure%20through%20the%20decoder)).

Итак, на этом этапе мы превращаем сжатые CMSampleBuffer (например, H.264) в последовательность декодированных кадров `CVPixelBuffer` (например, формат YUV). Далее нужно вывести эти кадры на экран.

## 5. Отображение видеокадров: AVSampleBufferDisplayLayer или Metal/OpenGL

После получения декодированных видеокадров (`CVPixelBuffer`) возникает задача их отрисовки на экран в нужное время. Существует два основных подхода:

**A. Использование AVSampleBufferDisplayLayer:**  
Этот класс (подкласс `CALayer`) специально создан для отображения медиа-данных и умеет самостоятельно декодировать и показывать видео. Он появился на iOS 8 и существенно упрощает вывод потока видео в реальном времени ([Direct Access to Video Encoding and Decoding - WWDC14 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2014/513/#:~:text=AVSampleBufferDisplayLayer%20shipped%20in%20Mavericks%2C%20and,it%27s%20new%20in%20iOS8)). `AVSampleBufferDisplayLayer` принимает на вход объекты `CMSampleBuffer` (с прикрепленным `CMVideoFormatDescription`) – обычно сжатые кадры – и внутренне производит декодирование и отображение ([Direct Access to Video Encoding and Decoding - WWDC14 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2014/513/#:~:text=As%20I%20mentioned%2C%20it%20takes,need%20to%20be%20in%20CMSampleBuffers)). Процесс такой:

- Создаётся экземпляр `AVSampleBufferDisplayLayer` и добавляется в иерархию слоёв (например, на view).
- По мере получения видеосэмплов (CMSampleBuffer) мы вызываем метод `enqueue(_:)` этого слоя, передавая буфер:
  ```objective-c
  if ([videoLayer isReadyForMoreMediaData]) {
      [videoLayer enqueueSampleBuffer:sampleBuffer];
  }
  ```
  Слой хранит внутреннюю очередь кадров. Внутри он имеет видеодекодер, декодирует поступающие кадры в `CVPixelBuffer` и помещает их в очередь на показ ([Direct Access to Video Encoding and Decoding - WWDC14 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2014/513/#:~:text=As%20I%20mentioned%2C%20it%20takes,need%20to%20be%20in%20CMSampleBuffers)).
- У `AVSampleBufferDisplayLayer` есть собственный таймбейс (CMTimebase), синхронизированный с системным временем по умолчанию. Он использует метки времени (PTS) каждого буфера, чтобы решить, когда отобразить декодированный кадр на экране. Проще говоря, вы можете добавить несколько CMSampleBuffer с опережением, и слой сам покажет каждый кадр в нужный момент времени.

Этот подход минимизирует наше участие в синхронизации видео – мы делегируем это AVFoundation. Однако есть нюанс: `AVSampleBufferDisplayLayer` не занимается аудио, поэтому синхронизацию с аудио всё равно надо учитывать (см. следующий раздел). Чаще `AVSampleBufferDisplayLayer` применяется для быстрого отображения сетевого видеопотока или когда хотим получить готовое решение «всё в одном». В контексте нашей задачи (низкоуровневый цикл воспроизведения с явным использованием VideoToolbox) **мы уже декодировали видео самостоятельно**, поэтому использование этого слоя может быть избыточным. Тем не менее, его стоит упомянуть как альтернативу: мы могли бы вовсе *не создавать VTDecompressionSession*, а просто отправлять `CMSampleBuffer` напрямую в `AVSampleBufferDisplayLayer` – тогда он сам проделал бы декодирование аппаратным кодеком ([ffmpeg - How to decode a H.264 frame on iOS by hardware decoding? - Stack Overflow](https://stackoverflow.com/questions/25197169/how-to-decode-a-h-264-frame-on-ios-by-hardware-decoding#:~:text=,to%20create%20your%20own%20decoder)).

**B. Рендеринг через Metal или OpenGL:**  
Этот путь требует немного больше кода, но дает полную гибкость в отображении. Имея декодированные `CVPixelBuffer`, мы можем отображать их с помощью графического API. В iOS современный выбор – **Metal**, для более старых приложений – **OpenGL ES** (или через Core Image). Основная задача – передать содержимое `CVPixelBuffer` в текстуру GPU и нарисовать на экране квадрат с этой текстурой (или использовать как хотите).

- **Metal:** Чтобы работать с `CVPixelBuffer` в Metal, Apple рекомендует использовать **`CVMetalTextureCache`**. Это кэш, который умеет создавать `MTLTexture`, ссылающуюся на буфер без копирования. Сначала создается кэш через `CVMetalTextureCacheCreate`. Затем для каждого нового кадра вызывается `CVMetalTextureCacheCreateTextureFromImage`, куда передается `CVPixelBuffer` и описание желаемого текстурного формата. Например, если декодер выдает YCbCr (NV12), можно создать два-плоскую текстуру (через формат `.bgra8Unorm` или специальный YUV-пиксель формат – Metal поддерживает YCbCr текстуры). После получения `MTLTexture` можно в рендер-проходе Metal отрисовать её на экран (например, используя `MTKView` или собственный CAMetalLayer). Важно не забывать, что `CVPixelBuffer` может переиспользоваться из пула (CVPixelBufferPool) декодера, поэтому после того, как Metal закончил с текстурой, нужно уведомить кэш (через `CVMetalTextureCacheFlush`) или убедиться, что буфер не освободится раньше времени ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=As%20described%20before%2C%20when%20a,used%20for%20the%20new%20CVPixelBuffer)) ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=There%20are%20two%20main%20approaches,a%20bit%20of%20extra%20care)). Обычно достаточно удерживать `CVPixelBuffer` (или полученную `CVMetalTexture`) до завершения рендеринга текущего кадра.

- **OpenGL ES:** Аналогично, используется **`CVOpenGLESTextureCache`** для создания текстур из `CVPixelBuffer`. Процесс похож: `CVOpenGLESTextureCacheCreateTextureFromImage` возвращает `CVOpenGLESTexture` (которая представляет GL-текстуру, привязанную к содержимому CVPixelBuffer). Далее обычными вызовами OpenGL можно нарисовать прямоугольник, заполнив его этой текстурой. Поскольку OpenGL ES теперь устарел на iOS, подробности опускаем, но суть такая же – безкопировочное отображение.

- **Core Image / UIImageView:** Для полноты можно упомянуть, что если не требуется сверхпроизводительность, `CVPixelBuffer` можно преобразовать в CIImage и отрисовать через `CIContext` на экран, или даже в CGImage/UIImage. Однако для видео 60fps это неоправданно медленно. В рамках реального времени лучше прямой GPU-рендеринг.

**Выбор подхода:** Использование `AVSampleBufferDisplayLayer` проще, но ограничено возможностями этого слоя. Если требуется накладывать эффекты, рисовать UI поверх видео, применять шейдеры и т.п., то Metal/OpenGL дают больше контроля. В наших целях учебного воспроизведения мы могли бы выбрать любой способ. Например, для простоты далее представим, что мы воспользуемся `AVSampleBufferDisplayLayer` **или** у нас уже есть собственный Metal-рендер, готовый принимать `CVPixelBuffer`.

Если использовать `AVSampleBufferDisplayLayer` после собственного декодирования, возможно, стоит не декодировать самим, а сразу отправлять туда compressed buffers. Но *можно* передавать и декодированные: слой также умеет принимать *уже декодированные* CMSampleBuffer, если правильно создать CMSampleBuffer с прикрепленным CVPixelBuffer. Однако практичнее либо полностью доверить слой декодирование, либо полностью контролировать самому. Для демонстрации баланса возможностей мы описали оба подхода.

## 6. Декодирование и воспроизведение аудио (AVAudioEngine или AudioQueue)

Одновременно с видео нужно воспроизводить аудио. Мы настроили `AVAssetReaderTrackOutput` для аудио выдавать несжатый PCM, так что на выходе `copyNextSampleBuffer()` для аудио получаем CMSampleBuffer с аудио-фреймами (например, 1024 или 4096 сэмплов PCM за раз, в зависимости от кодека). Задача – подать эти PCM-данные на аудиовыход устройства в реальном времени.

В iOS высокоуровневый способ – использовать **`AVAudioEngine`** с **`AVAudioPlayerNode`**, а низкоуровневый – **AudioQueue**. Расскажем об обоих.

**AVAudioEngine + AVAudioPlayerNode:** Этот вариант появился начиная с iOS 8 и предоставляет объектно-ориентированный интерфейс над CoreAudio. План такой:
- Создаем `AVAudioEngine` и `AVAudioPlayerNode`. PlayerNode – это узел, который умеет проигрывать аудио буферы.
- Прикрепляем узел к движку (`engine.attach(node)`) и подключаем его к выходному узлу (например, к главному микшеру или непосредственно к `engine.outputNode`), указав формат. Формат должен соответствовать формату наших PCM-данных (частота дискретизации, число каналов). Мы можем создать `AVAudioFormat` из тех же параметров, что использовали в outputSettings.
- Готовим `AVAudioEngine` к работе (`engine.prepare()`), затем `engine.start()` запускает аудиосистему.
- Теперь, по мере получения аудио-выборок, нужно конвертировать `CMSampleBuffer` в формат, понятный AVAudioEngine – то есть в объект `AVAudioPCMBuffer`. Можно воспользоваться методами CoreAudio: например, `CMSampleBufferGetAudioBufferListWithRetainedBlockBuffer` для доступа к AudioBufferList внутри сэмплбуфера, а затем скопировать данные в `AVAudioPCMBuffer`. Либо напрямую скопировать из `CMBlockBuffer` в `.floatChannelData` или `.int16ChannelData` буфера.
- Когда получен `AVAudioPCMBuffer`, его следует *запланировать* на воспроизведение: `playerNode.scheduleBuffer(pcmBuffer, completionHandler: nil)`. Если не указано время, буфер встанет в очередь после ранее поставленных буферов.
- После того, как накоплено достаточно буферов (или хотя бы один), вызываем `playerNode.play()` – аудио начнет воспроизводиться. Далее можно в реальном времени дозакидывать новые аудиобуферы через `scheduleBuffer`, даже во время проигрывания. PlayerNode будет их выводить последовательно, без разрывов, если всегда есть следующий буфер в очереди.

Ниже упрощенный пример цикла чтения аудио и подачи в AudioEngine (без обработки синхронизации пока):

```swift
// Предполагается, что engine и playerNode уже настроены и запущены.
while reader.status == .reading {
    if let audioSample = audioOutput.copyNextSampleBuffer() {
        // Преобразуем CMSampleBuffer в AVAudioPCMBuffer
        guard let pcmBuffer = sampleBufferToPCMBuffer(audioSample) else { continue }
        playerNode.scheduleBuffer(pcmBuffer, completionHandler: nil)
    } else {
        break // аудио трек закончился
    }
}
// Начинаем проигрывание (если не сделали ранее)
playerNode.play()
```

Функция `sampleBufferToPCMBuffer` в псевдокоде должна извлечь из CMSampleBuffer аудиоданные. Можно, например, сделать так (для 16-бит PCM):
```swift
func sampleBufferToPCMBuffer(_ sampleBuffer: CMSampleBuffer) -> AVAudioPCMBuffer? {
    guard let format = sampleBuffer.formatDescription else { return nil }
    let asbd = CMAudioFormatDescriptionGetStreamBasicDescription(format)!.pointee
    let audioFormat = AVAudioFormat(commonFormat: .pcmFormatInt16,
                                    sampleRate: asbd.mSampleRate,
                                    channels: asbd.mChannelsPerFrame,
                                    interleaved: true)
    let frameCount = CMItemCount(CMSampleBufferGetNumSamples(sampleBuffer))
    guard let pcmBuffer = AVAudioPCMBuffer(pcmFormat: audioFormat, frameCapacity: frameCount) else { return nil }
    pcmBuffer.frameLength = frameCount

    // Копируем данные из CMSampleBuffer в pcmBuffer
    if let dataBuffer = CMSampleBufferGetDataBuffer(sampleBuffer) {
        let length = pcmBuffer.byteLength
        var audioDataPointer: UnsafeMutablePointer<Int8>? = nil
        CMBlockBufferGetDataPointer(dataBuffer, atOffset: 0, lengthAtOffsetOut: nil,
                                    totalLengthOut: nil, dataPointerOut: &audioDataPointer)
        if let src = audioDataPointer {
            // копируем length байт из src в pcmBuffer.mutableBytes
            pcmBuffer.byteLength = length
            memcpy(pcmBuffer.int16ChannelData![0], src, Int(length))
        }
    }
    return pcmBuffer
}
```
*(Приведенный код носит иллюстративный характер; на практике нужно учесть формат (float/int) и интерливинг каналов.)*

AVAudioEngine автоматически выводит звук на устройство с минимальными задержками. Также он умеет добавлять эффекты, микшировать и т.д., но в нашем случае это не нужно. Главное – у нас появилась **непрерывная аудиовоспроизводящаяся линия**, на которую мы можем ориентироваться для синхронизации видео.

**AudioQueue:** Более низкоуровневый вариант – напрямую использовать **AudioQueue Services** из CoreAudio. Это C-API, где разработчик сам управляет буферами. Процедура:
- Определяем формат (AudioStreamBasicDescription) – тот же PCM 16-bit, например.
- Создаем выходную очередь через `AudioQueueNewOutput`, передавая формат и callback-функцию.
- Выделяем несколько буферов AudioQueue (`AudioQueueAllocateBuffer`) – обычно 3 или 4, чтобы один игрался, другой заполнялся и т.д.
- Заполняем буферы первыми порциями аудио данных из `CMSampleBuffer` (копируем PCM байты).
- Ставим их в очередь на проигрывание (`AudioQueueEnqueueBuffer`).
- Стартуем очередь (`AudioQueueStart`).
- Далее, в callback (вызывается, когда очередь освободила очередной буфер после воспроизведения) мы получаем указатель на пустой буфер. Нужно снова заполнить его следующей порцией PCM-данных и вновь ставить в очередь. Этот цикл продолжается, пока не кончатся данные.
- По окончании вызываем `AudioQueueStop` и освобождаем ресурсы.

AudioQueue требует тщательной работы с потоками и памятью, но даёт контроль на уровне буферов и синхронизации. Практическое преимущество AVAudioEngine – он сам реализует внутри, по сути, то же, но с более удобным интерфейсом (к тому же, AVAudioEngine работает на основе AudioUnits, обеспечивая низкую задержку).

Для нашей цели можно выбрать любой метод. **В контексте iOS, AVAudioEngine – предпочтительный способ**, если нет особых причин использовать AudioQueue. Поэтому дальше будем считать, что аудио играет через AVAudioEngine.

## 7. Синхронизация аудио и видео потоков во времени

Добиться одновременного и согласованного воспроизведения аудио и видео – одна из сложнейших задач мультимедийного плеера. Несинхронное воспроизведение приводит к рассинхрону («audio/video drift»), когда звук «убегает» вперед или отстает от картинки. В нашем низкоуровневом цикле мы должны сами обеспечить синхронизацию, используя метки времени сэмплов и системные таймеры или часы.

**Использование таймстампов CMSampleBuffer:** Каждый `CMSampleBuffer` содержит временную метку **PTS (Presentation Timestamp)** – время, когда данный фрейм должен быть представлен (показан или проигран) относительно начала медиапотока. Эти метки времени заданы в единицах `CMTime` (числитель/знаменатель). Например, видеоPTS 1.500 при таймскейле 600 означает 1.5 секунды. Аудио сэмплы могут идти с другой частотой дискретизации, но тоже имеют свой PTS.

Синхронизация сводится к следующему: **воспроизводить аудио и видео так, чтобы их PTS шли синхронно по единой временной шкале.** Например, если сейчас проигрывается аудио с меткой 10.0с, то на экране должен быть кадр с ~10.0с (учитывая допустимые отклонения, обычно не более 40-50 мс).

Практически, можно выбрать один из потоков **мастером времени**. Чаще выбирают аудио в роли ведущего (master), а видео синхронизируют под него. Причины: аудио более чувствительно к джиттеру (нельзя резко ускорять/замедлять звук без артефактов), а небольшие пропуски видео кадров менее заметны, чем щелчки или рассинхрон звука.

Если мы используем **AVAudioEngine**, у нас звук идет по системному **реальному времени** – движок сам привязан к аудиооборудованию, которое обычно опирается на аппаратный тактовый генератор. Мы не управляем напрямую скоростью – оно всегда 1x (если не применять спецэффекты). Значит, аудио PTS можно считать равномерно движущимся вперед со скоростью 1 секунда в секунду. Начало воспроизведения аудио соответствует PTS первого аудио буфера (например, 0 или другое значение если трек не с нуля).

У `AVAudioPlayerNode` есть методы для синхронизации, например `play(at:)`, позволяющий задать момент начала в определённом времени. Но часто можно обойтись и без этого: при старте воспроизведения аудио считать текущее время как точку отсчета.

**Стратегия синхронизации:** Предположим, первый аудиосемпл имеет PTS = 0 (обычно так и есть, если дорожка с начала). Когда мы вызываем `playerNode.play()`, аудио начнет звучать почти сразу (с небольшой системной задержкой, ~табличной). Мы можем получить *абсолютное время запуска* аудио, например через `playerNode.lastRenderTime` + `playerNode.outputPresentationTime` или с помощью AVAudioTime. Однако проще поступить так:
- Зафиксировать время запуска как текущее *системное время* (например, `mach_absolute_time` или `CACurrentMediaTime()` в секундах). Допустим, `startTimeSystem = CACurrentMediaTime()`.
- Сопоставить его с **медиа-временем** = 0 (начало потока). То есть считаем, что медиа-время 0 соответствует системному времени startTimeSystem.
- Теперь для каждого видеокадра с PTS = T (секундах) можно вычислить его **целевое системное время отображения**: `targetTime = startTimeSystem + T`.
- Текущего системного времени можно получать постоянно (например, опять же CACurrentMediaTime). И когда `currentSystemTime >= targetTime`, значит, пора показывать этот кадр.

Конкретная реализация может быть разной:
- **С использованием дисплейного таймера**: на iOS можно запустить `CADisplayLink`, который вызывается каждую vsync (например, 60 раз/сек). В его обработчике сверять время: брать PTS следующего кадра и сравнивать с `CACurrentMediaTime()`. Если уже пора (или чуть просрочено) – отобразить кадр (или сразу несколько, если накопились). Это распространенный метод в игровых и видео движках.
- **С использованием аудио clock**: у AudioQueue есть функция `AudioQueueGetCurrentTime` (в AVAudioEngine тоже можно вычленить время через AVAudioNode.lastRenderTime). Этот метод дает текущее время воспроизведения на аудио выходе. Можно спросить у аудиосистемы: "какой PTS аудио сейчас играет?". Получив, скажем, 5.2 секунды, мы можем сразу отобразить все видео с PTS вплоть до 5.2с. Это даже надежнее, так как основывается на реально сыгранном звуке. У `AVAudioPlayerNode` есть метод `playerTime(for:)`, возвращающий AVAudioTime, который можно перевести в секундах.
- **Общий CMTimebase:** продвинутый способ – создать `CMTimebase` и `CMClock`. Например, можно взять аудио устройство (AudioOutputUnit) clock или системные часы `CMClockGetHostTimeClock()` и связать с `AVSampleBufferDisplayLayer` или своим рендерером. AVFoundation позволяет установить для `AVSampleBufferDisplayLayer` свой `controlTimebase` ([Direct Access to Video Encoding and Decoding - WWDC14 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2014/513/#:~:text=So%20we%20allow%20you%20to,clock%20with%20your%20own%20timebase)). Если бы мы шли путем AVSampleBufferDisplayLayer, мы могли бы привязать его таймбейс к аудио (но это выходит за рамки текущей реализации без этого слоя).

Для простоты далее предположим, что мы будем использовать системные часы и аудио как опорные. Практически, можно сделать так: когда начали аудио (`playerNode.play()`), тут же запланировать показ первого кадра. Первый видеокадр обычно имеет PTS 0 и должен появиться сразу. Затем мы можем вычислять задержку до следующего кадра: `delay = nextFramePTS - currentFramePTS` и на это время ставить таймер/слайп, по истечении которого показать следующий кадр. Но у видео частота может не быть постоянной, лучше всегда смотреть на PTS.

**Пример реализации синхронизации (псевдокод):**

```swift
var videoFramesQueue: [(pts: CMTime, pixelBuffer: CVPixelBuffer)] = ...

// Допустим, audioStartedTime = CACurrentMediaTime() в момент запуска аудио
while !videoFramesQueue.isEmpty {
    let nextFramePTS = videoFramesQueue.first!.pts
    let nextFrameTimeSec = CMTimeGetSeconds(nextFramePTS)
    let targetDisplayTime = audioStartedTime + nextFrameTimeSec

    // Ждем, пока системное время приблизится к targetDisplayTime
    let now = CACurrentMediaTime()
    if now >= targetDisplayTime {
        let frame = videoFramesQueue.removeFirst()
        displayVideoFrame(frame.pixelBuffer)  // выводим кадр (через Metal/GL UI)
    } else {
        // спим/ждем минимально, либо продолжим цикл
        usleep(1000) // sleep 1ms, или использовать семафор/вейт на точное время
    }
}
```

В реальности `usleep` не очень точный, лучше использовать `RunLoop` или `DispatchSourceTimer` с нужной точностью, или `CADisplayLink` как упомянуто. Но суть: мы сопоставляем медиавремя кадра с реальным временем, основываясь на том, когда стартовало аудио.

Если аудио немного задержалось при старте, то первый кадр может показаться чуть раньше аудио – но человеческий слух/зрение не заметит небольших <50мс рассинхронов. Важно, чтобы на протяженности воспроизведения скорость не расходилась.

**Синхронизация при использовании AVSampleBufferDisplayLayer:** Если бы мы использовали этот слой, он сам отсчитывает время (по умолчанию относительно `mach_absolute_time`). Нужно лишь убедиться, что аудио и он стартуют синхронно. Одно решение: установить у `AVSampleBufferDisplayLayer` тот же `CMTimebase`, что и у аудио (хотя AVAudioEngine не предоставляет прямого CMTimebase). Либо вручную задать слой начать с времени 0 в момент T. Например, можно сделать `layer.controlTimebase = CMTimebaseCreateWithMasterClock(hostClock)` и установить временную метку: `CMTimebaseSetTime(layer.controlTimebase, CMTimeMake(0, 1000))` и `CMTimebaseSetAnchorTime(layer.controlTimebase, CMTime(seconds:0, preferredTimescale:600), CMClockGetHostTimeClock().convertHostTimeToCMTime(startHostTime))`. В общем, это сложнее, поэтому проще было бы или не декодировать вручную, или синхронизовать «вручную» как описано выше.

Резюмируя: **аудио ведет, видео следует.** Мы используем PTS видеокадров и либо ждём соответствующего аудио времени, либо планируем кадры на определенный момент.

## 8. Буферизация, обработка пропуска кадров и работа в реальном времени

Реальный мир не идеален: декодирование может задерживаться, а устройство может испытывать нагрузку. Чтобы воспроизведение было плавным, внедряются механизмы буферизации и компенсации.

**Буферизация (prefetching):** В нашем цикле мы не обязаны строго чередовать decode->display кадр за кадром. Лучше всегда иметь небольшой запас декодированных кадров. Например, можно читать и декодировать на несколько кадров вперед относительно текущего отображаемого. Видеодекодер (VTDecompressionSession) сам буферизует некоторый конвейер кадров, особенно при асинхронном декодировании. Но и на уровне нашей логики полезно иметь очередь декодированных `pixelBuffer` (скажем, на 0.5 секунды вперед). Аналогично, аудио мы тоже обычно планируем на несколько буферов вперед. Это обеспечивает сглаживание, если в какой-то момент декодер притормозит, у нас уже есть кадры, которые можно отобразить из очереди.

**Дроп кадров (frame dropping):** Если система не успевает декодировать или отрисовывать видео с нужной скоростью, возникает риск, что видеопоток начнет отставать от аудио. В такой ситуации плееры прибегают к пропуску кадров. Стратегия проста: если кадр пришло время показывать, а он еще не декодирован или не готов – его следует пропустить, чтобы догнать аудио. Проигнорированный кадр не покажется, пользователь увидит небольшой скачок, но синхронность сохранится. Обычно пропускают не ключевые кадры (не I-frames), чтобы не потерять важную сцену, но в реальном времени может не быть выбора. VideoToolbox не делает автоматического дропа за нас (разве что если указать специальные флаги real-time). Поэтому логика пропуска лежит на нашем коде: например, в шаге синхронизации, если обнаруживаем, что `now` значительно превышает `targetDisplayTime` кадра (т.е. мы «прозевали» его), значит, этот кадр уже устарел – вынимаем его из очереди без отображения и смотрим следующий. Так можно отбросить несколько подряд, пока не поймаем кадр, чей targetTime чуть в будущем или почти настоящее.

**Реальное время vs. обработка на максимум:** `AVAssetReader` с outputs позволит читать из файла очень быстро (быстрее реального времени). Нам важно не прочитать и не декодировать всё сразу, иначе заполнится память и нарушится синхронность. Нужно **дросселировать** чтение/декодирование под скорость воспроизведения (~1x). Наш цикл фактически управляется аудио: мы не можем воспроизвести звук быстрее, чем реальное время. Значит, и видео декодировать стоит с небольшим опережением, но не мгновенно. Одно решение: читая сэмплы, проверять PTS и не уходить слишком далеко вперед от текущего воспроизводимого времени (например, держать опережение не более 1-2 секунд). 

**Обработка пауз и сбоев:** Если аудио временно закончилось (например, тишина, или мы нажали паузу), видео тоже должно приостановиться. В случае паузы, можно приостановить таймеры/дисплейлинк и остановить `AVAudioEngine`. При возобновлении – проделать синхронизацию заново (вплоть до пересоздания timebase).

**Конфигурация VideoToolbox для реального времени:** Есть опция при декодировании – флаг `VTDecodeFrameFlags._1xRealTimePlayback` (в C: `kVTDecodeFrame_EnableTemporalProcessing`). Он сообщает декодеру, что поток воспроизводится в реальном времени (а не для оффлайн обработки). Это может побудить внутренние оптимизации, напр., снизить качество при пропуске. Однако в документации Apple указывается лишь флаг асинхронности, про 1xRealTime явно не упоминается, возможно, VideoToolbox сам адаптируется.

**Вывод аудио и видео без задержек:** Стоит упомянуть, что на iOS аудио имеет небольшую буферизацию (обычно 256-512 сэмплов, то есть 5-10ms для 44.1kHz) – это практически ничто. Видео же буферизуется по кадрам (например, один кадр ~16ms при 60fps). Таким образом, вполне реально добиться синхронизации с точностью ~±20мс. Это достаточная точность (человек начинает замечать рассинхрон звук/картинка начиная с ~40-50мс).

**Многопоточность:** Воспроизведение должно использовать фоновые потоки: например, чтение и декодирование видео можно делать на глобальной очереди QoS.userInteractive, а отображение кадров – на основном (UI) потоке или через Metal (который сам синхронизируется с дисплеем). АудиоEngine сам использует аудио-потоки реального времени. Важно не блокировать основную очередь длительными операциями.

Подводя итог, реализация полного цикла воспроизведения с помощью AVAssetReader и VideoToolbox требует учета многих деталей: чтение данных, декодирование видео, вывод на экран, проигрывание аудио и синхронизация. Однако такой подход дает большой контроль над процессом и возможен даже без использования высокоуровневых компонентов вроде `AVPlayer`. Мы разобрали, как с помощью низкоуровневых API iOS можно построить свой мини-плеер, обрабатывающий видео и аудио в реальном времени и синхронно.

**Источники и ссылки:**

- Официальная документация Apple и сессии WWDC по работе с медиаданными, AVFoundation и VideoToolbox, включая советы по извлечению сэмплов и использованию декодеров ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=In%20order%20to%20get%20raw,dictionary%20specifying%20a%20pixel%20format)) ([Direct Access to Video Encoding and Decoding - WWDC14 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2014/513/#:~:text=As%20I%20mentioned%2C%20it%20takes,need%20to%20be%20in%20CMSampleBuffers)).
- Статьи и примеры, демонстрирующие использование `VTDecompressionSession` для декодирования кадров ([ffmpeg - How to decode a H.264 frame on iOS by hardware decoding? - Stack Overflow](https://stackoverflow.com/questions/25197169/how-to-decode-a-h-264-frame-on-ios-by-hardware-decoding#:~:text=,to%20create%20your%20own%20decoder)) ([Decode ProRes with AVFoundation and VideoToolbox - WWDC20 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2020/10090/#:~:text=Once%20the%20VTDecompressionSession%20is%20created%2C,be%20set%20in%20the%20decodeFlags)).
- Документация по AVAssetReader и настройке выходных параметров (PCM для аудио, nil для видео) ([sdks/iPhoneOS9.3.sdk/System/Library/Frameworks/AVFoundation.framework/Headers/AVAssetReaderOutput.h at master · theos/sdks · GitHub](https://github.com/theos/sdks/blob/master/iPhoneOS9.3.sdk/System/Library/Frameworks/AVFoundation.framework/Headers/AVAssetReaderOutput.h#:~:text=A%20value%20of%20nil%20for,used%20with%20the%20specified%20track)) ([AVAssetReaderTrackOutput | Apple Developer Documentation](https://developer.apple.com/documentation/avfoundation/avassetreadertrackoutput#:~:text=AVAssetReaderTrackOutput%20,For%20video%20output%20settings%2C)), а также по `AVSampleBufferDisplayLayer` и синхронизации с CMSampleBuffer PTS ([Direct Access to Video Encoding and Decoding - WWDC14 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2014/513/#:~:text=As%20I%20mentioned%2C%20it%20takes,need%20to%20be%20in%20CMSampleBuffers)).
