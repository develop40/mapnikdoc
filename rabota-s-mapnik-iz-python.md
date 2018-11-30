# Быстрый старт



**Mapnik** поддерживает два способа конфигурирования - с помощью команд Python и с помощью специального \*.xml файла.

### Первый способ

#### Импорт Mapnik

Импортируем связку mapnik-python

```python
import mapnik
```

#### Создание карты

```python
map = mapnik.Map(600, 300) #создание карты с заданной шириной и высотой
map.background = mapnik.Color('steelblue') #зададим цвет фона
```

#### Создание стилей

Создадим стили, которые будут определять способ  отображения данных:

```python
style = mapnik.Style() #объект стиля для хранения правил отображения
rule = mapnik.Rule() #объект хранения правил отображения
```

Для отображения разных типов геометрий используется Symbolizer:

```python
polygon_symbolizer = mapnik.PolygonSymbolizer()
polygon_symbolizer.fill = mapnik.Color('#f2eff9') #зададим цвет заливки для полигона
rule.symbols.append (polygon_symbolizer) #добавляем symbolizer к правилам
#чтобы добавить контуры к полигону, создадим LineSymbolizer
line_symbolizer = mapnik.LineSymbolizer()
line_symbolizer.stroke = mapnik.Color('rgb(50%, 50%, 50%)') #задаем цвет линии
line_symbolizer.stroke_width =  0,1 #толщина линии в пикселях
rule.symbols.append(line_symbolizer)
```

Подробнее о том, какие Symbolizer поддерживает Mapnik, вы найдете в [SupportSymbology](https://github.com/mapnik/mapnik/wiki/SymbologySupport).

Теперь добавим правило в стиль, который мы объявили ранее:

```python
style.rules.append(rule)
```

И добавим стиль к карте:

```python
map.append_style ('My Style', style) #обязательно указав имя стиля для карты
```

#### Создание источника данных

Полный список форматов, которые может извлекать Mapnik представлен по ссылке: [MapnikPluginArchitecture](https://github.com/mapnik/mapnik/wiki/PluginArchitecture)

В примере ниже мы будем извлекать данные из базы данных PostGIS.

```python
ds = mapnik.PostGIS(host='127.0.0.1',
                    dbname='mydb',
                    user='postgres',
                    password='pass',
                    table='spatial_table')
```

Примечание: чтобы узнать охват своих данных, можно вызвать метод `envelope()` , который вернет полные координаты границ данных в системе координат таблицы.

#### Создание слоев

«layer.srs» является исходной проекцией источника данных и _должен_ соответствовать проекции координат этих данных, иначе ваша карта, скорее всего, будет пустой. Mapnik использует строки [Proj.4](http://trac.osgeo.org/proj/wiki/FAQ) для указания системы пространственных ссылок. В этом случае по умолчанию `srs`Mapnik предполагает \( `+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs`\), что соответствует данным. Если это не так, вы должны установить для параметра layer.srs правильное значение. Например:

```python
layer = mapnik.Layer('world')
layer.srs = '+init=epsg:4326'
```

Теперь мы можем присоединить источник данных к слою и укажем, чтобы созданный ранее стиль карты применится к слою:

```python
layer.datasource = ds
layer.styles.append('My style')
```

#### Рендерим карту в файл

Добавим ранее созданный слой на карту и отрендерим карту в файл:

```python
map.layers.append (layer) 
map.zoom_all ()#определяет совокупную протяженность всех слоев, прикрепленных к карте
mapnik.render_to_file(map, 'filename.png', 'png')
```

После запуска скрипта будет создан файл filename.png с отрисованной геометрией из вашей базы данных.

#### Второй способ

Второй способ заключается в использовании таблицы стилей XML. Для начала нам понадобится скрипт python, который устанавливает основные параметры карты и указывает какой XML использовать для стилей. 

```python
import mapnik
stylesheet =  'style.xml' 
image =  'world_style.png' 
map = mapnik.Map(600, 300)
mapnik.load_map (map, stylesheet)
map.zoom_all() 
mapnik.render_to_file (map, image)
```

Затем создадим сам «style.xml»

```markup
<?xml version="1.0" encoding="utf-8"?>
<Map srs="+init=epsg:4326" background-color="steelblue">
    <Style name="Admin style">
        <Rule>
            <PolygonSymbolizer fill="#ffffff"/>
            <LineSymbolizer stroke="#85c5d3" stroke-width="2" stroke-linejoin="round" />
        </Rule>
    </Style>
    <Layer name="admin" srs="+init=epsg:4326">
        <StyleName>Admin style</StyleName>
        <Datasource>
            <Parameter name="dbname">mydb</Parameter>
            <Parameter name="geometry_field">geom</Parameter>
            <Parameter name="host">127.0.0.1</Parameter>
            <Parameter name="password">pass</Parameter>
            <Parameter name="table">spatial_table</Parameter>
            <Parameter name="type">postgis</Parameter>
            <Parameter name="user">postgres</Parameter>
        </Datasource>
    </Layer>
</Map>
```

Получаем тот же результат, что и в первом случае.

