## S02E10. Ассемблер x86, загрузка ОС

### Необходимое ПО

Для работы всем понадобится эмулятор компьютера **QEMU**; пожалуйста, установите его для своей операционной системы (на сайте QEMU описана установка для Linux, Mac и Windows).

Ещё нам понадобится собственно ассемблер — установите **nasm** из вашего менеджера пакетов.

(Как пользоваться менеджерами пакетов, я объяснял на первом модуле — это brew для macOS, apt для Ubuntu/Linux, и pacman/msys2 для Windows.)

### Ассемблер x86

Существуют множество процессоров и их архитектур (а существовало ещё больше), но по ряду причин в 2019 году мы можем столкнуться на практике с двумя: **x86** и **ARM**. Первая придумана американской компанией Intel, и на ней работают процессоры в ваших десктопах, ноутбуках и серверах. Вторая была создана невесть когда в Великобритании, и из-за цепочки случайностей получила распространение почти во всех нынешних мобильных устройствах (телефонах и планшетах) — как c iOS, так и с Android. Обе архитектуры не самые изящные, и несут на себе огромный багаж обратной совместимости, но так получилось. Тысячи цветов не расцвели. 

Мы будем писать на ассемблере x86, который описывает машинный код, понимаемый процессорами Intel.

Чтобы нашу чистоту разума ничего не замутнило, мы отбросим все огромные культурные слои, предоставляемые операционной системой, библиотеками, шеллами, браузером и т.п., и попробуем написать код, который будет исполняться процессором непосредственно и эксклюзивно. Иными словами, перезагрузимся без операционной системы. Не то чтобы мы много успеем, но именно с этого начинается написание ОС, и когда-то Линус Торвальдс и Билл Гейтс сидели над кодом загрузчика и дописывали туда ассемблерные команды, так же, как это будете делать сейчас вы.

Поскольку загрузить современные компьютеры не так-то просто, я попросил вас поставить эмулятор QEMU (факультативно потом вы сможете записать получившийся под на флешку и попробовать загрузиться на настоящем компьютере, но это нетривиально и может не получиться по ряду причин). Ещё нам понадобится собственно ассемблер — установите nasm из вашего менеджера пакетов.

Итак, как многие знают, x86-компьютеры оснащены такой штукой как BIOS (про EFI я сейчас не буду говорить, это то же самое только хуже и монстроподобнее). **BIOS** — это небольшая встроенная (то есть всегда присутствуюшая) программка, которая предоставляет очень простой API к устройствам компьютера (на уровне "поставить сюда символ", "зажечь пиксель", и "прочитать с диска байт по такому-то смещению"), и умеет загружать первые 512 байт кода с дисков (а также флешек и других носителей). Эти 512 байт называются "загрузочным сектором", и в основном служат для того чтобы найти на диске ещё код и начать его выполнять (передать управление операционной системе). Но у нас ОС не будет, мы будем писать команды на ассемблере прямо туда. 512 байт это не так и мало.

Чтобы проверить, что перед нами на диске именно бут-сектор, BIOS проверяет, что байты 511 и 512 содержат значения `55` и `aa` соответственно. Случайно они вряд ли там окажутся, а если их туда записали намеренно — значит, давайте попробуем выполнить код всего бутсектора целиком.

Процессор исполняет **машинный код** — числа, которые он интепретирует как команды и аргументы. Страшную ужасную таблицу, какое число какой команде соответствует, можно посмотреть например здесь: http://ref.x86asm.net/coder32.html. В незапамятные времена программисты так и писали программы, сверяясь с таблицей и вручную добавляя команды байт за байтом. Мы, понятно, этого делать не будем и будем писать на ассемблере — текстовом представлении команд процессора (и некоторых удобных средств управления вроде меток), которое потом транслируется в машинный код.

Напишем простейший дзен-бутсектор, который ничего не делает (зависает в бесконечном цикле). Для этого нам нужно:

1. Написать команду бесконечного цикла. Команда безусловного перехода в x86-ассемблере называется `jmp`, и если её аргументом дать адрес самой себя (или метку, чтобы ассемблер сам вычислил адрес), мы зациклимся.

2. Заполнить остальные байты бутсектора, кроме последних двух, чем угодно, например нулями.

3. Байты 511 и 512 заполнить 16-ричными числами `55` и `aa`. Для этого в ассемблере есть команды вставления констант — по причине того, что байты в числах в Intel-архитектуре идут наоборот (как я объяснял в первом модуле), это число будет 0xaa55 (шестнадцатиричное). Константа соответствущего размера в ассемблере обычно обозначается `dw`.

Так и запишем. Создаём файлик boot.asm, и пишем там следующее:

    loop:
        jmp loop

    times 510-($-$$) db 0
    dw 0xaa55

(ужасная строка `times` высчитывает, сколько осталось до 510 байт, то есть 512 без последних, и заполняет их нулями)

Транслируем в машинный код: `nasm -f bin boot.asm`,
убеждаемся, что получившийся файлик занимает ровно 512 байт,
и загружаем виртуальный компьютер:

    qemu-system-x86_64 boot

(boot это ваш файлик с машинным кодом бутсектора, а команда `qemu` у вас может выглядеть иначе, посмотрите как именно).

Если всё прошло успешно, виртуальный компьютер напишет, что он загружается с диска, и зависнет. Так и должно быть!

Если всё получилось, ваша задача написать в бут-секторе `hello world` (поскольку делать это придётся посимвольно, обойдёмся просто `Hello`). Сейчас расскажу то, что для этого нужно знать.

Как мы знаем из прошлого раза, у процессора есть регистры (места, куда можно записывать числа, ну и буквы в байтовом представлении тоже), а также прерывания.

Команда, которая перемещает значения из одного места в другое (а точнее, копирует), называется `mov`. 

выглядит она так:

    mov (куда), (что)

например, положить в регистр ah нулевой байт это:

    mov ah, 0x00

а положить в регистр `al` байт, соответствующий букве 'H', это:

    mov al, 'H'

(немного про регистры. В самом начале истории Intel они были 8-битными, вмещали 1 байт и  назывались, видимо, a, b, c и d. Уже почти сразу они стали 16-битными (вмещать 2 байта) и называться ax, bx, cx и dx (есть и другие) при этом "старший" байт каждого из них обозначется например как ah, а "младший" — al.)

Потом компьютеры стали 32-битными (регистры eax, ebx...), а потом и 64-битными (rax, rbx, ...) Нам пока такие большие числа не нужны. :)

Итак, чтобы выводить буковки на экран, в байте регистра ah должно находиться магическое значение 0x0e (режим терминала). 
А потом можно по очереди класть в регистр al байты буковок, и после каждой вызывать прерывание 10, которое говорит BIOS, что вот эту буковку надо вывести на экран. И так одну за одной.

Прерывания вызываются командой `int`, в нашем случае

    int 0x10

## ЗАДАНИЕ:

1. Написать бутсектор, который пишет на экран слово Hello (в гугл по возможности не смотреть).
2. Загрузиться с ним в qemu.
