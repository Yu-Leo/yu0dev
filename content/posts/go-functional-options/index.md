---
title: Паттерн "Functional Options" в Go
slug: go-functional-options
date: 2023-11-20
tags:
  - go
---
Паттерн **"Functional Options"** ("Функциональные параметры") в Go - что это такое и ~~с чем его едят~~ как его применять? Давайте разбираться!

Рассмотрение данного паттерна я начну с примера, основанного на объектах реального мира, поскольку он будет наиболее понятен новичкам. После чего приведу пример практического применения этого подхода в контексте Backend-разработки на Go.
## Пример
### Постановка задачи
Предположим, что нам понадобилось создать структуру, хранящую информацию о конкретном автомобиле. Нас будет интересовать следующий набор параметров:
- Марка (`string`)
- Тип кузова: кабриолет/нет (`bool`)
- Объём двигателя (`float64`)
- Расход топлива в л/100км (`float64`)
- Владельцы (`[]string`)

```go
type Car struct {
	brand           string
	convertible     bool
	engineCapacity  float64
	fuelConsumption float64
	owners          []string
}
```

Перейдём к созданию экземпляров данной структуры.
### Первый вариант
```go
car := &Car{"Lada", false, 1.4, 7.4, []string{}}
```

Основным минусом такого способа является зависимость порядка значений от порядка объявления полей в структуре. Следствия:
1) При каждом создании нового экземпляра необходимо думать о том, в каком порядке были объявлены поля. Крайне высока вероятность ошибиться. Например, передать сначала значение поля `fuelConsumption`, а затем `engineCapacity`, а не наоборот, как требуется.
2) Если в начало/середину списка полей в объявлении структуры добавится новое поле, в каждом создании экземпляра необходимо будет менять порядок значений.
### Второй вариант
Конечно, можно создавать экземпляры структуры с указанием названий полей перед значениями. Код будет выглядеть следующим образом:
```go
car := &Car{
	brand: "Lada",
	convertible: false,
	engineCapacity: 1.4,
	fuelConsumption: 7.4,
	owners: []string{},
}
```

В таком случае мы избавляемся от минусов первого варианта. Однако этот способ по-прежнему имеет недостатки. Если необходимо, чтобы одно или несколько полей этой структуры имели значения по умолчанию, придётся дублировать их при каждом создании экземпляров данной структуры.
### Третий вариант
Напишем конструктор структуры `Car`.

Конечно, в Go нет встроенных конструкторов для структур, в отличие от того же Python с его ООП и методом `__init__`. Однако по общепринятым правилам конструктором структуры в Go считается ф-ция вида  `func NewType(<агрументы>) *Type` (где `Type` - имя структуры), возвращающая указатель на созданный объект. В случае, если пакет содержит всего одну структуру, название функции можно сократить с `NewType` до `New`.

В нашем случае такая функция-конструктор будет выглядеть следующим образом:
```go
func NewCar(brand string, convertible bool,
			engineCapacity, fuelConsumption float64, owners []string) *Car {
	return &Car{
		brand:           brand,
		convertible:     convertible,
		engineCapacity:  engineCapacity,
		fuelConsumption: fuelConsumption,
		owners:          owners,
	}
}
```
Создание экземпляра:
```go
car := NewCar("Lada", 1.4, 7.4, []string{})
```
Если мы захотим, чтобы поле `owners` по умолчанию было пустым слайсом, конструктор можно будет изменить следующим образом:
```go
func NewCar(brand string, convertible bool,
			engineCapacity, fuelConsumption float64) *Car {
	return &Car{
		brand:           brand,
		convertible:     convertible,
		engineCapacity:  engineCapacity,
		fuelConsumption: fuelConsumption,
		owners:          []string{},
	}
}
```
Создание экземпляра изменится на:
```go
car := NewCar("Lada", 1.4, 7.4)
```

Помимо этого конструктор может быть полезен в том случаях, когда перед непосредственным созданием экземпляра структуры необходима дополнительная обработка параметров, а так же когда после создания экземпляра структуры необходима валидация его полей. Прописанная один раз в конструкторе логика будет выполняться при каждом создании объекта.

Казалось бы, всё отлично: конструктор написан, объекты создаются, мы счастливы. Однако данный вариант опять имеет несколько недостатков.

