# OS_IHW2
# Габдуллина Алия Маратовна БПИ214
# Вариант 26. Условие задачи: 
Задача о производстве булавок. В цехе по заточке булавок все
необходимые операции осуществляются тремя рабочими. Первый
из них берет булавку и проверяет ее на предмет кривизны. Если
булавка не кривая, то рабочий передает ее своему напарнику. Иначе выбрасывает в отбраковку. Напарник осуществляет собственно
заточку и передает заточенную булавку третьему рабочему, который осуществляет контроль качества операции бракуя булавку или
отдавая на упаковку. Требуется создать приложение, моделирующее работу цеха. При решении использовать парадигму
«производитель-потребитель». Следует учесть, что каждая из операций выполняется за случайное время которое не связано с конкретным рабочим. Возможны различные способы реализации передачи (на усмотрение разработчика). Либо непосредственно по одной булавке, либо через ящики, в которых буферизируется некоторое конечное количество булавок.


Задание выполнено на 8

# Код на 4: 
Программа создает один родительский и три дочерних процесса, каждый из которых выполняет свою функцию:

0. Родительский процесс - цех
1. Первый процесс проверяет булавку на кривизну.
2. Второй процесс затачивает булавку.
3. Третий процесс контролирует качество булавки.
Процессы взаимодействуют с использованием неименованных POSIX семафоров
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#define BUFFER_SIZE 10

struct shared_memory {
    int buffer[BUFFER_SIZE];
    int in;
    int out;
};

// Функция для уменьшения значения семафора на 1
void sem_wait(int sem_id, int sem_num) {
    struct sembuf operation = {sem_num, -1, 010000};
    semop(sem_id, &operation, 1);
}

// Функция для увеличения значения семафора на 1
void sem_signal(int sem_id, int sem_num) {
    struct sembuf operation = {sem_num, 1, 010000};
    semop(sem_id, &operation, 1);
}

// Функция для записи данных в разделяемую память
void write_data(struct shared_memory *shm, int data, int sem_id) {
    sem_wait(sem_id, 0);
    sem_wait(sem_id, 2);
    shm->buffer[shm->in] = data; // Запись данных
    printf("Производитель записал булавку %d в буфер по индексу %d\n", data, shm->in);
    shm->in = (shm->in + 1) % BUFFER_SIZE;
    sem_signal(sem_id, 1); // есть данные для чтения
    sem_signal(sem_id, 0);
}

// Функция для чтения данных из разделяемой памяти
int read_data(struct shared_memory *shm, int sem_id) {
    int data;
    sem_wait(sem_id, 0);
    sem_wait(sem_id, 1);
    data = shm->buffer[shm->out]; // Чтение данных
    printf("Потребитель прочитал булавку %d из буфера по индексу %d\n", data, shm->out);
    shm->out = (shm->out + 1) % BUFFER_SIZE; // Обновление указателя на чтение
    sem_signal(sem_id, 2);
    sem_signal(sem_id, 0);
    return data;
}

// Функция для проверки булавки на кривизну
void check_pin(struct shared_memory *shm, int sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из разделяемой памяти
    if (pin % 2 == 0) {
        printf("Булавка %d прямая\n", pin);
    } else {
        printf("Булавка %d кривая\n", pin);
    }
}

// Функция для заточки булавки
void sharpen_pin(struct shared_memory *shm, int sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из разделяемой памяти
    printf("Заточка булавки %d\n", pin);
    write_data(shm, pin, sem_id); // Запись данных в разделяемую память
}

// Функция для контроля качества
void quality_control(struct shared_memory *shm, int sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из памяти
    if (pin % 2 == 0) {
        printf("Булавка %d прошла контроль качества\n", pin);
    } else {
        printf("Булавка %d не прошла контроль качества\n", pin);
    }
}

