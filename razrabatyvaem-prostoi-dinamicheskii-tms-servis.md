# Разрабатываем простой динамический TMS-сервис

Библиотеке Mapnik посвящен ряд документации, расположенной на http://mapnik.org/docs, которая специально предназначена для разработчиков на Python. К сожалению, она довольно скудная и запутанная. Дело в том, что Python-ориентированная документация вытекает из документации для программистов на C++ и сфокусирована на описании того, каким образом реализованы привязки Python, а не на том, каким образом конечный пользователь будет работать с библиотекой, используя этот язык. Вытекающее из этого обилие технических подробностей, которые не относятся к программированию на Python, и отсутствие необходимого объема специфических для Python описаний делают ее не особо полезной.
Начинать работу с библиотекой Mapnik лучше всего, следуя инструкциям по ее установке и затем обратившись к двум предоставленным учебным руководствам. Затем можно просмотреть страницу Learning Mapnik («И зучаем библиотеку Mapnik») в вики-книге по библиотеке Mapnik ( https://github.com/mapnik/mapnik/wiki/LearningMapnik) - примечания на этих страницах довольно краткие и непонятные, но там имеется полезная информация, в случае если вы готовы потратить время, чтобы ее найти.

Также стоит обратить внимание на документацию по API Python, несмотря на ее ограниченность. На главной странице перечислены различные классы и ряд полезных функций, многие из которых задокументированы. В самих классах перечислены доступные методы и атрибуты, и даже при том, что при описании многих из них ощущается нехватка в Python-ориентированной документации, как правило, в их функционале можно разобраться из контекста. В этом руководстве частично будет описана работа с классами.

Инструменты для разработки:
- Django
- Mapnik3
- PostGIS

Подробнее о том, как работают TMS-сервисы, вы можете узнать в [Основы работы динамических TMS-сервисов](http://gis-lab.info/qa/dynamic-tms.html)

Инициализируем проект Django и подключим все необходимые модули в INSTALLED_APPS. Не забываем указать подключение к нашей PostGIS базе.

Согласно спецификации OGC URL тайла должен выглядеть следующим образом:

```
/<str:service>/1.0.0/<str:mapname>/<z>/<int:x>/<int:y>.png
```

По ссылке будут передаваться координаты и номер масштабного уровня для дальнейшего вычисления охвата тайла. Подробнее смотрите в [Основы конфигурирования тайловых сеток](http://wiki.gis-lab.info/w/%D0%9E%D1%81%D0%BD%D0%BE%D0%B2%D1%8B_%D0%BA%D0%BE%D0%BD%D1%84%D0%B8%D0%B3%D1%83%D1%80%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F_%D1%82%D0%B0%D0%B9%D0%BB%D0%BE%D0%B2%D1%8B%D1%85_%D1%81%D0%B5%D1%82%D0%BE%D0%BA)

В файле views.py напишем вью, которая будет принимать значения по ссылке и вызывать функцию utils.tms, которую мы опишем далее

```python
from django.http import HttpResponse
from rest_framework.views import APIView
from . import utils

class MyView(APIView):
    def get(self, request, *args, **kwargs):
        service= self.kwargs['service']
        z= float(self.kwargs['z'])
        x=self.kwargs['x']
        y=self.kwargs['y']
        output = utils.tms(z, x, y, service)
        return HttpResponse(bytes(output), content_type="image/png")

```
В функции utils.tms для расчета охвата тайла мы должны указать размер карты. Размер карты указывается в той же системе координат, что и данные, которые мы будем получать из базы данных. В нашем случае, для проекции 4326, мы указываем максимальный размер карты:

```python
	bbox = dict(minx=-180, miny=-90, maxx=180, maxy=90)
```
Далее по формулам рассчитаем охват будущего тайла в зависимости от типа тайловой сетки (xyz, tms) :

```python
	step = max(bbox['maxx'] - bbox['minx'], bbox['maxy'] - bbox['miny']) / 2 ** z
    extents = dict()
    extents['tms'] = (
        bbox['minx'] + x * step,
        bbox['miny'] + y * step,
        bbox['minx'] + (x + 1) * step,
        bbox['miny'] + (y + 1) * step
    )   
    
    extents['xyz'] = (
        bbox['minx'] + x * step,
        bbox['maxy'] - (y + 1) * step,
        bbox['minx'] + (x + 1) * step,
        bbox['maxy'] - y * step
    )
```
Экспериментально было установлено, что оптимальный размер тайла 256х256 пикселей. Далее создадим наш тайл с заданными размерами:

```python
	tile = dict(width=256, height=256)
    map = mapnik.Map(tile['width'], tile['height'])
    map.background = mapnik.Color('steelblue')
```

И подготовим его к отрисовке, задав ограничительную рамку и загрузив стили из style.xml:

```python
	mapnik.load_map(map, 'style/style.xml')
    box = mapnik.Box2d(*extents.get(service))
    map.zoom_to_box(box)
    mapnik.render_to_file(map, 'world.png', 'png') #сохранение тайла в файл просто для тестов
    im = mapnik.Image(map.width, map.height) #создаем объект картинки, в который будет отрисовываться наш тайл
    mapnik.render(map, im)
    output = im.tostring('png')
    return output #отдаем клиенту картинку в байтах
```
style.xml с описанием стилей и слоев
```xml
<?xml version="1.0" encoding="utf-8"?>
<Map srs="+init=epsg:4326" background-color="transparent">
    <Style name="Admin style">
        <Rule>
            <PolygonSymbolizer fill="#ffffff"/>
            <LineSymbolizer stroke="#85c5d3" stroke-width="2" stroke-linejoin="round" />
        </Rule>
    </Style>
    <Style name="Ecoregions style">
        <Rule>
            <PolygonSymbolizer fill="pink"/>
            <LineSymbolizer stroke="black" stroke-width="0.5" stroke-linejoin="round" />
        </Rule>
    </Style>
     <Style name="Roads style">
        <Rule>
            <LineSymbolizer stroke="red" stroke-width="0.5" smooth="0.1"/>
        </Rule>
    </Style>
     <Style name="Points style">
        <Rule>
            <PointSymbolizer/>
        </Rule>
    </Style>
     <Layer name="poly" srs="+init=epsg:4326">
        <StyleName>Ecoregions style</StyleName>
        <Datasource>
            <Parameter name="dbname">mrsk</Parameter>
            <Parameter name="geometry_field">geom</Parameter>
            <Parameter name="host">127.0.0.1</Parameter>
            <Parameter name="password">tO8Qa3ng</Parameter>
            <Parameter name="table">mrsk_area_energy</Parameter>
            <Parameter name="type">postgis</Parameter>
            <Parameter name="user">postgres</Parameter>
        </Datasource>
    </Layer>
    <Layer name="points" srs="+init=epsg:4326">
        <StyleName>Points style</StyleName>
        <Datasource>
            <Parameter name="dbname">mrsk</Parameter>
            <Parameter name="geometry_field">geom</Parameter>
            <Parameter name="host">127.0.0.1</Parameter>
            <Parameter name="password">tO8Qa3ng</Parameter>
            <Parameter name="table">mrsk_accident_avar</Parameter>
            <Parameter name="type">postgis</Parameter>
            <Parameter name="user">postgres</Parameter>
        </Datasource>
    </Layer>
     <Layer name="lines" srs="+init=epsg:4326">
        <StyleName>Roads style</StyleName>
        <Datasource>
            <Parameter name="dbname">mrsk</Parameter>
            <Parameter name="geometry_field">geom</Parameter>
            <Parameter name="host">127.0.0.1</Parameter>
            <Parameter name="password">tO8Qa3ng</Parameter>
            <Parameter name="table">mrsk_communication_line</Parameter>
            <Parameter name="type">postgis</Parameter>
            <Parameter name="user">postgres</Parameter>
        </Datasource>
    </Layer>
</Map>
```
Если вы не хотите использовать xml-стили, то задавать стили можно непосредственно в коде:
```python
	#...
	style = mapnik.Style()
    rule = mapnik.Rule()
    point_symbolizer= mapnik.PointSymbolizer()
    rule.symbols.append(point_symbolizer)

    line_symbolizer = mapnik.LineSymbolizer()
    line_symbolizer.stroke = mapnik.Color('red')
    line_symbolizer.stroke_width = 1
    rule.symbols.append(line_symbolizer)  # добавить символ в объект правила
    style.rules.append(rule)  # теперь добавить правило в стиль, и мы закончили
    map.append_style('My Style', style)

    layer = mapnik.Layer(mapname)
    ds = mapnik.PostGIS(host='127.0.0.1',
                        dbname='mrsk',
                        user='postgres',
                        password='pass',
                        table=mapname)
    layer.datasource = ds
    layer.styles.append('My Style')
    map.layers.append(layer)
    #...
```

Для представления результатов работы TMS-сервиса будем использовать библиотеку OpenLayers. Создадим файл index.js и напишем следующий код:
```javascript
 var map = new ol.Map({
    target: 'map',
    layers: [

      new ol.layer.Tile({
        source: new ol.source.OSM(),
      }),
      new ol.layer.Tile({
        source: new ol.source.XYZ({
            url: 'http://localhost:8000/myview/xyz/1.0.0/land/{z}/{x}/{y}.png',
            projection: 'EPSG:4326'
        }),
          extent: ol.proj.transformExtent([38.18, 44.65, 50, 51.28], 'EPSG:4326', 'EPSG:3857'),
      }),

    ],
    view: new ol.View({
      center: ol.proj.fromLonLat([37.41, 44.82]),
      zoom: 5
    })
  });
```
Запускаем python manage.py runserver и тестируем в браузере. Во избежание загрузки огромного количества пустых тайлов, мы указали параметр extent, который определяет охват слоя.