# Процессы

В UNIX-системах поддерживается многозадачность, которая обеспечивается: параллельным выполнением нескольких задач, изоляцией адресного пространства каждой задачи.

Полный список процессов можно получить с помощью команды `ps -A`.

Убить какой-то процесс можно с помощью команды `kill`.

## Свойства процессов

У каждого процесса существует свои обособленные:
 * адресное пространство начиная с `0x00000000`;
 * набор файловых дескрипторов для открытых файлов.

Кроме того, каждый процесс может находиться в одном из состояний:
 * работает (Running);
 * приостановлен до возникновения определенного события (Suspended);
 * приостановлен до явного сигнала о том, что нужно продолжить работу (sTopped);
 * более не функционирует, не занимает память, но при этом не удален из таблицы процессов (Zombie).

Каждый процесс имеет свой уникальный идентификатор - Process ID (PID), который присваивается системой инкрементально. Множество доступных PID является ограниченным, и его исчерпание проводит к невозможности создания нового процесса (что является механизмом действия форк-бомбы).

## Иерархия процессов

Между процессами существуют родственные связи "предок-потомок", таким образом, иерархия процессов представляет собой древовидную структуру. Корнем дерева процессов является процесс с PID=1, который называется `init` (в классических UNIX-системах, в том числе xBSD или консервативных Linux-дистрибутивах), либо `systemd`.

Родительские процессы могут завершаться раньше, чем завершаются их потомки. В этом случае, осиротевшие процессы становятся прямыми потомками процесса с PID=1, то есть `init` или `systemd`.

Процессы можно объединять в *группы процессов* (process group), - множества процессов, которым доставляются *сигналы* о некоторых событиях. Например, в одну группу могут объединяться все процессы, запущенные из одной вкладки приложения-терминала.

Объединение нескольких групп процессов называется *сеансом* (session). Как правило, в сеансы объединяются группы процессов в рамках одного входа пользователя в систему (их может быть несколько, например, несколько входов по `ssh`).

## Системный вызов `fork`

Создание нового процесса осуществляется с помощью системного вызова `fork`, который создаёт почти точную копию текущего процесса, причём оба процесса продолжают своё выполнение со следующей, после вызова `fork`, инструкции. Различить родительский процесс от его копии - дочернего процесса, можно по возвращаемому значению `fork`: для родительского процесса возвращается PID вновь созданного процесса, а для дочернего - число 0.

```
pid_t process_id; // в большинстве систем pid_t совпадает с int
if ( -1 == ( process_id=fork() ) ) {
  perror("fork"); // ошибка создания нового процесса
}
else if ( 0 == process_id ) {
  printf("I'm a child process!");
}
else {
  printf("I'm a parent process, created child %d", process_id);
}
```

Ситуация, когда `fork` возвращает значение -1, как правило означает, что система исчерпала допустимый лимит ресурсов на создание новых процессов.

Пример реализации форк-бомбы, назначение которой - исчерпать лимит на количество запущенных процессов:

```
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sched.h>

int main()
{
    char * are_you_sure = getenv("ALLOW_FORK_BOMB");
    if (!are_you_sure || 0!=strcmp(are_you_sure, "yes")) {
        fprintf(stderr, "Fork bomb not allowed!\n");
        exit(127);
    }

    pid_t pid;
    do {
        pid = fork();
    } while (-1 != pid);

    printf("Process %d reached out limit on processes\n", getpid());
    while (1) {
        sched_yield();
    }
}
```

## Завершение работы процесса

Любой процесс, при добровольном завершении работы, должен явным образом сообщить об этом системе с помощью системного вызова `exit`.

При написании программ на языках Си или С++ этот системный вызов выполняется после завершения работы функции `main`, вызванной из функции `_start`, которая неявным образом генерируется компилятором.

Обратите внимание, что в стандартной библиотеке языка Си уже существует одноименная функция `exit`, поэтому название Си-оболочки для системного вызова начинается с символа подчеркивания: `_exit`.

Функция `exit` отличается от системного вызова `_exit` тем, что предварительно сбрасывает содержимое буферов вывода, а также последовательно вызывает все функции, зарегистрированные с помощью `atexit`.

В качестве аргумента принимается целое число, - код возврата из программы. Несмотря на то, что код возврата является 32-разрядным, к нему применяется операция поразрядного "и" с маской `0xFF`. Таким образом, диапазон кодов возврата находится от 0 до 255.

Код возврата предназначен для того, чтобы сообщить родительскому процессу причину завершения своей работы.

## Чтение кода возврата дочернего процесса

Семейство системных вызовов `wait*` предназначено для ожидания завершения работы процесса, и получения информации о том, как процесс жил и умер.

 * `wait(int *wstatus)` - ожидание завершения любого дочернего процесса, возвращает информацию о завершении работы;
 * `waitpid(pid_t pid, int *wstatus, int options)` - ожидание (возможно неблокирующее) завершения работы конкретного процесса, возвращает информации о завершении работы;
 * `wait3(int *wstatus, int options, struct rusage *rusage)` - ожидание (возможно неблокирующее) завершения любого дочернего процесса, возвращает информацию о завершении работы и статистике использования ресурсов;
 * `wait4(pid_t pid, int *wstatus, int options, struct rusage *rusage)` - ожидание (возможно неблокирующее) завершения конкретного процесса, возвращает информацию о завершении работы и статистике использования ресурсов.

Если в программе предусмотрено создание более одного дочернего процесса, то использовать системные вызовы `wait` и `wait3` настоятельно не рекомендуется, поскольку дочерние процессы могут завершать свою работу произвольным образом, и это может привести к неоднозначному поведению. Вместо них нужно использовать `waitpid` или `wait4`.

Состояние возврата, которое можно прочитать из таблицы процессов после того, как процесс перестал функционировать, - это причина завершения работы, код возврата, если процесс завершился через `_exit`, или номер убившего его сигнала, если процесс был принудительно завершён. Это состояние закодировано в 32-битном значении, формат которого строго не определен стандартом POSIX. Для извлечения информации используется нобор макросов:
 * `WIFEXITED(wstatus)` - возвращает значение, отличное от 0, если процесс был завершен с помощью системного вызова `_exit`;
 * `WIFSIGNALED(wstatus)` - возвращает значение, отличное от 0, если процесс был завершен принудительно;
 * `WEXITSTATUS(wstatus)` - выделяет код возврата в диапазоне от 0 до 255;
 * `WTERMSIG(wstatus)` - выделяет номер сигнала, если процесс был завершён принудительно.

Чтение кода возврата, - это не право, а обязанность родительского процесса. В противном случае, дочерний процесс, который завершил свою работу, становится процессом-зомби, информация о завершении которого продолжает занимать место в таблице процессов. Завершение работы родительского процесса при работающих дочерних приводит к тому, что код возврата будет прочитан процессом с PID=1, который автоматически становится родительским для "осиротевших" процессов.