int main() {
    int sem_id = semget(((key_t)0), 3, 001000 | 0666);
    int shm_id = shmget(((key_t)0), sizeof(struct shared_memory), 001000 | 0666);
    struct shared_memory *shm = shmat(shm_id, NULL, 0);

    semctl(sem_id, 0, 8, 1); // Семафор для доступа к разделяемой памяти
    semctl(sem_id, 1, 8, 0); // Семафор для наличия данных для чтения
    semctl(sem_id, 2, 8, BUFFER_SIZE); // Семафор для наличия места для записи

    // Инициализация булавок
    for (int i = 1; i <= BUFFER_SIZE; i++) {
        write_data(shm, i, sem_id);
    }

    // Создание процессов
    for (int i = 0; i < 3; i++) {
        pid_t pid = fork();

        if (pid == 0) {
            switch (i) {
                case 0:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        check_pin(shm, sem_id);
                    }
                    break;
                case 1:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        sharpen_pin(shm, sem_id);
                    }
                    break;
                case 2:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        quality_control(shm, sem_id);
                    }
                    break;
            }

            exit(0);
        }
    }

    // Ожидание завершения процессов
    for (int i = 0; i < 3; i++) {
        wait(NULL);
    }

    shmdt(shm);
    shmctl(shm_id, 0, NULL);
    semctl(sem_id, 0, 0, 0);
    semctl(sem_id, 1, 0, 0);
    semctl(sem_id, 2, 0, 0);

    return 0;
}
```
В данном примере использовалось количество булавок = 10, программа вывела: 
```
Производитель записал булавку 1 в буфер по индексу 0
Производитель записал булавку 2 в буфер по индексу 1
Производитель записал булавку 3 в буфер по индексу 2
Производитель записал булавку 4 в буфер по индексу 3
Производитель записал булавку 5 в буфер по индексу 4
Производитель записал булавку 6 в буфер по индексу 5
Производитель записал булавку 7 в буфер по индексу 6
Производитель записал булавку 8 в буфер по индексу 7
Производитель записал булавку 9 в буфер по индексу 8
Производитель записал булавку 10 в буфер по индексу 9
Потребитель прочитал булавку 1 из буфера по индексу 0
Заточка булавки 1
Потребитель прочитал булавку 2 из буфера по индексу 1
Заточка булавки 2
Потребитель прочитал булавку 3 из буфера по индексу 2
Заточка булавки 3
Потребитель прочитал булавку 4 из буфера по индексу 3
Заточка булавки 4
Потребитель прочитал булавку 5 из буфера по индексу 4
Заточка булавки 5
Потребитель прочитал булавку 6 из буфера по индексу 5
Заточка булавки 6
Потребитель прочитал булавку 7 из буфера по индексу 6
Заточка булавки 7
Потребитель прочитал булавку 8 из буфера по индексу 7
Заточка булавки 8
Потребитель прочитал булавку 9 из буфера по индексу 8
Заточка булавки 9
Потребитель прочитал булавку 10 из буфера по индексу 9
Заточка булавки 10
Булавка 1 прямая
Булавка 2 кривая
Булавка 3 прямая
Булавка 4 кривая
Булавка 5 прямая
Булавка 6 кривая
Булавка 7 прямая
Булавка 8 кривая
Булавка 9 прямая
Булавка 10 кривая
Булавка 1 прошла контроль качества
Булавка 2 не прошла контроль качества
Булавка 3 прошла контроль качества
Булавка 4 не прошла контроль качества
Булавка 5 прошла контроль качества
Булавка 6 не прошла контроль качества
Булавка 7 прошла контроль качества
Булавка 8 не прошла контроль качества
Булавка 9 прошла контроль качества
Булавка 10 не прошла контроль качества


