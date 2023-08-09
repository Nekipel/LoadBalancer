# LoadBalancer
КурсыJava Advanced Отказоустойчивость
Java Advanced . Лекция 7.1.1
LoadBalancer

В прошлых уроках мы разобрали базовый стэк, который необходим для создания простейших микросервисов. Т уже можешь создать свой API Gateway, наладить связь с помощью Feign Client между микросервисами — и все это под контролем Eureka Server.

Но в нынешних реалиях недостаточно просто написать и соединить, нужно быть уверенным, что приложение работает стабильно и не дает никаких side-эффектов при возникновении проблем, например, при перегруженности одного или нескольких/микросервисов.
Представь, что один из сервисов не выдержал нагрузки и просто упал — теперь он полностью недоступен.
В таких ситуациях на помощь приходит такой подход, как балансировка нагрузки и несколько инстансов одного и того же сервиса.
Балансировка нагрузки — это процесс распределения трафика между различными экземплярами одного и того же приложения.

Существует два вида балансировки:
server-side — когда запросы поступают от Client, они придут к балансировке нагрузки, и она определит один сервер для этого запроса. Самый простой алгоритм, используемый балансировкой нагрузки — это случайное распределение.
![image](https://github.com/Nekipel/LoadBalancer/assets/88710417/618675bd-58ee-4cab-b9a2-3d7aa890c4a6)
client-side — балансировка нагрузки находится на стороне клиента (Client side), она сама решает, к какому серверу отправить запрос на основании  критериев, например, доступность, зона, быстродействие и т. д.
![image](https://github.com/Nekipel/LoadBalancer/assets/88710417/aa527245-9dd7-4ffe-94e8-0d0861076b6d)

Как работать с балансировкой в Spring Cloud

На момент написания лекции в стэке Spring Cloud, а конкретно — в Feign Client и Eureka Server все максимально упрощено. Из Spring Cloud были удалены или перемещены под капот ribbon и hystrix, которые использовались в более ранних версиях для балансировки  нагрузки, теперь балансировка работает «из коробки .

Давай. в этом убедимся. Для этого нужно взять за основу наш client-service и создать его реплику. Как это сделать и убедиться, что балансировка работает?

Алгоритм прост:
-Создаем client-service2 с аналогичными зависимостями и таким же application name, как и у client-service, который используется в эврике.
-Создаем в первом и втором client-service тестовый end-point (метод в существующем контроллере), который отдает разные строки с одинаковыми маппингами, например /api/client/test.
-Полностью копировать client-service не обязательно, достаточно контроллера с одним тестовым методом, главное, чтобы имена сервиса совпадали.
-Запускаем eureka server, а затем — наш gateway и два клиент сервиса.
-Заходим на страницу Eureka и убеждаемся, что у нас две реплики client-service.
![image](https://github.com/Nekipel/LoadBalancer/assets/88710417/23622fa2-f632-40c6-b378-5a939b53d0c3)

В статусе напротив  client-service видим надпись UP(2) … — это значит, что ты все сделал правильно.

С помощью любой утилиты, например, Postman, шлем запрос наgateway с маппингом /api/client/test. Сделайте это несколько раз.

Убедись в результате — с каждым новым запросом будет отдаваться строка то с первого клиент сервиса, то со второго.

Теперь разберемся с Feign Client. Дело в том, что когда мы указываем value в аннотации @FeignClient, например, @FeignClient(name = "CLIENT-SERVICE"), Spring автоматически создает под него Spring Cloud LoadBalancer client и балансировка также автоматически будет работать. Давай убедимся и в этом:

Будем считать, что у нас запущены все приложения из прошлого примера.

Создадим еще один сервис, который будет называться client-updater (является клиентом еврики), он будет вызывать наш тестовый end-point с маппингом /api/client из прошлого примера при помощи Feign Client. Не забудь дать сервису уникальное имя, например, client-update.

Создадим в нем RestController, а в нем — один метод с маппингом /api/update, который будет отдавать результат вызова Feign Client.

Добавим новый route в gateway с предикатом /api/update/** и сервисом lb://CLIENT-UPDATE.

Сделаем пару запросов на контроллер в client-update сервисе и убедимся в результате — будут отдаваться разные строки.

Все сервисы с необходимыми зависимостями уже есть в заготовках, ты можешь воспользоваться ими.

user avatar
Владислав Шилов
28.02.2023 13:22
https://www.youtube.com/watch?v=2ZK80tRDmc8&list=PL8X2nqRlWfaZcyrJrsrWmQ17vtagWKv3f&index=19

user avatar
Олег Комиссаров
07.02.2023 22:55
порт не забудьте поменять на втором сервисе, а то конфликт будет


Circuit Breaker

В прошлом уроке мы разобрались с репликами и балансировкой нагрузки, но что если в нашем системе упали все реплики сервиса? На помощь приходит шаблон под названием Circuit Breaker, он подразумевает механизм предохранителя, который срабатывает от задаваемыхи критериев и позволяет обработать ситуацию сбоя.

Например, возьмем наш client-service и book-service из прошлых заданий.

Мы успешно соединили их с помощью Feign Client, и тот единственный метод, который запрашивает все книги, является потенциально опасным, ведь book-service может упасть.

Давайте разберемся, как зможно обработать эту ситуацию.

Сейчас spring-cloud поддерживает и рекомендует resilience4j в качестве реализации шаблона Circuit Breaker взамен устаревшего Hystrix, поэтому для начала добавим зависимость в проект.


<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>

Теперь давай сделаем наш Feign из client-service отказоустойчивым, для этого нужно лишь дописать в параметрах аннотации @FeignClient класс, который будет указывать, что делать в случае сбоя.


@FeignClient(name = "BOOK-SERVICE", fallback = BookServiceConnector.Fallback.class)
    public interface BookServiceConnector {
      @GetMapping("/api/books")
      List<Book> getAllBooks();
    }

    @Component
    class Fallback implements BookServiceConnector {
        @Override
        public List<Book> getAllBooks() {
            return Collections.emtyList();
        }
    }

Как видно из примера, мы просто создали рядом класс, который реализует сам интерфейс BookServiceConnector. Мы переопределяем метод getAllBooks так, как нам нужно, в данном случае просто возвращаем пустой лист. Также нам надо добавить в файл свойства.

feign:
 circuitbreaker:
  enabled: true
 

 

Это необходимо, поскольку по умолчанию circuitbeaker для feign выключен.

Но на этом мы не остановимся. Что если мы захотим узнать причину сбоя? Feign всегда выбрасывает исключение с информацией об ошибке, и мы можем использовать его, например, для логгирования. Сделаем это, тем более в следующем модуле нам это понадобится.

Теперь вместо параметра fallback нужно добавить параметр fallbackFactory.

 

@FeignClient(name = "BOOK-SERVICE", fallbackFactory = BookServiceConnector.BookServiceConnectorFallbackFactory.class

)


public interface BookServiceConnector {

 @GetMapping("/api/books")
      List<Book> getAllBooks();
}


@Component
class BookServiceConnectorFallbackFactory implements FallbackFactory<FallbackWithFactory> {

  @Override
  public FallbackWithFactory create(Throwable cause) {
      return new FallbackWithFactory(cause.getMessage());
  }
}

@Slf4j
record FallbackWithFactory(String reason) implements BookServiceConnector {
  @Override
  public List<Book> getAllBooks() {
      log.info("Failed send request on book service, reason {}", reason);
      return Collections.emptyList();
  }
}
Как видно из примера, код стал немного сложнее. Разберемся, что мы сделали в первую очередь:

Создали новый класс BookServiceConnectorFallbackFactory, который имплементирует параметризованный интерфейс FallbackFactory.

В интерфейсе есть один метод create, мы переопределяем его под себя. Как понятно из названия, это фабрика, которая создает наши fallbacks, и в методе create мы уже можем получить доступ исключению.

Следующий класс FallbackWithFactory — это аналог нашего прошлого Fallback, только с дополнительным полем String reason, которое хранит информацию из исключения. Фабрика будет отдавать объекты на основе именно этого класса, а не те, что предлагает Spring по умолчанию.

В этом классе мы также имплементируем наш интерфейс BookServiceConnector и переопределяем метод getAllBooks — он также отдает пустой лист при сбое, но теперь еще и логгирует информацию об этом сбое.

Обрати внимание, что FallbackWithFactory является record, тут я воспользовался возможностями Java 17, чтобы сократить код.

Добавь Fallback в свой client-service и не запускай BookService. Протестируй работоспособность механизма Fallbacks. 

