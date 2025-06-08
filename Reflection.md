# Reflection

***Рефлексия*** - механизм, позволяющий программам инспектировать свои структуры и модифицировать поведение в runtime. Этот механизм реализован во многих языках программирования, в Go для рефлексии используется пакет `reflect`.

## Введение в пакет reflect

Пакет `reflect` имеет интерфейсы и функции для динамического анализа типов и значений на стадии выполнения программы. В `reflect` активно используются понятия `Type` и `Value`:

- `Type`: представляет описание типа в Go. С помощью `reflect.Type` можно узнать о характеристиках типа, таких как его имя, размер, количество полей (если это структура), и т.п.

- `Value`: представляет собой значение переменной. С помощью reflect.Value можно получить и изменять данные, хранящиеся в переменной.

В этом примере представлены базовые методы из `reflect`:

```golang
type Todo struct {
	ID   uint
	Task string
}

func (t *Todo) Noop() {
	fmt.Println("Noop")
}

func main() {
	todo := Todo{1, "Hello"}

	var somebody interface{}
	somebody = &todo

	// Получение типа
	todoType := reflect.TypeOf(somebody)
	// Получение значения
	todoValue := reflect.ValueOf(somebody)

	fmt.Println("Тип:", todoType)
	fmt.Println("Значение:", todoValue)

    // Kind - это перечисление со всеми базовыми типами Go
	fmt.Println(todoType)
	if todoType.Elem().Kind() != reflect.Struct {
		fmt.Println("Это не структура!")
		return
	}

	// Получить поля
	fmt.Println("Количество полей:", todoType.Elem().NumField())
	for i := 0; i < todoType.Elem().NumField(); i++ {
		fmt.Println("Поле:", todoType.Elem().Field(i))
	}

	// Установить значение
	nameFieldValue := todoValue.Elem().FieldByName("Task")
	if nameFieldValue.CanSet() {
		nameFieldValue.SetString("Goodbye")
	}

	fmt.Println("Новое значение", todo)

	// Работа с методами
	fmt.Println("Количество методов:", todoType.NumMethod())
	for i := 0; i < todoType.NumMethod(); i++ {
		method := todoType.Method(i)
		fmt.Println("Метод:", method)

		// Вызвать метод
		methodValue := todoValue.Method(i)
		args := []reflect.Value{}
		methodValue.Call(args)
	}
}
```

Рефлексия мощный инструмент, она позволяет, например:

1) Реализовать считывание тегов структур
2) Вызвать приватный метод
3) Динамически работать с объектами

Однако ее неправильное использование чревато паниками в приложении и в целом ухудшению производительности приложения.

## Внутренняя реализация рефлексии

Компилятор анализирует исходный код и для каждого типа генерирует метаданные, которые включают:

- Размер типа (size).
- Список методов (methods).
- Структуру полей (для struct).
- Дополнительные флаги (например, является ли тип каналом, слайсом и т.д.).

Эти метаданные попадают в секцию `.rodata` (read-only data) исполняемого файла. Когда приложение запускается, метаданные загружаются в память, и runtime использует их для работы `reflect`.