```

# Код на 5:

Все аналогично предыдущему, только процессы взаимодействуют с использованием именованных POSIX семафоров

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/sem.h>
#include <fcntl.h>
#include <string.h>
#include <semaphore.h>
#include <sys/mman.h>
#define BUFFER_SIZE 10

struct shared_memory {
    int buffer[BUFFER_SIZE];
    int in;
    int out;
};

// Функция для уменьшения значения семафора на 1
void sem_wait_my(sem_t *sem_id, int sem_num) {
    sem_wait(&sem_id[sem_num]);
}

// Функция для увеличения значения семафора на 1
void sem_signal(sem_t *sem_id, int sem_num) {
    sem_post(&sem_id[sem_num]);
}

// Функция для записи данных в разделяемую память
void write_data(struct shared_memory *shm, int data, sem_t *sem_id) {
    sem_wait_my(sem_id, 0);
    sem_wait_my(sem_id, 2);
    shm->buffer[shm->in] = data; // Запись данных
    printf("Производитель записал булавку %d в буфер по индексу %d\n", data, shm->in);
    shm->in = (shm->in + 1) % BUFFER_SIZE; // Обновление указателя на запись
    sem_signal(sem_id, 1);
    sem_signal(sem_id, 0);
}

// Функция для чтения данных из разделяемой памяти
int read_data(struct shared_memory *shm, sem_t *sem_id) {
    int data;
    sem_wait_my(sem_id, 0);
    sem_wait_my(sem_id, 1);
    data = shm->buffer[shm->out]; // Чтение данных
    printf("Потребитель прочитал булавку %d из буфера по индексу %d\n", data, shm->out);
    shm->out = (shm->out + 1) % BUFFER_SIZE; // Обновление указателя на чтение
    sem_signal(sem_id, 2);
    sem_signal(sem_id, 0);
    return data;
}

// Функция для проверки булавки на кривизну
void check_pin(struct shared_memory *shm, sem_t *sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из разделяемой памяти
    if (pin % 2 == 0) {
        printf("Булавка %d прямая\n", pin);
    } else {
        printf("Булавка %d кривая\n", pin);
    }
}

// Функция для заточки булавки
void sharpen_pin(struct shared_memory *shm, sem_t *sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из памяти
    printf("Заточка булавки %d\n", pin);
    pin++;
    write_data(shm, pin, sem_id); // Запись обновленных данных в память
}

// Функция для контроля качества булавки
void quality_control(struct shared_memory *shm, sem_t *sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из памяти
    if (pin % 2 == 0) {
        printf("Булавка %d прошла контроль качества\n", pin);
    } else {
        printf("Булавка %d не прошла контроль качества\n", pin);
    }
}

int main() {
    sem_t *sem1_id = sem_open("sem1", O_CREAT, 0666, 1);
    sem_t *sem2_id = sem_open("sem2", O_CREAT, 0666, 0);
    sem_t *sem3_id = sem_open("sem3", O_CREAT, 0666, BUFFER_SIZE);
    int shm_id = shm_open("shm", O_CREAT | O_RDWR, 0666);
    ftruncate(shm_id, sizeof(struct shared_memory));
    struct shared_memory *shm = mmap(NULL, sizeof(struct shared_memory), PROT_READ | PROT_WRITE, MAP_SHARED, shm_id, 0);

    sem_init(sem1_id, 1, 0);
    sem_init(sem2_id, 1, 0);
    sem_init(sem3_id, 1, BUFFER_SIZE);

    // Инициализация булавок
    for (int i = 1; i <= BUFFER_SIZE; i++) {
        write_data(shm, i, sem3_id);
    }

    // Создание процессов
    for (int i = 0; i < 3; i++) {
        pid_t pid = fork();

        if (pid == 0) {
            switch (i) {
                case 0:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        check_pin(shm, sem1_id);
                    }
                    break;
                case 1:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        sharpen_pin(shm, sem1_id);
                    }
                    break;
                case 2:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        quality_control(shm, sem1_id);
                    }
                    break;
            }

            exit(0);
        }
    }

    // Ожидание завершения процессов
    for (int i = 0; i < 3; i++) {
        wait(NULL);
    }

    munmap(shm, sizeof(struct shared_memory));
    close(shm_id);
    shm_unlink("shm");
    sem_close(sem1_id);
    sem_close(sem2_id);
    sem_close(sem3_id);
    sem_unlink("sem1");
    sem_unlink("sem2");
    sem_unlink("sem3");

    return 0;
}
```
Вывод в консоль такой же.

#Код на 6:

Аналогично предыдущим двум, только используется стандарт SYSTEM V.

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <string.h>
#include <sys/mman.h>
#define BUFFER_SIZE 10

struct shared_memory {
    int buffer[BUFFER_SIZE];
    int in;
    int out;
};

// Функция для уменьшения значения семафора на 1
void sem_wait_my(int sem_id, int sem_num) {
    struct sembuf sem_buf = {sem_num, -1, 0};
    semop(sem_id, &sem_buf, 1);
}