1) Мы вернулись к проблеме, от которой уже пытались уйти. Если ранее порядок передаваемых значений зависел от порядка объявления полей в структуре, то теперь он зависит от сигнатуры функции-конструктора.
2) Если спустя какое-то время мы захотим добавить ещё одно поле в структуру `Car`, нам необходимо будет не только поменять саму структуру и сигнатуру функции-конструктора, но и добавить значение нового параметра в **каждом** существующем вызове конструктора.
### Используем Functional Options
Изменим функцию-конструктор и допишем следующий код:
```go
func NewCar(opts ...CarOption) *Car {
	c := &Car{
		brand: "",
		convertible: false,
		engineCapacity: 0.0,
		fuelConsumption: 0.0,
		owners: []string{},
	}
	for _, opt := range opts {
		opt(c)
	}
	return c
}

type CarOption func(*Car)

func WithBrand(brand string) CarOption {
	return func(c *Car) {
		c.brand = brand
	}
}

func WithConvertible(convertible string) CarOption {
	return func(c *Car) {
		c.convertible = convertible
	}
}

func WithEngineCapacity(engineCapacity float64) CarOption {
	return func(c *Car) {
		c.engineCapacity = engineCapacity
	}
}

func WithFuelConsumption(fuelConsumption float64) CarOption {
	return func(c *Car) {
		c.fuelConsumption = fuelConsumption
	}
}

func WithOwners(owners []string) CarOption {
	return func(c *Car) {
		c.owners = owners
	}
}
```
Код создания экземпляра структуры, очевидно, также изменится:
```go
car := NewCar(WithBrand("Lada"),
			  WithConvertible(true),
			  WithEngineCapacity(1.4),
			  WithFuelConsumption(7.4),
			  WithOwners([]string{"Василий", "Пётр"}))
```
Помимо этого можно написать функции, задающие полям структуры какие-либо часто используемые, но отличные от дефолтных значения.

Например, мы можем создать функцию, которая будет устанавливать значение поля `convertible` в `true`:
```go
func Convertible() CarOption {
	return func(c *Car) {
		c.convertible = true
	}
}
```
После чего код создания нового экземпляра структуры изменится на:
```go
car := NewCar(WithBrand("Lada"),
			  Convertible(),
			  WithEngineCapacity(1.4),
			  WithFuelConsumption(7.4),
			  WithOwners([]string{"Василий", "Пётр"}))
```
#### Объяснение
При использовании паттерна "Functional Options" все или часть параметров в функции-конструкторе заменяется с "сырых" значений типов `string`, `bool` и т. д. на функции, которые устанавливают значения соответствующих полей.

В конструкторе сначала создаётся экземпляр структуры со значениями полей по умолчанию, после чего на него "применяются" все переданные функции.
#### Преимущества
1) Возможность инициализировать все или часть полей структуры значениями по умолчанию и передавать в функцию-конструктор только те параметры, значения которых отличаются от дефолтных.
2) После добавления нового поля в структуру необходимо будет поменять только функцию-конструктор, задав в ней дефолтное значение для нового поля. При этом все существующие вызовы функции-конструктора не "сломаются" и их не нужно будет исправлять, если добавленное поле имеет "адекватное" значение по умолчанию.
3) По коду вызова функции-конструктора легко понять, к какому полю относится каждое переданное значение.
4) При изменении порядка полей в структуре не потребуется править никакой иной код, помимо, собственно, самого объявления структуры.
#### Недостатки
Очевидным недостатком использования данного подхода является необходимость создания функций, устанавливающих значения отдельным полям структуры, что приводит к увеличению общего количества кода в системе.
## Применение на практике
Выше я привел простой для понимания, но отчасти искусственно созданный пример. Теперь же рассмотрим применение паттерна "Functional Options" в контексте разработки Backend-сервисов.

Создание HTTP API сервисов - довольно частый случай использования языка Go. Поэтому со временем в комьюнити разработчиков выработались определённые подходы к проектированию архитектуры таких проектов и, как следствие, целые шаблоны репозиториев.

Один из таких шаблонов: https://github.com/evrone/go-clean-template.

В этой статье я не буду рассматривать вопросы, связанные с архитектурой backend-сервисов на Go, однако порекомендую вам ознакомиться содержимым данного репозитория после прочтения этой статьи.

