### 1

```
operation ApplyBasicGates() : Unit {
    // Создаём кубит
    using (q = Qubit()) {
        H(q);     // создаём суперпозицию
        Z(q);     // добавляем фазу
        X(q);     // инвертируем состояние
        Reset(q); // возвращаем в исходное состояние
    }
}
```

### 2

```
namespace SuperpositionTest {

    @EntryPoint()
    operation SuperpositionTest() : Result {
        use q = Qubit();
        H(q);                      // создаём суперпозицию (|0> → (|0> + |1>)/√2)
        let res = M(q);            // измеряем кубит
        Reset(q);                  // возвращаем кубит в исходное |0>
        
        return res;
    }
}
```

### 3

```
namespace RandomBit {

    @EntryPoint()
    operation Random(): Int {
        use q = Qubit();

        // перевод кубита в состояние суперпозиции
        H(q);

        // измерение кубита 
        let res = (M(q) == Zero) ? 0 | 1;

        Reset(q);
        
        return res;
    }
}
```

### 4, 5

```
namespace Teleport {

    import Std.Diagnostics.DumpMachine;

    @EntryPoint()
    operation Main() : Unit {
    let divider = "--------------------------";

    use (qAlice, qBob) = (Qubit(), Qubit());

    DumpMachine();
    Message(divider);

    Entangle(qAlice, qBob);
    DumpMachine();

    use qMessage = Qubit() {
        Message("Hello from Alice");

        SendMessage(qAlice, qMessage);

        Message(divider);
        DumpMachine();

        Reset(qAlice);
        Reset(qBob);
        Reset(qMessage);
    }
}
    
    // task 4
    operation Entangle(qAlice : Qubit, qBob : Qubit) : Unit {
        H(qAlice);
        CNOT(qAlice, qBob);
}
    // task 5
    operation SendMessage(qAlice : Qubit, qMessage : Qubit) : (Bool, Bool) {
        CNOT(qMessage, qAlice);
        H(qMessage);

    return (M(qMessage) == One, M(qAlice) == One);
}
}
```

### Выводы

Начал выполнение задания с прохождения основ на сайте https://quantum.microsoft.com/en-us/tools/quantum-katas

А именно: прошел курс по комплексным числам, в процессе прохождения изучил основы Q#.
B изучил вводный курс в кубиты.

Далее брал задачки из предоставленного репозитория, завел свой репозиторий с решением задачек.

Выполнение задания расширило кругозор и разожгло интерес к квантовому исчислению.
Работа с базовыми квантовыми воротами помогла лучше понять фундаментальный принцип квантовых вычислений - управление состоянием кубита и формирование суперпозиции. 

Пока что я в процессе изучения этой новой области и какие-то вещи могут пониматься неправильно.
Например, изначально не совсем корректно для себя определил значение кубита и наткнулся на сноску на курсе Microsoft:

>"A common misconception about quantum computing is that a qubit is always in state
or state
, and we just don't know which one until we "measure" it. That's not the case. A qubit in a superposition is in a linear combination of the states 0 and 1. When a qubit is measured, it's forced to collapse into one state or the other - in other words, measuring a qubit is an irreversible process that changes its initial state."
 
Также меня увлекла применимость комплексных чисел и линейной алгебры в квантовых вычислениях, такая увлеченность позволила восполнить некоторые пробелы в знаниях и прорешать задачи и по математике.

Буду стараться пройти полностью Katas и прорешать все задачки.