// Функция для увеличения значения семафора на 1
void sem_signal(int sem_id, int sem_num) {
    struct sembuf sem_buf = {sem_num, 1, 0};
    semop(sem_id, &sem_buf, 1);
}

// Функция для записи данных в разделяемую память
void write_data(struct shared_memory *shm, int data, int sem_id) {
    sem_wait_my(sem_id, 0);
    sem_wait_my(sem_id, 2);
    shm->buffer[shm->in] = data; // Запись данных
    printf("Производитель записал булавку %d в буфер по индексу %d\n", data, shm->in);
    shm->in = (shm->in + 1) % BUFFER_SIZE; // Обновление указателя на запись
    sem_signal(sem_id, 1);
    sem_signal(sem_id, 0);
}

// Функция для чтения данных из разделяемой памяти
int read_data(struct shared_memory *shm, int sem_id) {
    int data;
    sem_wait_my(sem_id, 0); // Блокировка доступа к разделяемой памяти
    sem_wait_my(sem_id, 1); // Ожидание данных для чтения
    data = shm->buffer[shm->out]; // Чтение данных
    printf("Потребитель прочитал булавку %d из буфера по индексу %d\n", data, shm->out);
    shm->out = (shm->out + 1) % BUFFER_SIZE; // Обновление указателя на чтение
    sem_signal(sem_id, 2); // Сигнализация о том, что есть место для записи
    sem_signal(sem_id, 0); // Разблокировка доступа к разделяемой памяти
    return data;
}

// Функция для проверки булавки на кривизну
void check_pin(struct shared_memory *shm, int sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из разделяемой памяти
    if (pin % 2 == 0) {
        printf("Булавка %d прямая\n", pin);
    } else {
        printf("Булавка %d кривая\n", pin);
    }
}

// Функция для заточки булавки
void sharpen_pin(struct shared_memory *shm, int sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из разделяемой памяти
    printf("Заточка булавки %d\n", pin);
    pin++;
    write_data(shm, pin, sem_id); // Запись обновленных данных в разделяемую память
}

// Функция для контроля качества булавки
void quality_control(struct shared_memory *shm, int sem_id) {
    int pin = read_data(shm, sem_id); // Чтение данных из разделяемой памяти
    if (pin % 2 == 0) {
        printf("Булавка %d прошла контроль качества\n", pin);
    } else {
        printf("Булавка %d не прошла контроль качества\n", pin);
    }
}

