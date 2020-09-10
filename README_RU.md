# Детектирование через сегментацию

Реализована нейросетевая модель, основанная на решении задачи сегментации, 
способная находить объекты на изображениях, с возможностью
(опционально) также определять класс найденных объктов.

Принципиальные отличия от других детекторов:
1. Находит объекты произвольной формы (например, YOLO плохо справляется с сильно вытянутыми объектами)
2. Очень быстрое решение - максимум 50 мс на изображение на cpu (4 cores) с макс стороной 512 
и 150 мс с макс стороной 1024

По ходу обучения сохраняются логи, которые удобно смотреть в tensorboard

Есть визуализации всех этапов - сегментации, постпроцессинга, классификации (+аугментации).

С помощью данного проекта была получена модель, способная находить одновременно все типы баркодов
и с хорошей точностью их классифицировать

### Как происодит детекция?

Устройство детектора сейчас следующее - 
1. **Масштабирование изображений.**
Большие изображения ресайзятся по максимальной стороне до max_image_side (по умолчанию 1024) 
сохраняя aspect ratio, 
небольшие изображения, у которых бОльшая сторона меньше max_image_side подаются как есть. 
В любом случае каждая размерность изображения округляется кратно side_multiple (по умолчанию 64)

2. **Предобработка (preprocessing)**. Сейчас поддерживается два варианта - без предобработки 
(изображение подается как есть) или как в mobilenet (x=(x-127.5)/127.5), можно указать в 
--preprocessing аргументе при запуске обучения, по умолчанию используется предобработка mobilenet

3. **Работа нейросети**. Текущая архитектура состоит из 3х сверток 
уменьшающих пространственное разрешение в 4 раза (со stride=2; stride=1; stride=2), 
затем кучи сепарабельных сверток, каждая последующая из которых имеет dilation в 2 раза 
больше предыдущей и завершающая свертка после которой для каждого пикселя
в получившейся (уменьшенной в 4 раза по каждой размерности относительно исходного изображения)
карте сегментации соответствует 1 + n_classes значений, первый канал отвечает за вероятность детекции,
последующие за выбор класса в случае успешной детекции

4. **Postprocessing**. Из полученной карты сегментации выделяются связные компоненты
по детекциям (detection_p > threshold), затем находится обрамляющий прямоугольник являющийся 
финальной детекцией объекта. Класс объекта определяется как argmax(mean(pixel_class_p)) где среднее
берется для всех пикселей в компаненте связности
 
 
### Структура файлов

Запустить обучение - train.py, все модели, аргументы и логи сохранятся в директории с логами
которую достаточно указать в predict.py чтобы все параметры совпадали

predict.py - запустить уже имеющуюся модель, необходимо указать директорию с логами, 
откуда эту модель брать

augmentation.py аугментация

data_generators.py генераторы для обучения и inference

data_markup.py типы разметки (объект без типа или с типом)

evaluation.py целевые метрики

keras_callbacks.py все что помогает максимально удобно мониторить обучение

keras_metrics.py метрики для оценки качества модели

losses.py функции потерь для обучения

model_runner.py прогон модели на произвольном датасете 
(возможно с сохранением результатов и отрисовкой визуализаций)

markup_readers.py ридеры для чтения различных форматов разметки,
если вам нужно обучить модель на какую-то другую задачу,
просто допишите свой ридер здесь

net.py здесь лежит config класс отвечающий за относительную воспроизводимость -
все основные параметры с которыми запускалась сеть при обучении (за исключением random seed, 
фиксирование которого при обучении на gpu все равно не позволяет гарантировать 100% воспроизводимость),
этот config сохраняется в папке с логами и при загрузке модели загружается вместе с ней.
Также в этом файле лежат сами архитектуры нейросетей и все что необходимо для их построения.

segmap_manager.py работа с картами сегментации - построение, постпроцессинг

visualizations.py отрисовка визуализаций

utils.py всякие вспомогательные функции

### Что нужно сделать, чтобы обучить сеть на своих данных?

Все просто - допишите ридер для своих данных (в markup_readers.py),
добавьте его в supported_markup_types (в data_generators.py) и всё, profit.

Известная возможная проблема: сейчас стоит перекос по лоссам в recall, поэтому, если
вы видите в логах тензорборда, что закрашивается вся картинка и ситуация не меняется,
стоит залезть в losses.py и уменьшить коэффициент L_POSITIVE_WEIGHT
