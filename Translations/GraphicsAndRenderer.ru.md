# Визуализация/Рендеринг (Rendering) в Urho3D: подсистема графики (Graphics) и рендеринга (Renderer).
Эта статья - перевод [статьи] (http://urho3d.github.io/documentation/HEAD/_rendering.html) с моими комментариями

## Подистема графики (Graphics)

Подистема графики включает в себя низкоуровневые функции, такие как:
- Создание окон и рендеринг контекста
- Задание экранного режима
- Отслеживание GPU ресурсов
- Отслеживание визуализации контекстного состояния (включает в себя текущую цель визализации (rendertarget), буфер вершин и индексов (vertex and index buffers), текстуры, шейдеры и состояния визализации (renderstates))
- Загрузка шейдеров
- Выполнение примитивных операций визуализации (primitive rendering operations)
- Обрабатывание потеряных устройств (?)

Функция **SetMode()** графической подсистемы устанавливает такие параметры, как расширение экрана, полноэкранный/оконный режим, вертикальная синхронизация и уровень мультисемплинга (множественной выборки) на графическом процессоре. Кроме этого существует экспериментальная возможность рендеринга в уже открытом окне путем передачи его обработчика в функцию **SetExternalWindow()** до установки начальных экранных параметров.

При установке начальных экранных параметров подсистема графики производит следующий проверки:
- проверка на наличие поддержки моделей шейдеров 3.0 для систем с **Direct3D9**
- проверка на наличие поддержки моделей шейдеров 3.2 для систем с **OpenGL**, при ее отсутствии, программа проверяет на наличие модели шейдеров 2.0 с расширенями **EXT_framebuffer_object**[1], **EXT_packed_depth_stencil**[2] и **EXT_texture_filter_anisotropic**[3] и использует её. Программа также проверяет расширение **ARB_instanced_arrays**[4], но это расширение опционально. Если **ARB_instanced_arrays** присутствует, то программа включит GPU поддержку для него.
- проверка на наличие GPU поддержки карт теней (поддерживаются AMD и NVidia). Если поддержки нет, то программа не будет поддрживать рендеринг теней.
- проверка на поддержку препрохода света (light pre-pass[6]) и рендеринга в режиме задержки (deferred[5]). Эти режимы также требуют поддержки множественных целей для рендеринга (multiple rendertarget support) и формата текстур R32F

## Подсистема рендеринга (класс Renderer)

Подсистема рендеринга занимается рендерингом 3D объектов покадрово (each frame), контролирует глобальные настройки такие как качество текстуры, материала, отраженного света и основного разрешения карты теней.

Подсистеме требется сцена (Scene) с октодеревом (Octree[7]) и камера (Camera), которая не обязательно должна быть привязана к сцене. Октодерево сохраняет все видимые компоненты (наследники класса Drawable) чтобы облегчить доступ и запросы к ним. Вся необходимая информация собрана в объекте порта просмотра (viewport), который передается в класс Renderer через функцию **SetViewport()**.

По умолчанию существует всего один Viewport, но количество портов просмотра может быть увеличено функцией **SetNumViewports()**. Порт или порты просмотра должы использовать весь экран, иначе может наблюдаться эффект зала с зеркалами (hall-of-mirrors artifacts [8]). Порты просмотра будут проходить через рендер от 0-го и дальше по увеличению, так, если требуется нарисовать небольшой порт просмотра на поверхности главного порта просмотра, то порт с индексом 0 должен быть главным портом, а с индексом 1 - небольшим.

Порты просмотра могут использоваться как текстуры для рендеринга (rendertargets) (см. Вспомогательные порты просмотра[9]).

Каждый порт просмотра определяет порядок команд для рендеринга сцены, так называемый путь рендеринга (renderpath). С программой уже поставляются стандартные пути рендеринга прямой (forward), с поддержкой препрохода света (light pre-pass) и в режиме задержки (deferred), пути рендеринга располагаются в папке bin/CoreData/RenderPaths, функцией SetDefaultRenderPath() можно задать путь рендеринга по умолчанию.  Если путь не задан, то используется прямой путь рендеринга (forward) по умолчанию. Однако, путь рендеринга с задержкой (Deferred) обладает преимуществом для сцен с большой плотностью освещения на пиксель каждого объекта, путь рендеринга с задержкой обладает также и недостатками, как то недостаток поддержки со стороны GPU мультисэмплинга (multisampling, MSAA[10]) и невозможность выбора световой модели с привязкой к материалу. Вместо MSAA можно использовать FXAA[11] (см пример bin/Data/Scripts/09_MultipleViewports.as).

Шаги рэндеринга на каждый порт вывода и на каждый кадр можно описать следующим образом:
- Запрос у октодереву с целью нахождения видимых объектов и и источников света в видимой часте сцены (camera's view frustum[12])
- Проверка влияния каждого видимого источника света на объекты. Если свет отбрасывает тень, запрос к октодереву на поиск объектов с тенью
- Конструирование рендеринговых операций (batches) для видимых объектов в соответствии с проходами сцены (scene passes) в пути рендеринга (renderpath)
- Выполнение команд путя рендеринга в конце каждого кадра
- Если в сцене определен DebugRenderer компонент и в порте вывода разрешен debug rendering (функция **SetDrawDebug()**), то, в самую последнюю очередь, производится рендеринг геометрии для DebugRenderer

В пути рендеринга по умолчанию, операции рендеринга происходят следующим образом:
- Проход для рассчета непрозрачной геометрии окружения или рассета G-буфера[13] если ренединг идет в режиме задержки  
- Проход для рассчета непрозрачной геометрии с попиксельным проходом света. В случае с рендерингом теней вначале рендерятся тени
- (В случае поддержки препрохода света light pre-pass) Проход для рассчета непрозрачной геометрией материала, этот проход также рендерит объекты с попиксельным освещением (objects with accumulated per-pixel lighting)
- Проход для пост-непрозрачных (post-opaque) деталей для специального рендеринга некоторых объектов, например неба
- Проход геометрии преломления (refractive geometry)
- Проход для прозрачной геометрии. Прозрачные, альфа-смешанные (alpha-blended[14]) объекты отсортированы по дистанции и отрендерены начиная от удаленных объектов и к ближним (back-to-front) чтобы получить корректную смешанность (blending)
- Проход пост-альфа, используется для 3D наложений (overlays), которые должны произойти после всех основных проходов


## Компоненты рендеринга (Rendering components)

Следующие компоненты связаны с ренедингом (определены в Grpahics и UI библиотеках):
- Octree: пространственное разделение Drawables объектов для ускоренных запросов и поиска. Должен быть создан в качестве покомпонента для Scene
- Camera: описывает порт просмотра для рендеринга, включает параметры проекции (поле видимости (FOV[15]), ближняя и дальняя дистанции, перспектива/ортографическая проекция[16])
- Drawable: Базовый класс для любвх видимых объектов
- StaticModel: геометрия модели без текстуры/материала (non-skinned). Может менять уровень деталей (LOD) в зависимости от дистанции
- StaticModelGroup: несколько объектов рендерятся, выбраковываются (culling - полигоны исезают, если пропадают из видимости) и получают освещение как один объект
- Skybox: подкласс StaticModel, который не будет перемещаться
- AnimatedModel: геоматрия с текстурами и материалом, которая может выполнять скелетную анимацию и анимацию изменения вершин (vertex morph)
- AnimationController: запускает и контролирует анимацию
- BillboardSet: Группа направленных в камеру плакатов (billboards), которые могут иметь изменяющийся размр, углы вращения и координаты текстур
- ParticleEmmitter: подкласс класса BillboardSet который испускает частицы плакаты (particle billboards)
- Light: освещение сцены. Может создавать тени
- Terrain: рендерит высотную карту местности
- CustomGeometry: Рендерит определяемую во время работы программы неиндексируемую геометрию. Геометрия не сериализирована или копирована (replicated) по сети
- DecalSet: Рендерит переводную картинку (decal[17]) на 3D геометрии
- Zone: определяет свет окружения и настройки тумана для объектов в зоне
- Text3D: текст, который рендерится в 3D

Дополнительно есть еще 2D компоненты, определенные в Urho2D.

## Оптимизация

Следующие техники использованы программой для оптимизации использования ресурсов ЦПУ и ГПУ:

- Програмно расстеризованное отсечение (Software rasterized occlusion): после того, как видимые объекты запрашиваются по октодереву, объекты, которые помечены как отсеченные (occluders) рендерятся на ЦПУ в маленький буфер с глубинной иерархией, этот буфер будет использован для определения видимости не отсеченных объектов. Используйте SetMaxOccluderTriangles() и SetOccluderSizeThreshold() чтобы сконфигурировать рендеринг отсечения. Тест на отсечение всегда многотредовый, однако рендеринг отсечения - однотредовый, причина: чтобы разрешить убирание отсеченных объектов при рендеринге от приближенных объектов к удаленным. Используйте SetThreadedOcclusion() чтобы разрешить трединг так же и для рендеринга, хотя иногда это приводит к худшей производительности, например на сценах с картой местности, где части местности работают как отсекатели.

- Дублирование геометрии: операция рендеринга с подобной геометрией, материалом или светом будут сгруппированы вместе и выполнены за один вызов. Даже если подобное не поддерживается графической картой, все равно рендеринг на ЦПУ будет сделан подобным образом

- Трафаретное маскирование света: В прямом рендеринге, перед тем, как объекты будут освещены источником света и перерендерены с добавлением света, граница света будет отрендерена в буфер трафарета, чтобы отсечь места, куда свет не доходит

Заметим, что есть еще другие способы и возможности оптимизации использования ресурсов на уровне работы с контентом, например использование геометрии и материалов с разным уровнем детализации, группирование статических объектов в один объект, минимизация полигональной сетки (meshes), использование атласа текстур чтобы избежать изменения состояния, импользование скомпрессированных или маленьких текстур, установку максимальной дистанции для рисования объектов, света и теней.


## Использование подготовленного изображения для нескольких портов просмотра

В некоторых приложениях, например стереоскопический рендеринг для виртуальной реальности, требуется отрендерить слегка измененное изображение в отдельных портах просмотра (viewports). Обычно подобная операция выполняется для каждого порта просмотра отдельно, что конечно очень ресурсозатратно. 

Чтобы избежать дублирования операции, можно использовать **SetCullCamera()** метод чтобы указать Viewport использовать одну камеру для рендеринга, а другую для отбраковки (culling). Когда разные порты просмотра используют одну и ту же камеру для отбраковки, подготовка изображения произойдет только один раз.

Чтобы этот метод работал правильно, надо чтобы камера для отбраковки изображения покрывала изображение для всех портов одинаково, иначе не все объекты, предназначенные для скрытия будут скрыты. Так же не рекомендуется использовать масштабирование для камеры отбраковки, по причине описанной выше.


## Работа с потерей ГПУ ресурсов

При использовании Direct3D9 и Android OpenGL ES2.0 есть возможность потерять отрендеренный контекст (и ГПУ ресурсы), такое происходит, когда окно приложения минимизируется в фон. Эффект потери и воссоздания контекста также запускается при изменении оконного режима или переходе в полноэкранный режим (это сделано, чтобы обойти возможные баги драйверов). Так или иначе, разработчик должен быть подготовлен к потере ГПУ ресурсов.

Текстуры, которые были подгружены из файла, также как и вершины и буферы индексов, которые используются с разрешенным теневым сохраненем (shadowing enabled) будут восстановлены автоматически, остальное будет восстановленно в ручную. При использовании Direct3D9 нединамические текстуры и буферы не будут потеряны, так как система сохраняет их автоматически в память.

Для определения потерянного контекста существует функция IsDataLost() в классах VertexBuffer, IndexBuffer, Texture2D, TextureCube и Texture3D. Классы Model, BillboardSet и Font тоже обладают механизмом восстановления ресурса, таким образом имеет смысл проверять потерю ресурсов только для созданных буферов и текстур.  Особенно опасны потери ресурсов в индексируемых буферах, неиниуиализированные и не восстановленные данные в таких буферах могут вызвать поломку программы на уровне ГПУ драйвера.


## Комментарии:

[1] EXT_framebuffer_object - Определяет простой интерфейс для рендеринга в заданный буфер, отличный от обычных окон. Рендеринг будет просиходить в "framebuffer-attachable" изображения. Расширение открывает возможность так называемого рендеринга в текстуру (когда изображение генерируется не на экран, а в текстуру).

[2] EXT_packed_depth_stencil - Объединенный буфер трафарета (stencil) и глубины (depth). Трафарет использует 8 бит, глубина же задается 24 битами, таким образом в 32 бита можно записать и трафарет и глубину.

[3] EXT_texture_filter_anisotropic - Обычное отображение тестур изотропно (проецируется квадратное поле пикселов), такой подход приводит к размыванию изображения на поверхности находящейся под углом от наблюдателя. Это расширение помогает ввести уровень анизотропии.

[4] ARB_instanced_arrays - Для некоторых программ необходимо рисовать одни и те же объекты или группы похожих объектов (у которых общие вершины, примитивы и типы) множество раз. Расширение ARB_instanced_arrays позволяет ускорить подобные операции путем менне частых вызовов к API, также это расширение позволяет избегать дубликатов объектов в памяти.

[5] Deferred rendering - 2х уровневый процесс - 1. рисование непрозрачной (opaque) геометрии с записью необходимых для освещения аттрибутов (например, глубины, нормалей, цвета альбедо (albedo color, это характеристика отражательной способности поверхности), отраженного света и других свойств материала) в буферах экрана (обычно в 3х - 4х) 2. Для каждого источника света, рисование объема света (lightvolumes) и накапливание освещения поверхности конечной цели рендеринга (пример: CoreData\RenderPath\Defererred.xml).

[6] Light pre-pass - 3х уровневый процесс - 1. рисование непрозрачной (opaque) геометрии с записью необходимых для освещения аттрибутов (например, глубины, нормалей, цвета альбедо (albedo color, это характеристика отражательной способности поверхности), отраженного света и других свойств материала) 2. для каждого источника света, рисование объема света и накапливание только освещения поверхности в дополнительном буфере. 3. рисование непрозрачной геометрии опять, рассчет полного шейдера материала и наложение светового сэмпла из буфера сгенерированного на втором шаге (так называемый, forward shader).

[7] Octree - тип древовидной структуры данных, в которой у каждого внутреннего узла ровно 8 потомков.

[8] hall-of-mirrors artifacts TODO: надо приложить картинку

[9] http://urho3d.github.io/documentation/HEAD/_auxiliary_views.html

[10] MSAA - Мультисэмплтнговое сглаживание (сглаживание множественной выборки) является оптимизацией суперсэмплингового сглаживание (где изображение рендерится с большим разрешением, а потом уменьшается в размере). В мультисэплинговом сглаживании само изображение рендерится в нормальном разрешении, а глубина и трафарет изображения генерятся в суперсэмпле, а потом уменьшаются в размере. Большим недостатком как суперсэмплинга, так и мультисэмплинга является то, что сглаживается вся сцена, хотя сглаживание нужно как правило не везде. Этот недостаток убирают шейдерные алгоритмы сглаживания, например FXAA

[11] FXAA - FXAA (Fast approXimate Anti-Aliasing) — метод сглаживания от Nvidia, представляющий собой однопроходный пиксельный шейдер, который обсчитывает результирующий кадр на этапе постобработки. Является более производительным решением, по сравнению с традиционным MSAA (Multi-Sampling Anti-Aliasing), что, однако, сказывается на точности работы и качестве изображения

[12] camera's view frustum - часть пирамиды которая заключает в себе видимую часть сцены, начинается от ближнего отсечения (near clip) (по умолчанию 0.1) и до дальнего отсечения  (far clip) (по умолчанию 1000.0)

[13] G-buffer буфер геометрии, содержащий набор текстур состоящий из позиции, нормалей, материала для каждой поверхности, используется для рендеринга с задержкой, так как в случае рендеринга с задержкой никакого шейдинга не происходит в первом проходе, а вначале рассчитываются непрозрачные геометрии, нормали, глубины и прочее, а шейдеры накладываются на втором проходе

[14] alpha-blending или alpha compositing - процесс комбинирования изображения с фоном с целью создать частичную или полную прозрачность объекта

[15] FOV - поле видимости/зрения - угловое пространство, видимое глазом при фиксированном взгляде и неподвижной голове

[16] Orthographic projection - представление трехмерного объекта в двухмерном пространстве

[17] Decal - 2D изображение сгенерированное в 3D пространстве, например след от пули