int main() {
    // Инициализация именованных семафоров и разделяемой памяти
    int sem_id = semget(IPC_PRIVATE, 3, IPC_CREAT | 0666);
    semctl(sem_id, 0, SETVAL, 1);
    semctl(sem_id, 1, SETVAL, 0);
    semctl(sem_id, 2, SETVAL, BUFFER_SIZE);
    int shm_id = shmget(IPC_PRIVATE, sizeof(struct shared_memory), IPC_CREAT | 0666);
    struct shared_memory *shm = shmat(shm_id, NULL, 0);

    // Инициализация булавок
    for (int i = 1; i <= BUFFER_SIZE; i++) {
        write_data(shm, i, sem_id);
    }

    // Создание процессов
    for (int i = 0; i < 3; i++) {
        pid_t pid = fork();

        if (pid == 0) {
            switch (i) {
                case 0:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        check_pin(shm, sem_id);
                    }
                    break;
                case 1:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        sharpen_pin(shm, sem_id);
                    }
                    break;
                case 2:
                    for (int j = 0; j < BUFFER_SIZE; j++) {
                        quality_control(shm, sem_id);
                    }
                    break;
            }

            exit(0);
        }
    }

    // Ожидание завершения процессов
    for (int i = 0; i < 3; i++) {
        wait(NULL);
    }

    shmdt(shm);
    shmctl(shm_id, 0, NULL);
    semctl(sem_id, 0, 0);
    semctl(sem_id, 1, 0);
    semctl(sem_id, 2, 0);

    return 0;
}
```
Вывод в консоль такой же.

# Код на 7:

Логика программы остается прежней, только теперь приложение разделено на две части - клиентская (родительский процесс - цех, отправляет запросы рабочим) и серверная (три рабочих, исполняющих свои обязанности: изготовление булавок). Процессы взаимодействуют с помощью именнованных POSIX семафоров.

Client:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <signal.h>
#define SEM_KEY 0x1234
#define SHM_KEY 0x5678
#define BUFFER_SIZE 10

typedef struct {
    int produced; // кол-во произведенных булавок
    int ready; // кол-во готовых булавок для отправки
    int sent; // кол-во отправленных булавок
} SharedData;

typedef struct {
    int id;
    int semid;
    SharedData* shared_data;
} Client;

int sem_wait(int semid, int sem_num) {
    struct sembuf op;
    op.sem_num = sem_num;
    op.sem_op = -1;
    op.sem_flg = 0;
    return semop(semid, &op, 1);
}

int sem_signal(int semid, int sem_num) {
    struct sembuf op;
    op.sem_num = sem_num;
    op.sem_op = 1;
    op.sem_flg = 0;
    return semop(semid, &op, 1);
}

void sigint_handler(int signum) {
    exit(0);
}

int main() {
    // Создание семафоров
    int semid = semget(SEM_KEY, 2, IPC_CREAT | 0666);
    if (semid == -1) {
        perror("semget error");
        exit(1);
    }

    union semun {
        int val;
        struct semid_ds *buf;
        unsigned short *array;
    } arg;

    arg.val = 1; // семафор 0 - для доступа к разделяемой памяти
    if (semctl(semid, 0, 8, arg) == -1) {
        perror("semctl error");
        exit(1);
    }

    arg.val = 0; // семафор 1 - для ожидания готовых булавок
    if (semctl(semid, 1, 8, arg) == -1) {
        perror("semctl error");
        exit(1);
    }

    // Создание разделяемой памяти
    int shmid = shmget(SHM_KEY, sizeof(SharedData), IPC_CREAT | 0666);
    if (shmid == -1) {
        perror("shmget error");
        exit(1);
    }

    // Присоединение разделяемой памяти
    SharedData* shared_data = (SharedData*) shmat(shmid, NULL, 0);
    if (shared_data == (SharedData*) -1) {
        perror("shmat error");
        exit(1);
    }

    shared_data->produced = 0;
    shared_data->ready = 0;
    shared_data->sent = 0;

    // Создание клиента
    Client client;
    client.id = getpid();
    client.semid = semid;
    client.shared_data = shared_data;

    signal(SIGINT, sigint_handler);

    // Основной цикл клиента
    for (int i = 0; i < BUFFER_SIZE; ++i) {
        sem_wait(semid, 0);

        if (client.shared_data->sent >= client.shared_data->produced) {
            // если все булавки отправлены
            printf("Клиент %d: Все булавки были отправлены.\\n", client.id);
            sem_signal(semid, 0);
            sleep(1);
            continue;
        }

        if (client.shared_data->ready > 0) {
            // если есть готовые булавки
            client.shared_data->sent++;
            client.shared_data->ready--;
            printf("Клиент %d: Отправил булавку. Отправлено: %d, Готово: %d\\n", client.id, client.shared_data->sent, client.shared_data->ready);
            sem_signal(semid, 1);
        } else {
            // иначе ожидаем готовых булавок
            printf("Клиент %d: Ожидает булавки\\n", client.id);
            sem_signal(semid, 0);
            sem_wait(semid, 1);
        }

        sleep(1);
    }

    return 0;
}
```

server:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <signal.h>
#define SEM_KEY 0x1234
#define SHM_KEY 0x5678
#define BUFFER_SIZE 10

typedef struct {
    int produced; // кол-во произведенных булавок
    int ready; // кол-во готовых булавок для отправки
    int sent; // кол-во отправленных булавок
} SharedData;

int sem_wait(int semid, int sem_num) {
    struct sembuf op;
    op.sem_num = sem_num;
    op.sem_op = -1;
    op.sem_flg = 0;
    return semop(semid, &op, 1);
}

int sem_signal(int semid, int sem_num) {
    struct sembuf op;
    op.sem_num = sem_num;
    op.sem_op = 1;
    op.sem_flg = 0;
    return semop(semid, &op, 1);
}

void sigint_handler(int signum) {
    exit(0);
}

