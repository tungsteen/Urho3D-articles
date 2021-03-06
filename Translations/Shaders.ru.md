# Шейдеры (Shaders) в Urho3D
Эта статья - перевод [статьи] (http://urho3d.github.io/documentation/HEAD/_shaders.html) с моими комментариями

Urho3D использует uber-shader подход [1] в котором различные вариации каждого шейдера будут скомпилированы с различными параметрами, чтобы получить, например: статический шейдер или тестурированыый шейдер, ренедринг с задержкой, прямой рендеринг или рендеринг с тенью или без.

Компилирование таких шейдеров происходит по запросу: техника (technique) и пути рендеринга (renderpaths) определяют вызов шейдера и задание параметров к нему. В добавок, игровой движок добавит переменные в шейдер, которые связаны с типом геометрии и освещения. Как правило, невозможно зараннее перечислить все возможные варианты, которые будут откомпилированы для шейдера.

В Direct3D полученные после компиляции шейдера байткоды будут сохранены в поддиректорию "Cache" рядом с исходным кодом шейдера. В OpenGL подобной возможности нет.

## Встроенные параметры шейдера

Для ренедринга объекта сцены, игровой движок требует чтобы определенные варианты шейдера существовали для различных геометрий и условий света. При написании шейдера, для этой цели используются следующие параметры:

### Параметры шейдера вершин (Vertex Shaders)
- NUMVERTEXLIGHTS=1,2,3 или 4: число освещения падающего на каждую вершину объекта
- DIRLIGHT, SPOTLIGHT, POINTLIGHT: тип освещения падающего на вершину. Обычно идет в сочетании с PERPIXEL
- SHADOW: направленный на вершину свет создает тень
- NORMALOFFSET: UV координаты получателя тени должны быть отрегулированны согласно нормалям
- SKINNED [2], INSTANCED [3], BILLBOARD [4]: тип геометрии

### Параметры шейдера пикселей (Pixel Shaders)

- DIRLIGHT, SPOTLIGHT, POINTLIGHT: тип освещения падающего на пиксель. Обычно идет в сочетании с PERPIXEL
- CUBEMASK: кубическая маска для точечного света
- SPEC: от (specular) отражение для точечного света
- SHADOW: тень для точечного света
- SIMPLE_SHADOW, PCF_SHADOW, VSM_SHADOW: используемый уровень качества тени
- SHADOWCMP: сравнение глубины света заданной вручную, в Direct3D9 только для DF16 и DF24 формата теней
- HEIGHTFOG: режим тумана для зоны объект 

## Встроенные общие переменные (uniforms) для шейдеров

Когда объекта или четырехугольнии (quads [6]) рендерятся по проходам, различные встроенные переменные (uniforms) ипсользуются для передачи данных. Далее приведен список переменных для Direct3D HLSL шейдеров. Соответствующие параметры для GLSL могут быть найдены в файле Uniforms.glsl

### Униформы шейдера вершин (Vertex Shaders)

- float3 cAmbientStartColor: начальное значение света для градиента окружения зоны (ambient gradient)
- float3 cAmbientEndColor: конечное значение света для градиента окружения зоны
- float3 cCameraPos: позиция камеры в системе координат мира
- float cNearClip: расстояние до ближайщей плоскости отсечения (near clip plane)
- float cFarClip: расстояние до дальней плоскости отсечения (far clip plane)
- float cDeltaTime: временной шаг (timestep) для текущего кадра
- float4 cDepthMode: параметры для рассчета линейной глубины (от 0 до 1) чтобы передать в шейдер пикселя в режиме интерполяции
- float cElapsedTime: время прошедшее с момента рендеринга сцены. Как правило, используется для анимации материала
- float4x3 cModel: матрица трансформирования из системы координат модели в систему координат мира (model matrix [5])
- float4x3 cView: матрица камеры
- float4x3 cViewInv: инвертированная матрица камеры
- float4x4 cViewProj: объединенная матрица состоящая из произведения матрицы перехода из координат мира в координаты камеры и матрицы проекции трехмерных координта камеры в двухмерные координаты экрана (projection * view [5])
- float4x3 cZone: трансформированная матрица зоны: используется для расчета градинета окружения

### Униформы шейдера пикселей (Pixels Shaders)

- float3 cAmbientColor: цвет окружения, для зоны в которой градиент окружения не задан
- float3 cCameraPosPS: позиция камеры в системе координат мира
- float4 cDepthReconstruct: параметры для реконструкции значения линейной глубины (значение от 0 до 1) с ииспользованием нелинейной глубины (hardware depth) образца текстуры.
- float cDeltaTimePS: временной шаг (timestep) для текущего кадра
- float cElapsedTimePS: время прошедшее с момента рендеринга сцены. Как правило, используется для анимации материала
- float3 cFogColor: цвет тумана зоны
- float4 cFogParams: параметры расссчета тумана (см. Batch.cpp и Fog.hlsl для точного определения)
- float cNearClipPS: расстояние до ближайщей плоскости отсечения (near clip plane)
- float cFarClipPS: расстояние до дальней плоскости отсечения (far clip plane)

## Написание шейдеров
Шейдеры должны быть написаны в двух вариантах HLSL (для Direct3D) и GLSL (для OpenGL). Оба варианта шейдеров должны обладать одинаковой функциональностью (на сколько это возможно).

Для того, чтобы начать писать шейдеры, имеет смысл ознакомиться с наиболее простыми, уже написанными вариантами: Basic, Shadow и Unit. Заметьте, что эти (как и другие) шейдеры включают общие файлы Uniforms._lsl, Samplers._lsl & Transform._lsl для доступа к общим функциям.

Трансформирование вершин делается с применением трюка, который комбинирует макросы и функции. Самое простое, скопировать следующие куски кода в свой шейдер:

Для HLSL:
```c
float4x3 modelMatrix = iModelMatrix;
float3 worldPos = GetWorldPos(modelMatrix);
oPos = GetClipPos(worldPos);
```

Для GLSL:
```c
mat4 modelMatrix = iModelMatrix;
vec3 worldPos = GetWorldPos(modelMatrix);
gl_Position = GetClipPos(worldPos);
```

Как при использовании Direct3D так и OpenGL шейдеры вершин и пикселей записываются в один и тот же файл, функции входа должны называться VS() и PS(). В OpenGL одна из функций входа будет потом трансформирована в main(). При компиляции шейдера вершин COMPILEVS всегда задана, то же самое с COMPILEPS при компиляции пиксель шейдеров. Эти define макросы часто используются при включении нужных файлов для компиляции шейдера, таким образом можно включать и выключать нелегальные для данного этапа компиляции include директивы, а также уменьшать время компиляции.

Для правильного рендеринга, входные параметры шейдера верщин должны соответствовать семантике. В HLSL семантике, параметры входа определеяются для каждого шейдера используя слова в верхнем регистре (POSITION, NORMAL, TEXCOORD0 итд.), в то время как в GLSL, параметры по умолчанию заданы в Transform.glsl и проверяются оператором "contains", который игнорирует регистр, к тому же иногда добавляется число в конце. Например iTexCoord задает координаты первой текстуры (семантический индекс 0), а iTexCoord1 - второй (семантический индекс 1).

Униформы должы быть заданы следующим способом:
- префикс "c" должен быть использован для униформы-константы, например cMatDiffColor. При использовании того же параметра внутри движка (не в шейдере), префикс "с" не нужен, например MatDiffColor в *SetShaderParameter()*
- префикс "s" для текстурных сэмплов [7], например sDiffMap

В GLSL шейдерах важно, чтобы сэмплы были привязаны к правильным текстурным единицам. Если не используете предоставленные движком имена для сэмплов (такие как sDiffMap), то убедитесь, что в имени сэмпла содержится номер, так как этот номер будет использоваться в качестве номера текстурной единицы [8]. Например, шейдер местности может использовать текстурные единицы с номером от 0 до 3 следующим образом

```c
uniform sampler2D sWeightMap0;
uniform sampler2D sDetailMap1;
uniform sampler2D sDetailMap2;
uniform sampler2D sDetailMap3;
```

Максимальное количество костей (bones) поддерживаемых процессором зависит от графического API и опирается на шейдерный код в MAXBONES define. Обычно максимум - это 64, но для Raspberry Pi максимум - 32, для Direct3D и OpenGL 3 максимум - 128. Для дальнейшей информации см. *GetMaxBones()*.

## Разница в API
Direct3D9 и Direct3D11 используют один и тот же код из hlsl файлов, так же OpenGL 2, OpenGL 3, OpenGL ES 2 и WebGL используют один и тот же код из glsl файлов. Макросы и коды выполняющиеся при определенном условии используются для скрытия кода, там, где это необходимо.

При компиляции HLSL шейдеров под Direct3D11, define D3D11 всегда задан, и следующие детали должны быть учтены:
- Униформы организованы в постоянные буферы. См. Uniforms.hlsl. Так же см. Terrain.hlsl для примера определения нужной униформы в нужном постоянном буфере.
- Как текстуры, так и сэмплы заданы для каждой текстурной единицы (unit). Макрос Samplers.hlsl (Sample2D, SampleCube etc.) может быть использован для написания кода, который работает как под D3D9, так и под D3D11. Этот макрос получает текстурную единицу без префикса "s"
- Позиция, как параметр выхода из шейдера вершин и  цвет, как параметр выхода из шейдера пикселей могут использовать defines SV_POSITION и SV_TARGET для определения семантики. Макросы OUTPOSITION и OUTCOLOR0-3 так же могут буть использованы для выбоа нужной семантики при использовании этих API. В шейдере вершин, позиция выхода должна быть определена последней, иначе семантика выхода не будет работать корректно. В целом, необходимо чтобы семантика параметров выхода, определяемая в шейдере вершин, была определена в том же порядке, как и в шейдере пикселей. Иначе Direct3D может неправильно переопределить семантику.
- При использовании Direct3D11 плоскости отсечения должны быть рассчитаны вручную. Эта часть кода должна заключаться в define CLIPPLANE. См. например  LitSolid.hlsl
- Direct3D11 не поддерживает форматы тектстур с яркостью (luminance) и яркостью с альфой (luminance-alpha), но использует R и RG каналы. Таким образом, придется прибегнуть к трюку переключения (swizzling) при чтении текстуры.
- Direct3D11 не сможет отрендерить объект, если шейдер вершин содержит ссылку на элемент вершины, которого нет в буфере вершин.

Для OpenGL, define GL3 задан в GLSL шейдере во время компиляции под OpenGL3+, definr GL_ES задан при компиляции под OpenGL ES 2, define WEBGL для WebGL и RPI для Raspberry Pi. Ознакомьтесь со следующим списком различий:
- OpenGL3 GLSL версии 150 [7] будет использована по умолчанию (если в коде не указана другая версия). Функции для текстурных сэмплингов отличаются, используются функции из Samplers.glsl. Файл Transform.glsl содержит макросы для скрытия различий в определении свойств вершин, интерполяции и выходных параметров пиксельного шейдера.
- В OpenGL3 яркость, альфа и яркость с альфой форматы текстур считаются устаревшими и заменены на R и RG форматы. Таким образом, придется опять прибегнуть к трюку переключения при чтении текстур.
- В OpenGL ES2 следует заданнать квалификатор точности.

## Кэширование шэйдеров
Различные шейдеры, полученные при компиляции с учетом различной техники, условий освещения и проходов рендеринга, собираются вместе во время загрузки материала, но так как таких шейдеров становится очень много, то компиляция и загрузка из байткода нужных шейдеров выполняется только по мере необходимости. Замечено, что, например в OpenGL, компиляция большого числа шейдеров может приводить к заторможенному обновлению экрана. Чтобы избежать этого, комбинации нужных шейдеров можно записать в XML файл, затем подгрузить их перед использованием. Функции *BeginDumpShaders()*, *EndDumpShaders()*, *PrecacheShaders()* в графической системе призваны помочь в этом. Так же, праметр командной строки -ds <имя файла> может быть использован для кэширования шейдеров при запуске движка.

Заметьте, что использование различных вариаций шейдеров так же зависит от настроек графики, например от качества теней (simple/PCF/VSM) или включения функции instanced.



## Комментарии
[1] uber-shader подход - создание одного большого шейдера, оторый компилируется несколько раз с различными #define для различных нужд. Например #define USE_NORMAL_MAP будет включать фцнкции нормалей карты, или NUM_SPOTLIGHTS определит сколько исуточников света будет обработано шейдером. Проблема в таком подходе, код становится клубком из различных вызовов функций и становится сложным для поддержки и отладки.

[2] SKINNED - "кожанная" геометрия, названа так, потому что поверхность-"кожа" фактически "надета" поверх "палочной" фигуры-"скелета"

[3] INSTANCED - геометрия объекта будет подготовлена и сохранена в буфере, чтобы потом один и тот же объект размножить много раз при рендеринге (такой подход экономит время и использование памяти, при работе с множеством одинаковых объектов)

[4] BILLBOARD - "плакаты" - 2D объекты встроенные в 3D мир

[5] подробнее [здесь](http://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/)

[6] quads - GL_QUADS четырехугольники, яаляются наряду с треугольниками, точками и линиями так называемыми примитивами в OpenGL. Часто используются для рассчета освещения. Подробнее [здесь](https://www.ntu.edu.sg/home/ehchua/programming/opengl/CG_BasicsTheory.html).

[7] Спецификация OpenGL 3+ версии 150 дана [здесь] (https://www.opengl.org/registry/doc/GLSLangSpec.1.50.11.pdf)

[8] сэмплы текстур - это цветовая информация текстуры в двух мерной системе координат UV