Runtime оперирует структурой `_type` из файла [/src/runtime/type.go](https://github.com/golang/go/blob/master/src/runtime/type.go#L26C1-L27C1), которая является псевдонимом `abi.Type`:

```golang
// https://github.com/golang/go/blob/master/src/internal/abi/type.go#L20C1-L20C5

type Type struct {
	Size_       uintptr     // размер типа
	PtrBytes    uintptr     // кол-во байт с указателями
	Hash        uint32      // хеш типа
	TFlag       TFlag       // флаги типа
	Align_      uint8       // выравнивание переменной
	FieldAlign_ uint8       // выравнивание поля структуры
	Kind_       Kind        // идентификатор типа
	Equal func(unsafe.Pointer, unsafe.Pointer) bool // функция сравнения
	GCData    *byte         // данные для сборщика мусора (битовая маска указателей)
	Str       NameOff       // имя типа
	PtrToThis TypeOff       // тип указателя на этот тип (может быть 0)
}
```

`reflect.Type` — это интерфейс, который внутри содержит указатель на `runtime._type`:

```golang
// https://github.com/golang/go/blob/master/src/runtime/type.go#L29-L31

// rtype - обертка, позволяющая определять дополнительные методы.
type rtype struct {
	*abi.Type
}
```

Когда вызывается `reflect.TypeOf(x)`, Go извлекает `*rtype` из `interface{}` (через `eface`).

> Для составных типов (struct, map, func) метаданные расширяются дополнительными структурами.

## Утечки памяти

При вызове метода `ValueOf` значение `_type` берётся из `eface`, а данные копируются или сохраняются через указатель:

```golang
// https://github.com/golang/go/blob/master/src/reflect/value.go#L154-L166

func unpackEface(i any) Value {
	e := (*emptyInterface)(unsafe.Pointer(&i))
	// NOTE: don't read e.word until we know whether it is really a pointer or not.
	t := e.typ
	if t == nil {
		return Value{}
	}
	f := flag(t.Kind())
	if t.IfaceIndir() {
		f |= flagIndir
	}
	return Value{t, e.word, f}
}
```

Здесь `unsafe.Pointer` используется для доступа к внутренностям интерфейса, но сам по себе он не игнорируется GC. Однако `reflect.Value` сохраняет ссылку на переданные данные, и если это значение остаётся в глобальной переменной, то исходный объект не будет освобождён.

```golang
var globalVar reflect.Value

func init() {
	data := make([]byte, 100<<20)     // 100 MB
	globalVar = reflect.ValueOf(data) // data теперь "залипла" в памяти
}
```

Здесь `data` не освободится, потому что `globalVar` хранит копию структуры слайса, включая указатель на массив в куче. Это не баг, а особенность работы `reflect` — он явно сохраняет ссылки, чтобы данные не исчезли неожиданно. Если `reflect.Value` остаётся в долгоживущей переменной, то переданные в неё большие объекты могут удерживаться в памяти неопределённо долго.

## Плохая производительность

Рассмотрим этот бенчмарк:

```golang
// Прямой доступ к полю
func BenchmarkDirectFieldAccess(b *testing.B) {
	t := &Todo{ID: 1, Task: "test"}
	for i := 0; i < b.N; i++ {
		_ = t.ID
	}
}

// Доступ к полю через рефлексию
func BenchmarkReflectFieldAccess(b *testing.B) {
	t := &Todo{ID: 1, Task: "test"}
	val := reflect.ValueOf(t).Elem()
	field := val.FieldByName("ID")
	for i := 0; i < b.N; i++ {
		_ = field.Uint()
	}
}

// Прямой вызов метода
func BenchmarkDirectMethodCall(b *testing.B) {
	t := &Todo{ID: 1, Task: "test"}
	for i := 0; i < b.N; i++ {
		t.Noop()
	}
}

// Вызов метода через рефлексию
func BenchmarkReflectMethodCall(b *testing.B) {
	t := &Todo{ID: 1, Task: "test"}
	val := reflect.ValueOf(t)
	method := val.MethodByName("Noop")
	for i := 0; i < b.N; i++ {
		method.Call(nil)
	}
}
```

Результаты на MacBook Pro M1 Pro:

```bash
goos: darwin
goarch: arm64
BenchmarkDirectFieldAccess-8            1000000000               0.3173 ns/op
BenchmarkReflectFieldAccess-8           1000000000               0.5222 ns/op
BenchmarkDirectMethodCall-8             1000000000               0.3183 ns/op
BenchmarkReflectMethodCall-8            11703524               101.6 ns/op
PASS
ok      command-line-arguments  2.775s
```

Доступ полям к 1,6 раз медленее, вызов методов в 300 раз! 

> На самом деле, этот результат является очень хорошим, так как reflect Go для ARM архитектуры очень хорошо оптимизирован. Для x86 результаты могут быть хуже.

## Альтернативы

[Сами разработчики Go](https://go.dev/blog/laws-of-reflection) рекомендуют использовать рефлексию как последнее средство. Сейчас лучшим решением будет использовать дженерики в таких ситуациях. Также можно воспользоваться кодогенерацией и в крайнем случае `unsafe.Pointer`.

## Источники

- [Правила рефлексии](https://go.dev/blog/laws-of-reflection)
- [Исходный код Go 1.24](https://github.com/golang/go/releases/tag/go1.24.2)