int main() {
    // Создание семафоров
    int semid = semget(SEM_KEY, 2, IPC_CREAT | 0666);
    if (semid == -1) {
        perror("semget error");
        exit(1);
    }

    // Инициализация семафоров
    union semun {
        int val;
        struct semid_ds *buf;
        unsigned short *array;
    } arg;

    arg.val = 1; // семафор 0 - для доступа к разделяемой памяти
    if (semctl(semid, 0, 8, arg) == -1) {
        perror("semctl error");
        exit(1);
    }

    arg.val = 0; // семафор 1 - для ожидания новых булавок
    if (semctl(semid, 1, 8, arg) == -1) {
        perror("semctl error");
        exit(1);
    }

    // Создание разделяемой памяти
    int shmid = shmget(SHM_KEY, sizeof(SharedData), IPC_CREAT | 0666);
    if (shmid == -1) {
        perror("shmget error");
        exit(1);
    }

    // Присоединение разделяемой памяти
    SharedData* shared_data = (SharedData*) shmat(shmid, NULL, 0);
    if (shared_data == (SharedData*) -1) {
        perror("shmat error");
        exit(1);
    }

    // Инициализация разделяемой памяти
    shared_data->produced = 0;
    shared_data->ready = 0;
    shared_data->sent = 0;

    signal(SIGINT, sigint_handler);

    // Основной цикл сервера
    for (int i = 0; i < BUFFER_SIZE; ++i) {
        sem_wait(semid, 0);

        if (shared_data->produced >= BUFFER_SIZE) {
            // если буфер полный
            printf("Буфер заполнен.\\n");
            sem_signal(semid, 0);
            sleep(1);
            continue;
        }

        // производим булавку
        shared_data->produced++;
        printf("Производитель произвел булавку. Произведено: %d\\n", shared_data->produced);

        // если буфер не пустой, увеличиваем кол-во готовых булавок
        if (shared_data->ready < BUFFER_SIZE) {
            shared_data->ready++;
            printf("Готовых булавок: %d\\n", shared_data->ready);
            sem_signal(semid, 1);
        } else {
            printf("Буфер заполнен.\\n");
            sem_signal(semid, 0);
        }

        sleep(1);
    }

    return 0;
}
```

Код на 8:
Аналогично коду на 7, только взаимодействие процессов через SYSTEM V семафоры:

client:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <signal.h>

#define SEM_KEY 0x1234
#define SHM_KEY 0x5678
#define BUFFER_SIZE 10

typedef struct {
    int produced; // кол-во произведенных булавок
    int ready; // кол-во готовых булавок для отправки
    int sent; // кол-во отправленных булавок
} SharedData;

typedef struct {
    int id;
    int semid;
    SharedData* shared_data;
} Client;

int sem_wait(int semid, int sem_num) {
    struct sembuf op;
    op.sem_num = sem_num;
    op.sem_op = -1;
    op.sem_flg = 0;
    return semop(semid, &op, 1);
}

int sem_signal(int semid, int sem_num) {
    struct sembuf op;
    op.sem_num = sem_num;
    op.sem_op = 1;
    op.sem_flg = 0;
    return semop(semid, &op, 1);
}

void sigint_handler(int signum) {
    exit(0);
}

int main() {
    // Создание семафоров
    int semid = semget(SEM_KEY, 2, IPC_CREAT | 0666);
    if (semid == -1) {
        perror("semget error");
        exit(1);
    }

    union semun {
        int val;
        struct semid_ds *buf;
        unsigned short *array;
    } arg;

    arg.val = 1; // семафор 0 - для доступа к разделяемой памяти
    if (semctl(semid, 0, 8, arg) == -1) {
        perror("semctl error");
        exit(1);
    }

    arg.val = 0; // семафор 1 - для ожидания готовых булавок
    if (semctl(semid, 1, 8, arg) == -1) {
        perror("semctl error");
        exit(1);
    }

    // Создание разделяемой памяти
    int shmid = shmget(SHM_KEY, sizeof(SharedData), IPC_CREAT | 0666);
    if (shmid == -1) {
        perror("shmget error");
        exit(1);
    }

    // Присоединение разделяемой памяти
    SharedData* shared_data = (SharedData*) shmat(shmid, NULL, 0);
    if (shared_data == (SharedData*) -1) {
        perror("shmat error");
        exit(1);
    }

    shared_data->produced = 0;
    shared_data->ready = 0;
    shared_data->sent = 0;

    // Создание клиента
    Client client;
    client.id = getpid();
    client.semid = semid;
    client.shared_data = shared_data;

    signal(SIGINT, sigint_handler);

    // Основной цикл клиента
    for (int i = 0; i < BUFFER_SIZE; ++i) {
        sem_wait(semid, 0);

        if (client.shared_data->sent >= client.shared_data->produced) {
            // если все булавки отправлены
            printf("Производитель %d: Все булавки были отправлены.\\n", client.id);
            sem_signal(semid, 0);
            sleep(1);
            continue;
        }

        if (client.shared_data->ready > 0) {
            // если есть готовые булавки
            client.shared_data->sent++;
            client.shared_data->ready--;
            printf("Производитель %d: Отправил булавку. Отправлено: %d, Готово: %d\\n", client.id, client.shared_data->sent, client.shared_data->ready);
            sem_signal(semid, 1);
        } else {
            // иначе ожидаем готовых булавок
            printf("Потребитель %d: Ожидает булавки\\n", client.id);
            sem_signal(semid, 0);
            sem_wait(semid, 1);
        }

        sleep(1);
    }

    return 0;
}
```

server:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <signal.h>

#define SEM_KEY 0x1234
#define SHM_KEY 0x5678
#define BUFFER_SIZE 10

typedef struct {
    int produced; // кол-во произведенных булавок
    int ready; // кол-во готовых булавок для отправки
    int sent; // кол-во отправленных булавок
} SharedData;

int sem_wait(int semid, int sem_num) {
    struct sembuf op;
    op.sem_num = sem_num;
    op.sem_op = -1;
    op.sem_flg = 0;
    return semop(semid, &op, 1);
}

int sem_signal(int semid, int sem_num) {
    struct sembuf op;
    op.sem_num = sem_num;
    op.sem_op = 1;
    op.sem_flg = 0;
    return semop(semid, &op, 1);
}

void sigint_handler(int signum) {
    exit(0);
}

int main() {
    // Создание семафоров
    int semid = semget(SEM_KEY, 2, IPC_CREAT | 0666);
    if (semid == -1) {
        perror("semget error");
        exit(1);
    }

    // Инициализация семафоров
    union semun {
        int val;
        struct semid_ds *buf;
        unsigned short *array;
    } arg;

    arg.val = 1; // семафор 0 - для доступа к разделяемой памяти
    if (semctl(semid, 0, 8, arg) == -1) {
        perror("semctl error");
        exit(1);
    }

    arg.val = 0; // семафор 1 - для ожидания новых булавок
    if (semctl(semid, 1, 8, arg) == -1) {
        perror("semctl error");
        exit(1);
    }

    // Создание разделяемой памяти
    int shmid = shmget(SHM_KEY, sizeof(SharedData), IPC_CREAT | 0666);
    if (shmid == -1) {
        perror("shmget error");
        exit(1);
    }

    // Присоединение разделяемой памяти
    SharedData* shared_data = (SharedData*) shmat(shmid, NULL, 0);
    if (shared_data == (SharedData*) -1) {
        perror("shmat error");
        exit(1);
    }

    // Инициализация разделяемой памяти
    shared_data->produced = 0;
    shared_data->ready = 0;
    shared_data->sent = 0;

    signal(SIGINT, sigint_handler);

    // Основной цикл сервера
    for (int i = 0; i < BUFFER_SIZE; ++i) {
        sem_wait(semid, 0);

        if (shared_data->produced >= BUFFER_SIZE) {
            // если буфер полный
            printf("Server: Buffer is full.\\n");
            sem_signal(semid, 0);
            sleep(1);
            continue;
        }

        // производим булавку
        shared_data->produced++;
        printf("Производитель произвел булавку. Произведено: %d\\n", shared_data->produced);

        // если буфер не пустой, увеличиваем кол-во готовых булавок
        if (shared_data->ready < BUFFER_SIZE) {
            shared_data->ready++;
            printf("Готовых булавок: %d\\n", shared_data->ready);
            sem_signal(semid, 1);
        } else {
            printf("Буфер заполнен.\\n");
            sem_signal(semid, 0);
        }

        sleep(1);
    }

    return 0;
}

```