Сейчас нас будет интересовать лишь директория [`httpserver`](https://github.com/evrone/go-clean-template/tree/master/pkg/httpserver). В ней располагаются два файла: 
- [`server.go`](https://github.com/evrone/go-clean-template/blob/master/pkg/httpserver/server.go) - описание структуры `Server`, являющейся оберткой над структурой [`http.Server`](https://pkg.go.dev/net/http#Server).
- [`options.go`](https://github.com/evrone/go-clean-template/blob/master/pkg/httpserver/options.go) - параметры структуры `Server`.

Как можно заменить, в этих файлах присутствует реализация паттерна "Functional Options".

В функцию-конструктор [`New`](https://github.com/evrone/go-clean-template/blob/master/pkg/httpserver/server.go#L25) передается один обязательный параметр `handler` и несколько опциональных, которые могут задавать таймауты для запросов и адрес сервера. Если какой-либо из опциональных параметров не передан, соответствующее поле будет инициализировано значением по умолчанию, которое описано в [отдельном блоке](https://github.com/evrone/go-clean-template/blob/master/pkg/httpserver/server.go#L10C1-L15C2).

В файле [`app.go`](https://github.com/evrone/go-clean-template/blob/master/internal/app/app.go) приведён пример создания  экземпляр структуры `Server`:
```go
httpServer := httpserver.New(handler, httpserver.Port(cfg.HTTP.Port))
```
Если же нам понадобится создать объект сервера, имеющий значения таймаутов, отличные от дефолтных, мы можем сделать это следующим образом:
```go
httpServer := httpserver.New(handler,
							 httpserver.Port(cfg.HTTP.Port),
							 httpserver.ReadTimeout(1 * time.Second),
							 httpserver.WriteTimeout(2 * time.Second))
```
## Кратко
```go
package main

type SomeStruct struct {
	requiredField  string
	optionalField1 string
	optionalField2 string
}

type SomeStructOption func(*SomeStruct)

func NewSomeStruct(requiredField string, opts ...SomeStructOption) *SomeStruct {
	s := &SomeStruct{
			requiredField: requiredField,
			optionalField1: "default value 1",
			optionalField2: "default value 2",
		 }
	for _, opt := range opts {
		opt(s)
	}
	return s
}

func WithOptionalField1(optionalField1 string) SomeStructOption {
	return func(s *SomeStruct) {
		s.optionalField1 = optionalField1
	}
}

func WithOptionalField2(optionalField2 string) SomeStructOption {
	return func(s *SomeStruct) {
		s.optionalField2 = optionalField2
	}
}

func OptionalField1ValueA() SomeStructOption {
	return func(s *SomeStruct) {
		s.optionalField1 = "value a"
	}
}

func OptionalField1ValueB() SomeStructOption {
	return func(s *SomeStruct) {
		s.optionalField1 = "value b"
	}
}

func main() {
	s1 := NewSomeStruct("some value")

	s2 := NewSomeStruct("some value", WithOptionalField1("another value 1"))

	s3 := NewSomeStruct("some value", WithOptionalField1("another value 1"),
						WithOptionalField2("another value 2"))

	s4 := NewSomeStruct("some value", OptionalField1ValueA())

	s5 := NewSomeStruct("some value", OptionalField1ValueB(),
						WithOptionalField2("another value 3"))
}
```
## Выводы
На первый взгляд любой паттерн может показаться "волшебной таблеткой", однако на практике это далеко не так. В каждой ситуации необходимо отдельно оценивать целесообразность применения тех или иных подходов. Каким бы замечательным и удобным не казался изученный вами паттерн, вовсе не обязательно использовать его в каждом своём проекте. "Functional Options" - не исключение.
### Когда следует задуматься о применении паттерна "Functional Options":
- Все или часть полей структуры опциональные; имеются "адекватные" значения по умолчанию, которые могут использоваться в создаваемых объектах.
- Все или часть полей структуры имеют несколько часто используемых при создании экземпляров значений.
- Структура находится в составе библиотеки, при этом в будущих версиях предполагается расширение кол-ва её полей (и, как следствие, кол-ва значений, которые будут передаваться при создании экземпляров данной структуры).
### Когда не следует применять паттерн "Functional Options":
- Структура имеет не большое число полей.
- Все поля структуры обязательны.
- Значения полей структуры часто различаются у различных её экземпляров.