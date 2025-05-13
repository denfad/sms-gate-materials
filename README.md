# Шлюз сообщений для облачной технологии 1С:Предприятие.Элемент

В даннном документе представлены дополнительные материалы к основной презентации.

## Система задач
### Подробная архитектура системы задач

![Архитектура](/res/архитектура.png "Архитектура системы задач")

На изображении представлено подробное описание обязанностей различных компонентов менеджера задач и то, какие вызовы между ними совершаются и с какой целью.

### Жизненный цикл задачи

![Задача](/res/жизненный_цикл.png "Жизненный цикл задачи")

Через жизненный цкил задач планировщик может отслеживать процесс выполнения задачи и переводить её в новые состояния в зависимости от этого. Благодаря этому не возникает забытых задач, которые по какой-либо причине не были приняты в работу.

### Чтение из очереди задач

![Очередь](/res/очередь.png "SQL запрос чтения из очереди")

Так как очередь задач реализована в СУБД Postgres, то был разработан данный SQL-запрос, позволяющий оптимизированно считывать из таблицы записи по принципу FIFO, при этом не вызывая конфликтов чтения между несколькими экземплярами шлюза

### Исполнитель задач

```Java
public interface TaskExecutor<T extends Task> {
    Class<T> appliesTo();
    ApplicationStatisticDto.StatisticDto getTaskStatistic(T task);
    T executeTask(T task);
}
```

Все исполнители задач реализуют данный интерфейс, позволяя конкретизировать особенности исполнения каждой задачи, при этом позволяя взаимодействовать с их исполнением через абстракцию.

### Сущность задачи

```Java

@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@Data
@NoArgsConstructor
public abstract class Task {
    @GeneratedValue(generator = "uuid2")
    @GenericGenerator(name = "uuid2", strategy = "uuid2")
    @Id
    private String id;
    @Enumerated(EnumType.STRING)
    private TaskStatus status;
    private String statusMessage;
    private Date createDate;
    private String message;
    private String providerName;
    @OneToMany(mappedBy = "task", fetch = FetchType.EAGER, cascade = CascadeType.ALL)
    private List<Destination> destinations;

    public abstract void init(MessageDto dto);
    public abstract TaskDto toTaskDto();
    public abstract ClientMessageDto toClientMessage();
}

```

Каждая задача на отправку конкретного типа сообщений наследуется от абстрактного класса задачи, дополняя его своими свойствами. При этом каждый тип задачи имеет собственную таблицу в СУБД.

### Создание пула потоков

```Java
@Configuration
public class ExecutorsConfiguration implements AsyncConfigurer {

    @Value("${threads-max-count}")
    private int threadsMaxCount;

    @Override
    @Qualifier("asyncExecutor")
    @Bean(name = "asyncExecutor")
    public ThreadPoolTaskExecutor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3); // Установка минимального размера пула потоков
        executor.setMaxPoolSize(threadsMaxCount); // Установка макисмального размера пула потоков
        executor.setQueueCapacity(0); // Размер очереди потоков
        executor.setThreadNamePrefix("task-async-"); // Установка префикса имён потоков
        executor.initialize();
        return executor;
    }
}
```

Все потоки исполнения хранятся в пуле потоков, который определяется в классах конфигурации Spring Boot.

### Создание исполнителей задач

```Java
@Configuration
public class TaskExecutorsConfiguration {

    @Bean(name = "executorsMap")
    public Map<Class<? extends Task>, TaskExecutor<? extends Task>> executors(List<TaskExecutor> executors) {
        var executorMap = new HashMap<Class<? extends Task>, TaskExecutor<? extends Task>>();

        for(TaskExecutor taskExecutor: executors) {
            executorMap.put(taskExecutor.appliesTo(), taskExecutor);
        }
        return executorMap;
    }
}
```

Также определяется словарь, ставящий в соответствие каждой задаче своего исполнителя. По даннойму словарю менеджер задач будет искать исполнителя для задач, прочитанных из очереди.

## Мультиарендность

### Принцип работы

![RLS-схема](/res/RLS-схема.png "Принцип работы")

На схеме представлен общий принцип работы механизма мультиарендности, показанный через взаимодействие компонентво шлюза при обработке запросов пользователей.

### Конфигурирование СУБД

![RLS](/res/RLS.png "Конфигурирование СУБД")

На данном изображении показан фрагмент файла конфигурирования СУБД. В нём демонстрируется объявление политик безопасности для таблицы настроек провайдеров.

### Переопределение взаимодействия с СУБД

```Java
@Slf4j
public class TenantHikariDataSource extends HikariDataSource {
    @Override
    public Connection getConnection() throws SQLException {
        Connection connection = super.getConnection();

        try (Statement sql = connection.createStatement()) {
            sql.execute("SET tenant_id = '" + ThreadLocalStorage.getTenantName() + "'");
        }

        return connection;
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        Connection connection = super.getConnection(username, password);

        try (Statement sql = connection.createStatement()) {
            sql.execute("SET tenant_id = '" + ThreadLocalStorage.getTenantName() + "'");
        }

        return connection;
    }
}
```

Для использования возможностей RLS был переопределён базовый класс соединения с СУБД, чтобы при каждом SQL-запросе к ниму приписывался специальный постфикс, необходимый для работы политик безопасности

### Пример API запроса

![API](/res/APIv3.png "Пример API запроса")

На данном изображении показн JSON-тело запроса на отправку сообщения. В нём содержится ключ идемпотентности для исключения повторных отправко одинаковых сообщений, массив адресов получателей сообщения, а также тип сообщения и через какого провайдера его необходимо отправить





